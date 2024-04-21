---
title: Chromium-sandbox-Job-analysis
date: 2018-05-14 20:17:11
categories: 源码剖析
tags:
	- chromium
	- chromium-sandbox
---

本篇是sandbox源码剖析的第五篇，主要分析了windows平台下，Chromium包装的Job对象。阅读本篇前，最好阅读前四篇（本篇相对独立）。

想要流程的阅读本系列你需要以下几个条件：
1. 较高水平的C++编码能力（至少通读C++ Primer 5th，刷过课后题，有一定编码量）。
2. 熟悉Windows API编程，尤其是安全相关的内容。
3. 对二进制安全有一定了解，熟悉各类型安全漏洞成因、漏洞缓解措施及bypass手法。

<!--more-->

# chromium-sandbox-Job-analysis

Windows本身有一种称为Job的对象，用于管理多个进程，做资源限制、访问控制等。

> Jeffery的《Windows核心编程》专门用了一章的篇幅来讲述这个Job作业对象。

在剖析`TargetPolicy`时，看到了3大组件+IL+Mitigation+other的组织结构。而target进程的job句柄是在`BrokerServicesBase::SpawnTarget`中通过`PolicyBase::MakeJobObject` new出来的。实际上这里new出来的对象并不是Windows原生的job对象，而是chrome的Job类，它封装了windows的job对象。

## `sandbox.Job`

头文件中类定义可以看到很多有用的信息。

```cpp
// Handles the creation of job objects based on a security profile.
// Sample usage:
//   Job job;
//   job.Init(JOB_LOCKDOWN, nullptr);  //no job name
//   job.AssignProcessToJob(process_handle);
class Job {
 public:
  Job();

  ~Job();

  // Initializes and creates the job object. The security of the job is based
  // on the security_level parameter.
  // job_name can be nullptr if the job is unnamed.
  // If the chosen profile has too many ui restrictions, you can disable some
  // by specifying them in the ui_exceptions parameters.
  // If the function succeeds, the return value is ERROR_SUCCESS. If the
  // function fails, the return value is the win32 error code corresponding to
  // the error.
  // job也是经典的constructor + init
  DWORD Init(JobLevel security_level,	//决定Job对象安全级别，还记得chrome设定的JobLevel枚举量吗？
             const wchar_t* job_name,	//名字可以没有，即匿名
             DWORD ui_exceptions,		//根据profile，个别UI限制可以开绿灯
             size_t memory_limit);		//应该是限制最大内存

  // Assigns the process referenced by process_handle to the job.
  // If the function succeeds, the return value is ERROR_SUCCESS. If the
  // function fails, the return value is the win32 error code corresponding to
  // the error.
  // 把进程归属到Job对象，使用句柄HANDLE操控进程
  // 注意此前在TargetProcess::Create中调用的是WinAPI AssignProcessToJobObject
  // 巧在该函数实际上也是对该API的封装，作为一个Job类的预留接口，我在跟踪代码时除了测试单元并没有发现其他地方对此接口引用
  DWORD AssignProcessToJob(HANDLE process_handle);

  // Grants access to "handle" to the job. All processes in the job can
  // subsequently recognize and use the handle.
  // If the function succeeds, the return value is ERROR_SUCCESS. If the
  // function fails, the return value is the win32 error code corresponding to
  // the error.
  //应该是调用之后，Job内的所有进程都可以访问传入的HANDLE
  DWORD UserHandleGrantAccess(HANDLE handle);

  // Revokes ownership to the job handle and returns it.
  // If the object is not yet initialized, it returns an invalid handle.
  // 转移owner
  base::win::ScopedHandle Take();

 private:
  // Handle to the job referenced by the object.
  base::win::ScopedHandle job_handle_;	//看起来是引用Windows Job对象的成员

  DISALLOW_COPY_AND_ASSIGN(Job);	//禁用默认拷贝复制和默认赋值操作
};
```

Chrome的注释一贯很nice，对Job的使用给出了范例。

