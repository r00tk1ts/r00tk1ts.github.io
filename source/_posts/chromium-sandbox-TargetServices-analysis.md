---
title: Chromium-sandbox-TargetServices-analysis
date: 2018-05-13 18:40:11
categories: 源码剖析
tags:
	- chromium
	- chromium-sandbox
---

本篇是sandbox源码剖析的第二篇，主要分析了windows平台下，chrome的target进程的API Interface类TargetServices。阅读本篇前，请先阅读第一篇。

想要流程的阅读本系列你需要以下几个条件：
1. 较高水平的C++编码能力（至少通读C++ Primer 5th，刷过课后题，有一定编码量）。
2. 熟悉Windows API编程，尤其是安全相关的内容。
3. 对二进制安全有一定了解，熟悉各类型安全漏洞成因、漏洞缓解措施及bypass手法。

<!--more-->

# chromium-sandbox-TargetServices-analysis

## `TargetServices`

broker进程对应的是`BrokerServices`，而像renderer进程这种target进程则对应`TargetServices`：

```cpp
// TargetServices models the current process from the perspective
// of a target process. To obtain a pointer to it use
// Sandbox::GetTargetServices(). Note that this call returns a non-null
// pointer only if this process is in fact a target. A process is a target
// only if the process was spawned by a call to BrokerServices::SpawnTarget().
//
// This API allows the target to gain access to resources with a high
// privilege token and then when it is ready to perform dangerous activities
// (such as download content from the web) it can lower its token and
// enter into locked-down (sandbox) mode.
// The typical usage is as follows:
//
//   TargetServices* target_services = Sandbox::GetTargetServices();
//   if (target_services) {
//     // We are the target.
//     target_services->Init();
//     // Do work that requires high privileges here.
//     // ....
//     // When ready to enter lock-down mode call LowerToken:
//     target_services->LowerToken();
//   }
//
// For more information see the BrokerServices API documentation.
class TargetServices {
 public:
  // Initializes the target. Must call this function before any other.
  // returns ALL_OK if successful. All other return values imply failure.
  // If the return is ERROR_GENERIC, you can call ::GetLastError() to get
  // more information.
  // 经典的Init
  virtual ResultCode Init() = 0;

  // Discards the impersonation token and uses the lower token, call before
  // processing any untrusted data or running third-party code. If this call
  // fails the current process could be terminated immediately.
  // 用于切换init token到lower token，之所以有这样的设计，是因为在关小黑屋之前，
  // 有一些需要高权限处理的任务要预先处理
  // target进程不是一开始就关小黑屋，而是调用之后才关禁闭
  virtual void LowerToken() = 0;

  // Returns the ProcessState object. Through that object it's possible to have
  // information about the current state of the process, such as whether
  // LowerToken has been called or not.
  // 获取进程的状态，ProcessState本身是个类，描述了进程状态以及csrss的连接状况
  virtual ProcessState* GetState() = 0;

 protected:
  ~TargetServices() {}	//抽象基类，外部不会直接实例化基类
};
```

和`BrokerServices`的设计如出一辙，在阅读派生类`TargetServicesBase`前，先看看`ProcessState`。

## `ProcessState`

```cpp
class ProcessState {
 public:
  ProcessState();
  // 几个RT时作为参照的状态判断，这几个函数都不会修改对象成员
  // Returns true if kernel32.dll has been loaded.
  bool IsKernel32Loaded() const;
  // Returns true if main has been called.
  bool InitCalled() const;
  // Returns true if LowerToken has been called.
  bool RevertedToSelf() const;
  // Returns true if Csrss is connected.
  bool IsCsrssConnected() const;
  // 上面的可以看成是status的get，那么有get就会有set
  // Set the current state.
  void SetKernel32Loaded();
  void SetInitCalled();
  void SetRevertedToSelf();
  void SetCsrssConnected(bool csrss_connected);

 private:
  int process_state_;
  bool csrss_connected_;
  DISALLOW_COPY_AND_ASSIGN(ProcessState);
};
```

可以说是一目了然，`process_state`是个int值，几个状态的判定有着拓扑顺序，该值随着状态的设置而逐一递增，向下是包含关系：

