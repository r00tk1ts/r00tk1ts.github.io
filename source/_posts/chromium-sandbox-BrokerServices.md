---
title: Chromium-sandbox-BrokerServices-analysis
date: 2018-05-13 18:28:11
categories: 源码剖析
tags:
	- chromium
	- chromium-sandbox
---
最近一个半月一直在研究浏览器安全，在通读了浏览器安全白皮书，研读了各种优质相关资源后，将第一个目标锁定了chromium的sandbox。起初是对沙盒逃逸感兴趣，但深入到sandbox源码并分析了一些escape的writeup后，发现实际上沙盒逃逸的攻击面是不一而论的，仅仅搞懂了sandbox的原理依旧不够，sandbox的上下层接口往往才是安全焦点。
然而我一直信奉“知其所以然”这一信条，研读chrome源码一方面可以提高对C++编码的理解，另一方面也更有机会揪出隐藏较深的bug乃至安全漏洞，这是庞大的google fuzzer大军所做不到的。
本篇是sandbox源码剖析的第一篇，主要分析了windows平台下，chrome的broker进程API Interface类BrokerServices。

想要流程的阅读本系列你需要以下几个条件：
1. 较高水平的C++编码能力（至少通读C++ Primer 5th，刷过课后题，有一定编码量）。
2. 熟悉Windows API编程，尤其是安全相关的内容。
3. 对二进制安全有一定了解，熟悉各类型安全漏洞成因、漏洞缓解措施及bypass手法。

<!--more-->

# Chromium-sandbox-BrokerServices-analysis

Chrome的browser进程也就是主进程叫做broker。

在研读`BrokerServices`和`TargetServices`之前，一定要先阅读chrome的一些文档：

- 多进程架构
- Chromium是如何展示网页的
- 沙盒
- 沙盒_FAQ

## `sandbox.BrokerServices`

显然应该在sandbox namespace中，直接看类头定义：

```cpp
// BrokerServices exposes all the broker API.
// The basic use is to start the target(s) and wait for them to end.
//
// This API is intended to be called in the following order
// (error checking omitted):
//  BrokerServices* broker = SandboxFactory::GetBrokerServices();
//  broker->Init();
//  PROCESS_INFORMATION target;
//  broker->SpawnTarget(target_exe_path, target_args, &target);
//  ::ResumeThread(target->hThread);
//  // -- later you can call:
//  broker->WaitForAllTargets(option);
//
// 彻头彻尾的抽象基类，只提供了4个纯虚函数接口
class BrokerServices {
 public:
  // Initializes the broker. Must be called before any other on this class.
  // returns ALL_OK if successful. All other return values imply failure.
  // If the return is ERROR_GENERIC, you can call ::GetLastError() to get
  // more information.
  virtual ResultCode Init() = 0;//chrome经典的Init

  // Returns the interface pointer to a new, empty policy object. Use this
  // interface to specify the sandbox policy for new processes created by
  // SpawnTarget()
  // 对生成的target进程所应用的沙盒策略，这是标准的职能分离
  // scoped_refptr是个类模板，实现了智能指针机制，指向TargetPolicy对象
  // TargetPolicy是个管理target进程Policy的对象，policy相当复杂，暂不关心
  virtual scoped_refptr<TargetPolicy> CreatePolicy() = 0;

  // Creates a new target (child process) in a suspended state.
  // Parameters:
  //   exe_path: This is the full path to the target binary. This parameter
  //   can be null and in this case the exe path must be the first argument
  //   of the command_line.
  //   command_line: The arguments to be passed as command line to the new
  //   process. This can be null if the exe_path parameter is not null.
  //   policy: This is the pointer to the policy object for the sandbox to
  //   be created.
  //   last_warning: The argument will contain an indication on whether
  //   the process security was initialized completely, Only set if the
  //   process can be used without a serious compromise in security.
  //   last_error: If an error or warning is returned from this method this
  //   parameter will hold the last Win32 error value.
  //   target: returns the resulting target process information such as process
  //   handle and PID just as if CreateProcess() had been called. The caller is
  //   responsible for closing the handles returned in this structure.
  // Returns:
  //   ALL_OK if successful. All other return values imply failure.
  // 实际上就是CreateProcess的封装了
  virtual ResultCode SpawnTarget(const wchar_t* exe_path,
                                 const wchar_t* command_line,
                                 scoped_refptr<TargetPolicy> policy,
                                 ResultCode* last_warning,
                                 DWORD* last_error,
                                 PROCESS_INFORMATION* target) = 0;

  // This call blocks (waits) for all the targets to terminate.
  // Returns:
  //   ALL_OK if successful. All other return values imply failure.
  //   If the return is ERROR_GENERIC, you can call ::GetLastError() to get
  //   more information.
  // 实际上就是WaitForSingleObject的封装
  virtual ResultCode WaitForAllTargets() = 0;

 protected:
  //因为是虚基类，所以析构是protected，除了子类析构时会递归调用到，其他地方不该被调用到，这种提权可以在编译器发现一些语法错误
  ~BrokerServices() {}
};
```

软件设计层面实际上broker就是多个target的monitor进程，通过IPC通信。`BrokerServices`类实际上并不是broker进程，而是为broker进程提供的一个API调用面。broker可以通过它的API来完成创建policy、创建target进程等工作。头起始也给出了broker进程中使用它来创建target的用法范例，尽管相对于`SpawnTarget`来说，有些过时了。

### `SandboxFactory::GetBrokerServices()`

值得一提的是，范例中的broker对象创建使用的是`SandboxFactory::GetBrokerServices()`工厂方法，展开来看一下：

```cpp
// sandbox的工厂类
class SandboxFactory {
 public:
  // Returns the Broker API interface, returns nullptr if this process is the
  // target.
  static BrokerServices* GetBrokerServices();//生产broker

  // Returns the Target API interface, returns nullptr if this process is the
  // broker.
  static TargetServices* GetTargetServices();//生产target

 private:
  DISALLOW_IMPLICIT_CONSTRUCTORS(SandboxFactory);
  //禁用了隐式构造器，当然也包括默认赋值和默认复制构造
  //这是因为SandboxFactory类提供的两个public方法都是static，所以SandboxFactory无需实例化，这一操作同样可以在编译器发现问题。
};
```

我们暂时只关心`GetBrokerServices()`：