> 1. `base::win::ScopedHandle`是个很复杂的类模板，日后写专门的文章再分析，暂时理解成Windows句柄的封装。
>
> 2. DISALLOW_COPY_AND_ASSIGN是个C++很常见的技巧，展开实际上就是:
>
>    ```cpp
>    // Put this in the declarations for a class to be uncopyable and unassignable.
>    #define DISALLOW_COPY_AND_ASSIGN(TypeName) \
>      DISALLOW_COPY(TypeName);                 \
>      DISALLOW_ASSIGN(TypeName)
>
>    // Put this in the declarations for a class to be uncopyable.
>    #define DISALLOW_COPY(TypeName) \
>      TypeName(const TypeName&) = delete
>
>    // Put this in the declarations for a class to be unassignable.
>    #define DISALLOW_ASSIGN(TypeName) TypeName& operator=(const TypeName&) = delete
>    ```
>
>    如此，相当于告知编译器不要产生默认赋值构造和默认赋值操作，对于不需要这两个成员的类来说，这种做法可以将一些语法或逻辑错误提到编译器来排解。

### Constructor/Destructor

public的成员不多，对Job来说，构造和析构都是public，是个比较单纯的类：

```cpp
Job::Job() : job_handle_(nullptr){};	//只是简单的对私有成员变量零化
Job::~Job(){};	//这种纯粹就是好习惯，对C++来说，我个人的理解就是一定要不厌其烦
```

### `Init()`

`JobLevel`此前已经见过了，再贴一下：

```cpp
// The Job level specifies a set of decreasing security profiles for the
// Job object that the target process will be placed into.
// This table summarizes the security associated with each level:
//
//  JobLevel        |General                            |Quota               |
//                  |restrictions                       |restrictions        |
// -----------------|---------------------------------- |--------------------|
// JOB_NONE         | No job is assigned to the         | None               |
//                  | sandboxed process.                |                    |
// -----------------|---------------------------------- |--------------------|
// JOB_UNPROTECTED  | None                              | *Kill on Job close.|
// -----------------|---------------------------------- |--------------------|
// JOB_INTERACTIVE  | *Forbid system-wide changes using |                    |
//                  |  SystemParametersInfo().          | *Kill on Job close.|
//                  | *Forbid the creation/switch of    |                    |
//                  |  Desktops.                        |                    |
//                  | *Forbids calls to ExitWindows().  |                    |
// -----------------|---------------------------------- |--------------------|
// JOB_LIMITED_USER | Same as INTERACTIVE_USER plus:    | *One active process|
//                  | *Forbid changes to the display    |  limit.            |
//                  |  settings.                        | *Kill on Job close.|
// -----------------|---------------------------------- |--------------------|
// JOB_RESTRICTED   | Same as LIMITED_USER plus:        | *One active process|
//                  | * No read/write to the clipboard. |  limit.            |
//                  | * No access to User Handles that  | *Kill on Job close.|
//                  |   belong to other processes.      |                    |
//                  | * Forbid message broadcasts.      |                    |
//                  | * Forbid setting global hooks.    |                    |
//                  | * No access to the global atoms   |                    |
//                  |   table.                          |                    |
// -----------------|-----------------------------------|--------------------|
// JOB_LOCKDOWN     | Same as RESTRICTED                | *One active process|
//                  |                                   |  limit.            |
//                  |                                   | *Kill on Job close.|
//                  |                                   | *Kill on unhandled |
//                  |                                   |  exception.        |
//                  |                                   |                    |
// In the context of the above table, 'user handles' refers to the handles of
// windows, bitmaps, menus, etc. Files, treads and registry handles are kernel
// handles and are not affected by the job level settings.
enum JobLevel {
  JOB_LOCKDOWN = 0,
  JOB_RESTRICTED,
  JOB_LIMITED_USER,
  JOB_INTERACTIVE,
  JOB_UNPROTECTED,
  JOB_NONE
};
```

枚举量的范围对应`Job`的权限高低，注释同样非常友好，不必多言。