```cpp
ProcessState::ProcessState() : process_state_(0), csrss_connected_(true) {}

bool ProcessState::IsKernel32Loaded() const {
  return process_state_ != 0;
}

bool ProcessState::InitCalled() const {
  return process_state_ > 1;
}

bool ProcessState::RevertedToSelf() const {
  return process_state_ > 2;
}

bool ProcessState::IsCsrssConnected() const {
  return csrss_connected_;
}

void ProcessState::SetKernel32Loaded() {
  if (!process_state_)
    process_state_ = 1;
}

void ProcessState::SetInitCalled() {
  if (process_state_ < 2)
    process_state_ = 2;
}

void ProcessState::SetRevertedToSelf() {
  if (process_state_ < 3)
    process_state_ = 3;
}

void ProcessState::SetCsrssConnected(bool csrss_connected) {
  csrss_connected_ = csrss_connected;
}
```

可以看出`ProcessState`提供的仅仅是Set/Get接口罢了，真正的驱动者才有意义。

## `TargetServiceBase`

```cpp
// This class is an implementation of the  TargetServices.
// Look in the documentation of sandbox::TargetServices for more info.
// Do NOT add a destructor to this class without changing the implementation of
// the factory method.
class TargetServicesBase : public TargetServices {
 public:
  TargetServicesBase();

  // Public interface of TargetServices.
  // 三个纯虚函数的override
  ResultCode Init() override;
  void LowerToken() override;
  ProcessState* GetState() override;//TargetServiceBase才是ProcessState的真正驱动者

  // Factory method.
  // 静态工厂方法，提供给SandboxFactory类使用
  static TargetServicesBase* GetInstance();

  // Sends a simple IPC Message that has a well-known answer. Returns true
  // if the IPC was successful and false otherwise. There are 2 versions of
  // this test: 1 and 2. The first one send a simple message while the
  // second one send a message with an in/out param.
  // 一个用于测试IPC的函数，有两个版本
  bool TestIPCPing(int version);

 private:
  ~TargetServicesBase() {}	// 限制该类对象只能在堆上，或由静态方法析构，这里显然是后者
  ProcessState process_state_;	// 这个就是第三方驱动者了，target需要持有一个成员来描述进程状态
  DISALLOW_COPY_AND_ASSIGN(TargetServicesBase);//禁用拷贝构造和赋值操作
};
```

对比`BrokerServicesBase`，target显然要小一些，且target并没有应用单例模式，从Singleton继承`GetInstance`方法，所以自己提供了`GetInstance`方法供工厂类使用。

展开各成员看看具体做了什么。

### `构造和析构`

```cpp
TargetServicesBase::TargetServicesBase() {}
```

析构并未定义，只是以private权限做了声明，这里表示只能由static静态`GetInstance`方法操纵。根据类头的提示"Do NOT add a destructor to this class without changing the implementation of the factory method."，如果不修改工厂方法的实现机制，无需添加析构器。

###  `GetInstance()`

```cpp
TargetServicesBase* TargetServicesBase::GetInstance() {
  // Leak on purpose TargetServicesBase.
  if (!g_target_services)
    g_target_services = new (g_target_services_memory) TargetServicesBase;
  return g_target_services;
}
```

非常奇怪的方法，搞了个全局指针，而对象分配在了`g_target_services_memory`起始处的内存空间。所以这实际上也是单例模式，只是对象部署在了全局变量的内存空间。

```cpp
// Used as storage for g_target_services, because other allocation facilities
// are not available early. We can't use a regular function static because on
// VS2015, because the CRT tries to acquire a lock to guard initialization, but
// this code runs before the CRT is initialized.
char g_target_services_memory[sizeof(TargetServicesBase)];
TargetServicesBase* g_target_services = nullptr;
```

看起来和VS2015的CRT守护初始化时获取锁的机制有关，我也不懂。

### `Init()`

```cpp
ResultCode TargetServicesBase::Init() {
  process_state_.SetInitCalled();	//process_state_ => 2
  return SBOX_ALL_OK;
}
```

### `GetState()`

```cpp
ProcessState* TargetServicesBase::GetState() {
  return &process_state_;
}
```

### `LowerToken()`