```cpp
// GetBrokerServices: the current implementation relies on a shared section
// that is created by the broker and opened by the target.
BrokerServices* SandboxFactory::GetBrokerServices() {
  // Can't be the broker if the shared section is open.
  // 看起来这个HANDLE有效时，其调用进程不是broker，而是target
  // 根据g_shared_section的注释来看，它作为target/broker间IPC和policy交互的共享内存的句柄，所以我猜测它的初始化应该在broker进程调用该函数之后。
  if (g_shared_section)	
    return nullptr;
  // If the shared section does not exist we are the broker, then create
  // the broker object.
  // s_is_broker是个static全局量，看起来也只在两个工厂方法里用到，分别表示是否是broker进程，但因为是个static的全局变量，该文件的其他位置也没有用到这货，可能是个reserved备胎吧。
  s_is_broker = true;
  //这里就是关键的对象生成了，BrokerServices是抽象基类显然不干实业，BrokerServices的派生类是BrokerServicesBase，负责干活。
  return BrokerServicesBase::GetInstance();
}
```

## `sandbox.BrokerServicesBase`

抽象基类没什么好深入的，行至`BrokerServicesBase::GetInstance()`也就山穷水尽了，这波看派生类操作就行了：

```cpp
// BrokerServicesBase ---------------------------------------------------------
// Broker implementation version 0
//
// This is an implementation of the interface BrokerServices and
// of the associated TargetProcess interface. In this implementation
// TargetProcess is a friend of BrokerServices where the later manages a
// collection of the former.
// 公有多继承BrokerServices和SingletonBase<BrokerServicesBase>类
// SingletonBase是个类模板，这里是对BrokerServicesBase应用了单例模式
// final表示不可被派生，也就表示这是最终实现体。
class BrokerServicesBase final : public BrokerServices,
                                 public SingletonBase<BrokerServicesBase> {
 public:
  BrokerServicesBase();

  ~BrokerServicesBase();

  // BrokerServices interface.
  // 4个纯虚函数的override
  ResultCode Init() override;
  scoped_refptr<TargetPolicy> CreatePolicy() override;
  ResultCode SpawnTarget(const wchar_t* exe_path,
                         const wchar_t* command_line,
                         scoped_refptr<TargetPolicy> policy,
                         ResultCode* last_warning,
                         DWORD* last_error,
                         PROCESS_INFORMATION* target) override;
  ResultCode WaitForAllTargets() override;

  // Checks if the supplied process ID matches one of the broker's active
  // target processes
  // Returns:
  //   true if there is an active target process for this ID, otherwise false.
  // 检查指定进程ID是否在broker的活跃目标进程集中                             
  bool IsActiveTarget(DWORD process_id);

 private:
  // The routine that the worker thread executes. It is in charge of
  // notifications and cleanup-related tasks.
  // 工作线程routine，负责通知和清理相关的任务。
  // 应该是有谁用该static函数作为一个独立运行的线程，但既然是private，就应该源于内部
  static DWORD WINAPI TargetEventsThread(PVOID param);

  // The completion port used by the job objects to communicate events to
  // the worker thread.
  // base::win::ScopedHandle是对Windows句柄HANDLE的一个封装，暂不理
  base::win::ScopedHandle job_port_;	//Job对象用此I/O完成端口与工作线程进行事件通信

  // Handle to a manual-reset event that is signaled when the total target
  // process count reaches zero.
  base::win::ScopedHandle no_targets_;	//这是个手动重置event，当全部target进程计数降到0时，signaled置信

  // Handle to the worker thread that reacts to job notifications.
  base::win::ScopedHandle job_thread_;	//响应job通知的工作线程句柄

  // Lock used to protect the list of targets from being modified by 2
  // threads at the same time.
  CRITICAL_SECTION lock_;	//锁，防止target链同时被两个线程修改

  // Provides a pool of threads that are used to wait on the IPC calls.
  std::unique_ptr<ThreadProvider> thread_pool_;	//线程池，用于等待IPC调用

  // List of the trackers for closing and cleanup purposes.
  std::list<std::unique_ptr<JobTracker>> tracker_list_;	//关闭和清理的tracker list

  // Provides a fast lookup to identify sandboxed processes that belong to a
  // job. Consult |jobless_process_handles_| for handles of processes without
  // jobs.
  std::set<DWORD> child_process_ids_;//属于job对象的沙盒进程id集，可用于快速检索

  DISALLOW_COPY_AND_ASSIGN(BrokerServicesBase);//禁用默认复制构造和默认赋值操作，这是因为BrokerServicesBase是单例模式，只应该有构造和析构
};
```

### 构造和析构

```cpp
BrokerServicesBase::BrokerServicesBase() {}	// 纯粹是好习惯

// The destructor should only be called when the Broker process is terminating.
// Since BrokerServicesBase is a singleton, this is called from the CRT
// termination handlers, if this code lives on a DLL it is called during
// DLL_PROCESS_DETACH in other words, holding the loader lock, so we cannot
// wait for threads here.
BrokerServicesBase::~BrokerServicesBase() {
  // If there is no port Init() was never called successfully.
  if (!job_port_.IsValid())	
    return;

  // Closing the port causes, that no more Job notifications are delivered to
  // the worker thread and also causes the thread to exit. This is what we
  // want to do since we are going to close all outstanding Jobs and notifying
  // the policy objects ourselves.
  // 关闭I/O完成端口，Job通知不再转给工作线程，工作线程也会退出
  ::PostQueuedCompletionStatus(job_port_.Get(), 0, THREAD_CTRL_QUIT, nullptr);

  // 等待工作线程退出
  if (job_thread_.IsValid() &&
      WAIT_TIMEOUT == ::WaitForSingleObject(job_thread_.Get(), 1000)) {
    // Cannot clean broker services.
    NOTREACHED();
    return;
  }

  //资源清理
  tracker_list_.clear();
  thread_pool_.reset();

  ::DeleteCriticalSection(&lock_);
}
```

析构函数的调用时机仅仅在Broker进程被终止时。

Broker是单例，终止时要么由CRT的termination handler调用，要么在DLL_PROCESS_DETACH时机调用（这说明代码在DLL上）。

实际上直接看析构还是有点懵逼的，先看`Init`更好些。

