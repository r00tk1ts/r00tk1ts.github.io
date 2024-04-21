---
title: Chromium-sandbox-TargetPolicy-analysis
date: 2018-05-13 20:38:11
categories: 源码剖析
tags:
	- chromium
	- chromium-sandbox
---

本篇是sandbox源码剖析的第四篇，主要分析了windows平台下，对chrome的target进程应用安全策略的管家——TargetPolicy类。阅读本篇前，请先阅读前三篇。

想要流程的阅读本系列你需要以下几个条件：
1. 较高水平的C++编码能力（至少通读C++ Primer 5th，刷过课后题，有一定编码量）。
2. 熟悉Windows API编程，尤其是安全相关的内容。
3. 对二进制安全有一定了解，熟悉各类型安全漏洞成因、漏洞缓解措施及bypass手法。

<!--more-->

# chromium-sandbox-TargetPolicy-analysis

policy和target进程的关系在看过前三篇后，相必已是十分清楚了。`TargetPolicy`由`BrokerServicesBase::CreatePolicy`创建，policy对象管理`BrokerServicesBase::SpawnTarget`生成的`TargetProcess`对象。

而`TargetPolicy`实际上也是个抽象基类，chrome实际使用的是`PolicyBase`这个派生类对象，使用父类指针指向子类对象。

## `TargetPolicy`

先看看抽象基类，定义了哪些接口规范：