```cpp
// Failure here is a breach of security so the process is terminated.
// 出于安全性考虑，这个函数必须成功
void TargetServicesBase::LowerToken() {
  // IntegrityLevel是应用在sandbox上的一种策略，思想源于Windows OS的IL
  // 这里使用的IL也是个全局量，应该是在call入前由其他角色设定好了
  if (ERROR_SUCCESS !=
      SetProcessIntegrityLevel(g_shared_delayed_integrity_level))
    ::TerminateProcess(::GetCurrentProcess(), SBOX_FATAL_INTEGRITY);
  // 改ProcessState的标志
  process_state_.SetRevertedToSelf(); .// process_state_ => 3
  // If the client code as called RegOpenKey, advapi32.dll has cached some
  // handles. The following code gets rid of them.
  // 清空缓存的handle
  if (!::RevertToSelf())
    ::TerminateProcess(::GetCurrentProcess(), SBOX_FATAL_DROPTOKEN);
  if (!FlushCachedRegHandles())
    ::TerminateProcess(::GetCurrentProcess(), SBOX_FATAL_FLUSHANDLES);
  if (ERROR_SUCCESS != ::RegDisablePredefinedCache())
    ::TerminateProcess(::GetCurrentProcess(), SBOX_FATAL_CACHEDISABLE);
  if (!WarmupWindowsLocales())
    ::TerminateProcess(::GetCurrentProcess(), SBOX_FATAL_WARMUP);
  bool is_csrss_connected = true;
  if (!CloseOpenHandles(&is_csrss_connected))
    ::TerminateProcess(::GetCurrentProcess(), SBOX_FATAL_CLOSEHANDLES);
  process_state_.SetCsrssConnected(is_csrss_connected);
  // Enabling mitigations must happen last otherwise handle closing breaks
  // 如果有mitigation，激活，使用的依然是全局量g_shared_delayed_mitigations
  if (g_shared_delayed_mitigations &&
      !ApplyProcessMitigationsToCurrentProcess(g_shared_delayed_mitigations))
    ::TerminateProcess(::GetCurrentProcess(), SBOX_FATAL_MITIGATION);
}
```

IntegrityLevel是sandbox使用的一种划分权限的等级制度，应用到各个进程上，不同类型的进程有着不同的IL，思想源于Windows OS本身的IL，sandbox对他进行了包装：

```cpp
// List of all the integrity levels supported in the sandbox. This is used
// only on Windows Vista and newer. You can't set the integrity level of the
// process in the sandbox to a level higher than yours.
enum IntegrityLevel {
  INTEGRITY_LEVEL_SYSTEM,
  INTEGRITY_LEVEL_HIGH,
  INTEGRITY_LEVEL_MEDIUM,
  INTEGRITY_LEVEL_MEDIUM_LOW,
  INTEGRITY_LEVEL_LOW,
  INTEGRITY_LEVEL_BELOW_LOW,
  INTEGRITY_LEVEL_UNTRUSTED,
  INTEGRITY_LEVEL_LAST
};
```

展开看`SetProcessIntegrityLevel`:

```cpp
DWORD SetProcessIntegrityLevel(IntegrityLevel integrity_level) {
  // We don't check for an invalid level here because we'll just let it
  // fail on the SetTokenIntegrityLevel call later on.
  if (integrity_level == INTEGRITY_LEVEL_LAST) {
    // No mandatory level specified, we don't change it.
    return ERROR_SUCCESS;
  }

  HANDLE token_handle;
  if (!::OpenProcessToken(GetCurrentProcess(), TOKEN_ADJUST_DEFAULT,
                          &token_handle))
    return ::GetLastError();

  // 我不太懂为什么要多此一举搞个智能指针，直接传HANDLE不行吗？
  base::win::ScopedHandle token(token_handle);

  return SetTokenIntegrityLevel(token.Get(), integrity_level);
}
```

打开了进程Token后，对其进行IL进行调整：

```cpp
DWORD SetTokenIntegrityLevel(HANDLE token, IntegrityLevel integrity_level) {
  const wchar_t* integrity_level_str = GetIntegrityLevelString(integrity_level);//这个函数做了IL到SID字符串的映射
  if (!integrity_level_str) {
    // No mandatory level specified, we don't change it.
    return ERROR_SUCCESS;
  }

  // PSID就是Windows本身提供的SID，它是Token的子集。
  // Windows API常规套路，素质三连(convert->set->free)
  PSID integrity_sid = nullptr;
  if (!::ConvertStringSidToSid(integrity_level_str, &integrity_sid))
    return ::GetLastError();

  TOKEN_MANDATORY_LABEL label = {};
  label.Label.Attributes = SE_GROUP_INTEGRITY;
  label.Label.Sid = integrity_sid;

  DWORD size = sizeof(TOKEN_MANDATORY_LABEL) + ::GetLengthSid(integrity_sid);
  bool result = ::SetTokenInformation(token, TokenIntegrityLevel, &label, size);
  auto last_error = ::GetLastError();
  ::LocalFree(integrity_sid);

  return result ? ERROR_SUCCESS : last_error;
}

// SID str都是硬编码的
const wchar_t* GetIntegrityLevelString(IntegrityLevel integrity_level) {
  switch (integrity_level) {
    case INTEGRITY_LEVEL_SYSTEM:
      return L"S-1-16-16384";
    case INTEGRITY_LEVEL_HIGH:
      return L"S-1-16-12288";
    case INTEGRITY_LEVEL_MEDIUM:
      return L"S-1-16-8192";
    case INTEGRITY_LEVEL_MEDIUM_LOW:
      return L"S-1-16-6144";
    case INTEGRITY_LEVEL_LOW:
      return L"S-1-16-4096";
    case INTEGRITY_LEVEL_BELOW_LOW:
      return L"S-1-16-2048";
    case INTEGRITY_LEVEL_UNTRUSTED:
      return L"S-1-16-0";
    case INTEGRITY_LEVEL_LAST:
      return nullptr;
  }

  NOTREACHED();
  return nullptr;
}
```