```cpp
// The broker uses a dedicated worker thread that services the job completion
// port to perform policy notifications and associated cleanup tasks.
ResultCode BrokerServicesBase::Init() {
  if (job_port_.IsValid() || thread_pool_)	//防止二次Init
    return SBOX_ERROR_UNEXPECTED_CALL;

  //初始化用户态临界区锁
  ::InitializeCriticalSection(&lock_);

  //创建IO完成端口，job_port是个封装的windows HANDLE类，Set方法关联
  job_port_.Set(::CreateIoCompletionPort(INVALID_HANDLE_VALUE, nullptr, 0, 0));
  if (!job_port_.IsValid())
    return SBOX_ERROR_GENERIC;

  //同样的套路创建Event和Thread对象，都没有设置安全属性
  //event匿名，设置为手动reset，nonsignaled态
  no_targets_.Set(::CreateEventW(nullptr, true, false, nullptr));

  //创建的线程入口就是static成员TargetEventsThread，传入的参数是对象broker本身
  job_thread_.Set(::CreateThread(nullptr, 0,  // Default security and stack.
                                 TargetEventsThread, this, 0, nullptr));
  if (!job_thread_.IsValid())
    return SBOX_ERROR_GENERIC;

  return SBOX_ALL_OK;
}
```

可以看到`Init`函数就是对资源的初始化以及`TargetEventsThread`线程的启动。

> `ResultCode`作为sandbox中泛用的返回值，是个枚举类型，可参考sandbox_types.h文件定义。

### `SpawnTarget`

暂且先不管这个线程做什么，先看看Target进程是如何由broker生成的：

