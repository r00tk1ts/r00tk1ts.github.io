---
title: Chromium-sandbox-ThreadProcess-analysis
date: 2018-06-03 11:40:11
categories: 源码剖析
tags:
	- chromium
	- chromium-sandbox
---

本篇是sandbox源码剖析的第十五篇，主要分析了子系统ThreadProcess的三大组件。

Target进程执行9个进程线程相关API时，应用于此的三大组件进行了一些处理。该子系统和Filesystem类似，但9个API却分成了3个类别，具体内容请参考本文的详细解读。

阅读本篇前，请先阅读前面所有章节。

想要流程的阅读本系列你需要以下几个条件：
1. 较高水平的C++编码能力（至少通读C++ Primer 5th，刷过课后题，有一定编码量）。
2. 熟悉Windows API编程，尤其是安全相关的内容。
3. 对二进制安全有一定了解，熟悉各类型安全漏洞成因、漏洞缓解措施及bypass手法。

<!--more-->

# chromium-sandbox-ThreadProcess-analysis

上一篇分析了Filesystem子系统，通过对filesystem子系统三大组件的分析，我们已经理清了完整的脉络。本篇我们采用相同的方式，长驱直入，分析ThreadProcess子系统。

## 初始化设置

在分析三大组件前，先看看初始化时都置办了哪些物件。

### `TopLevelDispatcher::TopLevelDispatcher`

`TopLevelDispatcher`的构造器是在`PolicyBase`的构造器中调用的，构造器内部我们已经非常熟悉了，它会将`ipc_targets_`指针数组的每一个成员指向正确的dispatcher对象。而对于ThreadProcess子系统来说，这个dispatcher对象是一个`ThreadProcessDispatcher`对象：

```cpp
dispatcher = new ThreadProcessDispatcher(policy_);
ipc_targets_[IPC_NTOPENTHREAD_TAG] = dispatcher;
ipc_targets_[IPC_NTOPENPROCESS_TAG] = dispatcher;
ipc_targets_[IPC_CREATEPROCESSW_TAG] = dispatcher;
ipc_targets_[IPC_NTOPENPROCESSTOKEN_TAG] = dispatcher;
ipc_targets_[IPC_NTOPENPROCESSTOKENEX_TAG] = dispatcher;
ipc_targets_[IPC_CREATETHREAD_TAG] = dispatcher;
thread_process_dispatcher_.reset(dispatcher);
```

可以看到该子系统相关的IPC请求tag有6种，分别是NtOpenThread, NtOpenProcess, CreateProcess, NtOpenProcessToken, NtOpenProcessTokenEx, CreateThread。

### `PolicyBase::AddTarget`

`PolicyBase`的Rules是预先由操纵者设计好的，当在`BrokerServices::SpawnTarget`中调用`PolicyBase::AddTarget`时，会进行一大堆关键操作，此前我们已经分析过数个了。

```cpp
ResultCode PolicyBase::AddTarget(TargetProcess* target) {
  // 把AddRules添加的PolicyRule解析出来全部置入PolicyGlobal
  if (policy_)
    policy_maker_->Done();

  if (!ApplyProcessMitigationsToSuspendedProcess(target->Process(),
                                                 mitigations_)) {
    return SBOX_ERROR_APPLY_ASLR_MITIGATIONS;
  }

  // 我们本次对它感兴趣
  ResultCode ret = SetupAllInterceptions(target);

  if (ret != SBOX_ALL_OK)
    return ret;

  if (!SetupHandleCloser(target))
    return SBOX_ERROR_SETUP_HANDLE_CLOSER;

  DWORD win_error = ERROR_SUCCESS;
  // Initialize the sandbox infrastructure for the target.
  // TODO(wfh) do something with win_error code here.
  ret = target->Init(dispatcher_.get(), policy_, kIPCMemSize, kPolMemSize,
                     &win_error);

  if (ret != SBOX_ALL_OK)
    return ret;

  g_shared_delayed_integrity_level = delayed_integrity_level_;
  ret = target->TransferVariable("g_shared_delayed_integrity_level",
                                 &g_shared_delayed_integrity_level,
                                 sizeof(g_shared_delayed_integrity_level));
  g_shared_delayed_integrity_level = INTEGRITY_LEVEL_LAST;
  if (SBOX_ALL_OK != ret)
    return ret;

  // Add in delayed mitigations and pseudo-mitigations enforced at startup.
  g_shared_delayed_mitigations =
      delayed_mitigations_ | FilterPostStartupProcessMitigations(mitigations_);
  if (!CanSetProcessMitigationsPostStartup(g_shared_delayed_mitigations))
    return SBOX_ERROR_BAD_PARAMS;

  ret = target->TransferVariable("g_shared_delayed_mitigations",
                                 &g_shared_delayed_mitigations,
                                 sizeof(g_shared_delayed_mitigations));
  g_shared_delayed_mitigations = 0;
  if (SBOX_ALL_OK != ret)
    return ret;

  AutoLock lock(&lock_);
  targets_.push_back(target);
  return SBOX_ALL_OK;
}
```

`LowLevelPolicy::Done`操作没什么好说的，我们感兴趣的是`SetupAllInterceptions`：