```cpp
DWORD Job::Init(JobLevel security_level,
                const wchar_t* job_name,
                DWORD ui_exceptions,
                size_t memory_limit) {
  if (job_handle_.IsValid())	//防止二次初始化
    return ERROR_ALREADY_INITIALIZED;

  //这里果然用到了Windows API来创建Job对象，安全属性参数为nullptr实际上是表示使用默认安全属性，而非无
  //base::win::ScopedHandle的Set操作建立了Windows Job对象和job_handle_的联系
  job_handle_.Set(::CreateJobObject(nullptr,  // No security attribute
                                    job_name));
  if (!job_handle_.IsValid())
    return ::GetLastError();

  //这就是Windows Job对象的日常了，扩展限制信息和基础UI限制
  JOBOBJECT_EXTENDED_LIMIT_INFORMATION jeli = {};
  JOBOBJECT_BASIC_UI_RESTRICTIONS jbur = {};

  // Set the settings for the different security levels. Note: The higher levels
  // inherit from the lower levels.
  // 这里要根据传入的JobLevel，也就是第一个参数来分类
  switch (security_level) {
    case JOB_LOCKDOWN: {
      jeli.BasicLimitInformation.LimitFlags |=
          JOB_OBJECT_LIMIT_DIE_ON_UNHANDLED_EXCEPTION;	//最严格的权限，这里的flag标志也确实符合上面enum定义的描述，UNHANDLED_EXCEPTION时直接go die
      FALLTHROUGH;	//注意这里是穿透的，并非break，因为flag的设置从低到高有着子集的关系，所以这也算是一种编码时的小技巧。
    }
    case JOB_RESTRICTED: {
      jbur.UIRestrictionsClass |= JOB_OBJECT_UILIMIT_WRITECLIPBOARD;
      jbur.UIRestrictionsClass |= JOB_OBJECT_UILIMIT_READCLIPBOARD;//读写剪贴板
      jbur.UIRestrictionsClass |= JOB_OBJECT_UILIMIT_HANDLES;//UI HANDLES
      jbur.UIRestrictionsClass |= JOB_OBJECT_UILIMIT_GLOBALATOMS;//全局原子表
      FALLTHROUGH;
    }
    case JOB_LIMITED_USER: {
      jbur.UIRestrictionsClass |= JOB_OBJECT_UILIMIT_DISPLAYSETTINGS;
      jeli.BasicLimitInformation.LimitFlags |= JOB_OBJECT_LIMIT_ACTIVE_PROCESS;
      jeli.BasicLimitInformation.ActiveProcessLimit = 1;
      FALLTHROUGH;
    }
    case JOB_INTERACTIVE: {
      jbur.UIRestrictionsClass |= JOB_OBJECT_UILIMIT_SYSTEMPARAMETERS;
      jbur.UIRestrictionsClass |= JOB_OBJECT_UILIMIT_DESKTOP;
      jbur.UIRestrictionsClass |= JOB_OBJECT_UILIMIT_EXITWINDOWS;
      FALLTHROUGH;
    }
    case JOB_UNPROTECTED: {
      if (memory_limit) {
        jeli.BasicLimitInformation.LimitFlags |=
            JOB_OBJECT_LIMIT_PROCESS_MEMORY;
        jeli.ProcessMemoryLimit = memory_limit;//这一层面仅仅有着内存限制的设定
      }

      jeli.BasicLimitInformation.LimitFlags |=
          JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE;
      break;
    }
    default: { return ERROR_BAD_ARGUMENTS; }
  }

  //权限限制设定下去 
  if (!::SetInformationJobObject(job_handle_.Get(),
                                 JobObjectExtendedLimitInformation, &jeli,
                                 sizeof(jeli))) {
    return ::GetLastError();
  }

  //开绿灯的UI限制要放行
  jbur.UIRestrictionsClass = jbur.UIRestrictionsClass & (~ui_exceptions);
  if (!::SetInformationJobObject(job_handle_.Get(),
                                 JobObjectBasicUIRestrictions, &jbur,
                                 sizeof(jbur))) {
    return ::GetLastError();
  }

  return ERROR_SUCCESS;
}
```

> Init中Job对象的用法、权限的设置直接查MSDN吧。对这些Windows组合技定式的理解，没有什么捷径，多查MSDN+stackoverflow。

### `AssignProcessToJob()`

```cpp
DWORD Job::AssignProcessToJob(HANDLE process_handle) {
  if (!job_handle_.IsValid())	//心智健全检查
    return ERROR_NO_DATA;
	
  // 封装了AssignProcessToJobObject WinAPI
  // 实际上由于当前PolicyBase::MakeJobObject的实现返回的是HANDLE，而不是Job对象
  // 所以这个接口函数并没有被PolicyBase这个Policy实现体用到，它是直接调用了WinAPI
  if (!::AssignProcessToJobObject(job_handle_.Get(), process_handle))
    return ::GetLastError();

  return ERROR_SUCCESS;
}
```