到此也就搞懂了，chrome所谓的IntergrityLevel不过是折射成了Windows进程中Token的SID。

mitigation则表示待应用到进程的各缓解措施。`MitigationFlags`是一组缓解措施flag常量：

```cpp
// These flags correspond to various process-level mitigations (eg. ASLR and
// DEP). Most are implemented via UpdateProcThreadAttribute() plus flags for
// the PROC_THREAD_ATTRIBUTE_MITIGATION_POLICY attribute argument; documented
// here: http://msdn.microsoft.com/en-us/library/windows/desktop/ms686880
// Some mitigations are implemented directly by the sandbox or emulated to
// the greatest extent possible when not directly supported by the OS.
// Flags that are unsupported for the target OS will be silently ignored.
// Flags that are invalid for their application (pre or post startup) will
// return SBOX_ERROR_BAD_PARAMS.
typedef uint64_t MitigationFlags;

// 值的设定显然是“按位或”经典设计
// Permanently enables DEP for the target process. Corresponds to
// PROCESS_CREATION_MITIGATION_POLICY_DEP_ENABLE.
const MitigationFlags MITIGATION_DEP = 0x00000001;

// Permanently Disables ATL thunk emulation when DEP is enabled. Valid
// only when MITIGATION_DEP is passed. Corresponds to not passing
// PROCESS_CREATION_MITIGATION_POLICY_DEP_ATL_THUNK_ENABLE.
const MitigationFlags MITIGATION_DEP_NO_ATL_THUNK = 0x00000002;

// Enables Structured exception handling override prevention. Must be
// enabled prior to process start. Corresponds to
// PROCESS_CREATION_MITIGATION_POLICY_SEHOP_ENABLE.
const MitigationFlags MITIGATION_SEHOP = 0x00000004;

// Forces ASLR on all images in the child process. In debug builds, must be
// enabled after startup. Corresponds to
// PROCESS_CREATION_MITIGATION_POLICY_FORCE_RELOCATE_IMAGES_ALWAYS_ON .
const MitigationFlags MITIGATION_RELOCATE_IMAGE = 0x00000008;

// Refuses to load DLLs that cannot support ASLR. In debug builds, must be
// enabled after startup. Corresponds to
// PROCESS_CREATION_MITIGATION_POLICY_FORCE_RELOCATE_IMAGES_ALWAYS_ON_REQ_RELOCS.
const MitigationFlags MITIGATION_RELOCATE_IMAGE_REQUIRED = 0x00000010;

// Terminates the process on Windows heap corruption. Coresponds to
// PROCESS_CREATION_MITIGATION_POLICY_HEAP_TERMINATE_ALWAYS_ON.
const MitigationFlags MITIGATION_HEAP_TERMINATE = 0x00000020;

// Sets a random lower bound as the minimum user address. Must be
// enabled prior to process start. On 32-bit processes this is
// emulated to a much smaller degree. Corresponds to
// PROCESS_CREATION_MITIGATION_POLICY_BOTTOM_UP_ASLR_ALWAYS_ON.
const MitigationFlags MITIGATION_BOTTOM_UP_ASLR = 0x00000040;

// Increases the randomness range of bottom-up ASLR to up to 1TB. Must be
// enabled prior to process start and with MITIGATION_BOTTOM_UP_ASLR.
// Corresponds to
// PROCESS_CREATION_MITIGATION_POLICY_HIGH_ENTROPY_ASLR_ALWAYS_ON
const MitigationFlags MITIGATION_HIGH_ENTROPY_ASLR = 0x00000080;

// Immediately raises an exception on a bad handle reference. Must be
// enabled after startup. Corresponds to
// PROCESS_CREATION_MITIGATION_POLICY_STRICT_HANDLE_CHECKS_ALWAYS_ON.
const MitigationFlags MITIGATION_STRICT_HANDLE_CHECKS = 0x00000100;

// Sets the DLL search order to LOAD_LIBRARY_SEARCH_DEFAULT_DIRS. Additional
// directories can be added via the Windows AddDllDirectory() function.
// http://msdn.microsoft.com/en-us/library/windows/desktop/hh310515
// Must be enabled after startup.
const MitigationFlags MITIGATION_DLL_SEARCH_ORDER = 0x00000200;

// Changes the mandatory integrity level policy on the current process' token
// to enable no-read and no-execute up. This prevents a lower IL process from
// opening the process token for impersonate/duplicate/assignment.
const MitigationFlags MITIGATION_HARDEN_TOKEN_IL_POLICY = 0x00000400;

// Prevents the process from making Win32k calls. Corresponds to
// PROCESS_CREATION_MITIGATION_POLICY_WIN32K_SYSTEM_CALL_DISABLE_ALWAYS_ON.
//
// Applications linked to user32.dll or gdi32.dll make Win32k calls during
// setup, even if Win32k is not otherwise used. So they also need to add a rule
// with SUBSYS_WIN32K_LOCKDOWN and semantics FAKE_USER_GDI_INIT to allow the
// initialization to succeed.
const MitigationFlags MITIGATION_WIN32K_DISABLE = 0x00000800;

// Prevents certain built-in third party extension points from being used.
// - App_Init DLLs
// - Winsock Layered Service Providers (LSPs)
// - Global Windows Hooks (NOT thread-targeted hooks)
// - Legacy Input Method Editors (IMEs)
// I.e.: Disable legacy hooking mechanisms.  Corresponds to
// PROCESS_CREATION_MITIGATION_POLICY_EXTENSION_POINT_DISABLE_ALWAYS_ON.
const MitigationFlags MITIGATION_EXTENSION_POINT_DISABLE = 0x00001000;

// Prevents the process from generating dynamic code or modifying executable
// code. Second option to allow thread-specific opt-out.
// - VirtualAlloc with PAGE_EXECUTE_*
// - VirtualProtect with PAGE_EXECUTE_*
// - MapViewOfFile with FILE_MAP_EXECUTE | FILE_MAP_WRITE
// - SetProcessValidCallTargets for CFG
// Corresponds to
// PROCESS_CREATION_MITIGATION_POLICY_PROHIBIT_DYNAMIC_CODE_ALWAYS_ON and
// PROCESS_CREATION_MITIGATION_POLICY_PROHIBIT_DYNAMIC_CODE_ALWAYS_ON_ALLOW_OPT_OUT.
const MitigationFlags MITIGATION_DYNAMIC_CODE_DISABLE = 0x00002000;
const MitigationFlags MITIGATION_DYNAMIC_CODE_DISABLE_WITH_OPT_OUT = 0x00004000;
// The following per-thread flag can be used with the
// ApplyMitigationsToCurrentThread API.  Requires the above process mitigation
// to be set on the current process.
const MitigationFlags MITIGATION_DYNAMIC_CODE_OPT_OUT_THIS_THREAD = 0x00008000;

// Prevents the process from loading non-system fonts into GDI.
// Corresponds to
// PROCESS_CREATION_MITIGATION_POLICY_FONT_DISABLE_ALWAYS_ON
const MitigationFlags MITIGATION_NONSYSTEM_FONT_DISABLE = 0x00010000;

// Prevents the process from loading binaries NOT signed by MS.
// Corresponds to
// PROCESS_CREATION_MITIGATION_POLICY_BLOCK_NON_MICROSOFT_BINARIES_ALWAYS_ON
const MitigationFlags MITIGATION_FORCE_MS_SIGNED_BINS = 0x00020000;

// Blocks mapping of images from remote devices. Corresponds to
// PROCESS_CREATION_MITIGATION_POLICY_IMAGE_LOAD_NO_REMOTE_ALWAYS_ON.
const MitigationFlags MITIGATION_IMAGE_LOAD_NO_REMOTE = 0x00040000;

// Blocks mapping of images that have the low manditory label. Corresponds to
// PROCESS_CREATION_MITIGATION_POLICY_IMAGE_LOAD_NO_LOW_LABEL_ALWAYS_ON.
const MitigationFlags MITIGATION_IMAGE_LOAD_NO_LOW_LABEL = 0x00080000;

// Forces image load preference to prioritize the Windows install System32
// folder before dll load dir, application dir and any user dirs set.
// - Affects IAT resolution standard search path only, NOT direct LoadLibrary or
//   executable search path.
// PROCESS_CREATION_MITIGATION_POLICY_IMAGE_LOAD_PREFER_SYSTEM32_ALWAYS_ON.
const MitigationFlags MITIGATION_IMAGE_LOAD_PREFER_SYS32 = 0x00100000;

// Prevents hyperthreads from interfering with indirect branch predictions.
// (SPECTRE Variant 2 mitigation.)  Corresponds to
// PROCESS_CREATION_MITIGATION_POLICY2_RESTRICT_INDIRECT_BRANCH_PREDICTION_ALWAYS_ON.
const MitigationFlags MITIGATION_RESTRICT_INDIRECT_BRANCH_PREDICTION =
    0x00200000;
```