```cpp
class TargetPolicy {
 public:
  // Windows subsystems that can have specific rules.
  // Note: The process subsystem(SUBSY_PROCESS) does not evaluate the request
  // exactly like the CreateProcess API does. See the comment at the top of
  // process_thread_dispatcher.cc for more details.
  // Windows子系统包含的各种特定规则，包含文件、命名管道、进程、注册表、命名异步对象以及Win32K禁闭策略
  // Policy会分门别类的根据某个rule来判定动作的仲裁结果，以Semantics表示
  enum SubSystem {
    SUBSYS_FILES,           // Creation and opening of files and pipes.
    SUBSYS_NAMED_PIPES,     // Creation of named pipes.
    SUBSYS_PROCESS,         // Creation of child processes.
    SUBSYS_REGISTRY,        // Creation and opening of registry keys.
    SUBSYS_SYNC,            // Creation of named sync objects.
    SUBSYS_WIN32K_LOCKDOWN  // Win32K Lockdown related policy.
  };

  // Allowable semantics when a rule is matched.
  // rule仲裁结果，对于各个不同的子系统，都有对应的几个枚举值
  enum Semantics {
    FILES_ALLOW_ANY,       // Allows open or create for any kind of access that
                           // the file system supports.
    FILES_ALLOW_READONLY,  // Allows open or create with read access only.
    FILES_ALLOW_QUERY,     // Allows access to query the attributes of a file.
    FILES_ALLOW_DIR_ANY,   // Allows open or create with directory semantics
                           // only.
    NAMEDPIPES_ALLOW_ANY,  // Allows creation of a named pipe.
    PROCESS_MIN_EXEC,      // Allows to create a process with minimal rights
                           // over the resulting process and thread handles.
                           // No other parameters besides the command line are
                           // passed to the child process.
    PROCESS_ALL_EXEC,      // Allows the creation of a process and return full
                           // access on the returned handles.
                           // This flag can be used only when the main token of
                           // the sandboxed application is at least INTERACTIVE.
    EVENTS_ALLOW_ANY,      // Allows the creation of an event with full access.
    EVENTS_ALLOW_READONLY,  // Allows opening an even with synchronize access.
    REG_ALLOW_READONLY,     // Allows readonly access to a registry key.
    REG_ALLOW_ANY,          // Allows read and write access to a registry key.
    FAKE_USER_GDI_INIT,     // Fakes user32 and gdi32 initialization. This can
                            // be used to allow the DLLs to load and initialize
                            // even if the process cannot access that subsystem.
    IMPLEMENT_OPM_APIS      // Implements FAKE_USER_GDI_INIT and also exposes
                            // IPC calls to handle Output Protection Manager
                            // APIs.
  };

  // Increments the reference count of this object. The reference count must
  // be incremented if this interface is given to another component.
  // 下面两个是引用计数相关
  virtual void AddRef() = 0;

  // Decrements the reference count of this object. When the reference count
  // is zero the object is automatically destroyed.
  // Indicates that the caller is done with this interface. After calling
  // release no other method should be called.
  virtual void Release() = 0;

  // 下面分布了三大组件（token，job，alternative desktop）的接口函数
  // Sets the security level for the target process' two tokens.
  // This setting is permanent and cannot be changed once the target process is
  // spawned.
  // 两个token不可更改
  // initial: the security level for the initial token. This is the token that
  //   is used by the process from the creation of the process until the moment
  //   the process calls TargetServices::LowerToken() or the process calls
  //   win32's RevertToSelf(). Once this happens the initial token is no longer
  //   available and the lockdown token is in effect. Using an initial token is
  //   not compatible with AppContainer, see SetAppContainer.
  // lockdown: the security level for the token that comes into force after the
  //   process calls TargetServices::LowerToken() or the process calls
  //   RevertToSelf(). See the explanation of each level in the TokenLevel
  //   definition.
  // Return value: SBOX_ALL_OK if the setting succeeds and false otherwise.
  //   Returns false if the lockdown value is more permissive than the initial
  //   value.
  //
  // Important: most of the sandbox-provided security relies on this single
  // setting. The caller should strive to set the lockdown level as restricted
  // as possible.
  // 设置initial和lockdown Token的接口由policy提供，token作为安全限制当然应该由policy控制
  virtual ResultCode SetTokenLevel(TokenLevel initial, TokenLevel lockdown) = 0;

  // Returns the initial token level.
  virtual TokenLevel GetInitialTokenLevel() const = 0;

  // Returns the lockdown token level.
  virtual TokenLevel GetLockdownTokenLevel() const = 0;

  // Sets the security level of the Job Object to which the target process will
  // belong. This setting is permanent and cannot be changed once the target
  // process is spawned. The job controls the global security settings which
  // can not be specified in the token security profile.
  // job_level: the security level for the job. See the explanation of each
  //   level in the JobLevel definition.
  // ui_exceptions: specify what specific rights that are disabled in the
  //   chosen job_level that need to be granted. Use this parameter to avoid
  //   selecting the next permissive job level unless you need all the rights
  //   that are granted in such level.
  //   The exceptions can be specified as a combination of the following
  //   constants:
  // JOB_OBJECT_UILIMIT_HANDLES : grant access to all user-mode handles. These
  //   include windows, icons, menus and various GDI objects. In addition the
  //   target process can set hooks, and broadcast messages to other processes
  //   that belong to the same desktop.
  // JOB_OBJECT_UILIMIT_READCLIPBOARD : grant read-only access to the clipboard.
  // JOB_OBJECT_UILIMIT_WRITECLIPBOARD : grant write access to the clipboard.
  // JOB_OBJECT_UILIMIT_SYSTEMPARAMETERS : allow changes to the system-wide
  //   parameters as defined by the Win32 call SystemParametersInfo().
  // JOB_OBJECT_UILIMIT_DISPLAYSETTINGS : allow programmatic changes to the
  //  display settings.
  // JOB_OBJECT_UILIMIT_GLOBALATOMS : allow access to the global atoms table.
  // JOB_OBJECT_UILIMIT_DESKTOP : allow the creation of new desktops.
  // JOB_OBJECT_UILIMIT_EXITWINDOWS : allow the call to ExitWindows().
  //
  // Return value: SBOX_ALL_OK if the setting succeeds and false otherwise.
  //
  // Note: JOB_OBJECT_XXXX constants are defined in winnt.h and documented at
  // length in:
  //   http://msdn2.microsoft.com/en-us/library/ms684152.aspx
  //
  // Note: the recommended level is JOB_RESTRICTED or JOB_LOCKDOWN.
  // 除了token外，还有job的安全限制
  virtual ResultCode SetJobLevel(JobLevel job_level,
                                 uint32_t ui_exceptions) = 0;

  // Returns the job level.
  virtual JobLevel GetJobLevel() const = 0;

  // Sets a hard limit on the size of the commit set for the sandboxed process.
  // If the limit is reached, the process will be terminated with
  // SBOX_FATAL_MEMORY_EXCEEDED (7012).
  virtual ResultCode SetJobMemoryLimit(size_t memory_limit) = 0;

  // Specifies the desktop on which the application is going to run. If the
  // desktop does not exist, it will be created. If alternate_winstation is
  // set to true, the desktop will be created on an alternate window station.
  // windows下除了job、token外，还需要设置alternate desktop而不能用user desktop，这三大组件连同IntegrityLevel可以参考chrome的sandbox.md文档
  virtual ResultCode SetAlternateDesktop(bool alternate_winstation) = 0;

  // Returns the name of the alternate desktop used. If an alternate window
  // station is specified, the name is prepended by the window station name,
  // followed by a backslash.
  virtual base::string16 GetAlternateDesktop() const = 0;

  // Precreates the desktop and window station, if any.
  virtual ResultCode CreateAlternateDesktop(bool alternate_winstation) = 0;

  // Destroys the desktop and windows station.
  virtual void DestroyAlternateDesktop() = 0;

  // IntegrityLevel是三大组件以外的一个“冗余”安全项
  // Sets the integrity level of the process in the sandbox. Both the initial
  // token and the main token will be affected by this. If the integrity level
  // is set to a level higher than the current level, the sandbox will fail
  // to start.
  // 实际上最终是设置token的一个子集SID
  virtual ResultCode SetIntegrityLevel(IntegrityLevel level) = 0;

  // Returns the initial integrity level used.
  virtual IntegrityLevel GetIntegrityLevel() const = 0;

  // Sets the integrity level of the process in the sandbox. The integrity level
  // will not take effect before you call LowerToken. User Interface Privilege
  // Isolation is not affected by this setting and will remain off for the
  // process in the sandbox. If the integrity level is set to a level higher
  // than the current level, the sandbox will fail to start.
  // LowerToken中使用了全局变量g_shared_delayed_integrity_level来设置进程token的SID
  // 这货应该就是设置g_shared_delayed_integrity_level的，在LowerToken后这个IL才生效
  virtual ResultCode SetDelayedIntegrityLevel(IntegrityLevel level) = 0;

  // Sets the LowBox token for sandboxed process. This is mutually exclusive
  // with SetAppContainer method.
  // 这个是win8以上才有的lowbox token
  virtual ResultCode SetLowBox(const wchar_t* sid) = 0;

  // Sets the mitigations enabled when the process is created. Most of these
  // are implemented as attributes passed via STARTUPINFOEX. So they take
  // effect before any thread in the target executes. The declaration of
  // MitigationFlags is followed by a detailed description of each flag.
  // 除了3+1的安全限制外，进程的缓解措施部署也依靠policy维护
  virtual ResultCode SetProcessMitigations(MitigationFlags flags) = 0;

  // Returns the currently set mitigation flags.
  virtual MitigationFlags GetProcessMitigations() = 0;

  // Sets process mitigation flags that don't take effect before the call to
  // LowerToken().
  // 与IL一样，人性化的提供了LowerToken()前后的mitigation部署
  virtual ResultCode SetDelayedProcessMitigations(MitigationFlags flags) = 0;

  // Returns the currently set delayed mitigation flags.
  virtual MitigationFlags GetDelayedProcessMitigations() const = 0;

  // Disconnect the target from CSRSS when TargetServices::LowerToken() is
  // called inside the target.
  // 一旦LowerToken那么就与csrss断开
  virtual ResultCode SetDisconnectCsrss() = 0;

  // Sets the interceptions to operate in strict mode. By default, interceptions
  // are performed in "relaxed" mode, where if something inside NTDLL.DLL is
  // already patched we attempt to intercept it anyway. Setting interceptions
  // to strict mode means that when we detect that the function is patched we'll
  // refuse to perform the interception.
  // 拦截操作为严格模式，默认是宽松模式。
  virtual void SetStrictInterceptions() = 0;

  // Set the handles the target process should inherit for stdout and
  // stderr.  The handles the caller passes must remain valid for the
  // lifetime of the policy object.  This only has an effect on
  // Windows Vista and later versions.  These methods accept pipe and
  // file handles, but not console handles.
  // 句柄拥有者必须维持其生命周期，这里只接收管道和文件句柄，控制台句柄不行
  virtual ResultCode SetStdoutHandle(HANDLE handle) = 0;
  virtual ResultCode SetStderrHandle(HANDLE handle) = 0;

  // Adds a policy rule effective for processes spawned using this policy.
  // subsystem: One of the above enumerated windows subsystems.
  // semantics: One of the above enumerated FileSemantics.
  // pattern: A specific full path or a full path with wildcard patterns.
  //   The valid wildcards are:
  //   '*' : Matches zero or more character. Only one in series allowed.
  //   '?' : Matches a single character. One or more in series are allowed.
  // Examples:
  //   "c:\\documents and settings\\vince\\*.dmp"
  //   "c:\\documents and settings\\*\\crashdumps\\*.dmp"
  //   "c:\\temp\\app_log_?????_chrome.txt"
  // 这个就是结合上面两个枚举，给各种资源开红绿灯，Interception会用到
  virtual ResultCode AddRule(SubSystem subsystem,
                             Semantics semantics,
                             const wchar_t* pattern) = 0;

  // Adds a dll that will be unloaded in the target process before it gets
  // a chance to initialize itself. Typically, dlls that cause the target
  // to crash go here.
  // target初始化前需要unload的dll放在这儿
  virtual ResultCode AddDllToUnload(const wchar_t* dll_name) = 0;

  // Adds a handle that will be closed in the target process after lockdown.
  // A nullptr value for handle_name indicates all handles of the specified
  // type. An empty string for handle_name indicates the handle is unnamed.
  // lockdown之后target进程中需要关闭的句柄
  virtual ResultCode AddKernelObjectToClose(const wchar_t* handle_type,
                                            const wchar_t* handle_name) = 0;

  // Adds a handle that will be shared with the target process. Does not take
  // ownership of the handle.
  // target进程共享的句柄，但并非ownership
  virtual void AddHandleToShare(HANDLE handle) = 0;

  // Locks down the default DACL of the created lockdown and initial tokens
  // to restrict what other processes are allowed to access a process' kernel
  // resources.
  // 这几个不太了解
  virtual void SetLockdownDefaultDacl() = 0;

  // Enable OPM API redirection when in Win32k lockdown.
  virtual void SetEnableOPMRedirection() = 0;
  // Enable OPM API emulation when in Win32k lockdown.
  virtual bool GetEnableOPMRedirection() = 0;

  // Configure policy to use an AppContainer profile. |package_name| is the
  // name of the profile to use. Specifying True for |create_profile| ensures
  // the profile exists, if set to False process creation will fail if the
  // profile has not already been created.
  // 这个是AppContainer相关，暂时不管
  virtual ResultCode AddAppContainerProfile(const wchar_t* package_name,
                                            bool create_profile) = 0;

  // Get the configured AppContainerProfile.
  virtual scoped_refptr<AppContainerProfile> GetAppContainerProfile() = 0;

 protected:
  ~TargetPolicy() {}
};

```