```cpp
ResultCode PolicyBase::SetupAllInterceptions(TargetProcess* target) {
  InterceptionManager manager(target, relaxed_interceptions_);

  if (policy_) {
    // 对每个IPC tag，都要进行dispatcher_->SetupService
    // 此处的dispatcher_是个TopLevelDispatcher，内部会根据tag分发
    for (int i = 0; i < IPC_LAST_TAG; i++) {
      if (policy_->entry[i] && !dispatcher_->SetupService(&manager, i))
        return SBOX_ERROR_SETUP_INTERCEPTION_SERVICE;
    }
  }

  // 这些是target不允许加载的dll，加入manager管制
  if (!blacklisted_dlls_.empty()) {
    std::vector<base::string16>::iterator it = blacklisted_dlls_.begin();
    for (; it != blacklisted_dlls_.end(); ++it) {
      manager.AddToUnloadModules(it->c_str());
    }
  }

  // 非常关键的调用
  if (!SetupBasicInterceptions(&manager, is_csrss_connected_))
    return SBOX_ERROR_SETUP_BASIC_INTERCEPTIONS;

  ResultCode rc = manager.InitializeInterceptions();
  if (rc != SBOX_ALL_OK)
    return rc;

  // Finally, setup imports on the target so the interceptions can work.
  if (!SetupNtdllImports(target))
    return SBOX_ERROR_SETUP_NTDLL_IMPORTS;

  return SBOX_ALL_OK;
}

// Delegate to the appropriate dispatcher.
bool TopLevelDispatcher::SetupService(InterceptionManager* manager,
                                      int service) {
  if (IPC_PING1_TAG == service || IPC_PING2_TAG == service)
    return true;

  // GetDispatcher不过是取出ipc_targets_[service]，它在构造器中设置过了
  Dispatcher* dispatcher = GetDispatcher(service);
  if (!dispatcher) {
    NOTREACHED();
    return false;
  }
  return dispatcher->SetupService(manager, service);
}
```

上一篇分析的`FilesystemDispatcher`在此处对5个系统调用进行了Interceptions的安装。类比一下，其他的子系统应该也差不多。我们先看看`ThreadProcessDispatcher::SetupService`：

```cpp
bool ThreadProcessDispatcher::SetupService(InterceptionManager* manager,
                                           int service) {
  switch (service) {
    case IPC_NTOPENTHREAD_TAG:
    case IPC_NTOPENPROCESS_TAG:
    case IPC_NTOPENPROCESSTOKEN_TAG:
    case IPC_NTOPENPROCESSTOKENEX_TAG:
    case IPC_CREATETHREAD_TAG:
      // There is no explicit policy for these services.
      // 这里非常奇怪，SetupService对这5个IPC tag没有做任何的拦截，它们不需要拦截？
      // 还是说因为某种原因而不在此处处理？
      NOTREACHED();
      return false;

    case IPC_CREATEPROCESSW_TAG:
      // 只有CreateProcess这个kernel32.dll的函数在这里处理
      // 这里和filesystem不同，它是个eat类型的interception
      // eat是通过修改dll导入表地址的方式来hook的，且它的安装是broker交由target自己处理的
      // hook函数叫TargetCreateProcess
      return INTERCEPT_EAT(manager, kKerneldllName, CreateProcessW,
                           CREATE_PROCESSW_ID, 44) &&
             INTERCEPT_EAT(manager, L"kernel32.dll", CreateProcessA,
                           CREATE_PROCESSA_ID, 44);

    default:
      return false;
  }
}

#define INTERCEPT_EAT(manager, dll, function, id, num_params) \
  manager->AddToPatchedFunctions(                             \
      dll, #function, sandbox::INTERCEPTION_EAT,              \
      reinterpret_cast<void*>(MAKE_SERVICE_NAME(function)), id)
```

再看看此后的关键call——`SetupBasicInterceptions`:

```cpp
bool SetupBasicInterceptions(InterceptionManager* manager,
                             bool is_csrss_connected) {
  // Interceptions provided by process_thread_policy, without actual policy.
  // 这3个连同下面的NtOpenProcessTokenEx都是有对应IPC tag的，看来它们4个的拦截器实际上是装了的，只不过移到了此处
  // 根据注释，之所以移到此处是因为它们无需policy
  // 但是dispatcher也和policy没有直接关系啊，我不太理解这种设计，历史原因？
  if (!INTERCEPT_NT(manager, NtOpenThread, OPEN_THREAD_ID, 20) ||
      !INTERCEPT_NT(manager, NtOpenProcess, OPEN_PROCESS_ID, 20) ||
      !INTERCEPT_NT(manager, NtOpenProcessToken, OPEN_PROCESS_TOKEN_ID, 16))
    return false;

  // Interceptions with neither policy nor IPC.
  // 这两个函数没有对应的IPC tag，是因为它们无需IPC请求
  if (!INTERCEPT_NT(manager, NtSetInformationThread, SET_INFORMATION_THREAD_ID,
                    20) ||
      !INTERCEPT_NT(manager, NtOpenThreadToken, OPEN_THREAD_TOKEN_ID, 20))
    return false;

  // This one is also provided by process_thread_policy.
  if (!INTERCEPT_NT(manager, NtOpenProcessTokenEx, OPEN_PROCESS_TOKEN_EX_ID,
                    20))
    return false;

  // NtOpenThreadTokenEx也没有对应的IPC tag，它也无需IPC请求
  if (!INTERCEPT_NT(manager, NtOpenThreadTokenEx, OPEN_THREAD_TOKEN_EX_ID, 24))
    return false;

  // 如果csrss未连接，那么还要拦截kernel32.dll的CreateThread
  // 它是有IPC tag的，注意它是导入表hook
  if (!is_csrss_connected) {
    if (!INTERCEPT_EAT(manager, kKerneldllName, CreateThread, CREATE_THREAD_ID,
                       28))
      return false;
  }

  return true;
}
```

9个函数中有3个是不需要IPC请求的，其余6个仅有CreateProcess在dispatcher的`SetupService`中进行了设置，其他都是由`SetupBasicInterceptions`部署的。除了没有IPC请求的3个以及需要判定`is_csrss_connected`的CreateThread，其他的几个为什么不放入Dispatcher中的`SetupService`处理，我也不是很懂，字面的意思是因为它们无需实际的policy，但另一方面，dispatcher和policy并没有直接关联。

或许继续分析会找到答案也说不定。

## policy

和filesystem子系统大同小异，policy也提供了一个最为重要的接口`GenerateRules`以及6个在dispatcher中调用的action。

