---
title: Chromium-sandbox-TargetProcess-analysis
date: 2018-05-13 18:44:11
categories: 源码剖析
tags:
	- chromium
	- chromium-sandbox
---

本篇是sandbox源码剖析的第三篇，主要分析了windows平台下，chrome的target进程的核心类TargetProcess。阅读本篇前，请先阅读前两篇。

想要流程的阅读本系列你需要以下几个条件：
1. 较高水平的C++编码能力（至少通读C++ Primer 5th，刷过课后题，有一定编码量）。
2. 熟悉Windows API编程，尤其是安全相关的内容。
3. 对二进制安全有一定了解，熟悉各类型安全漏洞成因、漏洞缓解措施及bypass手法。

<!--more-->

# Chromium-sandbox-TargetProcess-analysis

## `TargetProcess`

TargetServices的全局对象实例只是用于提供服务的，设置token、IntegrityLevel、Mitigations等安全组件。

Target进程的生产此前已在`BrokerServicesBase::SpawnTarget`中看到了，`TargetProcess`对象由policy对象接管，而policy对象通过`BrokerServicesBase::tracker_list_`相联系。

现在就来看看`TargetProcess`：

```cpp
// TargetProcess models a target instance (child process). Objects of this
// class are owned by the Policy used to create them.
// TargetProcess对象表示一个target实例，即子进程，对象由创建它的Policy接管
class TargetProcess {
 public:
  // The constructor takes ownership of |initial_token| and |lockdown_token|
  TargetProcess(base::win::ScopedHandle initial_token,
                base::win::ScopedHandle lockdown_token,
                HANDLE job,
                ThreadProvider* thread_pool,
                const std::vector<Sid>& impersonation_capabilities);
  ~TargetProcess();

  // TODO(cpu): Currently there does not seem to be a reason to implement
  // reference counting for this class since is internal, but kept the
  // the same interface so the interception framework does not need to be
  // touched at this point.
  // 注释简述了对internal class定义空的引用计数接口函数的原因，和拦截框架有关
  void AddRef() {}
  void Release() {}

  // Creates the new target process. The process is created suspended.
  // 进程的创建，在BrokerServicesBase中我们看到new之后立即Create
  ResultCode Create(const wchar_t* exe_path,
                    const wchar_t* command_line,
                    bool inherit_handles,
                    const base::win::StartupInformation& startup_info,
                    base::win::ScopedProcessInformation* target_info,
                    DWORD* win_error);

  // Assign a new lowbox token to the process post creation. The process
  // must still be in its initial suspended state, however this still
  // might fail in the presence of third-party software.
  // 这个和lowbow token有关，调用时进程必须得是初始化时的挂起态
  ResultCode AssignLowBoxToken(const base::win::ScopedHandle& token);

  // Destroys the target process.
  // 终止
  void Terminate();

  // Creates the IPC objects such as the BrokerDispatcher and the
  // IPC server. The IPC server uses the services of the thread_pool.
  // IPC相关
  ResultCode Init(Dispatcher* ipc_dispatcher,
                  void* policy,
                  uint32_t shared_IPC_size,
                  uint32_t shared_policy_size,
                  DWORD* win_error);

  // 下面是一些资源获取相关函数
  // Returns the handle to the target process.
  HANDLE Process() const { return sandbox_process_info_.process_handle(); }

  // Returns the handle to the job object that the target process belongs to.
  HANDLE Job() const { return job_; }

  // Returns the address of the target main exe. This is used by the
  // interceptions framework.
  HMODULE MainModule() const {
    return reinterpret_cast<HMODULE>(base_address_);
  }

  // Returns the name of the executable.
  const wchar_t* Name() const { return exe_name_.get(); }

  // Returns the process id.
  DWORD ProcessId() const { return sandbox_process_info_.process_id(); }

  // Returns the handle to the main thread.
  HANDLE MainThread() const { return sandbox_process_info_.thread_handle(); }

  // broker和target间传输32位变量
  // Transfers a 32-bit variable between the broker and the target.
  ResultCode TransferVariable(const char* name, void* address, size_t size);

 private:
  // Details of the target process.
  base::win::ScopedProcessInformation sandbox_process_info_;
  // The token associated with the process. It provides the core of the
  // sbox security.
  // 禁闭锁，这个是沙盒限制中token限制的核心
  base::win::ScopedHandle lockdown_token_;
  // The token given to the initial thread so that the target process can
  // start. It has more powers than the lockdown_token.
  // 起始时使用的token，这个时候因为有很多需要高权限的任务要处理
  // TargetServices的LowerToken的一个应用就是从initial到lockdown token
  base::win::ScopedHandle initial_token_;
  // Kernel handle to the shared memory used by the IPC server.
  // IPC server用到的共享内存
  base::win::ScopedHandle shared_section_;
  // Job object containing the target process.
  // 包含该target进程的job
  HANDLE job_;
  // Reference to the IPC subsystem.
  // IPC server是SharedMemIPCServer对象，这里维护了该server地址
  std::unique_ptr<SharedMemIPCServer> ipc_server_;
  // Provides the threads used by the IPC. This class does not own this pointer.
  // IPC server会借用线程池对象来作为处理Msg的工具，这里维护了线程池对象地址
  ThreadProvider* thread_pool_;
  // Base address of the main executable
  void* base_address_;
  // Full name of the target executable.
  std::unique_ptr<wchar_t, base::FreeDeleter> exe_name_;
  /// List of capability sids for use when impersonating in an AC process.
  // 和AppContainer相关的一个结构，应用AC时会指定一个Sid列表
  std::vector<Sid> impersonation_capabilities_;

  // Function used for testing.
  friend TargetProcess* MakeTestTargetProcess(HANDLE process,
                                              HMODULE base_address);

  DISALLOW_IMPLICIT_CONSTRUCTORS(TargetProcess);//TargetProcess的构造非常严格，杜绝一切自作聪明的隐式转换
};
```