一个抽象基类就已经囊括四海了，与policy对象耦合的主要是两个模块，一个是target进程，另一个就是interceptions。

### `TokenLevel`

3+1组合中，alternative desktop没啥好说的，主要是防止窗口之间消息的苟且。而Job和Token则核心在于一个level：

```cpp
// The Token level specifies a set of  security profiles designed to
// provide the bulk of the security of sandbox.
//
//  TokenLevel                 |Restricting   |Deny Only       |Privileges|
//                             |Sids          |Sids            |          |
// ----------------------------|--------------|----------------|----------|
// USER_LOCKDOWN               | Null Sid     | All            | None     |
// ----------------------------|--------------|----------------|----------|
// USER_RESTRICTED             | RESTRICTED   | All            | Traverse |
// ----------------------------|--------------|----------------|----------|
// USER_LIMITED                | Users        | All except:    | Traverse |
//                             | Everyone     | Users          |          |
//                             | RESTRICTED   | Everyone       |          |
//                             |              | Interactive    |          |
// ----------------------------|--------------|----------------|----------|
// USER_INTERACTIVE            | Users        | All except:    | Traverse |
//                             | Everyone     | Users          |          |
//                             | RESTRICTED   | Everyone       |          |
//                             | Owner        | Interactive    |          |
//                             |              | Local          |          |
//                             |              | Authent-users  |          |
//                             |              | User           |          |
// ----------------------------|--------------|----------------|----------|
// USER_NON_ADMIN              | None         | All except:    | Traverse |
//                             |              | Users          |          |
//                             |              | Everyone       |          |
//                             |              | Interactive    |          |
//                             |              | Local          |          |
//                             |              | Authent-users  |          |
//                             |              | User           |          |
// ----------------------------|--------------|----------------|----------|
// USER_RESTRICTED_SAME_ACCESS | All          | None           | All      |
// ----------------------------|--------------|----------------|----------|
// USER_UNPROTECTED            | None         | None           | All      |
// ----------------------------|--------------|----------------|----------|
//
// The above restrictions are actually a transformation that is applied to
// the existing broker process token. The resulting token that will be
// applied to the target process depends both on the token level selected
// and on the broker token itself.
//
//  The LOCKDOWN and RESTRICTED are designed to allow access to almost
//  nothing that has security associated with and they are the recommended
//  levels to run sandboxed code specially if there is a chance that the
//  broker is process might be started by a user that belongs to the Admins
//  or power users groups.
enum TokenLevel {
  USER_LOCKDOWN = 0,
  USER_RESTRICTED,
  USER_LIMITED,
  USER_INTERACTIVE,
  USER_NON_ADMIN,
  USER_RESTRICTED_SAME_ACCESS,
  USER_UNPROTECTED,
  USER_LAST
};
```