```cpp
// This class centralizes most of the knowledge related to process execution.
class ProcessPolicy {
 public:
  // Creates the required low-level policy rules to evaluate a high-level.
  // policy rule for process creation
  // 'name' is the executable to be spawn.
  // 'semantics' is the desired semantics.
  // 'policy' is the policy generator to which the rules are going to be added.
  static bool GenerateRules(const wchar_t* name,
                            TargetPolicy::Semantics semantics,
                            LowLevelPolicy* policy);

  // Opens a thread from the child process and returns the handle.
  // client_info contains the information about the child process,
  // desired_access is the access requested by the child and thread_id
  // is the thread_id to be opened.
  // The function returns the return value of NtOpenThread.
  static NTSTATUS OpenThreadAction(const ClientInfo& client_info,
                                   uint32_t desired_access,
                                   uint32_t thread_id,
                                   HANDLE* handle);

  // Opens the process id passed in and returns the duplicated handle to
  // the child. We only allow the child processes to open themselves. Any other
  // pid open is denied.
  static NTSTATUS OpenProcessAction(const ClientInfo& client_info,
                                    uint32_t desired_access,
                                    uint32_t process_id,
                                    HANDLE* handle);

  // Opens the token associated with the process and returns the duplicated
  // handle to the child. We only allow the child processes to open its own
  // token (using ::GetCurrentProcess()).
  static NTSTATUS OpenProcessTokenAction(const ClientInfo& client_info,
                                         HANDLE process,
                                         uint32_t desired_access,
                                         HANDLE* handle);

  // Opens the token associated with the process and returns the duplicated
  // handle to the child. We only allow the child processes to open its own
  // token (using ::GetCurrentProcess()).
  static NTSTATUS OpenProcessTokenExAction(const ClientInfo& client_info,
                                           HANDLE process,
                                           uint32_t desired_access,
                                           uint32_t attributes,
                                           HANDLE* handle);

  // Processes a 'CreateProcessW()' request from the target.
  // 'client_info' : the target process that is making the request.
  // 'eval_result' : The desired policy action to accomplish.
  // 'app_name' : The full path of the process to be created.
  // 'command_line' : The command line passed to the created process.
  // 'current_dir' : The CWD with which to spawn the child process.
  static DWORD CreateProcessWAction(EvalResult eval_result,
                                    const ClientInfo& client_info,
                                    const base::string16& app_name,
                                    const base::string16& command_line,
                                    const base::string16& current_dir,
                                    PROCESS_INFORMATION* process_info);

  // Processes a 'CreateThread()' request from the target.
  // 'client_info' : the target process that is making the request.
  static DWORD CreateThreadAction(const ClientInfo& client_info,
                                  SIZE_T stack_size,
                                  LPTHREAD_START_ROUTINE start_address,
                                  PVOID parameter,
                                  DWORD creation_flags,
                                  LPDWORD thread_id,
                                  HANDLE* handle);
};
```

`GenerateRules`也是一个德行：

```cpp
bool ProcessPolicy::GenerateRules(const wchar_t* name,
                                  TargetPolicy::Semantics semantics,
                                  LowLevelPolicy* policy) {
  std::unique_ptr<PolicyRule> process;
  switch (semantics) {
    case TargetPolicy::PROCESS_MIN_EXEC: {
      // 如果是PROCESS_MIN_EXEC，那么rule的action opcode设定为资源只读
      process.reset(new PolicyRule(GIVE_READONLY));
      break;
    };
    case TargetPolicy::PROCESS_ALL_EXEC: {
      // 如果是PROCESS_ALL_EXEC，那么rule的action opcode设定位full access
      process.reset(new PolicyRule(GIVE_ALLACCESS));
      break;
    };
    default: { return false; };
  }

  // NameBased::Name这一参数需要是name
  if (!process->AddStringMatch(IF, NameBased::NAME, name, CASE_INSENSITIVE)) {
    return false;
  }
  // 绑定IPC_CREATEPROCESSW_TAG这一service id与process
  // 实际的意义就是如果此后service id为IPC_CREATEPROCESSW_TAG请求到来以后
  // 如果NameBased::NAME这一参数位与此处设置的name相同，那么审判的结果就是
  // GIVE_READONLY或GIVE_ALLACCESS（取决于AddRule时的semantics）
  if (!policy->AddRule(IPC_CREATEPROCESSW_TAG, process.get())) {
    return false;
  }
  return true;
}

	// 相关的两个semantics语义
	PROCESS_MIN_EXEC,      // Allows to create a process with minimal rights
                           // over the resulting process and thread handles.
                           // No other parameters besides the command line are
                           // passed to the child process.
    PROCESS_ALL_EXEC,      // Allows the creation of a process and return full
                           // access on the returned handles.
                           // This flag can be used only when the main token of
                           // the sandboxed application is at least INTERACTIVE.
```

实际上在`GenerateRules`中就可以发现，对于ThreadProcess子系统来说，所有的规则都只能限制CreateProcess。回想`SetupService`，它内部也仅对CreateProcess进行了拦截器的部署，其他的5个IPC请求的tag，由于没有实际的policy参数条件约束，所以设计上并未在此处进行拦截器部署。另外5个拦截器的部署是在一个全局的`SetupBasicInterceptions`中处理的。

所以，从设计者的角度来讲，dispatcher的`SetupService`和policy的`GenerateRules`有一定程度的耦合。

6个action函数先不关心，继续看dispatcher。

## dispatcher