大部分成员相对来说还是见名知意的，IPC以及拦截框架自成篇幅，这里暂时规避。

### 构造和析构

```cpp
/*
  参考SpawnTarget中的调用：
    TargetProcess* target = new TargetProcess(
      std::move(initial_token), std::move(lockdown_token), job.Get(),
      thread_pool_.get(),
      profile ? profile->GetImpersonationCapabilities() : std::vector<Sid>());
*/
TargetProcess::TargetProcess(base::win::ScopedHandle initial_token,
                             base::win::ScopedHandle lockdown_token,
                             HANDLE job,
                             ThreadProvider* thread_pool,
                             const std::vector<Sid>& impersonation_capabilities)
    // This object owns everything initialized here except thread_pool and
    // the job_ handle. The Job handle is closed by BrokerServices and results
    // eventually in a call to our dtor.
  // 实际上就是非常简单的成员初始化，lockdown/initial token，job, IPC server用到的thread_pool以及SID集合都是传进来的，base_address_此时未知所以先NULL
    : lockdown_token_(std::move(lockdown_token)),
      initial_token_(std::move(initial_token)),
      job_(job),
      thread_pool_(thread_pool),
      base_address_(nullptr),
      impersonation_capabilities_(impersonation_capabilities) {}
```

此前在`BrokerServicesBase::SpawnTarget`中的构造体如此：

```cpp
  base::win::ScopedProcessInformation process_info;
  TargetProcess* target = new TargetProcess(
      std::move(initial_token), std::move(lockdown_token), job.Get(),
      thread_pool_.get(),
      profile ? profile->GetImpersonationCapabilities() : std::vector<Sid>());
```

显然各个功能部件都是外部造出来的，但是`TargetProcess`内部或掌控或关联了这些外部组件。

再看析构：