```cpp
// SpawnTarget does all the interesting sandbox setup and creates the target
// process inside the sandbox.
// 负责沙盒的安装、target进程的创建，是个相当复杂的函数
ResultCode BrokerServicesBase::SpawnTarget(const wchar_t* exe_path,
                                           const wchar_t* command_line,
                                           scoped_refptr<TargetPolicy> policy,
                                           ResultCode* last_warning,
                                           DWORD* last_error,
                                           PROCESS_INFORMATION* target_info) {
  //target process二进制全路径，这里不能为NULL
  //实际上这里的实现体和虚基类的说法不太一致，虚基类的纯虚接口允许exe_path为NULL，但此时command_line的第一元必须为exe_path
  if (!exe_path)	
    return SBOX_ERROR_BAD_PARAMS;

  //应用于target进程的policy也不能为空，必须要提前通过CreatePolicy创建好，传入参数在该函数内关联，暂且先不管CreatePolicy做了什么。
  if (!policy)
    return SBOX_ERROR_BAD_PARAMS;

  // Even though the resources touched by SpawnTarget can be accessed in
  // multiple threads, the method itself cannot be called from more than
  // 1 thread. This is to protect the global variables used while setting up
  // the child process.
  // 这里的static局部变量以及锁是对线程重入做的限制，尽管SpawnTarget中资源可以被多个线程访问，但该方法本身一次只能有1个线程调用
  static DWORD thread_id = ::GetCurrentThreadId();
  // 到这里如果两次id不一致，说明是A->B->A，B改变了A设置的static thread_id
  // 这个时候A就应该终止，由B继续进行
  DCHECK(thread_id == ::GetCurrentThreadId());
  *last_warning = SBOX_ALL_OK;

  // 其实我不是很懂为什么要搞个static的thread_id，直接上锁不就行了吗，是为了记录thread_id吗？
  // 上锁，AutoLock是个自动获取和释放CRITICAL_SECTION锁的类
  // lock是个栈局部变量，函数返回后对象会析构，析构会release锁
  AutoLock lock(&lock_);

  // This downcast is safe as long as we control CreatePolicy()
  // policy尽管是 scoped_refptr<TargetPolicy>对象，但实际上CreatePolicy()时父类引用调用的是子类PolicyBase对象成员函数，标准的多态
  // 之所以做了向下转型，是因为虚基类TargetPolicy并没有定义要如何实现安全策略，使用了token、job等策略的PolicyBase只是一种实现体罢了。这就是OO的魅力，扩展性极强。
  // 当然，向下转型的使用前提是，必须已知父类引用指向的是某个具体的派生类，且只能是该派生类。
  scoped_refptr<PolicyBase> policy_base(static_cast<PolicyBase*>(policy.get()));

  // Construct the tokens and the job object that we are going to associate
  // with the soon to be created target process.
  // 构造三个信令token以及job对象，与即将创建的target进程相关联
  base::win::ScopedHandle initial_token;
  base::win::ScopedHandle lockdown_token;
  base::win::ScopedHandle lowbox_token;
  ResultCode result = SBOX_ALL_OK;

  // 可以看到三个信令token的部署是policy_base执行的，token是policy的三大核心之一（另外两个是job和alternative desktop）
  // 至于token是如何部署的，这个疑问留到解剖TargetPolicy时再探索
  result =
      policy_base->MakeTokens(&initial_token, &lockdown_token, &lowbox_token);
  if (SBOX_ALL_OK != result)
    return result;
  // lowbox_token必须要Win8以上才可用
  // 我觉着把这个逻辑放在MakeTokens后面挺奇怪的
  if (lowbox_token.IsValid() &&
      base::win::GetVersion() < base::win::VERSION_WIN8) {
    // We don't allow lowbox_token below Windows 8.
    return SBOX_ERROR_BAD_PARAMS;
  }

  // policy的三大核心之job部署，依然是依赖policy_base
  base::win::ScopedHandle job;
  result = policy_base->MakeJobObject(&job);
  if (SBOX_ALL_OK != result)
    return result;

  // StartupInformation是windows的STARTUPINFOEX的封装
  // spawn进程的时候会用到，这个结构理应耳熟能详
  // Initialize the startup information from the policy.
  base::win::StartupInformation startup_info;
  // The liftime of |mitigations|, |inherit_handle_list| and
  // |child_process_creation| have to be at least as long as
  // |startup_info| because |UpdateProcThreadAttribute| requires that
  // its |lpValue| parameter persist until |DeleteProcThreadAttributeList| is
  // called; StartupInformation's destructor makes such a call.
  DWORD64 mitigations[2];
  std::vector<HANDLE> inherited_handle_list;
  DWORD child_process_creation = PROCESS_CREATION_CHILD_PROCESS_RESTRICTED;

  //sandbox的进程还需要使用额外的AlternateDesktop，而非User的Desktop
  // alternate desktop就是policy三大核心的第三个
  base::string16 desktop = policy_base->GetAlternateDesktop();
  if (!desktop.empty()) {
    startup_info.startup_info()->lpDesktop =
        const_cast<wchar_t*>(desktop.c_str());//statup_info指定拿到的alternate desktop
  }

  bool inherit_handles = false;

  int attribute_count = 0;//这货记录policy三大核心之外的其他feature的个数，比如mitigation、target是否可生产子进程等

  size_t mitigations_size;
  //将sandbox flags转换成PROC_THREAD_ATTRIBUTE_SECURITY_CAPABILITIES policy flags，为UpdateProcThreadAttribute()所用
  // mitigation实际上也由policy掌控，但policy掌控的毕竟只是抽象的flag
  // 这里需要把他做一个转换，转换成windows API可以直接使用的结构
  // 而UpdateProcThreadAttribute不过是对API的一层封装
  ConvertProcessMitigationsToPolicy(policy_base->GetProcessMitigations(),
                                    &mitigations[0], &mitigations_size);
  if (mitigations[0] || mitigations[1])
    ++attribute_count;

  bool restrict_child_process_creation = false;
  //Win10_TH2以上，policy的Job权限低于JOB_LIMITED_USER则不允许target创建子进程
  if (base::win::GetVersion() >= base::win::VERSION_WIN10_TH2 &&
      policy_base->GetJobLevel() <= JOB_LIMITED_USER) {
    restrict_child_process_creation = true;
    ++attribute_count;
  }

  // 看起来policy的标准输出和错误句柄需要作为继承句柄
  HANDLE stdout_handle = policy_base->GetStdoutHandle();
  HANDLE stderr_handle = policy_base->GetStderrHandle();

  if (stdout_handle != INVALID_HANDLE_VALUE)
    inherited_handle_list.push_back(stdout_handle);

  // Handles in the list must be unique.
  if (stderr_handle != stdout_handle && stderr_handle != INVALID_HANDLE_VALUE)
    inherited_handle_list.push_back(stderr_handle);

  const auto& policy_handle_list = policy_base->GetHandlesBeingShared();

  //标准输出，标准错误和policy的所有共享句柄全部放入inherited_handle_list
  for (HANDLE handle : policy_handle_list)
    inherited_handle_list.push_back(handle);

  if (inherited_handle_list.size())
    ++attribute_count;

  //AppContainer可以在Win8以上使用
  scoped_refptr<AppContainerProfileBase> profile =
      policy_base->GetAppContainerProfileBase();
  if (profile) {
    if (base::win::GetVersion() < base::win::VERSION_WIN8)
      return SBOX_ERROR_BAD_PARAMS;
    ++attribute_count;
    if (profile->GetEnableLowPrivilegeAppContainer()) {
      // LPAC first supported in RS1.
      // Win10_RS1以上可以应用LPAC
      if (base::win::GetVersion() < base::win::VERSION_WIN10_RS1)
        return SBOX_ERROR_BAD_PARAMS;
      ++attribute_count;// LPAC与AC各算一次计数
    }
  }
	
  //初始化startup_info的线程属性列表，attribute_count记录了属性的个数
  if (!startup_info.InitializeProcThreadAttributeList(attribute_count))
    return SBOX_ERROR_PROC_THREAD_ATTRIBUTES;

  //UpdateProcThreadAttribute用以更新线程属性PROC_THREAD_ATTRIBUTE_MITIGATION_POLICY
  if (mitigations[0] || mitigations[1]) {
    if (!startup_info.UpdateProcThreadAttribute(
            PROC_THREAD_ATTRIBUTE_MITIGATION_POLICY, &mitigations[0],
            mitigations_size)) {
      return SBOX_ERROR_PROC_THREAD_ATTRIBUTES;
    }
  }

  //如果有子进程不允许创建的限制，那么还要更新PROC_THREAD_ATTRIBUTE_CHILD_PROCESS_POLICY这一属性
  if (restrict_child_process_creation) {
    if (!startup_info.UpdateProcThreadAttribute(
            PROC_THREAD_ATTRIBUTE_CHILD_PROCESS_POLICY, &child_process_creation,
            sizeof(child_process_creation))) {
      return SBOX_ERROR_PROC_THREAD_ATTRIBUTES;
    }
  }

  //更新PROC_THREAD_ATTRIBUTE_HANDLE_LIST属性
  if (inherited_handle_list.size()) {
    if (!startup_info.UpdateProcThreadAttribute(
            PROC_THREAD_ATTRIBUTE_HANDLE_LIST, &inherited_handle_list[0],
            sizeof(HANDLE) * inherited_handle_list.size())) {
      return SBOX_ERROR_PROC_THREAD_ATTRIBUTES;
    }
    startup_info.startup_info()->dwFlags |= STARTF_USESTDHANDLES;
    startup_info.startup_info()->hStdInput = INVALID_HANDLE_VALUE;
    // policy的标准输出和错误句柄为新进程使用，同时也在可继承句柄列表中
    startup_info.startup_info()->hStdOutput = stdout_handle;
    startup_info.startup_info()->hStdError = stderr_handle;
    // Allowing inheritance of handles is only secure now that we
    // have limited which handles will be inherited.
    inherit_handles = true;//仅在控制了哪些句柄可以被继承的情况下，允许继承句柄才是安全的
  }

  // Declared here to ensure they stay in scope until after process creation.
  std::unique_ptr<SecurityCapabilities> security_capabilities;
  DWORD all_applications_package_policy =
      PROCESS_CREATION_ALL_APPLICATION_PACKAGES_OPT_OUT;

  //如果应用了AppContainer，还要更新PROC_THREAD_ATTRIBUTE_SECURITY_CAPABILITIES
  // SecurityCapabilities对象源于profile，AC相关的部分我还没有看，暂时不清楚技术内幕
  if (profile) {
    security_capabilities = profile->GetSecurityCapabilities();
    if (!startup_info.UpdateProcThreadAttribute(
            PROC_THREAD_ATTRIBUTE_SECURITY_CAPABILITIES,
            security_capabilities.get(), sizeof(SECURITY_CAPABILITIES))) {
      return SBOX_ERROR_PROC_THREAD_ATTRIBUTES;
    }
    //如果应用了LPAC，还要更新PROC_THREAD_ATTRIBUTE_ALL_APPLICATION_PACKAGES_POLICY
    if (profile->GetEnableLowPrivilegeAppContainer()) {
      if (!startup_info.UpdateProcThreadAttribute(
              PROC_THREAD_ATTRIBUTE_ALL_APPLICATION_PACKAGES_POLICY,
              &all_applications_package_policy,
              sizeof(all_applications_package_policy))) {
        return SBOX_ERROR_PROC_THREAD_ATTRIBUTES;
      }
    }
  }

  // Construct the thread pool here in case it is expensive.
  // The thread pool is shared by all the targets
  // 首次创建target时，初始化线程池
  // Win2kThreadPool是Win2k线程池API的实现
  // 这里使用的是个std::unique_ptr，防止其他指针指向该对象
  if (!thread_pool_)
    thread_pool_ = std::make_unique<Win2kThreadPool>();

  // Create the TargetProcess object and spawn the target suspended. Note that
  // Brokerservices does not own the target object. It is owned by the Policy.
  // 创建TargetProcess对象，生成挂起的target进程。
  // target不属于BrokerServices，它实际上属于Policy。
  // base::win::ScopedProcessInformation是对PROCESS_INFORMATION的封装
  base::win::ScopedProcessInformation process_info;
  // 创建TargetProcess对象
  // 传递各种policy相关资源，注意如果没有用AppContainer就使用Sid机制
  TargetProcess* target = new TargetProcess(
      std::move(initial_token), std::move(lockdown_token), job.Get(),
      thread_pool_.get(),
      profile ? profile->GetImpersonationCapabilities() : std::vector<Sid>());

  // 创建target进程，传递命令行参数、继承句柄、启动信息、进程信息
  // 尽管此时还不了解TargetProcess的Create实现，但已经能嗅到CreateProcess的气息了。这些参数传的太堂而皇之了。
  result = target->Create(exe_path, command_line, inherit_handles, startup_info,
                          &process_info, last_error);

  if (result != SBOX_ALL_OK) {
    SpawnCleanup(target);
    return result;
  }

  // lowbox_token是win8以上独有的，单独处理，TargetProcess特意给了个接口
  if (lowbox_token.IsValid()) {
    *last_warning = target->AssignLowBoxToken(lowbox_token);
    // If this fails we continue, but report the error as a warning.
    // This is due to certain configurations causing the setting of the
    // token to fail post creation, and we'd rather continue if possible.
    // lowbox_token的失败是可以接受的，仅仅记录错误继续前行
    if (*last_warning != SBOX_ALL_OK)
      *last_error = ::GetLastError();
  }

  // Now the policy is the owner of the target.
  // policy正式接管target
  result = policy_base->AddTarget(target);

  if (result != SBOX_ALL_OK) {
    *last_error = ::GetLastError();
    SpawnCleanup(target);
    return result;
  }

  // We are going to keep a pointer to the policy because we'll call it when
  // the job object generates notifications using the completion port.
  // policy和target关联起来了，但broker生成的policy目前却没法找到
  // broker通过tracker_list_成员维护tracker，而tracker维护job和policy
  // 注意局部变量对象job通过std::move转移了owner
  // 通过JobTracker将job通知与job和policy对象联系起来
  if (job.IsValid()) {
    std::unique_ptr<JobTracker> tracker =
        std::make_unique<JobTracker>(std::move(job), policy_base);

    // There is no obvious recovery after failure here. Previous version with
    // SpawnCleanup() caused deletion of TargetProcess twice. crbug.com/480639
    // 将IO完成端口句柄job_port_与job对象相关联
    // tracker指针做key,IO完成端口句柄做value
    CHECK(AssociateCompletionPort(tracker->job.Get(), job_port_.Get(),
                                  tracker.get()));

    // Save the tracker because in cleanup we might need to force closing
    // the Jobs.
    // tracker是unique_ptr，所以不能直接复制，需要std::move右值引用来转换owner为tracker_list_成员
    tracker_list_.push_back(std::move(tracker));
    // 维护target进程的id
    child_process_ids_.insert(process_info.process_id());
  } else {
    // Leak policy_base. This needs to outlive the child process, but there's
    // nothing that tracks that lifetime properly if there's no job object.
    // TODO(wfh): Find a way to make this have the correct lifetime.
    policy_base->AddRef();

    // We have to signal the event once here because the completion port will
    // never get a message that this target is being terminated thus we should
    // not block WaitForAllTargets until we have at least one target with job.
    // 大概的意思就是，这个target没有job对象，所以终止时也没办法通知IO完成端口
    // 这就可能导致在当前没有任何一个其他拥有job对象的target时，一旦调用WaitForAllTargets
    // 就会因为收不到该进程终止的消息而永久阻塞，所以这里在没有其他拥有job对象的target进程存在的情况下启动这样一个进程时
    // 就手工置信no_targets_信号量，防止WaitForAllTargets的阻塞
    // 但这也意味着这种游离于IO完成端口管理之外的target，lifetime不受控制
    // 我暂时也不清楚什么样的target会落入到这里
    if (child_process_ids_.empty())
      ::SetEvent(no_targets_.Get());
  }

  //process_info转交给OUT型参数target_info（owner）
  *target_info = process_info.Take();
  return result;
}
```
因为涉及了大量的外部对象，所以在没有拆解耦合对象的情况下，很多细节暂时不纠结，看到了再说。