```cpp
// This class handles process and thread-related IPC calls.
class ThreadProcessDispatcher : public Dispatcher {
 public:
  explicit ThreadProcessDispatcher(PolicyBase* policy_base);
  ~ThreadProcessDispatcher() override {}

  // Dispatcher interface.
  // Setup已经见过了
  bool SetupService(InterceptionManager* manager, int service) override;

 private:
  // Processes IPC requests coming from calls to NtOpenThread() in the target.
  bool NtOpenThread(IPCInfo* ipc, uint32_t desired_access, uint32_t thread_id);

  // Processes IPC requests coming from calls to NtOpenProcess() in the target.
  bool NtOpenProcess(IPCInfo* ipc,
                     uint32_t desired_access,
                     uint32_t process_id);

  // Processes IPC requests from calls to NtOpenProcessToken() in the target.
  bool NtOpenProcessToken(IPCInfo* ipc,
                          HANDLE process,
                          uint32_t desired_access);

  // Processes IPC requests from calls to NtOpenProcessTokenEx() in the target.
  bool NtOpenProcessTokenEx(IPCInfo* ipc,
                            HANDLE process,
                            uint32_t desired_access,
                            uint32_t attributes);

  // Processes IPC requests coming from calls to CreateProcessW() in the target.
  bool CreateProcessW(IPCInfo* ipc,
                      base::string16* name,
                      base::string16* cmd_line,
                      base::string16* cur_dir,
                      base::string16* target_cur_dir,
                      CountedBuffer* info);

  // Processes IPC requests coming from calls to CreateThread() in the target.
  bool CreateThread(IPCInfo* ipc,
                    SIZE_T stack_size,
                    LPTHREAD_START_ROUTINE start_address,
                    LPVOID parameter,
                    DWORD creation_flags);

  PolicyBase* policy_base_;
  DISALLOW_COPY_AND_ASSIGN(ThreadProcessDispatcher);
};
```

构造器实际上也颇为重要，实际上他是在`PolicyBase::AddTarget`首次`GetTarget`时调用到的(还记得各子系统static局部dispatcher对象吗)：

```cpp
ThreadProcessDispatcher::ThreadProcessDispatcher(PolicyBase* policy_base)
    : policy_base_(policy_base) {
  // 做出6个IPCCall，把它们塞入dispatcher的ipc_calls_用于此后处理IPC请求时
  // 匹配请求，它们绑定的6个callback函数也在该类中定义，它们内部会进行broker端的处理
  static const IPCCall open_thread = {
      {IPC_NTOPENTHREAD_TAG, {UINT32_TYPE, UINT32_TYPE}},
      reinterpret_cast<CallbackGeneric>(
          &ThreadProcessDispatcher::NtOpenThread)};

  static const IPCCall open_process = {
      {IPC_NTOPENPROCESS_TAG, {UINT32_TYPE, UINT32_TYPE}},
      reinterpret_cast<CallbackGeneric>(
          &ThreadProcessDispatcher::NtOpenProcess)};

  static const IPCCall process_token = {
      {IPC_NTOPENPROCESSTOKEN_TAG, {VOIDPTR_TYPE, UINT32_TYPE}},
      reinterpret_cast<CallbackGeneric>(
          &ThreadProcessDispatcher::NtOpenProcessToken)};

  static const IPCCall process_tokenex = {
      {IPC_NTOPENPROCESSTOKENEX_TAG, {VOIDPTR_TYPE, UINT32_TYPE, UINT32_TYPE}},
      reinterpret_cast<CallbackGeneric>(
          &ThreadProcessDispatcher::NtOpenProcessTokenEx)};

  static const IPCCall create_params = {
      {IPC_CREATEPROCESSW_TAG,
       {WCHAR_TYPE, WCHAR_TYPE, WCHAR_TYPE, WCHAR_TYPE, INOUTPTR_TYPE}},
      reinterpret_cast<CallbackGeneric>(
          &ThreadProcessDispatcher::CreateProcessW)};

  // NOTE(liamjm): 2nd param is size_t: Using VOIDPTR_TYPE as they are
  // the same size on windows.
  static_assert(sizeof(size_t) == sizeof(void*),
                "VOIDPTR_TYPE not same size as size_t");
  static const IPCCall create_thread_params = {
      {IPC_CREATETHREAD_TAG,
       {VOIDPTR_TYPE, VOIDPTR_TYPE, VOIDPTR_TYPE, UINT32_TYPE}},
      reinterpret_cast<CallbackGeneric>(
          &ThreadProcessDispatcher::CreateThread)};

  ipc_calls_.push_back(open_thread);
  ipc_calls_.push_back(open_process);
  ipc_calls_.push_back(process_token);
  ipc_calls_.push_back(process_tokenex);
  ipc_calls_.push_back(create_params);
  ipc_calls_.push_back(create_thread_params);
}
```

Server端IPC的处理机制本篇就不再说了，我们直接来看看这些请求处理的callback函数是如何处理的：

```cpp
// 以下4个是ntdll的4个系统调用
bool ThreadProcessDispatcher::NtOpenThread(IPCInfo* ipc,
                                           uint32_t desired_access,
                                           uint32_t thread_id) {
  HANDLE handle;
  // 直接就调用了ProcessPolicy::OpenThreadAction，没有任何处理
  // 下面的3个函数也类似
  NTSTATUS ret = ProcessPolicy::OpenThreadAction(
      *ipc->client_info, desired_access, thread_id, &handle);
  ipc->return_info.nt_status = ret;
  ipc->return_info.handle = handle;
  return true;
}

bool ThreadProcessDispatcher::NtOpenProcess(IPCInfo* ipc,
                                            uint32_t desired_access,
                                            uint32_t process_id) {
  HANDLE handle;
  NTSTATUS ret = ProcessPolicy::OpenProcessAction(
      *ipc->client_info, desired_access, process_id, &handle);
  ipc->return_info.nt_status = ret;
  ipc->return_info.handle = handle;
  return true;
}

bool ThreadProcessDispatcher::NtOpenProcessToken(IPCInfo* ipc,
                                                 HANDLE process,
                                                 uint32_t desired_access) {
  HANDLE handle;
  NTSTATUS ret = ProcessPolicy::OpenProcessTokenAction(
      *ipc->client_info, process, desired_access, &handle);
  ipc->return_info.nt_status = ret;
  ipc->return_info.handle = handle;
  return true;
}

bool ThreadProcessDispatcher::NtOpenProcessTokenEx(IPCInfo* ipc,
                                                   HANDLE process,
                                                   uint32_t desired_access,
                                                   uint32_t attributes) {
  HANDLE handle;
  NTSTATUS ret = ProcessPolicy::OpenProcessTokenExAction(
      *ipc->client_info, process, desired_access, attributes, &handle);
  ipc->return_info.nt_status = ret;
  ipc->return_info.handle = handle;
  return true;
}
```