```cpp
TargetProcess::~TargetProcess() {
  // Give a chance to the process to die. In most cases the JOB_KILL_ON_CLOSE
  // will take effect only when the context changes. As far as the testing went,
  // this wait was enough to switch context and kill the processes in the job.
  // If this process is already dead, the function will return without waiting.
  // For now, this wait is there only to do a best effort to prevent some leaks
  // from showing up in purify.
  // 进程如果还在，就手动终止
  if (sandbox_process_info_.IsValid()) {
    ::WaitForSingleObject(sandbox_process_info_.process_handle(), 50);
    // Terminate the process if it's still alive, as its IPC server is going
    // away. 1 is RESULT_CODE_KILLED.
    ::TerminateProcess(sandbox_process_info_.process_handle(), 1);
  }

  // ipc_server_ references our process handle, so make sure the former is shut
  // down before the latter is closed (by ScopedProcessInformation).
  ipc_server_.reset();
}
```

### `Create()`

```cpp
// Creates the target (child) process suspended and assigns it to the job
// object.
// 构造函数中job已经进来了，所以这个函数会创建process对象并与之关联
ResultCode TargetProcess::Create(
    const wchar_t* exe_path,
    const wchar_t* command_line,
    bool inherit_handles,
    const base::win::StartupInformation& startup_info,
    base::win::ScopedProcessInformation* target_info,
    DWORD* win_error) {
  exe_name_.reset(_wcsdup(exe_path));

  // the command line needs to be writable by CreateProcess().
  std::unique_ptr<wchar_t, base::FreeDeleter> cmd_line(_wcsdup(command_line));

  // 基本的几个标志位，没什么好解释的
  // Start the target process suspended.
  DWORD flags =
      CREATE_SUSPENDED | CREATE_UNICODE_ENVIRONMENT | DETACHED_PROCESS;

  if (startup_info.has_extended_startup_info())
    flags |= EXTENDED_STARTUPINFO_PRESENT;

  if (job_ && base::win::GetVersion() < base::win::VERSION_WIN8) {
    // Windows 8 implements nested jobs, but for older systems we need to
    // break out of any job we're in to enforce our restrictions.
    // win8以后可以嵌套关联job，对于win8以下，属于job的父进程的子进程不属于该job
    flags |= CREATE_BREAKAWAY_FROM_JOB;
  }

  // 使用传入的参数创建进程
  PROCESS_INFORMATION temp_process_info = {};
  // 这个函数说来话长，以lockdown_token指定的安全上下文环境创建进程
  // 参考<https://msdn.microsoft.com/en-us/library/ms682429.aspx>
  if (!::CreateProcessAsUserW(lockdown_token_.Get(), exe_path, cmd_line.get(),
                              nullptr,  // No security attribute.
                              nullptr,  // No thread attribute.
                              inherit_handles, flags,
                              nullptr,  // Use the environment of the caller.
                              nullptr,  // Use current directory of the caller.
                              startup_info.startup_info(),
                              &temp_process_info)) {
    *win_error = ::GetLastError();
    return SBOX_ERROR_CREATE_PROCESS;
  }
  // 应该是用于OUT参数target_info
  base::win::ScopedProcessInformation process_info(temp_process_info);

  if (job_) {
    // Assign the suspended target to the windows job object.
    // 把target进程与job绑定
    if (!::AssignProcessToJobObject(job_, process_info.process_handle())) {
      *win_error = ::GetLastError();
      ::TerminateProcess(process_info.process_handle(), 0);
      return SBOX_ERROR_ASSIGN_PROCESS_TO_JOB_OBJECT;
    }
  }

  if (initial_token_.IsValid()) {
    HANDLE impersonation_token = initial_token_.Get();
    base::win::ScopedHandle app_container_token;
    if (GetAppContainerImpersonationToken(
            process_info.process_handle(), impersonation_token,
            impersonation_capabilities_, &app_container_token)) {
      impersonation_token = app_container_token.Get();
    }

    // Change the token of the main thread of the new process for the
    // impersonation token with more rights. This allows the target to start;
    // otherwise it will crash too early for us to help.
    HANDLE temp_thread = process_info.thread_handle();
    // 先用inititial token（如果有AppContainer，就用appcontainer那套）设置target进程
    if (!::SetThreadToken(&temp_thread, impersonation_token)) {
      *win_error = ::GetLastError();
      ::TerminateProcess(process_info.process_handle(), 0);
      return SBOX_ERROR_SET_THREAD_TOKEN;
    }
    initial_token_.Close();//从现在起initial_token就不会再用到了
  }

  // 这里果然把process_info给了OUT参数target_info
  if (!target_info->DuplicateFrom(process_info)) {
    *win_error = ::GetLastError();  // This may or may not be correct.
    ::TerminateProcess(process_info.process_handle(), 0);
    return SBOX_ERROR_DUPLICATE_TARGET_INFO;
  }

  // process对象已经有了，可以获取基址了
  base_address_ = GetProcessBaseAddress(process_info.process_handle());
  DCHECK(base_address_);
  if (!base_address_) {
    *win_error = ::GetLastError();
    ::TerminateProcess(process_info.process_handle(), 0);
    return SBOX_ERROR_CANNOT_FIND_BASE_ADDRESS;
  }

  // sandbox_process_info保存target的进程信息，这里直接转移ownership
  sandbox_process_info_.Set(process_info.Take());
  return SBOX_ALL_OK;
}
```