注释已经给出了非常详细的描述。

### `JobLevel`

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

### `IntegrityLevel`

IL实际上最终会折射成token的子集SID。

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

通过给不同进程不同的权限，也就做到了沙盒的资源限制的目的。这种机制的立足点在于，沙盒必须是进程粒度。

缓解措施相关此前已经看过了，都是些老生长谈的东西，不多说了。

## `PolicyBase`

真正的实现还是得看派生类：

```cpp
class PolicyBase final : public TargetPolicy {//不会再派生了
 public:
  PolicyBase();

  // TargetPolicy:
  // 这些都是必须要override的方法
  void AddRef() override;
  void Release() override;
  ResultCode SetTokenLevel(TokenLevel initial, TokenLevel lockdown) override;
  TokenLevel GetInitialTokenLevel() const override;
  TokenLevel GetLockdownTokenLevel() const override;
  ResultCode SetJobLevel(JobLevel job_level, uint32_t ui_exceptions) override;
  JobLevel GetJobLevel() const override;
  ResultCode SetJobMemoryLimit(size_t memory_limit) override;
  ResultCode SetAlternateDesktop(bool alternate_winstation) override;
  base::string16 GetAlternateDesktop() const override;
  ResultCode CreateAlternateDesktop(bool alternate_winstation) override;
  void DestroyAlternateDesktop() override;
  ResultCode SetIntegrityLevel(IntegrityLevel integrity_level) override;
  IntegrityLevel GetIntegrityLevel() const override;
  ResultCode SetDelayedIntegrityLevel(IntegrityLevel integrity_level) override;
  ResultCode SetLowBox(const wchar_t* sid) override;
  ResultCode SetProcessMitigations(MitigationFlags flags) override;
  MitigationFlags GetProcessMitigations() override;
  ResultCode SetDelayedProcessMitigations(MitigationFlags flags) override;
  MitigationFlags GetDelayedProcessMitigations() const override;
  ResultCode SetDisconnectCsrss() override;
  void SetStrictInterceptions() override;
  ResultCode SetStdoutHandle(HANDLE handle) override;
  ResultCode SetStderrHandle(HANDLE handle) override;
  ResultCode AddRule(SubSystem subsystem,
                     Semantics semantics,
                     const wchar_t* pattern) override;
  ResultCode AddDllToUnload(const wchar_t* dll_name) override;
  ResultCode AddKernelObjectToClose(const base::char16* handle_type,
                                    const base::char16* handle_name) override;
  void AddHandleToShare(HANDLE handle) override;
  void SetLockdownDefaultDacl() override;
  void SetEnableOPMRedirection() override;
  bool GetEnableOPMRedirection() override;
  ResultCode AddAppContainerProfile(const wchar_t* package_name,
                                    bool create_profile) override;
  scoped_refptr<AppContainerProfile> GetAppContainerProfile() override;
	
  // 除了override的纯虚接口，还有几个help函数或者是combo函数
  // Get the AppContainer profile as its internal type.
  // 这个应该就是GetAppContainerProfile内部所用，AppContainerProfileBase应该是干实业的派生类
  scoped_refptr<AppContainerProfileBase> GetAppContainerProfileBase();

  // Creates a Job object with the level specified in a previous call to
  // SetJobLevel().
  // 这个在SpawnTarget中调用，用于生成job对象
  ResultCode MakeJobObject(base::win::ScopedHandle* job);

  // Creates the two tokens with the levels specified in a previous call to
  // SetTokenLevel(). Also creates a lowbox token if specified based on the
  // lowbox SID.
  // 这个也在SpawnTarget中调用，用于生成三个token
  ResultCode MakeTokens(base::win::ScopedHandle* initial,
                        base::win::ScopedHandle* lockdown,
                        base::win::ScopedHandle* lowbox);

  PSID GetLowBoxSid() const;

  // Adds a target process to the internal list of targets. Internally a
  // call to TargetProcess::Init() is issued.
  // 多个target维护成内部的链表，通过这个接口来include TargetProcess
  // 要知道Policy管理TargetProcess，同一个Policy对象意味着同样的安全限制，很多target进程的安全限制当然是一致的，所以都由一个Policy来管理。
  ResultCode AddTarget(TargetProcess* target);

  // Called when there are no more active processes in a Job.
  // Removes a Job object associated with this policy and the target associated
  // with the job.
  // job内target process全部终止时的事件响应，那么它的驱动是谁呢？
  bool OnJobEmpty(HANDLE job);

  // 根据传入的params以及service id，进行可行性裁决
  // 这牵扯到PolicyProcessor、PolicyOpcode这一套Policy裁决引擎机制
  EvalResult EvalPolicy(int service, CountedParameterSetBase* params);

  HANDLE GetStdoutHandle();
  HANDLE GetStderrHandle();

  // Returns the list of handles being shared with the target process.
  const base::HandlesToInheritVector& GetHandlesBeingShared();//实际上就是std::vector<HANDLE>

 private:
  ~PolicyBase();	//外部只能new，不能delete。唯一的delete位置在Release中，private权限保护了自释放机制

  // Sets up interceptions for a new target.
  // 为target部署Interceptions
  ResultCode SetupAllInterceptions(TargetProcess* target);

  // Sets up the handle closer for a new target.
  bool SetupHandleCloser(TargetProcess* target);

  // AddRule内部所用helper
  ResultCode AddRuleInternal(SubSystem subsystem,
                             Semantics semantics,
                             const wchar_t* pattern);

  // This lock synchronizes operations on the targets_ collection.
  CRITICAL_SECTION lock_;//对targets_集合的异步锁
  // Maintains the list of target process associated with this policy.
  // The policy takes ownership of them.
  //这个targets_链表就是policy和TargetProcess的纽带了，保存了被该policy管理的所有target进程
  typedef std::list<TargetProcess*> TargetSet;
  TargetSet targets_;
  // Standard object-lifetime reference counter.
  volatile LONG ref_count;
  // The user-defined global policy settings.
  TokenLevel lockdown_level_;
  TokenLevel initial_level_;
  JobLevel job_level_;
  uint32_t ui_exceptions_;
  size_t memory_limit_;
  bool use_alternate_desktop_;
  bool use_alternate_winstation_;
  // Helps the file system policy initialization.
  bool file_system_init_;
  bool relaxed_interceptions_;
  HANDLE stdout_handle_;
  HANDLE stderr_handle_;
  IntegrityLevel integrity_level_;
  IntegrityLevel delayed_integrity_level_;
  MitigationFlags mitigations_;
  MitigationFlags delayed_mitigations_;
  bool is_csrss_connected_;
  // Object in charge of generating the low level policy.
  // Low level policy是架在PolicyOpcode和OpcodeFactory的一套rule裁决机制
  // 下面两个类型都是这套机制用到的非常那个重要的结构，前者负责生成low level policy，后者存储policy
  // 这些在日后分析policy engine及low-level policy时会详细展开。
  LowLevelPolicy* policy_maker_;
  // Memory structure that stores the low level policy.
  PolicyGlobal* policy_;
  // The list of dlls to unload in the target process.
  std::vector<base::string16> blacklisted_dlls_;
  // This is a map of handle-types to names that we need to close in the
  // target process. A null set means we need to close all handles of the
  // given type.
  HandleCloser handle_closer_;
  PSID lowbox_sid_;
  base::win::ScopedHandle lowbox_directory_;
  std::unique_ptr<Dispatcher> dispatcher_;
  bool lockdown_default_dacl_;

  static HDESK alternate_desktop_handle_;
  static HWINSTA alternate_winstation_handle_;
  static HDESK alternate_desktop_local_winstation_handle_;
  static IntegrityLevel alternate_desktop_integrity_level_label_;
  static IntegrityLevel
      alternate_desktop_local_winstation_integrity_level_label_;

  // Contains the list of handles being shared with the target process.
  // This list contains handles other than the stderr/stdout handles which are
  // shared with the target at times.
  // 保存那些可以继承的句柄，包括stderr/stdout（如果父进程IsValid的话）
  base::HandlesToInheritVector handles_to_share_;
  bool enable_opm_redirection_;	//这货我还不知道是干啥的

  scoped_refptr<AppContainerProfileBase> app_container_profile_;

  DISALLOW_COPY_AND_ASSIGN(PolicyBase);
};
```