另外两个kernel32.dll的函数则不太一致：

```cpp
bool ThreadProcessDispatcher::CreateProcessW(IPCInfo* ipc,
                                             base::string16* name,
                                             base::string16* cmd_line,
                                             base::string16* cur_dir,
                                             base::string16* target_cur_dir,
                                             CountedBuffer* info) {
  if (sizeof(PROCESS_INFORMATION) != info->Size())
    return false;

  // Check if there is an application name.
  base::string16 exe_name;
  if (!name->empty())
    exe_name = *name;
  else
    exe_name = GetPathFromCmdLine(*cmd_line);

  if (IsPathRelative(exe_name)) {
    if (!ConvertToAbsolutePath(*cur_dir, name->empty(), &exe_name)) {
      // Cannot find the path. Maybe the file does not exist.
      ipc->return_info.win32_result = ERROR_FILE_NOT_FOUND;
      return true;
    }
  }

  // 这里构造CountedParameterSet<NameBased>进行了审判
  const wchar_t* const_exe_name = exe_name.c_str();
  CountedParameterSet<NameBased> params;
  params[NameBased::NAME] = ParamPickerMake(const_exe_name);

  EvalResult eval =
      policy_base_->EvalPolicy(IPC_CREATEPROCESSW_TAG, params.GetBase());

  PROCESS_INFORMATION* proc_info =
      reinterpret_cast<PROCESS_INFORMATION*>(info->Buffer());
  // Here we force the app_name to be the one we used for the policy lookup.
  // If our logic was wrong, at least we wont allow create a random process.
  // 连同审判结果一起传入了ProcessPolicy::CreateProcessWAction
  DWORD ret = ProcessPolicy::CreateProcessWAction(
      eval, *ipc->client_info, exe_name, *cmd_line, *target_cur_dir, proc_info);

  ipc->return_info.win32_result = ret;
  return true;
}

bool ThreadProcessDispatcher::CreateThread(IPCInfo* ipc,
                                           SIZE_T stack_size,
                                           LPTHREAD_START_ROUTINE start_address,
                                           LPVOID parameter,
                                           DWORD creation_flags) {
  if (!start_address) {
    return false;
  }

  HANDLE handle;
  // CreateThread就很干脆，没有审判
  DWORD ret = ProcessPolicy::CreateThreadAction(
      *ipc->client_info, stack_size, start_address, parameter, creation_flags,
      nullptr, &handle);

  ipc->return_info.nt_status = ret;
  ipc->return_info.handle = handle;
  return true;
}
```

再次回到policy组件，最为关心的当属进行了Rule审判的`CreateProcessWAction`:

```cpp
DWORD ProcessPolicy::CreateProcessWAction(EvalResult eval_result,
                                          const ClientInfo& client_info,
                                          const base::string16& app_name,
                                          const base::string16& command_line,
                                          const base::string16& current_dir,
                                          PROCESS_INFORMATION* process_info) {
  // The only action supported is ASK_BROKER which means create the process.
  // 这里的注释有点问题，实际上匹配rule时EvalResult只能是GIVE_ALLACCESS或GIVE_READONLY
  if (GIVE_ALLACCESS != eval_result && GIVE_READONLY != eval_result) {
    return ERROR_ACCESS_DENIED;
  }

  // 如果通过了审判，那么会调用一个helper函数来进一步调用CreateProcess API
  STARTUPINFO startup_info = {0};
  startup_info.cb = sizeof(startup_info);
  std::unique_ptr<wchar_t, base::FreeDeleter> cmd_line(
      _wcsdup(command_line.c_str()));

  bool should_give_full_access = (GIVE_ALLACCESS == eval_result);

  const wchar_t* cwd = current_dir.c_str();
  if (current_dir.empty())
    cwd = nullptr;

  if (!CreateProcessExWHelper(client_info.process, should_give_full_access,
                              app_name.c_str(), cmd_line.get(), nullptr,
                              nullptr, false, 0, nullptr, cwd, &startup_info,
                              process_info)) {
    return ERROR_ACCESS_DENIED;
  }
  return ERROR_SUCCESS;
}
```

这个helper函数实际上就是一层封装：

```cpp
// Creates a child process and duplicates the handles to 'target_process'. The
// remaining parameters are the same as CreateProcess().
bool CreateProcessExWHelper(HANDLE target_process,
                            bool give_full_access,
                            LPCWSTR lpApplicationName,
                            LPWSTR lpCommandLine,
                            LPSECURITY_ATTRIBUTES lpProcessAttributes,
                            LPSECURITY_ATTRIBUTES lpThreadAttributes,
                            bool bInheritHandles,
                            DWORD dwCreationFlags,
                            LPVOID lpEnvironment,
                            LPCWSTR lpCurrentDirectory,
                            LPSTARTUPINFOW lpStartupInfo,
                            LPPROCESS_INFORMATION lpProcessInformation) {
  // 这里进行了CreateProcessW API的调用
  if (!::CreateProcessW(lpApplicationName, lpCommandLine, lpProcessAttributes,
                        lpThreadAttributes, bInheritHandles, dwCreationFlags,
                        lpEnvironment, lpCurrentDirectory, lpStartupInfo,
                        lpProcessInformation)) {
    return false;
  }

  DWORD process_access = kProcessRights;
  DWORD thread_access = kThreadRights;
  // 访问权限会受传入的give_full_access影响，而give_full_access对应宿主的两个EvalResult
  if (give_full_access) {
    process_access = PROCESS_ALL_ACCESS;
    thread_access = THREAD_ALL_ACCESS;
  }
  // 复制句柄给target进程，它会在IPC处理后填充到CrossCallResult中返还给target client
  if (!::DuplicateHandle(::GetCurrentProcess(), lpProcessInformation->hProcess,
                         target_process, &lpProcessInformation->hProcess,
                         process_access, false, DUPLICATE_CLOSE_SOURCE)) {
    ::CloseHandle(lpProcessInformation->hThread);
    return false;
  }
  if (!::DuplicateHandle(::GetCurrentProcess(), lpProcessInformation->hThread,
                         target_process, &lpProcessInformation->hThread,
                         thread_access, false, DUPLICATE_CLOSE_SOURCE)) {
    return false;
  }
  return true;
}
```