### `WaitForAllTargets()`

```cpp
ResultCode BrokerServicesBase::WaitForAllTargets() {
  ::WaitForSingleObject(no_targets_.Get(), INFINITE);
  return SBOX_ALL_OK;
}
```

这个就很简单了，`no_targets_`置信时`WaitForSingleObject`会返回，这就表示所有target都退出了。

### `IsActiveTarget()`

```cpp
bool BrokerServicesBase::IsActiveTarget(DWORD process_id) {
  // 上锁是为了在迭代child_process_ids_时，不受其他线程修改的影响
  AutoLock lock(&lock_);
  return child_process_ids_.find(process_id) != child_process_ids_.end();
}
```

### `CreatePolicy()`

作为`BrokerServicesBase`中非常重要的组成部分，对target进程的管理、安全策略都由policy控制。

```cpp
scoped_refptr<TargetPolicy> BrokerServicesBase::CreatePolicy() {
  // If you change the type of the object being created here you must also
  // change the downcast to it in SpawnTarget().
  // 父类指针引用子类对象，标准的多态
  scoped_refptr<TargetPolicy> policy(new PolicyBase);
  // PolicyBase starts with refcount 1.
  policy->Release();
  return policy;
}
```

实际上也没做什么，单纯的new出一个`PolicyBase`对象并返回其智能指针罢了。

而在`SpawnTarget`中也确实看到了使用的是`PolicyBase`对象的指针引用了这个对象，并做了各种安全设定、target绑定。

具体的安全设定是如何实现的，限于篇幅，这里就不展开了，本次分析旨在缕清Broker和Target的关系，以及各结构是如何编制在一起，形成了哪些效果等。具体的TargetProcess，TargetPolicy以及派生类PolicyBase等日后单独分析。

### `TargetEventsThread`

最后的重中之重，需要弄清楚target事件的处理线程都做了些什么。