### `构造和析构`

```cpp
PolicyBase::PolicyBase()
    : ref_count(1),//以1初始化
      lockdown_level_(USER_LOCKDOWN),
      initial_level_(USER_LOCKDOWN),
      job_level_(JOB_LOCKDOWN),
      ui_exceptions_(0),
      memory_limit_(0),
      use_alternate_desktop_(false),
      use_alternate_winstation_(false),
      file_system_init_(false),
      relaxed_interceptions_(true),
      stdout_handle_(INVALID_HANDLE_VALUE),
      stderr_handle_(INVALID_HANDLE_VALUE),
      integrity_level_(INTEGRITY_LEVEL_LAST),
      delayed_integrity_level_(INTEGRITY_LEVEL_LAST),
      mitigations_(0),
      delayed_mitigations_(0),
      is_csrss_connected_(true),
      policy_maker_(nullptr),
      policy_(nullptr),
      lowbox_sid_(nullptr),
      lockdown_default_dacl_(false),
      enable_opm_redirection_(false) {
  ::InitializeCriticalSection(&lock_);
  dispatcher_.reset(new TopLevelDispatcher(this));
}

PolicyBase::~PolicyBase() {
  TargetSet::iterator it;
  for (it = targets_.begin(); it != targets_.end(); ++it) {
    TargetProcess* target = (*it);
    delete target;
  }
  delete policy_maker_;
  delete policy_;

  if (lowbox_sid_)
    ::LocalFree(lowbox_sid_);

  ::DeleteCriticalSection(&lock_);
}
```