其他的5个action，就仅仅是封装罢了：

```cpp
NTSTATUS ProcessPolicy::OpenThreadAction(const ClientInfo& client_info,
                                         uint32_t desired_access,
                                         uint32_t thread_id,
                                         HANDLE* handle) {
  *handle = nullptr;

  NtOpenThreadFunction NtOpenThread = nullptr;
  ResolveNTFunctionPtr("NtOpenThread", &NtOpenThread);

  OBJECT_ATTRIBUTES attributes = {0};
  attributes.Length = sizeof(attributes);
  CLIENT_ID client_id = {0};
  client_id.UniqueProcess =
      reinterpret_cast<PVOID>(static_cast<ULONG_PTR>(client_info.process_id));
  client_id.UniqueThread =
      reinterpret_cast<PVOID>(static_cast<ULONG_PTR>(thread_id));

  HANDLE local_handle = nullptr;
  NTSTATUS status =
      NtOpenThread(&local_handle, desired_access, &attributes, &client_id);
  if (NT_SUCCESS(status)) {
    if (!::DuplicateHandle(::GetCurrentProcess(), local_handle,
                           client_info.process, handle, 0, false,
                           DUPLICATE_CLOSE_SOURCE | DUPLICATE_SAME_ACCESS)) {
      return STATUS_ACCESS_DENIED;
    }
  }

  return status;
}

NTSTATUS ProcessPolicy::OpenProcessAction(const ClientInfo& client_info,
                                          uint32_t desired_access,
                                          uint32_t process_id,
                                          HANDLE* handle) {
  *handle = nullptr;

  NtOpenProcessFunction NtOpenProcess = nullptr;
  ResolveNTFunctionPtr("NtOpenProcess", &NtOpenProcess);

  if (client_info.process_id != process_id)
    return STATUS_ACCESS_DENIED;

  OBJECT_ATTRIBUTES attributes = {0};
  attributes.Length = sizeof(attributes);
  CLIENT_ID client_id = {0};
  client_id.UniqueProcess =
      reinterpret_cast<PVOID>(static_cast<ULONG_PTR>(client_info.process_id));
  HANDLE local_handle = nullptr;
  NTSTATUS status =
      NtOpenProcess(&local_handle, desired_access, &attributes, &client_id);
  if (NT_SUCCESS(status)) {
    if (!::DuplicateHandle(::GetCurrentProcess(), local_handle,
                           client_info.process, handle, 0, false,
                           DUPLICATE_CLOSE_SOURCE | DUPLICATE_SAME_ACCESS)) {
      return STATUS_ACCESS_DENIED;
    }
  }

  return status;
}

NTSTATUS ProcessPolicy::OpenProcessTokenAction(const ClientInfo& client_info,
                                               HANDLE process,
                                               uint32_t desired_access,
                                               HANDLE* handle) {
  *handle = nullptr;
  NtOpenProcessTokenFunction NtOpenProcessToken = nullptr;
  ResolveNTFunctionPtr("NtOpenProcessToken", &NtOpenProcessToken);

  if (CURRENT_PROCESS != process)
    return STATUS_ACCESS_DENIED;

  HANDLE local_handle = nullptr;
  NTSTATUS status =
      NtOpenProcessToken(client_info.process, desired_access, &local_handle);
  if (NT_SUCCESS(status)) {
    if (!::DuplicateHandle(::GetCurrentProcess(), local_handle,
                           client_info.process, handle, 0, false,
                           DUPLICATE_CLOSE_SOURCE | DUPLICATE_SAME_ACCESS)) {
      return STATUS_ACCESS_DENIED;
    }
  }
  return status;
}

NTSTATUS ProcessPolicy::OpenProcessTokenExAction(const ClientInfo& client_info,
                                                 HANDLE process,
                                                 uint32_t desired_access,
                                                 uint32_t attributes,
                                                 HANDLE* handle) {
  *handle = nullptr;
  NtOpenProcessTokenExFunction NtOpenProcessTokenEx = nullptr;
  ResolveNTFunctionPtr("NtOpenProcessTokenEx", &NtOpenProcessTokenEx);

  if (CURRENT_PROCESS != process)
    return STATUS_ACCESS_DENIED;

  HANDLE local_handle = nullptr;
  NTSTATUS status = NtOpenProcessTokenEx(client_info.process, desired_access,
                                         attributes, &local_handle);
  if (NT_SUCCESS(status)) {
    if (!::DuplicateHandle(::GetCurrentProcess(), local_handle,
                           client_info.process, handle, 0, false,
                           DUPLICATE_CLOSE_SOURCE | DUPLICATE_SAME_ACCESS)) {
      return STATUS_ACCESS_DENIED;
    }
  }
  return status;
}

DWORD ProcessPolicy::CreateThreadAction(
    const ClientInfo& client_info,
    const SIZE_T stack_size,
    const LPTHREAD_START_ROUTINE start_address,
    const LPVOID parameter,
    const DWORD creation_flags,
    LPDWORD thread_id,
    HANDLE* handle) {
  *handle = nullptr;
  HANDLE local_handle =
      ::CreateRemoteThread(client_info.process, nullptr, stack_size,
                           start_address, parameter, creation_flags, thread_id);
  if (!local_handle) {
    return ::GetLastError();
  }
  if (!::DuplicateHandle(::GetCurrentProcess(), local_handle,
                         client_info.process, handle, 0, false,
                         DUPLICATE_CLOSE_SOURCE | DUPLICATE_SAME_ACCESS)) {
    return ERROR_ACCESS_DENIED;
  }
  return ERROR_SUCCESS;
}
```