```cpp
// The worker thread stays in a loop waiting for asynchronous notifications
// from the job objects. Right now we only care about knowing when the last
// process on a job terminates, but in general this is the place to tell
// the policy about events.
// 循环等待各个target的job对象的异步通知
// 当前仅关心最后一个进程何时退出，实际上它可以用来通知policy event的发生
// （这是因为绑定IO完成端口与job对象时，tracker指针做了key，而tracker维护了job和policy）
DWORD WINAPI BrokerServicesBase::TargetEventsThread(PVOID param) {
  if (!param)
    return 1;

  //broker中这个线程名叫BrokerEvent
  base::PlatformThread::SetName("BrokerEvent");

  //param实际上就是BrokerServicesBase对象，以this传入
  BrokerServicesBase* broker = reinterpret_cast<BrokerServicesBase*>(param);
  HANDLE port = broker->job_port_.Get();//IO完成端口句柄
  HANDLE no_targets = broker->no_targets_.Get();//no_targets_事件句柄

  int target_counter = 0;
  int untracked_target_counter = 0;
  ::ResetEvent(no_targets);

  while (true) {
    DWORD events = 0;
    ULONG_PTR key = 0;
    LPOVERLAPPED ovl = nullptr;

    // IO完成端口获取事件
    if (!::GetQueuedCompletionStatus(port, &events, &key, &ovl, INFINITE)) {
      // this call fails if the port has been closed before we have a
      // chance to service the last packet which is 'exit' anyway so
      // this is not an error.
      return 1;
    }

    // 具体key代表什么现在不得而知，得拆解IO完成端口的信源才行
    if (key > THREAD_CTRL_LAST) {
      // The notification comes from a job object. There are nine notifications
      // that jobs can send and some of them depend on the job attributes set.
      // 看起来key大于THREAD_CTRL_LAST时表示通知是从job对象发过来的，一共有9种通知
      // 原来这个key就是JobTracker指针，在前面SpawnTarget中创建JobTracker时，曾用此作为JOBOBJECT_ASSOCIATE_COMPLETION_PORT的第一元素，port作为第二元素
      // 而JobTracker本身是job和policy的关联
      JobTracker* tracker = reinterpret_cast<JobTracker*>(key);

      // events指定哪种通知
      switch (events) {
        case JOB_OBJECT_MSG_ACTIVE_PROCESS_ZERO: {
          // The job object has signaled that the last process associated
          // with it has terminated. Assuming there is no way for a process
          // to appear out of thin air in this job, it safe to assume that
          // we can tell the policy to destroy the target object, and for
          // us to release our reference to the policy object.
          // job对象告知最后一个进程已终止，通过tracker来释放所有相关资源。
          tracker->FreeResources();
          break;
        }

        case JOB_OBJECT_MSG_NEW_PROCESS: {
          DWORD handle = static_cast<DWORD>(reinterpret_cast<uintptr_t>(ovl));
          {
            AutoLock lock(&broker->lock_);
            size_t count = broker->child_process_ids_.count(handle);
            // Child process created from sandboxed process.
            // 如果新创建的进程不在broker的child_process_ids中，说明是沙盒中某个target进程创建的子进程，此时要untracked_target_counter递增以维护计数
            if (count == 0)
              untracked_target_counter++;
          }
          // 无论是target进程还是某个target创建的子进程，都要递增计数
          ++target_counter;
          // 当从0到1时，no_targets event也需要手工nonsignaled
          if (1 == target_counter) {
            ::ResetEvent(no_targets);
          }
          break;
        }

        case JOB_OBJECT_MSG_EXIT_PROCESS:
        case JOB_OBJECT_MSG_ABNORMAL_EXIT_PROCESS: {
          size_t erase_result = 0;
          {
            AutoLock lock(&broker->lock_);
            // 删除对应的process id
            erase_result = broker->child_process_ids_.erase(
                static_cast<DWORD>(reinterpret_cast<uintptr_t>(ovl)));
          }
          if (erase_result != 1U) {
            // The process was untracked e.g. a child process of the target.
            // 不在child_process_ids_中，说明是target生成的子进程
            --untracked_target_counter;
            DCHECK(untracked_target_counter >= 0);
          }
          --target_counter;
          // 和楼上相反的操作
          if (0 == target_counter)
            ::SetEvent(no_targets);

          DCHECK(target_counter >= 0);
          break;
        }

          // 这种情况为什么还要递增计数，我不太懂
          // 会不会有某种手法一直发这个消息，让int型的untracked_target_counter和target_counter溢出？
        case JOB_OBJECT_MSG_ACTIVE_PROCESS_LIMIT: {
          // A child process attempted and failed to create a child process.
          // Windows does not reveal the process id.
          untracked_target_counter++;
          target_counter++;
          break;
        }

          // 一旦超过了内存限制，直接终止job
        case JOB_OBJECT_MSG_PROCESS_MEMORY_LIMIT: {
          bool res = ::TerminateJobObject(tracker->job.Get(),
                                          SBOX_FATAL_MEMORY_EXCEEDED);
          DCHECK(res);
          break;
        }

        default: {
          NOTREACHED();
          break;
        }
      }
    } else if (THREAD_CTRL_QUIT == key) {
      // The broker object is being destroyed so the thread needs to exit.
      // 如果key是THREAD_CTRL_QUIT，那么broker对象就已经被销毁了，这个线程也就没有存在的必要了。（该线程是static成员，和broker实例的lifetime没关系）
      return 0;
    } else {
      // We have not implemented more commands.
      NOTREACHED();
    }
  }

  NOTREACHED();
  return 0;
}
```

到此，对`BrokerServices`有了一个大体的认识。

## Appendix

### `SingletonBase`

broker的工厂方法`SandboxFactory::GetBrokerServices()`最终是调用`BrokerServicesBase::GetInstance()`，这是继承单例类模板`SingletonBase`的方法。

```cpp
// Basic implementation of a singleton which calls the destructor
// when the exe is shutting down or the DLL is being unloaded.
template <typename Derived>
class SingletonBase {
 public:
 //相当标准的饿汉单例模式实现，SingletonBase没必要产生实例，所以直接将GetInstance作为static public方法公开即可
  static Derived* GetInstance() {
  	//这里如果没有static，问题就大了:)
    static Derived* instance = nullptr;
    if (!instance) {
      instance = new Derived();
      // Microsoft CRT extension. In an exe this this called after
      // winmain returns, in a dll is called in DLL_PROCESS_DETACH
      // 这个是MS的特权，在winmain返回或DLL_PROCESS_DETACH触发时调用注册的函数
      _onexit(OnExit);
    }
    return instance;
  }

 private:
  // this is the function that gets called by the CRT when the
  // process is shutting down.
  static int __cdecl OnExit() {
  	// 防止退出后继续使用GetInstance？这个不是太懂
    delete GetInstance();
    return 0;
  }
};
```