其中大部分漏洞缓解措施，搞二进制安全的老司机大都耳熟能详。

看看如何Apply：

```cpp
bool ApplyProcessMitigationsToCurrentProcess(MitigationFlags flags) {
  //判断startup后是否可以设置这些flags，实际上就是按位判断
  if (!CanSetProcessMitigationsPostStartup(flags))
    return false;

  base::win::Version version = base::win::GetVersion();
  HMODULE module = ::GetModuleHandleA("kernel32.dll");

  // 对各种flag进行处理
  // DLL search order，win8以上在kernel32.dll引入
  if (flags & MITIGATION_DLL_SEARCH_ORDER) {
    SetDefaultDllDirectoriesFunction set_default_dll_directories =
        reinterpret_cast<SetDefaultDllDirectoriesFunction>(
            ::GetProcAddress(module, "SetDefaultDllDirectories"));

    // Check for SetDefaultDllDirectories since it requires KB2533623.
    // 设置从默认目录搜索DLL，猜测是防止从当前目录加载恶意DLL吧
    if (set_default_dll_directories) {
      if (!set_default_dll_directories(LOAD_LIBRARY_SEARCH_DEFAULT_DIRS) &&
          ERROR_ACCESS_DENIED != ::GetLastError()) {
        return false;
      }
    }
  }

  // 堆污染时直接终止，这个往往非常头疼
  // Set the heap to terminate on corruption
  if (flags & MITIGATION_HEAP_TERMINATE) {
    if (!::HeapSetInformation(nullptr, HeapEnableTerminationOnCorruption,
                              nullptr, 0) &&
        ERROR_ACCESS_DENIED != ::GetLastError()) {
      return false;
    }
  }

  //阻止更低IL进程访问进程token
  if (flags & MITIGATION_HARDEN_TOKEN_IL_POLICY) {
    DWORD error = HardenProcessIntegrityLevelPolicy();//封装了win的设定原语
    if ((error != ERROR_SUCCESS) && (error != ERROR_ACCESS_DENIED))
      return false;
  }

#if !defined(_WIN64)  // DEP is always enabled on 64-bit.
  if (flags & MITIGATION_DEP) {
    DWORD dep_flags = PROCESS_DEP_ENABLE;

    if (flags & MITIGATION_DEP_NO_ATL_THUNK)
      dep_flags |= PROCESS_DEP_DISABLE_ATL_THUNK_EMULATION;

    // 利用的是SetProcessDEPPolicy
    SetProcessDEPPolicyFunction set_process_dep_policy =
        reinterpret_cast<SetProcessDEPPolicyFunction>(
            ::GetProcAddress(module, "SetProcessDEPPolicy"));
    if (set_process_dep_policy) {
      if (!set_process_dep_policy(dep_flags) &&
          ERROR_ACCESS_DENIED != ::GetLastError()) {
        return false;
      }
    } else
      return false;
  }
#endif

  // This is all we can do in Win7 and below.
  // Win7以下能设置的都设置了，使用SetProcessMitigationPolicy
  if (version < base::win::VERSION_WIN8)
    return true;

  SetProcessMitigationPolicyFunction set_process_mitigation_policy =
      reinterpret_cast<SetProcessMitigationPolicyFunction>(
          ::GetProcAddress(module, "SetProcessMitigationPolicy"));
  if (!set_process_mitigation_policy)
    return false;

  // Enable ASLR policies.
  // ASLR
  // 借助的是SetProcessMitigationPolicy
  if (flags & MITIGATION_RELOCATE_IMAGE) {
    PROCESS_MITIGATION_ASLR_POLICY policy = {};
    policy.EnableForceRelocateImages = true;
    policy.DisallowStrippedImages =
        (flags & MITIGATION_RELOCATE_IMAGE_REQUIRED) ==
        MITIGATION_RELOCATE_IMAGE_REQUIRED;

    if (!set_process_mitigation_policy(ProcessASLRPolicy, &policy,
                                       sizeof(policy)) &&
        ERROR_ACCESS_DENIED != ::GetLastError()) {
      return false;
    }
  }

  // Enable strict handle policies.
  // 句柄的严格控制，一旦出现bad handle引用，立即抛异常
  if (flags & MITIGATION_STRICT_HANDLE_CHECKS) {
    PROCESS_MITIGATION_STRICT_HANDLE_CHECK_POLICY policy = {};
    policy.HandleExceptionsPermanentlyEnabled =
        policy.RaiseExceptionOnInvalidHandleReference = true;

    if (!set_process_mitigation_policy(ProcessStrictHandleCheckPolicy, &policy,
                                       sizeof(policy)) &&
        ERROR_ACCESS_DENIED != ::GetLastError()) {
      return false;
    }
  }

  // Enable system call policies.
  // 系统调用的禁止
  if (flags & MITIGATION_WIN32K_DISABLE) {
    PROCESS_MITIGATION_SYSTEM_CALL_DISABLE_POLICY policy = {};
    policy.DisallowWin32kSystemCalls = true;

    if (!set_process_mitigation_policy(ProcessSystemCallDisablePolicy, &policy,
                                       sizeof(policy)) &&
        ERROR_ACCESS_DENIED != ::GetLastError()) {
      return false;
    }
  }

  // Enable extension point policies.
  if (flags & MITIGATION_EXTENSION_POINT_DISABLE) {
    PROCESS_MITIGATION_EXTENSION_POINT_DISABLE_POLICY policy = {};
    policy.DisableExtensionPoints = true;

    if (!set_process_mitigation_policy(ProcessExtensionPointDisablePolicy,
                                       &policy, sizeof(policy)) &&
        ERROR_ACCESS_DENIED != ::GetLastError()) {
      return false;
    }
  }

  if (version < base::win::VERSION_WIN8_1)
    return true;

  // Enable dynamic code policies.
  if (flags & MITIGATION_DYNAMIC_CODE_DISABLE ||
      flags & MITIGATION_DYNAMIC_CODE_DISABLE_WITH_OPT_OUT) {
    PROCESS_MITIGATION_DYNAMIC_CODE_POLICY policy = {};
    policy.ProhibitDynamicCode = true;

    // Per-thread opt-out is only supported on >= Anniversary.
    if (version >= base::win::VERSION_WIN10_RS1 &&
        flags & MITIGATION_DYNAMIC_CODE_DISABLE_WITH_OPT_OUT) {
      policy.AllowThreadOptOut = true;
    }

    if (!set_process_mitigation_policy(ProcessDynamicCodePolicy, &policy,
                                       sizeof(policy)) &&
        ERROR_ACCESS_DENIED != ::GetLastError()) {
      return false;
    }
  }

  //win10以下系统，能设定的也都设定了
  if (version < base::win::VERSION_WIN10)
    return true;

  // Enable font policies.
  // win10可以禁用非系统字体的加载
  if (flags & MITIGATION_NONSYSTEM_FONT_DISABLE) {
    PROCESS_MITIGATION_FONT_DISABLE_POLICY policy = {};
    policy.DisableNonSystemFonts = true;

    if (!set_process_mitigation_policy(ProcessFontDisablePolicy, &policy,
                                       sizeof(policy)) &&
        ERROR_ACCESS_DENIED != ::GetLastError()) {
      return false;
    }
  }

  if (version < base::win::VERSION_WIN10_TH2)
    return true;

  // Enable binary signing policies.
  if (flags & MITIGATION_FORCE_MS_SIGNED_BINS) {
    PROCESS_MITIGATION_BINARY_SIGNATURE_POLICY policy = {};
    // Allow only MS signed binaries.
    policy.MicrosoftSignedOnly = true;
    // NOTE: there are two other flags available to allow
    // 1) Only Windows Store signed.
    // 2) MS-signed, Win Store signed, and WHQL signed binaries.
    // Support not added at the moment.
    if (!set_process_mitigation_policy(ProcessSignaturePolicy, &policy,
                                       sizeof(policy)) &&
        ERROR_ACCESS_DENIED != ::GetLastError()) {
      return false;
    }
  }

  // Enable image load policies.
  if (flags & MITIGATION_IMAGE_LOAD_NO_REMOTE ||
      flags & MITIGATION_IMAGE_LOAD_NO_LOW_LABEL ||
      flags & MITIGATION_IMAGE_LOAD_PREFER_SYS32) {
    PROCESS_MITIGATION_IMAGE_LOAD_POLICY policy = {};
    if (flags & MITIGATION_IMAGE_LOAD_NO_REMOTE)
      policy.NoRemoteImages = true;
    if (flags & MITIGATION_IMAGE_LOAD_NO_LOW_LABEL)
      policy.NoLowMandatoryLabelImages = true;
    // PreferSystem32 is only supported on >= Anniversary.
    if (version >= base::win::VERSION_WIN10_RS1 &&
        flags & MITIGATION_IMAGE_LOAD_PREFER_SYS32) {
      policy.PreferSystem32Images = true;
    }

    if (!set_process_mitigation_policy(ProcessImageLoadPolicy, &policy,
                                       sizeof(policy)) &&
        ERROR_ACCESS_DENIED != ::GetLastError()) {
      return false;
    }
  }

  return true;
}
```