IPC请求处理的后半段相关部分已经分析过了，剩下的就是回去看看target client端是如何发起请求的了。

## interceptions

target在调用以上9个WIN API或系统调用时，实际上调用到的是broker或target自己为target部署的拦截器函数。其中系统调用的拦截器由broker为target部署，而除了ntdll的其他dll（这里仅有kernel32.dll）中函数的导入表(Eat)拦截器是target自己部署的（broker会在`AddToPatchFunction`和`AddUnloadModule`后把它们传给target，sidestep和smart_sidestep虽有组件但目前尚未用到）。

对于ThreadProcess子系统的拦截器来说，一共有3种：

1. broker部署的4个系统调用拦截器
2. target自身部署的2个kernel32.dll的导入表拦截器
3. 无需IPC请求的3个系统调用拦截器

其中前两种都在process_thread_interceptions.cc中定义，后一种则独立开来，在policy_target.cc中定义。policy_target.cc中不仅包含这3个interceptions，还包含一个我们上一篇分析过的函数`QueryBroker`。

我们先看第一种，以`TargetNtOpenThread`为例：

```cpp
// Hooks NtOpenThread and proxy the call to the broker if it's trying to
// open a thread in the same process.
NTSTATUS WINAPI TargetNtOpenThread(NtOpenThreadFunction orig_OpenThread,
                                   PHANDLE thread,
                                   ACCESS_MASK desired_access,
                                   POBJECT_ATTRIBUTES object_attributes,
                                   PCLIENT_ID client_id) {
  // x86下第一个参数是原始函数
  NTSTATUS status =
      orig_OpenThread(thread, desired_access, object_attributes, client_id);
  if (NT_SUCCESS(status))
    return status;

  do {
    if (!SandboxFactory::GetTargetServices()->GetState()->InitCalled())
      break;
    if (!client_id)
      break;

    uint32_t thread_id = 0;
    bool should_break = false;
    __try {
      // We support only the calls for the current process
      if (client_id->UniqueProcess)
        should_break = true;

      // Object attributes should be nullptr or empty.
      if (!should_break && object_attributes) {
        if (object_attributes->Attributes || object_attributes->ObjectName ||
            object_attributes->RootDirectory ||
            object_attributes->SecurityDescriptor ||
            object_attributes->SecurityQualityOfService) {
          should_break = true;
        }
      }

      thread_id = static_cast<uint32_t>(
          reinterpret_cast<ULONG_PTR>(client_id->UniqueThread));
    } __except (EXCEPTION_EXECUTE_HANDLER) {
      break;
    }

    if (should_break)
      break;

    if (!ValidParameter(thread, sizeof(HANDLE), WRITE))
      break;

    void* memory = GetGlobalIPCMemory();
    if (!memory)
      break;

    // 填充参数以后，发起具体的tag（service id）的IPC请求
    SharedMemIPCClient ipc(memory);
    CrossCallReturn answer = {0};
    ResultCode code = CrossCall(ipc, IPC_NTOPENTHREAD_TAG, desired_access,
                                thread_id, &answer);
    if (SBOX_ALL_OK != code)
      break;

    if (!NT_SUCCESS(answer.nt_status))
      // The nt_status here is most likely STATUS_INVALID_CID because
      // in the broker we set the process id in the CID (client ID) param
      // to be the current process. If you try to open a thread from another
      // process you will get this INVALID_CID error. On the other hand, if you
      // try to open a thread in your own process, it should return success.
      // We don't want to return STATUS_INVALID_CID here, so we return the
      // return of the original open thread status, which is most likely
      // STATUS_ACCESS_DENIED.
      break;

    __try {
      // Write the output parameters.
      *thread = answer.handle;
    } __except (EXCEPTION_EXECUTE_HANDLER) {
      break;
    }

    return answer.nt_status;
  } while (false);

  return status;
}
```

第二类有两个，`TargetCreateProcess`和`TargetCreateThread`，前者有ASCII和Unicode两个版本，且它是有Rule可match的，后者一无所有：

```cpp
BOOL WINAPI TargetCreateProcessW(CreateProcessWFunction orig_CreateProcessW,
                                 LPCWSTR application_name,
                                 LPWSTR command_line,
                                 LPSECURITY_ATTRIBUTES process_attributes,
                                 LPSECURITY_ATTRIBUTES thread_attributes,
                                 BOOL inherit_handles,
                                 DWORD flags,
                                 LPVOID environment,
                                 LPCWSTR current_directory,
                                 LPSTARTUPINFOW startup_info,
                                 LPPROCESS_INFORMATION process_information) {
  if (SandboxFactory::GetTargetServices()->GetState()->IsCsrssConnected() &&
      orig_CreateProcessW(application_name, command_line, process_attributes,
                          thread_attributes, inherit_handles, flags,
                          environment, current_directory, startup_info,
                          process_information)) {
    return true;
  }

  // We don't trust that the IPC can work this early.
  if (!SandboxFactory::GetTargetServices()->GetState()->InitCalled())
    return false;

  // Don't call GetLastError before InitCalled() succeeds because kernel32 may
  // not be mapped yet.
  DWORD original_error = ::GetLastError();

  do {
    if (!ValidParameter(process_information, sizeof(PROCESS_INFORMATION),
                        WRITE))
      break;

    void* memory = GetGlobalIPCMemory();
    if (!memory)
      break;

    const wchar_t* cur_dir = nullptr;

    wchar_t this_current_directory[MAX_PATH];
    DWORD result = ::GetCurrentDirectory(MAX_PATH, this_current_directory);
    if (0 != result && result < MAX_PATH)
      cur_dir = this_current_directory;

    // target端也仅仅是发起了IPC请求，没有像filesystem那样在target端自己还来了一次QueryBroker
    SharedMemIPCClient ipc(memory);
    CrossCallReturn answer = {0};

    InOutCountedBuffer proc_info(process_information,
                                 sizeof(PROCESS_INFORMATION));

    ResultCode code =
        CrossCall(ipc, IPC_CREATEPROCESSW_TAG, application_name, command_line,
                  cur_dir, current_directory, proc_info, &answer);
    if (SBOX_ALL_OK != code)
      break;

    ::SetLastError(answer.win32_result);
    if (ERROR_SUCCESS != answer.win32_result)
      return false;

    return true;
  } while (false);

  ::SetLastError(original_error);
  return false;
}
```