### `UserHandleGrantAccess()`

```cpp
DWORD Job::UserHandleGrantAccess(HANDLE handle) {
  if (!job_handle_.IsValid())	//心智健全检查
    return ERROR_NO_DATA;

  //依然是WinAPI，第三个参数为true表示允许
  if (!::UserHandleGrantAccess(handle, job_handle_.Get(),
                               true)) {  // Access allowed.
    return ::GetLastError();
  }

  return ERROR_SUCCESS;
}
```

### `Take()`

```cpp
base::win::ScopedHandle Job::Take() {
  return std::move(job_handle_);	
  //左值引用转右值引用，避免拷贝操作（对象没变化，只是转移了所有权），另一方面由于默认拷贝构造的禁用，直接return job_handle_会报错。
}
```
一旦调用了Take之后，Job对象就失去了job_handle_这个Windows Job对象的所有权，当然也就不能再使用了。

这个函数在`PolicyBase::MakeJobObject`结尾处调用。

## `BrokerServicesBase` related

尽管Job封装的人模狗样，但对于policy来说，policy是怎样使用的Job才是最关键的。

我们从`BrokerServicesBase::SpawnTarget`开始探索Job的身世。

首先步入`PolicyBase::MakeJobObject`：

```cpp
base::win::ScopedHandle job;
result = policy_base->MakeJobObject(&job);
/* ------------------------------------------------------------------ */
ResultCode PolicyBase::MakeJobObject(base::win::ScopedHandle* job) {
  if (job_level_ != JOB_NONE) {
    // Create the windows job object.
    // constructor + init在栈上部署Job对象
    Job job_obj;
    DWORD result =
        job_obj.Init(job_level_, nullptr, ui_exceptions_, memory_limit_);
    if (ERROR_SUCCESS != result)
      return SBOX_ERROR_GENERIC;
	// 立即剥离了HANDLE的所有权，作为OUT型参数返回
    *job = job_obj.Take();
  } else {
    // 处理不需要job的情况，空的ScopedHandle对象实际上是INVALID_HANDLE
    *job = base::win::ScopedHandle();
  }
  return SBOX_ALL_OK;
}
```

此后在new TargetProcess对象时传入Job HANDLE，进入构造器：

```cpp
 TargetProcess* target = new TargetProcess(
      std::move(initial_token), std::move(lockdown_token), job.Get(),
      thread_pool_.get(),
      profile ? profile->GetImpersonationCapabilities() : std::vector<Sid>());
/* ------------------------------------------------------------------ */
TargetProcess::TargetProcess(base::win::ScopedHandle initial_token,
                             base::win::ScopedHandle lockdown_token,
                             HANDLE job,
                             ThreadProvider* thread_pool,
                             const std::vector<Sid>& impersonation_capabilities)
    // This object owns everything initialized here except thread_pool and
    // the job_ handle. The Job handle is closed by BrokerServices and results
    // eventually in a call to our dtor.
    : lockdown_token_(std::move(lockdown_token)),
      initial_token_(std::move(initial_token)),
      job_(job),	//所以这个HANDLE最终由TargetProcess的job_成员存储
      thread_pool_(thread_pool),
      base_address_(nullptr),
      impersonation_capabilities_(impersonation_capabilities) {}
```

从设计的角度来看，job对象本身就是管理target进程的，target进程对象存储job HANDLE也是天经地义。Policy作为中间介入层，它控制了target对象最终拿到的job是个什么样的job，而是什么样的job又取决于policy设定的`JobLevel`。

然而job的处理并没有结束，上面函数传参的时候，仅仅是复制了HANDLE值，`TargetProcess`存储的也仅仅是个值罢了。job的归属究竟在何处？

```cpp
 if (job.IsValid()) {
    std::unique_ptr<JobTracker> tracker =
        std::make_unique<JobTracker>(std::move(job), policy_base);

    // There is no obvious recovery after failure here. Previous version with
    // SpawnCleanup() caused deletion of TargetProcess twice. crbug.com/480639
    CHECK(AssociateCompletionPort(tracker->job.Get(), job_port_.Get(),
                                  tracker.get()));
 	tracker_list_.push_back(std::move(tracker));
/* ------------------------------------------------------------------ */
bool AssociateCompletionPort(HANDLE job, HANDLE port, void* key) {
  JOBOBJECT_ASSOCIATE_COMPLETION_PORT job_acp = {key, port};
  return ::SetInformationJobObject(job,
                                   JobObjectAssociateCompletionPortInformation,
                                   &job_acp, sizeof(job_acp))
             ? true
             : false;
}
```