每一种缓解措施，详细展开怕是篇幅过巨，本文旨在理解代码的架构。

### `TestIPCPing()`

```cpp
// The broker services a 'test' IPC service with the IPC_PING_TAG tag.
// broker有个test IPC服务器，使用IPC_PING_TAG标记
bool TargetServicesBase::TestIPCPing(int version) {
  void* memory = GetGlobalIPCMemory();
  if (!memory)
    return false;
  // SharedMemIPCClient是共享内存的IPC client，CrossCallReturn是返回时server填充的结构体
  SharedMemIPCClient ipc(memory);
  CrossCallReturn answer = {0};

  if (1 == version) {
    uint32_t tick1 = ::GetTickCount();
    uint32_t cookie = 717115;
    // CrossCall完成IPC请求，返回时answer已经被server填充好了
    // CrossCall封装了很多细节，具体分析到IPC机制时再展开
    ResultCode code = CrossCall(ipc, IPC_PING1_TAG, cookie, &answer);

    if (SBOX_ALL_OK != code) {
      return false;
    }
    // We should get two extended returns values from the IPC, one is the
    // tick count on the broker and the other is the cookie times two.
    if ((answer.extended_count != 2)) {
      return false;
    }
    // We test the first extended answer to be within the bounds of the tick
    // count only if there was no tick count wraparound.
    uint32_t tick2 = ::GetTickCount();
    if (tick2 >= tick1) {
      if ((answer.extended[0].unsigned_int < tick1) ||
          (answer.extended[0].unsigned_int > tick2)) {
        return false;
      }
    }

    if (answer.extended[1].unsigned_int != cookie * 2) {
      return false;
    }
  } else if (2 == version) {
    // 版本2的处理只是带上了参数，参数是个counted_buffer，携带了值cookie 717111
    // 参数值具体意义不明，这个得分析对端server才懂
    uint32_t cookie = 717111;
    InOutCountedBuffer counted_buffer(&cookie, sizeof(cookie));
    ResultCode code = CrossCall(ipc, IPC_PING2_TAG, counted_buffer, &answer);

    if (SBOX_ALL_OK != code) {
      return false;
    }
    if (cookie != 717111 * 3) {
      return false;
    }
  } else {
    return false;
  }
  return true;
}
```

这货和类主体架构关系不大，牵扯的大都是进程通信IPC的内容，以后分析到了再回头看。

## `GetTargetSerivces()`

再看工厂方法：

```cpp
// GetTargetServices implementation must follow the same technique as the
// GetBrokerServices, but in this case the logic is the opposite.
TargetServices* SandboxFactory::GetTargetServices() {
  // Can't be the target if the section handle is not valid.
  // 这里就与BrokerServices相反了，也就是说执行到这里时，g_shared_section应该已经被初始化过了
  if (!g_shared_section)
    return nullptr;
  // We are the target
  s_is_broker = false;
  // Creates and returns the target services implementation.
  return TargetServicesBase::GetInstance();
}
```

总之，`TargetServices`相关的目的就是对target进程部署IntegrityLevel、Mitigation选项等。`LowerToken()`即是降权的核心接口。 