构造器没啥好说的，不过是成员变量初始化。至于析构，首先要删除所有的target对象，然后清理内部的policy相关对象以及其他资源。

### `引用计数`

```cpp
void PolicyBase::AddRef() {
  ::InterlockedIncrement(&ref_count);
}

void PolicyBase::Release() {
  if (0 == ::InterlockedDecrement(&ref_count))
    delete this;
}
```

都是原子操作，调用Release时，如果减到0就自我delete（还记得private析构吗）。

### token相关

```cpp
ResultCode PolicyBase::SetTokenLevel(TokenLevel initial, TokenLevel lockdown) {
  // initial token必须得比lockdown token权限高
  if (initial < lockdown) {
    return SBOX_ERROR_BAD_PARAMS;
  }
  // 仅仅只是设定成员罢了，没有Windows系统上做实际生效的操作
  initial_level_ = initial;
  lockdown_level_ = lockdown;
  return SBOX_ALL_OK;
}

TokenLevel PolicyBase::GetInitialTokenLevel() const {
  return initial_level_;
}

TokenLevel PolicyBase::GetLockdownTokenLevel() const {
  return lockdown_level_;
}

ResultCode PolicyBase::SetLowBox(const wchar_t* sid) {
  if (base::win::GetVersion() < base::win::VERSION_WIN8)
    return SBOX_ERROR_UNSUPPORTED;

  DCHECK(sid);
  // 如果app_container_profile_存在的话，是不能自行设置lowbox的，走app_container_profile_自己的那套体系
  if (lowbox_sid_ || app_container_profile_)
    return SBOX_ERROR_BAD_PARAMS;

  if (!ConvertStringSidToSid(sid, &lowbox_sid_))
    return SBOX_ERROR_GENERIC;

  return SBOX_ALL_OK;
}

// 对外直接接口，这是抽象基类没有定义的，也就是说抽象基类本不关心三个token是如何做出来的
// 但派生类的实现中，将三个token的make也放在了policy中
ResultCode PolicyBase::MakeTokens(base::win::ScopedHandle* initial,
                                  base::win::ScopedHandle* lockdown,
                                  base::win::ScopedHandle* lowbox) {
  // Create the 'naked' token. This will be the permanent token associated
  // with the process and therefore with any thread that is not impersonating.
  // CreateRestrictedToken是个很复杂的东西，整集了tokenLevel、token和IL所有相关内容
  // lockdown是OUT型参数，这里暂时简单理解成根据lockdown_level_和integrity_level_等条件做出了一个
  // 需要的lockdown token，lockdown为句柄
  // 实际上就是根据chrome的level分类填充info，最终通过CreateRestrictedToken API来创建windows的token
  DWORD result =
      CreateRestrictedToken(lockdown_level_, integrity_level_, PRIMARY,
                            lockdown_default_dacl_, lockdown);
  if (ERROR_SUCCESS != result)
    return SBOX_ERROR_GENERIC;

  // alternate desktop相关，想要理解这些代码，需要先理解Windows这方面的安全机制
  // If we're launching on the alternate desktop we need to make sure the
  // integrity label on the object is no higher than the sandboxed process's
  // integrity level. So, we lower the label on the desktop process if it's
  // not already low enough for our process.
  if (use_alternate_desktop_ && integrity_level_ != INTEGRITY_LEVEL_LAST) {
    // Integrity label enum is reversed (higher level is a lower value).
    static_assert(INTEGRITY_LEVEL_SYSTEM < INTEGRITY_LEVEL_UNTRUSTED,
                  "Integrity level ordering reversed.");
    HDESK desktop_handle = nullptr;
    IntegrityLevel desktop_integrity_level_label;
    if (use_alternate_winstation_) {
      desktop_handle = alternate_desktop_handle_;
      desktop_integrity_level_label = alternate_desktop_integrity_level_label_;
    } else {
      desktop_handle = alternate_desktop_local_winstation_handle_;
      desktop_integrity_level_label =
          alternate_desktop_local_winstation_integrity_level_label_;
    }
    // If the desktop_handle hasn't been created for any reason, skip this.
    if (desktop_handle && desktop_integrity_level_label < integrity_level_) {
      result =
          SetObjectIntegrityLabel(desktop_handle, SE_WINDOW_OBJECT, L"",
                                  GetIntegrityLevelString(integrity_level_));
      if (ERROR_SUCCESS != result)
        return SBOX_ERROR_GENERIC;

      if (use_alternate_winstation_) {
        alternate_desktop_integrity_level_label_ = integrity_level_;
      } else {
        alternate_desktop_local_winstation_integrity_level_label_ =
            integrity_level_;
      }
    }
  }

  // lowbox相关
  if (lowbox_sid_) {
    if (!lowbox_directory_.IsValid()) {
      result =
          CreateLowBoxObjectDirectory(lowbox_sid_, true, &lowbox_directory_);
      DCHECK(result == ERROR_SUCCESS);
    }

    // The order of handles isn't important in the CreateLowBoxToken call.
    // The kernel will maintain a reference to the object directory handle.
    HANDLE saved_handles[1] = {lowbox_directory_.Get()};
    DWORD saved_handles_count = lowbox_directory_.IsValid() ? 1 : 0;

    Sid package_sid(lowbox_sid_);
    SecurityCapabilities caps(package_sid);
    if (CreateLowBoxToken(lockdown->Get(), PRIMARY, &caps, saved_handles,
                          saved_handles_count, lowbox) != ERROR_SUCCESS) {
      return SBOX_ERROR_GENERIC;
    }
  }

  // Create the 'better' token. We use this token as the one that the main
  // thread uses when booting up the process. It should contain most of
  // what we need (before reaching main( ))
  // 创建initial token
  result =
      CreateRestrictedToken(initial_level_, integrity_level_, IMPERSONATION,
                            lockdown_default_dacl_, initial);
  if (ERROR_SUCCESS != result)
    return SBOX_ERROR_GENERIC;

  return SBOX_ALL_OK;
}

PSID PolicyBase::GetLowBoxSid() const {
  return lowbox_sid_;
}
```