中规中矩的`TargetCreateThread`：

```cpp
HANDLE WINAPI TargetCreateThread(CreateThreadFunction orig_CreateThread,
                                 LPSECURITY_ATTRIBUTES thread_attributes,
                                 SIZE_T stack_size,
                                 LPTHREAD_START_ROUTINE start_address,
                                 LPVOID parameter,
                                 DWORD creation_flags,
                                 LPDWORD thread_id) {
  HANDLE hThread = nullptr;

  TargetServices* target_services = SandboxFactory::GetTargetServices();
  if (!target_services || target_services->GetState()->IsCsrssConnected()) {
    hThread = orig_CreateThread(thread_attributes, stack_size, start_address,
                                parameter, creation_flags, thread_id);
    if (hThread)
      return hThread;
  }

  DWORD original_error = ::GetLastError();
  do {
    if (!target_services)
      break;

    // We don't trust that the IPC can work this early.
    if (!target_services->GetState()->InitCalled())
      break;

    __try {
      if (thread_id && !ValidParameter(thread_id, sizeof(*thread_id), WRITE))
        break;

      if (!start_address)
        break;
      // We don't support thread_attributes not being null.
      if (thread_attributes)
        break;
    } __except (EXCEPTION_EXECUTE_HANDLER) {
      break;
    }

    void* memory = GetGlobalIPCMemory();
    if (!memory)
      break;

    SharedMemIPCClient ipc(memory);
    CrossCallReturn answer = {0};

    // NOTE: we don't pass the thread_attributes through. This matches the
    // approach in CreateProcess and in CreateThreadInternal().
    ResultCode code = CrossCall(ipc, IPC_CREATETHREAD_TAG,
                                reinterpret_cast<LPVOID>(stack_size),
                                reinterpret_cast<LPVOID>(start_address),
                                parameter, creation_flags, &answer);
    if (SBOX_ALL_OK != code)
      break;

    ::SetLastError(answer.win32_result);
    if (ERROR_SUCCESS != answer.win32_result)
      return nullptr;

    __try {
      if (thread_id)
        *thread_id = ::GetThreadId(answer.handle);
      return answer.handle;
    } __except (EXCEPTION_EXECUTE_HANDLER) {
      break;
    }
  } while (false);

  ::SetLastError(original_error);
  return nullptr;
}
```

第三种就有点意思了，我们都展开看看：

```cpp
// Hooks NtSetInformationThread to block RevertToSelf from being
// called before the actual call to LowerToken.
// 看起来之所以要hook这个函数，是为了防止调用LowerToken前，进行RevertToSelf
NTSTATUS WINAPI TargetNtSetInformationThread(
    NtSetInformationThreadFunction orig_SetInformationThread,
    HANDLE thread,
    NT_THREAD_INFORMATION_CLASS thread_info_class,
    PVOID thread_information,
    ULONG thread_information_bytes) {
  do {
    // 判断LowerToken是否被掉用过，如果已经被掉用过就跳出去
    if (SandboxFactory::GetTargetServices()->GetState()->RevertedToSelf())
      break;
    // 传入的thread_info_class是否是Impersonate token，如果是就跳出去
    if (ThreadImpersonationToken != thread_info_class)
      break;
    // This is a revert to self.
    // 如果上述两种情况不满足，说明是个RevertToSelf，抱歉，不放行。
    return STATUS_SUCCESS;
  } while (false);

  // 放行
  return orig_SetInformationThread(
      thread, thread_info_class, thread_information, thread_information_bytes);
}
```

另外两个是和thread token相关：

```cpp
// Hooks NtOpenThreadToken to force the open_as_self parameter to be set to
// false if we are still running with the impersonation token. open_as_self set
// to true means that the token will be open using the process token instead of
// the impersonation token. This is bad because the process token does not have
// access to open the thread token.
NTSTATUS WINAPI
TargetNtOpenThreadToken(NtOpenThreadTokenFunction orig_OpenThreadToken,
                        HANDLE thread,
                        ACCESS_MASK desired_access,
                        BOOLEAN open_as_self,
                        PHANDLE token) {
  // 这个拦截函数纯粹是为了在LowerToken调用前，防止以open_as_self=true的形式获取
  // 线程token，此时target仍以impersonate这个高权限token在运行，所以不能让操作得逞
  // 此时获取thread token必须得是lockdown token而不能是当前的impersonate token
  if (!SandboxFactory::GetTargetServices()->GetState()->RevertedToSelf())
    open_as_self = false;

  return orig_OpenThreadToken(thread, desired_access, open_as_self, token);
}

// See comment for TargetNtOpenThreadToken
// 这个同理，双胞胎同等对待
NTSTATUS WINAPI
TargetNtOpenThreadTokenEx(NtOpenThreadTokenExFunction orig_OpenThreadTokenEx,
                          HANDLE thread,
                          ACCESS_MASK desired_access,
                          BOOLEAN open_as_self,
                          ULONG handle_attributes,
                          PHANDLE token) {
  if (!SandboxFactory::GetTargetServices()->GetState()->RevertedToSelf())
    open_as_self = false;

  return orig_OpenThreadTokenEx(thread, desired_access, open_as_self,
                                handle_attributes, token);
}
```

到此，我们就缕清了ThreadProcess子系统的相关内容，尽管和filesystem子系统有一定的不同，但整体的架构是一致的，只是具体的应用因场景而有所差异。