### `AutoLock`

```cpp
// Automatically acquires and releases a lock when the object is
// is destroyed.
class AutoLock {
 public:
  // Acquires the lock.
  // 这个构造器是显式的，防止了某些情况下编译器自作聪明的CRITICAL_SECTION对象到AutoLock的转换。
  explicit AutoLock(CRITICAL_SECTION* lock) : lock_(lock) {
    ::EnterCriticalSection(lock);//上锁
  }

  // Releases the lock;
  ~AutoLock() { ::LeaveCriticalSection(lock_); //释放锁}

 private:
  CRITICAL_SECTION* lock_;
  DISALLOW_IMPLICIT_CONSTRUCTORS(AutoLock);//禁用了默认构造器、复制构造、赋值操作
};
```

`AutoLock`简化了编码工作，也防止了编码时忘记release而编译不会报错的尴尬。实际上就是利用栈上对象会在函数返回后自动析构的特性来实现自动release，另一方面，构造函数中直接上锁也省去了调用enter的麻烦。

### `ScopedHandle`

实际上它是`GenericScopedHandle<HandleTraits, VerifierTraits>`的typedef。

```cpp
// The traits class for Win32 handles that can be closed via CloseHandle() API.
// 该类仅有3个public static函数，所以不会用于实例化
class HandleTraits {
 public:
  typedef HANDLE Handle;

  // 用于关闭句柄的static public成员
  // Closes the handle.
  static bool BASE_EXPORT CloseHandle(HANDLE handle);

  // 判断句柄是否有效，NULL或INVALID_HANDLE_VALUE均表示无效
  // Returns true if the handle value is valid.
  static bool IsHandleValid(HANDLE handle) {
    return handle != NULL && handle != INVALID_HANDLE_VALUE;
  }

  // 返回NULL
  // Returns NULL handle value.
  static HANDLE NullHandle() {
    return NULL;
  }

 private:
  DISALLOW_IMPLICIT_CONSTRUCTORS(HandleTraits);//拒绝所有隐式构造器
};

// Performs actual run-time tracking.
class BASE_EXPORT VerifierTraits {
 public:
  typedef HANDLE Handle;
	
  //同样只有两个static public方法，不用于实例化
  static void StartTracking(HANDLE handle, const void* owner,
                            const void* pc1, const void* pc2);
  static void StopTracking(HANDLE handle, const void* owner,
                           const void* pc1, const void* pc2);

 private:
  DISALLOW_IMPLICIT_CONSTRUCTORS(VerifierTraits);
};

// 几个static函数的实现体，全都用到了ScopedHandleVerifier::Get()
// 看起来ScopedHandleVerifier做了各种功能的抽离，这几个static仅仅只是壳子
// Static.
bool HandleTraits::CloseHandle(HANDLE handle) {
  return ScopedHandleVerifier::Get()->CloseHandle(handle);
}

// Static.
void VerifierTraits::StartTracking(HANDLE handle, const void* owner,
                                   const void* pc1, const void* pc2) {
  return ScopedHandleVerifier::Get()->StartTracking(handle, owner, pc1, pc2);
}

// Static.
void VerifierTraits::StopTracking(HANDLE handle, const void* owner,
                                  const void* pc1, const void* pc2) {
  return ScopedHandleVerifier::Get()->StopTracking(handle, owner, pc1, pc2);
}
// ScopedHandleVerifier相当复杂，这里就不展开了

// Generic wrapper for raw handles that takes care of closing handles
// automatically. The class interface follows the style of
// the ScopedFILE class with two additions:
//   - IsValid() method can tolerate multiple invalid handle values such as NULL
//     and INVALID_HANDLE_VALUE (-1) for Win32 handles.
//   - Set() (and the constructors and assignment operators that call it)
//     preserve the Windows LastError code. This ensures that GetLastError() can
//     be called after stashing a handle in a GenericScopedHandle object. Doing
//     this explicitly is necessary because of bug 528394 and VC++ 2015.
// windows原生HANDLE的包裹，它会负责自动关闭句柄。
template <class Traits, class Verifier>
class GenericScopedHandle {
 public:
  typedef typename Traits::Handle Handle;

  // 各种构造器
  GenericScopedHandle() : handle_(Traits::NullHandle()) {}

  explicit GenericScopedHandle(Handle handle) : handle_(Traits::NullHandle()) {
    Set(handle);
  }

  // 移动构造函数，other得是右值引用
  GenericScopedHandle(GenericScopedHandle&& other)
      : handle_(Traits::NullHandle()) {
    Set(other.Take());
  }

  ~GenericScopedHandle() {
    Close();
  }

  bool IsValid() const {
    return Traits::IsHandleValid(handle_);
  }

  // 移动赋值操作，other得是右值引用
  GenericScopedHandle& operator=(GenericScopedHandle&& other) {
    DCHECK_NE(this, &other);
    Set(other.Take());
    return *this;
  }

  void Set(Handle handle) {
    if (handle_ != handle) {
      // Preserve old LastError to avoid bug 528394.
      auto last_error = ::GetLastError();
      Close();

      if (Traits::IsHandleValid(handle)) {
        handle_ = handle;
        Verifier::StartTracking(handle, this, BASE_WIN_GET_CALLER,
                                GetProgramCounter());
      }
      ::SetLastError(last_error);
    }
  }

  Handle Get() const {
    return handle_;
  }

  // Transfers ownership away from this object.
  Handle Take() {
    Handle temp = handle_;
    handle_ = Traits::NullHandle();
    if (Traits::IsHandleValid(temp)) {
      Verifier::StopTracking(temp, this, BASE_WIN_GET_CALLER,
                             GetProgramCounter());
    }
    return temp;
  }

  // Explicitly closes the owned handle.
  void Close() {
    if (Traits::IsHandleValid(handle_)) {
      Verifier::StopTracking(handle_, this, BASE_WIN_GET_CALLER,
                             GetProgramCounter());

      Traits::CloseHandle(handle_);
      handle_ = Traits::NullHandle();
    }
  }

 private:
  FRIEND_TEST_ALL_PREFIXES(ScopedHandleTest, ActiveVerifierWrongOwner);
  FRIEND_TEST_ALL_PREFIXES(ScopedHandleTest, ActiveVerifierUntrackedHandle);
  Handle handle_;

  DISALLOW_COPY_AND_ASSIGN(GenericScopedHandle);//默认赋值操作和复制构造被禁
};
```

每个包裹HANDLE的对象都应该是唯一的，可以转换所有者，但不能复制，所以才有了移动复制构造和移动赋值，但却禁用默认赋值和默认赋值构造。