### Job相关

```cpp
ResultCode PolicyBase::SetJobLevel(JobLevel job_level, uint32_t ui_exceptions) {
  if (memory_limit_ && job_level == JOB_NONE) {
    return SBOX_ERROR_BAD_PARAMS;
  }
  job_level_ = job_level;
  ui_exceptions_ = ui_exceptions;	//job的ui绿灯，在分析job时会看到
  return SBOX_ALL_OK;
}

JobLevel PolicyBase::GetJobLevel() const {
  return job_level_;
}

ResultCode PolicyBase::SetJobMemoryLimit(size_t memory_limit) {
  memory_limit_ = memory_limit;
  return SBOX_ALL_OK;
}

//对外直接接口，job的生成也被派生类归纳到了policy中
ResultCode PolicyBase::MakeJobObject(base::win::ScopedHandle* job) {
  if (job_level_ != JOB_NONE) {
    // Create the windows job object.
    Job job_obj;
    DWORD result =
        job_obj.Init(job_level_, nullptr, ui_exceptions_, memory_limit_);
    if (ERROR_SUCCESS != result)
      return SBOX_ERROR_GENERIC;

    *job = job_obj.Take();//转移owner
  } else {
    *job = base::win::ScopedHandle();
  }
  return SBOX_ALL_OK;
}
```

### Alternate Desktop相关

```cpp
ResultCode PolicyBase::SetAlternateDesktop(bool alternate_winstation) {
  use_alternate_desktop_ = true;
  use_alternate_winstation_ = alternate_winstation;
  return CreateAlternateDesktop(alternate_winstation);
}

base::string16 PolicyBase::GetAlternateDesktop() const {
  // No alternate desktop or winstation. Return an empty string.
  if (!use_alternate_desktop_ && !use_alternate_winstation_) {
    return base::string16();
  }

  if (use_alternate_winstation_) {
    // The desktop and winstation should have been created by now.
    // If we hit this scenario, it means that the user ignored the failure
    // during SetAlternateDesktop, so we ignore it here too.
    if (!alternate_desktop_handle_ || !alternate_winstation_handle_) {
      return base::string16();
    }
    return GetFullDesktopName(alternate_winstation_handle_,
                              alternate_desktop_handle_);
  } else {
    if (!alternate_desktop_local_winstation_handle_) {
      return base::string16();
    }
    return GetFullDesktopName(nullptr,
                              alternate_desktop_local_winstation_handle_);
  }
}

ResultCode PolicyBase::CreateAlternateDesktop(bool alternate_winstation) {
  if (alternate_winstation) {
    // Check if it's already created.
    if (alternate_winstation_handle_ && alternate_desktop_handle_)
      return SBOX_ALL_OK;

    DCHECK(!alternate_winstation_handle_);
    // Create the window station.
    ResultCode result = CreateAltWindowStation(&alternate_winstation_handle_);
    if (SBOX_ALL_OK != result)
      return result;

    // Verify that everything is fine.
    if (!alternate_winstation_handle_ ||
        GetWindowObjectName(alternate_winstation_handle_).empty())
      return SBOX_ERROR_CANNOT_CREATE_DESKTOP;

    // Create the destkop.
    result = CreateAltDesktop(alternate_winstation_handle_,
                              &alternate_desktop_handle_);
    if (SBOX_ALL_OK != result)
      return result;

    // Verify that everything is fine.
    if (!alternate_desktop_handle_ ||
        GetWindowObjectName(alternate_desktop_handle_).empty())
      return SBOX_ERROR_CANNOT_CREATE_DESKTOP;
  } else {
    // Check if it already exists.
    if (alternate_desktop_local_winstation_handle_)
      return SBOX_ALL_OK;

    // Create the destkop.
    // 实际上底层用到了SetProcessWindowStation和CreateDesktop API
    ResultCode result =
        CreateAltDesktop(nullptr, &alternate_desktop_local_winstation_handle_);
    if (SBOX_ALL_OK != result)
      return result;

    // Verify that everything is fine.
    if (!alternate_desktop_local_winstation_handle_ ||
        GetWindowObjectName(alternate_desktop_local_winstation_handle_).empty())
      return SBOX_ERROR_CANNOT_CREATE_DESKTOP;
  }

  return SBOX_ALL_OK;
}

void PolicyBase::DestroyAlternateDesktop() {
  if (use_alternate_winstation_) {
    if (alternate_desktop_handle_) {
      ::CloseDesktop(alternate_desktop_handle_);
      alternate_desktop_handle_ = nullptr;
    }

    if (alternate_winstation_handle_) {
      ::CloseWindowStation(alternate_winstation_handle_);
      alternate_winstation_handle_ = nullptr;
    }
  } else {
    if (alternate_desktop_local_winstation_handle_) {
      ::CloseDesktop(alternate_desktop_local_winstation_handle_);
      alternate_desktop_local_winstation_handle_ = nullptr;
    }
  }
}
```