### `TransferVariable()`

```cpp
ResultCode TargetProcess::TransferVariable(const char* name,
                                           void* address,
                                           size_t size) {
  if (!sandbox_process_info_.IsValid())
    return SBOX_ERROR_UNEXPECTED_CALL;

  void* child_var = address;

#if SANDBOX_EXPORTS
  HMODULE module = ::LoadLibrary(exe_name_.get());
  if (!module)
    return SBOX_ERROR_GENERIC;

  child_var = ::GetProcAddress(module, name);
  ::FreeLibrary(module);

  if (!child_var)
    return SBOX_ERROR_GENERIC;

  // 因为偏移相对基址固定，所以可以计算运行时的child_var地址
  size_t offset =
      reinterpret_cast<char*>(child_var) - reinterpret_cast<char*>(module);
  child_var = reinterpret_cast<char*>(MainModule()) + offset;
#endif

  //address和size是传入的，写入地址值到address起始处
  //注释说是broker和target传数据用，那address应该是共享内存
  SIZE_T written;
  if (!::WriteProcessMemory(sandbox_process_info_.process_handle(), child_var,
                            address, size, &written))
    return SBOX_ERROR_GENERIC;

  if (written != size)
    return SBOX_ERROR_GENERIC;

  return SBOX_ALL_OK;
}
```

这个函数实际上非常有用，只是它的精髓在于参数的来源，某个很重要的组件会用到它。

### `Terminate()`

```cpp
void TargetProcess::Terminate() {
  if (!sandbox_process_info_.IsValid())
    return;

  ::TerminateProcess(sandbox_process_info_.process_handle(), 0);
}
```

### `AssignLowBoxToken()`

```cpp
ResultCode TargetProcess::AssignLowBoxToken(
    const base::win::ScopedHandle& token) {
  // 专门给Win8以上用的第三个token——lowbox
  // 这个lowbox具体限制了什么，甚至initial和lockdown具体的限制等到分析Policy时才清楚
  if (!token.IsValid())
    return SBOX_ALL_OK;
  PROCESS_ACCESS_TOKEN process_access_token = {};
  process_access_token.token = token.Get();

  NtSetInformationProcess SetInformationProcess = nullptr;
  ResolveNTFunctionPtr("NtSetInformationProcess", &SetInformationProcess);

  //使用NtSetInformationProcess来替换
  NTSTATUS status = SetInformationProcess(
      sandbox_process_info_.process_handle(),
      static_cast<PROCESS_INFORMATION_CLASS>(NtProcessInformationAccessToken),
      &process_access_token, sizeof(process_access_token));
  if (!NT_SUCCESS(status)) {
    ::SetLastError(GetLastErrorFromNtStatus(status));
    return SBOX_ERROR_SET_LOW_BOX_TOKEN;
  }
  return SBOX_ALL_OK;
}
```