### `scoped_refptr`

一个封装好的智能(smart)指针类，用于那些需要引用计数的对象，自动的对象计数可以防止内存泄露（忘记release）：

```cpp
//
// A smart pointer class for reference counted objects.  Use this class instead
// of calling AddRef and Release manually on a reference counted object to
// avoid common memory leaks caused by forgetting to Release an object
// reference.  Sample usage:
//
//   class MyFoo : public RefCounted<MyFoo> {
//    ...
//    private:
//     friend class RefCounted<MyFoo>;  // Allow destruction by RefCounted<>.
//     ~MyFoo();                        // Destructor must be private/protected.
//   };
//
//   void some_function() {
//     scoped_refptr<MyFoo> foo = MakeRefCounted<MyFoo>();
//     foo->Method(param);
//     // |foo| is released when this function returns
//   }
//
//   void some_other_function() {
//     scoped_refptr<MyFoo> foo = MakeRefCounted<MyFoo>();
//     ...
//     foo = nullptr;  // explicitly releases |foo|
//     ...
//     if (foo)
//       foo->Method(param);
//   }
//
// The above examples show how scoped_refptr<T> acts like a pointer to T.
// Given two scoped_refptr<T> classes, it is also possible to exchange
// references between the two objects, like so:
//
//   {
//     scoped_refptr<MyFoo> a = MakeRefCounted<MyFoo>();
//     scoped_refptr<MyFoo> b;
//
//     b.swap(a);
//     // now, |b| references the MyFoo object, and |a| references nullptr.
//   }
//
// To make both |a| and |b| in the above example reference the same MyFoo
// object, simply use the assignment operator:
//
//   {
//     scoped_refptr<MyFoo> a = MakeRefCounted<MyFoo>();
//     scoped_refptr<MyFoo> b;
//
//     b = a;
//     // now, |a| and |b| each own a reference to the same MyFoo object.
//   }
//
// Also see Chromium's ownership and calling conventions:
// https://chromium.googlesource.com/chromium/src/+/lkgr/styleguide/c++/c++.md#object-ownership-and-calling-conventions
// Specifically:
//   If the function (at least sometimes) takes a ref on a refcounted object,
//   declare the param as scoped_refptr<T>. The caller can decide whether it
//   wishes to transfer ownership (by calling std::move(t) when passing t) or
//   retain its ref (by simply passing t directly).
//   In other words, use scoped_refptr like you would a std::unique_ptr except
//   in the odd case where it's required to hold on to a ref while handing one
//   to another component (if a component merely needs to use t on the stack
//   without keeping a ref: pass t as a raw T*).
template <class T>
class scoped_refptr {
 public:
  typedef T element_type;

  //下面都是封装指针的常规操作，可以参考STL中iterator对指针的封装
  constexpr scoped_refptr() = default;

  // Constructs from raw pointer. constexpr if |p| is null.
  // 原生指针做参数的构造器
  constexpr scoped_refptr(T* p) : ptr_(p) {
    if (ptr_)
      AddRef(ptr_);
  }

  // Copy constructor. This is required in addition to the copy conversion
  // constructor below.
  // 拷贝构造，直接中继原生指针构造
  scoped_refptr(const scoped_refptr& r) : scoped_refptr(r.ptr_) {}

  // Copy conversion constructor.
  // 拷贝转换构造，因为指针可能类型不同，这需要引入另一个类模板，且需要判断U到T是否允许转换
  // 依然中继原生指针构造
  template <typename U,
            typename = typename std::enable_if<
                std::is_convertible<U*, T*>::value>::type>
  scoped_refptr(const scoped_refptr<U>& r) : scoped_refptr(r.ptr_) {}

  // Move constructor. This is required in addition to the move conversion
  // constructor below.
  // 移动构造
  scoped_refptr(scoped_refptr&& r) noexcept : ptr_(r.ptr_) { r.ptr_ = nullptr;//注意要清空右值引用r本体 }

  // Move conversion constructor.
  // 移动转换构造
  template <typename U,
            typename = typename std::enable_if<
                std::is_convertible<U*, T*>::value>::type>
  scoped_refptr(scoped_refptr<U>&& r) noexcept : ptr_(r.ptr_) {
    r.ptr_ = nullptr;
  }

  //析构
  ~scoped_refptr() {
    static_assert(!base::subtle::IsRefCountPreferenceOverridden(
                      static_cast<T*>(nullptr), static_cast<T*>(nullptr)),
                  "It's unsafe to override the ref count preference."
                  " Please remove REQUIRE_ADOPTION_FOR_REFCOUNTED_TYPE"
                  " from subclasses.");
    if (ptr_)
      Release(ptr_);
  }

  T* get() const { return ptr_; }

  // 解引用
  T& operator*() const {
    DCHECK(ptr_);
    return *ptr_;
  }

  // 解引用
  T* operator->() const {
    DCHECK(ptr_);
    return ptr_;
  }

  // 赋值操作
  scoped_refptr& operator=(T* p) { return *this = scoped_refptr(p); }

  // Unified assignment operator.
  // 指针直接赋值也可以，给了特权，实际上内部做了交换
  scoped_refptr& operator=(scoped_refptr r) noexcept {
    swap(r);
    return *this;
  }

  void swap(scoped_refptr& r) noexcept { std::swap(ptr_, r.ptr_); }

  explicit operator bool() const { return ptr_ != nullptr; }

  template <typename U>
  bool operator==(const scoped_refptr<U>& rhs) const {
    return ptr_ == rhs.get();
  }

  template <typename U>
  bool operator!=(const scoped_refptr<U>& rhs) const {
    return !operator==(rhs);
  }

  template <typename U>
  bool operator<(const scoped_refptr<U>& rhs) const {
    return ptr_ < rhs.get();
  }

 protected:
  T* ptr_ = nullptr;

 private:
  template <typename U>
  friend scoped_refptr<U> base::AdoptRef(U*);//通过原生指针创建scoped_refptr而不增加引用计数
  scoped_refptr(T* p, base::subtle::AdoptRefTag) : ptr_(p) {}

  // Friend required for move constructors that set r.ptr_ to null.
  template <typename U>
  friend class scoped_refptr;	//移动构造器中对r.ptr置空，需要声明friend

  // Non-inline helpers to allow:
  //     class Opaque;
  //     extern template class scoped_refptr<Opaque>;
  // Otherwise the compiler will complain that Opaque is an incomplete type.
  static void AddRef(T* ptr);
  static void Release(T* ptr);
};
```