具体的WinAPI封装细节就不展开了，固定的概念固定的使用套路，我个人称之为“原语”。

比如以`GetFullDesktopName`做例子，实际上非常简单：

```cpp
base::string16 GetFullDesktopName(HWINSTA winsta, HDESK desktop) {
  if (!desktop) {
    NOTREACHED();
    return base::string16();
  }

  base::string16 name;
  if (winsta) {
    name = GetWindowObjectName(winsta);
    name += L'\\';
  }

  name += GetWindowObjectName(desktop);
  return name;
}

base::string16 GetWindowObjectName(HANDLE handle) {
  // Get the size of the name.
  DWORD size = 0;
  ::GetUserObjectInformation(handle, UOI_NAME, nullptr, 0, &size);

  if (!size) {
    NOTREACHED();
    return base::string16();
  }

  // Create the buffer that will hold the name.
  std::unique_ptr<wchar_t[]> name_buffer(new wchar_t[size]);

  // Query the name of the object.
  if (!::GetUserObjectInformation(handle, UOI_NAME, name_buffer.get(), size,
                                  &size)) {
    NOTREACHED();
    return base::string16();
  }

  return base::string16(name_buffer.get());
}
```

归咎于API GetUserObjectInformation的强大。

### IntegrityLevel相关

```cpp
ResultCode PolicyBase::SetIntegrityLevel(IntegrityLevel integrity_level) {
  if (app_container_profile_)
    return SBOX_ERROR_BAD_PARAMS;
  integrity_level_ = integrity_level;
  return SBOX_ALL_OK;
}

IntegrityLevel PolicyBase::GetIntegrityLevel() const {
  return integrity_level_;
}

ResultCode PolicyBase::SetDelayedIntegrityLevel(
    IntegrityLevel integrity_level) {
  delayed_integrity_level_ = integrity_level;
  return SBOX_ALL_OK;
}
```

只是简单的传值，另外注意使用了appcontainer，就不能用integrity_level（实际上也没必要）。

### 缓解措施相关

```cpp
ResultCode PolicyBase::SetProcessMitigations(MitigationFlags flags) {
  if (app_container_profile_ || !CanSetProcessMitigationsPreStartup(flags))
    return SBOX_ERROR_BAD_PARAMS;
  mitigations_ = flags;
  return SBOX_ALL_OK;
}

MitigationFlags PolicyBase::GetProcessMitigations() {
  return mitigations_;
}

ResultCode PolicyBase::SetDelayedProcessMitigations(MitigationFlags flags) {
  if (!CanSetProcessMitigationsPostStartup(flags))
    return SBOX_ERROR_BAD_PARAMS;
  delayed_mitigations_ = flags;
  return SBOX_ALL_OK;
}

MitigationFlags PolicyBase::GetDelayedProcessMitigations() const {
  return delayed_mitigations_;
}
```

### Target管理

```cpp
ResultCode PolicyBase::AddTarget(TargetProcess* target) {
  if (policy_)
    policy_maker_->Done();

  // 先部署target的一些基础设施
  if (!ApplyProcessMitigationsToSuspendedProcess(target->Process(),
                                                 mitigations_)) {
    return SBOX_ERROR_APPLY_ASLR_MITIGATIONS;
  }

  // 部署target的Interceptions
  ResultCode ret = SetupAllInterceptions(target);

  if (ret != SBOX_ALL_OK)
    return ret;

  // 关闭句柄
  if (!SetupHandleCloser(target))
    return SBOX_ERROR_SETUP_HANDLE_CLOSER;

  DWORD win_error = ERROR_SUCCESS;
  // Initialize the sandbox infrastructure for the target.
  // TODO(wfh) do something with win_error code here.
  ret = target->Init(dispatcher_.get(), policy_, kIPCMemSize, kPolMemSize,
                     &win_error);

  if (ret != SBOX_ALL_OK)
    return ret;

  // 注意这里把delayed_integrity_level_给了g_shared_delayed_integrity_level
  // 并把这个g_shared_delayed_integrity_level传给了broker，所以broker的LowerToken才能使用
  // 合法的g_shared_delayed_integrity_level值
  g_shared_delayed_integrity_level = delayed_integrity_level_;
  ret = target->TransferVariable("g_shared_delayed_integrity_level",
                                 &g_shared_delayed_integrity_level,
                                 sizeof(g_shared_delayed_integrity_level));
  g_shared_delayed_integrity_level = INTEGRITY_LEVEL_LAST;
  if (SBOX_ALL_OK != ret)
    return ret;

  // Add in delayed mitigations and pseudo-mitigations enforced at startup.
  // 这个和g_shared_delayed_integrity_level同理，TransferVariable的作用也就显现了出来
  // 跨进程传递全局变量值（两个进程都有这个全局变量，当然不是共享，只是同名）
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
  targets_.push_back(target);	//终于放进去了
  return SBOX_ALL_OK;
}
```

### `OnJobEmpty()`

这个应该是收到job释放的消息，此时target就不要了，通过他的job来检索。

```cpp
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

本章节因为牵扯了大量的下层模块，所以初学者一定是一脸懵逼的（我第一次看这部分代码亦是如此），等到各个模块逐个击破后，也就豁然开朗了。

了解Policy这个安全管家的结构即可，这也是本次分析之旅的目的。