还记得`BrokerServices::tracker_list_`吗？它是个`std::list<std::unique_ptr<JobTracker>>`。这货将所有的`JobTracker`对象指针集成在一起管理。`std::move(job)`意味着局部对象job的所属正式移交给了tracker，而tracker本身又把job和policy_base对象关联在一起。

进一步，`AssociateCompletionPort`把该job和IO完成端口联系起来。

### `JobTracker`

```cpp
// Helper structure that allows the Broker to associate a job notification
// with a job object and with a policy.
struct JobTracker {
  JobTracker(base::win::ScopedHandle job,
             scoped_refptr<sandbox::PolicyBase> policy)
      : job(std::move(job)), policy(policy) {}// job owner再次转移给内部成员job
  ~JobTracker() { FreeResources(); }

  // Releases the Job and notifies the associated Policy object to release its
  // resources as well.
  // 这才是关联policy的真实目的，在BrokerServicesBase::TargetEventsThread中可以看到
  // JOB_OBJECT_MSG_ACTIVE_PROCESS_ZERO发生时，就要释放它的policy的资源
  // 因为policy此时不再管辖任何target进程了
  void FreeResources();

  base::win::ScopedHandle job;
  scoped_refptr<sandbox::PolicyBase> policy;
};
```

封装这样一个东西，实际上是为了broker可以把job的通知事件同时与该job对象和制定job规格的policy联系起来。

#### `FreeResources`

那么，展开来看就很清晰了：

```cpp
void JobTracker::FreeResources() {
  if (policy) {
    // 先终止Job对象
    bool res = ::TerminateJobObject(job.Get(), sandbox::SBOX_ALL_OK);
    DCHECK(res);
    // Closing the job causes the target process to be destroyed so this needs
    // to happen before calling OnJobEmpty().
    HANDLE stale_job_handle = job.Get();
    job.Close();

    // In OnJobEmpty() we don't actually use the job handle directly.
    // 原来OnJobEmpty这个事件的驱动者在这儿
    policy->OnJobEmpty(stale_job_handle);
    policy = nullptr;
  }
}

// 揪出policy管理的targets_中，已经GG的target并删除
bool PolicyBase::OnJobEmpty(HANDLE job) {
  AutoLock lock(&lock_);
  TargetSet::iterator it;
  for (it = targets_.begin(); it != targets_.end(); ++it) {
    if ((*it)->Job() == job)
      break;
  }
  if (it == targets_.end()) {
    return false;
  }
  TargetProcess* target = *it;
  targets_.erase(it);
  delete target;
  return true;
}
```

## `TargetPolicy` related

Job最为policy的3大组件之一，在`PolicyBase`中也有丰富的接口函数，比如上面已经看到的`MakeJobObject`和`OnJobEmpty`。

其他的几个set/get方法只是单纯的成员传递：

```cpp
ResultCode PolicyBase::SetJobLevel(JobLevel job_level, uint32_t ui_exceptions) {
  if (memory_limit_ && job_level == JOB_NONE) {
    return SBOX_ERROR_BAD_PARAMS;
  }
  job_level_ = job_level;
  ui_exceptions_ = ui_exceptions;
  return SBOX_ALL_OK;
}

JobLevel PolicyBase::GetJobLevel() const {
  return job_level_;
}

ResultCode PolicyBase::SetJobMemoryLimit(size_t memory_limit) {
  memory_limit_ = memory_limit;
  return SBOX_ALL_OK;
}
```

这些已经在分析`TargetPolicy`一节中看到过了。

## `TargetProcess` related

如上分析，`TargetProcess`仅仅保存了一个所属job对象的HANDLE。而该类中涉及到job的接口只有：

```cpp
// Returns the handle to the job object that the target process belongs to.
HANDLE Job() const { return job_; }
```

这个函数在`PolicyBase::OnJobEmpty`中用来迭代`PolicyBase::targets_`，以job匹配。

到此，Job相关的内容目前都已经搞清楚了。