---
title: Chromium-sandbox-Tokens-analysis
date: 2018-05-19 08:14:11
categories: 源码剖析
tags:
	- chromium
	- chromium-sandbox
---

本篇是sandbox源码剖析的第六篇，主要分析了windows平台下，Chromium对Token的封装与使用。阅读本篇前，最好阅读前四篇（本篇相对独立）。

想要流程的阅读本系列你需要以下几个条件：
1. 较高水平的C++编码能力（至少通读C++ Primer 5th，刷过课后题，有一定编码量）。
2. 熟悉Windows API编程，尤其是安全相关的内容。
3. 对二进制安全有一定了解，熟悉各类型安全漏洞成因、漏洞缓解措施及bypass手法。

<!--more-->

# chromium-sandbox-Tokens-analysis

上一节分析了Job，本节分析policy三大组件之Token。Token实际上是Windows环境下的一种安全信令，它可以作为判定资源是否可以访问的凭证。Chrome的sandbox使用了token，以此来限制target进程的访问权限。

这一节我们从已知的几个疑惑点入手，层层深入。

由于Token牵扯到了Windows系统安全鉴权方面的知识，所以建议对token不了解的读者优先阅读《Windows Internals》第六章。在搞清楚下面的问题之后，你就可以看这部分代码了。

1. token的结构是什么样的、拥有者是谁、用来干什么？personation token是谁的，用来干什么？restricted token又是什么？
2. Integrity Level是什么，用来区分什么，SID相同时，低Level进程可以访问高Level对象吗？
3. privilege和account right是什么，有什么区别？
4. SID(Security Identifier)代表什么？
5. SD(Security Descriptor)是什么，DACL又是什么，进程用token访问对象时，如何利用SD裁决请求的？

## `BrokerServicesBase` related

三个token最初是在`BrokerServicesBase::SpawnTarget`中make出来的。

```cpp
base::win::ScopedHandle initial_token;
base::win::ScopedHandle lockdown_token;
base::win::ScopedHandle lowbox_token;
ResultCode result = SBOX_ALL_OK;

result =
  policy_base->MakeTokens(&initial_token, &lockdown_token, &lowbox_token);
```

步入到`MakeToken`:

```cpp
ResultCode PolicyBase::MakeTokens(base::win::ScopedHandle* initial,
                                  base::win::ScopedHandle* lockdown,
                                  base::win::ScopedHandle* lowbox) {
  // Create the 'naked' token. This will be the permanent token associated
  // with the process and therefore with any thread that is not impersonating.
  // 这是个很关键的API，在进程token基础上，做出一个受限信令，使用的是默认DACL
  // restricted token是Windows的一个重要概念
  DWORD result =
      CreateRestrictedToken(lockdown_level_, integrity_level_, PRIMARY,
                            lockdown_default_dacl_, lockdown);
  if (ERROR_SUCCESS != result)
    return SBOX_ERROR_GENERIC;

  // If we're launching on the alternate desktop we need to make sure the
  // integrity label on the object is no higher than the sandboxed process's
  // integrity level. So, we lower the label on the desktop process if it's
  // not already low enough for our process.
  // 这一部分是对alternative desktop IL的调整，它不应该高于target的IL，所以如果
  // IL不够低，就要削成一样低
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

  // lowbox和另外两个不太一样，它的设定依赖于lowbox object
  // 它是Win8以上才能用的技术，我目前亯܌将target交给了policy来管理(`AddTarget`)。

展开看看`AddTarget`:

​```cpp
ResultCode PolicyBase::AddTarget(TargetProcess* target) {
  if (policy_)
    policy_maker_->Done();

  // mitigation实装
  if (!ApplyProcessMitigationsToSuspendedProcess(target->Process(),
                                                 mitigations_)) {
    return SBOX_ERROR_APPLY_ASLR_MITIGATIONS;
  }

  // 对target设置所有的Interceptions，Interception是一种对target进程的Hook机制
  // 内容非常丰富，以后会分析
  ResultCode ret = SetupAllInterceptions(target);

  if (ret != SBOX_ALL_OK)
    return ret;

  // broker为target进程开辟一块空间，存储传过去的handles
  // 还要把g_handles_to_close值传过去
  if (!SetupHandleCloser(target))
    return SBOX_ERROR_SETUP_HANDLE_CLOSER;

  DWORD win_error = ERROR_SUCCESS;
  // Initialize the sandbox infrastructure for the target.
  // TODO(wfh) do something with win_error code here.
  // 这里面别有洞天，现在知道传过去的都是什么鬼了
  ret = target->Init(dispatcher_.get(), policy_, kIPCMemSize, kPolMemSize,
                     &win_error);

  if (ret != SBOX_ALL_OK)
    return ret;

  // 通过broker把delayed_integrity_level_传给target
  // 这货在SpawnTarget前就由传入的policy参数对象设置好了
  // 在TargetServicesBase::LowerToken中用于降权
  g_shared_delayed_integrity_level = delayed_integrity_level_;
  ret = target->TransferVariable("g_shared_delayed_integrity_level",
                                 &g_shared_delayed_integrity_level,
                                 sizeof(g_shared_delayed_integrity_level));
  g_shared_delayed_integrity_level = INTEGRITY_LEVEL_LAST;
  if (SBOX_ALL_OK != ret)
    return ret;

  // 传入g_shared_delayed_mitigations
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

现在再看`TargetProcess::Init`:

```cpp
// Construct the IPC server and the IPC dispatcher. When the target does
// an IPC it will eventually call the dispatcher.
ResultCode TargetProcess::Init(Dispatcher* ipc_dispatcher,
                               void* policy,
                               uint32_t shared_IPC_size,
                               uint32_t shared_policy_size,
                               DWORD* win_error) {
  // We need to map the shared memory on the target. This is necessary for
  // any IPC that needs to take place, even if the target has not yet hit
  // the main( ) function or even has initialized the CRT. So here we set
  // the handle to the shared section. The target on the first IPC must do
  // the rest, which boils down to calling MapViewofFile()

  // We use this single memory pool for IPC and for policy.
  // 使用了shared_mem的IPC通信方式，以后会详细分析这些内容
  DWORD shared_mem_size =
      static_cast<DWORD>(shared_IPC_size + shared_policy_size);
  shared_section_.Set(::CreateFileMappingW(INVALID_HANDLE_VALUE, nullptr,
                                           PAGE_READWRITE | SEC_COMMIT, 0,
                                           shared_mem_size, nullptr));
  if (!shared_section_.IsValid()) {
    *win_error = ::GetLastError();
    return SBOX_ERROR_CREATE_FILE_MAPPING;
  }

  DWORD access = FILE_MAP_READ | FILE_MAP_WRITE | SECTION_QUERY;
  HANDLE target_shared_section;
  if (!::DuplicateHandle(::GetCurrentProcess(), shared_section_.Get(),
                         sandbox_process_info_.process_handle(),
                         &target_sharedࠤ؍是很清楚它的作用
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
  // initial token的创建使用的是和lockdown同样的接口，但它是IMPERSONATION而lockdown是PRIMARY
  result =
      CreateRestrictedToken(initial_level_, integrity_level_, IMPERSONATION,
                            lockdown_default_dacl_, initial);
  if (ERROR_SUCCESS != result)
    return SBOX_ERROR_GENERIC;

  return SBOX_ALL_OK;
}
```

## `RestrictedToken`

想要看懂`CreateRestrictedToken`，首先得搞清楚`RestrictedToken`类存在的意义。所以在步入`CreateRestrictedToken`之前，先拆解一下该类。

```cpp
// Handles the creation of a restricted token using the effective token or
// any token handle.
// 在一个有效的token句柄之上，创建出一个权限更为严格的token
// 在Windows中这种token叫受限令牌
// 可能从特权集中删除一些特权；该令牌SID可以被标记成Deny-only；该令牌SID可以被标记为
// restricted
// Sample usage:
//    RestrictedToken restricted_token;
//    DWORD err_code = restricted_token.Init(nullptr);  // Use the current
//                                                   // effective token
//    if (ERROR_SUCCESS != err_code) {
//      // handle error.
//    }
//
//    restricted_token.AddRestrictingSid(ATL::Sids::Users().GetPSID());
//    base::win::ScopedHandle token_handle;
//    err_code = restricted_token.GetRestrictedToken(&token_handle);
//    if (ERROR_SUCCESS != err_code) {
//      // handle error.
//    }
//    [...]
class RestrictedToken {
 public:
  // Init() has to be called before calling any other method in the class.
  RestrictedToken();
  ~RestrictedToken();

  // Initializes the RestrictedToken object with effective_token.
  // If effective_token is nullptr, it initializes the RestrictedToken object
  // with the effective token of the current process.
  // 构造器接Init素质二连，如果effective_token是nullptr，就使用当前进程的token
  DWORD Init(HANDLE effective_token);

  // Creates a restricted token.
  // If the function succeeds, the return value is ERROR_SUCCESS. If the
  // function fails, the return value is the win32 error code corresponding to
  // the error.
  // 这个就是在应用了所有的规则之后，创建出来的restricted token，在最后一步使用
  // token显然是个OUT型参数
  DWORD GetRestrictedToken(base::win::ScopedHandle* token) const;

  // Creates a restricted token and uses this new token to create a new token
  // for impersonation. Returns this impersonation token.
  //
  // If the function succeeds, the return value is ERROR_SUCCESS. If the
  // function fails, the return value is the win32 error code corresponding to
  // the error.
  //
  // The sample usage is the same as the GetRestrictedToken function.
  // 与上面类似，只是生成的是个impersonation token
  DWORD GetRestrictedTokenForImpersonation(
      base::win::ScopedHandle* token) const;

  // Lists all sids in the token and mark them as Deny Only except for those
  // present in the exceptions parameter. If there is no exception needed,
  // the caller can pass an empty list or nullptr for the exceptions
  // parameter.
  // 除了exceptions参数中指定的白名单Sid列表，token中的所有sid都被设置为Deny Only
  // 
  // If the function succeeds, the return value is ERROR_SUCCESS. If the
  // function fails, the return value is the win32 error code corresponding to
  // the error.
  //
  // Sample usage:
  //    std::vector<Sid> sid_exceptions;
  //    sid_exceptions.push_back(ATL::Sids::Users().GetPSID());
  //    sid_exceptions.push_back(ATL::Sids::World().GetPSID());
  //    restricted_token.AddAllSidsForDenyOnly(&sid_exceptions);
  // Note: A Sid marked for Deny Only in a token cannot be used to grant
  // access to any resource. It can only be used to deny access.
  DWORD AddAllSidsForDenyOnly(std::vector<Sid>* exceptions);

  // Adds a user or group SID for Deny Only in the restricted token.
  // Parameter: sid is the SID to add in the Deny Only list.
  // The return value is always ERROR_SUCCESS.
  //
  // Sample Usage:
  //    restricted_token.AddSidForDenyOnly(ATL::Sids::Admins().GetPSID());
  // 某个SID设为Deny Only
  DWORD AddSidForDenyOnly(const Sid& sid);

  // Adds the user sid of the token for Deny Only in the restricted token.
  // If the function succeeds, the return value is ERROR_SUCCESS. If the
  // function fails, the return value is the win32 error code corresponding to
  // the error.
  // 这个是user SID的版本
  DWORD AddUserSidForDenyOnly();

  // Lists all privileges in the token and add them to the list of privileges
  // to remove except for those present in the exceptions parameter. If
  // there is no exception needed, the caller can pass an empty list or nullptr
  // for the exceptions parameter.
  //
  // If the function succeeds, the return value is ERROR_SUCCESS. If the
  // function fails, the return value is the win32 error code corresponding to
  // the error.
  //
  // Sample usage:
  //    std::vector<base::string16> privilege_exceptions;
  //    privilege_exceptions.push_back(SE_CHANGE_NOTIFY_NAME);
  //    restricted_token.DeleteAllPrivileges(&privilege_exceptions);
  // 删除所有特权，除了白名单的那些privilege
  DWORD DeleteAllPrivileges(const std::vector<base::string16>* exceptions);

  // Adds a privilege to the list of privileges to remove in the restricted
  // token.
  // Parameter: privilege is the privilege name to remove. This is the string
  // representing the privilege. (e.g. "SeChangeNotifyPrivilege").
  // If the function succeeds, the return value is ERROR_SUCCESS. If the
  // function fails, the return value is the win32 error code corresponding to
  // the error.
  //
  // Sample usage:
  //    restricted_token.DeletePrivilege(SE_LOAD_DRIVER_NAME);
  // 删除某个特权
  DWORD DeletePrivilege(const wchar_t* privilege);

  // Adds a SID to the list of restricting sids in the restricted token.
  // Parameter: sid is the sid to add to the list restricting sids.
  // The return value is always ERROR_SUCCESS.
  //
  // Sample usage:
  //    restricted_token.AddRestrictingSid(ATL::Sids::Users().GetPSID());
  // Note: The list of restricting is used to force Windows to perform all
  // access checks twice. The first time using your user SID and your groups,
  // and the second time using your list of restricting sids. The access has
  // to be granted in both places to get access to the resource requested.
  // Restricting SID强制windows对所有检查进行两次。
  // 第一次使用你的用户和组SID，第二次使用restricting SID列表的SID。
  // 两次必须都被授予可访问，才能通过请求
  DWORD AddRestrictingSid(const Sid& sid);

  // Adds the logon sid of the token in the list of restricting sids for the
  // restricted token.
  //
  // If the function succeeds, the return value is ERROR_SUCCESS. If the
  // function fails, the return value is the win32 error code corresponding to
  // the error.
  // Logon SID是个特殊的SID，是登录会话进行时系统随机生成的（SID最后一节）
  DWORD AddRestrictingSidLogonSession();

  // Adds the owner sid of the token in the list of restricting sids for the
  // restricted token.
  //
  // If the function succeeds, the return value is ERROR_SUCCESS. If the
  // function fails, the return value is the win32 error code corresponding to
  // the error.
  // 把所有者SID也加到restricting SID列表
  DWORD AddRestrictingSidCurrentUser();

  // Adds all group sids and the user sid to the restricting sids list.
  //
  // If the function succeeds, the return value is ERROR_SUCCESS. If the
  // function fails, the return value is the win32 error code corresponding to
  // the error.
  // token的所有用户/组 SID都加入到restricting sid列表
  DWORD AddRestrictingSidAllSids();

  // Sets the token integrity level. This is only valid on Vista. The integrity
  // level cannot be higher than your current integrity level.
  // IL是用于隔离同一所有者不同权限资源的机制，实际上windows上最终也会折射成一个SID
  // 只是SID的不同节值表示不同含义，并以此区分开
  DWORD SetIntegrityLevel(IntegrityLevel integrity_level);

  // Set a flag which indicates the created token should have a locked down
  // default DACL when created.
  void SetLockdownDefaultDacl();

 private:
  // The list of restricting sids in the restricted token.
  std::vector<Sid> sids_to_restrict_;
  // The list of privileges to remove in the restricted token.
  std::vector<LUID> privileges_to_disable_;
  // The list of sids to mark as Deny Only in the restricted token.
  std::vector<Sid> sids_for_deny_only_;
  // The token to restrict. Can only be set in a constructor.
  base::win::ScopedHandle effective_token_;
  // The token integrity level. Only valid on Vista.
  IntegrityLevel integrity_level_;
  // Tells if the object is initialized or not (if Init() has been called)
  bool init_;
  // Lockdown the default DACL when creating new tokens.
  bool lockdown_default_dacl_;

  DISALLOW_COPY_AND_ASSIGN(RestrictedToken);
};
```

### `构造器+Init`

```cpp
RestrictedToken::RestrictedToken()
    : integrity_level_(INTEGRITY_LEVEL_LAST),
      init_(false),
      lockdown_default_dacl_(false) {}

RestrictedToken::~RestrictedToken() {}

DWORD RestrictedToken::Init(const HANDLE effective_token) {
  if (init_)
    return ERROR_ALREADY_INITIALIZED;

  HANDLE temp_token;
  if (effective_token) {
    // We duplicate the handle to be able to use it even if the original handle
    // is closed.
    // 使用受限令牌的WinAPI定式，复制句柄->打开进程令牌
    if (!::DuplicateHandle(::GetCurrentProcess(), effective_token,
                           ::GetCurrentProcess(), &temp_token, 0, false,
                           DUPLICATE_SAME_ACCESS)) {
      return ::GetLastError();
    }
  } else {
    if (!::OpenProcessToken(::GetCurrentProcess(), TOKEN_ALL_ACCESS,
                            &temp_token)) {
      return ::GetLastError();
    }
  }
  // 到这里，effective_token_成员就是当前或某个进程的句柄了（取决于传入参数是不是空）
  effective_token_.Set(temp_token);

  init_ = true;
  return ERROR_SUCCESS;
}
```

### `SID related`

```cpp
DWORD RestrictedToken::AddAllSidsForDenyOnly(std::vector<Sid>* exceptions) {
  DCHECK(init_);	//确保干这事的时候已经初始化过了
  if (!init_)
    return ERROR_NO_TOKEN;

  DWORD error;
  // Windows API GetTokenInformation的封装
  // TokenGroups这个枚举值表示所有组SID
  std::unique_ptr<BYTE[]> buffer =
      GetTokenInfo(effective_token_, TokenGroups, &error);

  if (!buffer)
    return error;

  //通过获取的group SID，构筑deny only group SID列表
  TOKEN_GROUPS* token_groups = reinterpret_cast<TOKEN_GROUPS*>(buffer.get());

  // Build the list of the deny only group SIDs
  for (unsigned int i = 0; i < token_groups->GroupCount; ++i) {
    if ((token_groups->Groups[i].Attributes & SE_GROUP_INTEGRITY) == 0 &&
        (token_groups->Groups[i].Attributes & SE_GROUP_LOGON_ID) == 0) {
      bool should_ignore = false;
      if (exceptions) {
        for (unsigned int j = 0; j < exceptions->size(); ++j) {
          if (::EqualSid((*exceptions)[j].GetPSID(),
                         token_groups->Groups[i].Sid)) {
            should_ignore = true;
            break;
          }
        }
      }
      if (!should_ignore) {
        // 装入group SID
        sids_for_deny_only_.push_back(
            reinterpret_cast<SID*>(token_groups->Groups[i].Sid));
      }
    }
  }

  return ERROR_SUCCESS;
}
// 所以上面这货是装所有组SID的
// user sid只可能有一个，但组SID则可能有多个（用户在多个组）

// 载入特定SID
// 注意没有白名单的拦截
DWORD RestrictedToken::AddSidForDenyOnly(const Sid& sid) {
  DCHECK(init_);
  if (!init_)
    return ERROR_NO_TOKEN;

  sids_for_deny_only_.push_back(sid);
  return ERROR_SUCCESS;
}

DWORD RestrictedToken::AddUserSidForDenyOnly() {
  DCHECK(init_);
  if (!init_)
    return ERROR_NO_TOKEN;

  DWORD size = sizeof(TOKEN_USER) + SECURITY_MAX_SID_SIZE;
  std::unique_ptr<BYTE[]> buffer(new BYTE[size]);
  TOKEN_USER* token_user = reinterpret_cast<TOKEN_USER*>(buffer.get());

  // TokenUser枚举量则表示获取User SID。这次没封装。
  // 因为SID仅有一个，所以知道buffer的合适大小，调用一次就行
  bool result = ::GetTokenInformation(effective_token_.Get(), TokenUser,
                                      token_user, size, &size);

  if (!result)
    return ::GetLastError();

  Sid user = reinterpret_cast<SID*>(token_user->User.Sid);
  sids_for_deny_only_.push_back(user);

  return ERROR_SUCCESS;
}
```

有意思的是获取组SID封装了一个外部函数：

```cpp
// Calls GetTokenInformation with the desired |info_class| and returns a buffer
// with the result.
std::unique_ptr<BYTE[]> GetTokenInfo(const base::win::ScopedHandle& token,
                                     TOKEN_INFORMATION_CLASS info_class,
                                     DWORD* error) {
  // Get the required buffer size.
  // 因为组SID是不定长的，无法预先知道长度，GetTokenInformation这个API允许通过传入null的方式来获取长度，然后再二次调用
  // 很多Windows API都有这个特性，好比炉石的暗影步刷一个范克里夫
  DWORD size = 0;
  ::GetTokenInformation(token.Get(), info_class, nullptr, 0, &size);
  if (!size) {
    *error = ::GetLastError();
    return nullptr;
  }

  std::unique_ptr<BYTE[]> buffer(new BYTE[size]);
  if (!::GetTokenInformation(token.Get(), info_class, buffer.get(), size,
                             &size)) {
    *error = ::GetLastError();
    return nullptr;
  }

  *error = ERROR_SUCCESS;
  return buffer;
}
```

### privilege related

```cpp
DWORD RestrictedToken::DeleteAllPrivileges(
    const std::vector<base::string16>* exceptions) {
  DCHECK(init_);
  if (!init_)
    return ERROR_NO_TOKEN;

  DWORD error;
  // Privilege是token组成的一部分，通过TokenPrivileges枚举来获取
  // 不定个privilege同样也是未知长度buffer
  std::unique_ptr<BYTE[]> buffer =
      GetTokenInfo(effective_token_, TokenPrivileges, &error);

  if (!buffer)
    return error;

  TOKEN_PRIVILEGES* token_privileges =
      reinterpret_cast<TOKEN_PRIVILEGES*>(buffer.get());

  // Build the list of privileges to disable
  for (unsigned int i = 0; i < token_privileges->PrivilegeCount; ++i) {
    bool should_ignore = false;
    if (exceptions) {
      for (unsigned int j = 0; j < exceptions->size(); ++j) {
        // 特权在windows中以LUID值存在，这里通过比较LUID来判断是不是在白名单中
        LUID luid = {0};
        ::LookupPrivilegeValue(nullptr, (*exceptions)[j].c_str(), &luid);
        if (token_privileges->Privileges[i].Luid.HighPart == luid.HighPart &&
            token_privileges->Privileges[i].Luid.LowPart == luid.LowPart) {
          should_ignore = true;
          break;
        }
      }
    }
    if (!should_ignore) {
      // 装入
      privileges_to_disable_.push_back(token_privileges->Privileges[i].Luid);
    }
  }

  return ERROR_SUCCESS;
}

// 删除某条具体的特权，这个就很easy了，注意没有白名单的拦截
DWORD RestrictedToken::DeletePrivilege(const wchar_t* privilege) {
  DCHECK(init_);
  if (!init_)
    return ERROR_NO_TOKEN;

  LUID luid = {0};
  if (LookupPrivilegeValue(nullptr, privilege, &luid))
    privileges_to_disable_.push_back(luid);
  else
    return ::GetLastError();

  return ERROR_SUCCESS;
}
```

### Restricting SID list related

```cpp
// 载入特定sid，没啥好说的
DWORD RestrictedToken::AddRestrictingSid(const Sid& sid) {
  DCHECK(init_);
  if (!init_)
    return ERROR_NO_TOKEN;

  sids_to_restrict_.push_back(sid);  // No attributes
  return ERROR_SUCCESS;
}

DWORD RestrictedToken::AddRestrictingSidLogonSession() {
  DCHECK(init_);
  if (!init_)
    return ERROR_NO_TOKEN;

  // 先找所有的group SID
  DWORD error;
  std::unique_ptr<BYTE[]> buffer =
      GetTokenInfo(effective_token_, TokenGroups, &error);

  if (!buffer)
    return error;

  TOKEN_GROUPS* token_groups = reinterpret_cast<TOKEN_GROUPS*>(buffer.get());

  SID* logon_sid = nullptr;
  // 在group SID中找那个SE_GROUP_LOGON_ID属性的SID，这个就是logon SID
  for (unsigned int i = 0; i < token_groups->GroupCount; ++i) {
    if ((token_groups->Groups[i].Attributes & SE_GROUP_LOGON_ID) != 0) {
      logon_sid = static_cast<SID*>(token_groups->Groups[i].Sid);
      break;
    }
  }

  if (logon_sid)
    sids_to_restrict_.push_back(logon_sid);

  return ERROR_SUCCESS;
}

// 这个和上面的DenyOnly设定类似
DWORD RestrictedToken::AddRestrictingSidCurrentUser() {
  DCHECK(init_);
  if (!init_)
    return ERROR_NO_TOKEN;

  DWORD size = sizeof(TOKEN_USER) + SECURITY_MAX_SID_SIZE;
  std::unique_ptr<BYTE[]> buffer(new BYTE[size]);
  TOKEN_USER* token_user = reinterpret_cast<TOKEN_USER*>(buffer.get());

  bool result = ::GetTokenInformation(effective_token_.Get(), TokenUser,
                                      token_user, size, &size);

  if (!result)
    return ::GetLastError();

  Sid user = reinterpret_cast<SID*>(token_user->User.Sid);
  sids_to_restrict_.push_back(user);

  return ERROR_SUCCESS;
}

// 这个也和Deny only的类似，对所有group SID都加入到restricting SID list
DWORD RestrictedToken::AddRestrictingSidAllSids() {
  DCHECK(init_);
  if (!init_)
    return ERROR_NO_TOKEN;

  // Add the current user to the list.
  DWORD error = AddRestrictingSidCurrentUser();
  if (ERROR_SUCCESS != error)
    return error;

  std::unique_ptr<BYTE[]> buffer =
      GetTokenInfo(effective_token_, TokenGroups, &error);

  if (!buffer)
    return error;

  TOKEN_GROUPS* token_groups = reinterpret_cast<TOKEN_GROUPS*>(buffer.get());

  // Build the list of restricting sids from all groups.
  for (unsigned int i = 0; i < token_groups->GroupCount; ++i) {
    if ((token_groups->Groups[i].Attributes & SE_GROUP_INTEGRITY) == 0)
      AddRestrictingSid(reinterpret_cast<SID*>(token_groups->Groups[i].Sid));
  }

  return ERROR_SUCCESS;
}
```

> 关于SID的意义，以及chrome对它的封装类Sid，就不展开了，实际上只是把各种类型各种渠道的SID使用不同的static接口汇集在了一起，最终都变成了一个Sid对象。具体请自己阅读Sid.h/Sid.cc。

### Integrity Level related

```cpp
DWORD RestrictedToken::SetIntegrityLevel(IntegrityLevel integrity_level) {
  // 仅仅只是个成员set，实际上IL到Token SID的转换，我们在前面已经见过了。
  integrity_level_ = integrity_level;
  return ERROR_SUCCESS;
}
```

### GetRestrictedToken related

```cpp
DWORD RestrictedToken::GetRestrictedToken(
    base::win::ScopedHandle* token) const {
  DCHECK(init_);
  if (!init_)
    return ERROR_NO_TOKEN;

  // 调用Get的时机，对三个代表的容器应该都已经设置好了
  size_t deny_size = sids_for_deny_only_.size();
  size_t restrict_size = sids_to_restrict_.size();
  size_t privileges_size = privileges_to_disable_.size();

  // SID_AND_ATTRIBUTES和LUID_AND_ATTRIBUTES是token存SID和privilege的结构体
  SID_AND_ATTRIBUTES* deny_only_array = nullptr;
  if (deny_size) {
    deny_only_array = new SID_AND_ATTRIBUTES[deny_size];

    for (unsigned int i = 0; i < sids_for_deny_only_.size(); ++i) {
      deny_only_array[i].Attributes = SE_GROUP_USE_FOR_DENY_ONLY;//这个是key
      deny_only_array[i].Sid = sids_for_deny_only_[i].GetPSID();
    }
  }

  SID_AND_ATTRIBUTES* sids_to_restrict_array = nullptr;
  if (restrict_size) {
    sids_to_restrict_array = new SID_AND_ATTRIBUTES[restrict_size];

    for (unsigned int i = 0; i < restrict_size; ++i) {
      sids_to_restrict_array[i].Attributes = 0;//同理
      sids_to_restrict_array[i].Sid = sids_to_restrict_[i].GetPSID();
    }
  }

  // 特权的结构和SID不一样
  LUID_AND_ATTRIBUTES* privileges_to_disable_array = nullptr;
  if (privileges_size) {
    privileges_to_disable_array = new LUID_AND_ATTRIBUTES[privileges_size];

    for (unsigned int i = 0; i < privileges_size; ++i) {
      privileges_to_disable_array[i].Attributes = 0;
      privileges_to_disable_array[i].Luid = privileges_to_disable_[i];
    }
  }

  bool result = true;
  HANDLE new_token_handle = nullptr;
  // The SANDBOX_INERT flag did nothing in XP and it was just a way to tell
  // if a token has ben restricted given the limiations of IsTokenRestricted()
  // but it appears that in Windows 7 it hints the AppLocker subsystem to
  // leave us alone.
  // Windows API真身在此
  if (deny_size || restrict_size || privileges_size) {
    result = ::CreateRestrictedToken(
        effective_token_.Get(), SANDBOX_INERT, static_cast<DWORD>(deny_size),
        deny_only_array, static_cast<DWORD>(privileges_size),
        privileges_to_disable_array, static_cast<DWORD>(restrict_size),
        sids_to_restrict_array, &new_token_handle);
  } else {
    // Duplicate the token even if it's not modified at this point
    // because any subsequent changes to this token would also affect the
    // current process.
    result = ::DuplicateTokenEx(effective_token_.Get(), TOKEN_ALL_ACCESS,
                                nullptr, SecurityIdentification, TokenPrimary,
                                &new_token_handle);
  }
  auto last_error = ::GetLastError();

  // 清理资源
  if (deny_only_array)
    delete[] deny_only_array;

  if (sids_to_restrict_array)
    delete[] sids_to_restrict_array;

  if (privileges_to_disable_array)
    delete[] privileges_to_disable_array;

  if (!result)
    return last_error;

  base::win::ScopedHandle new_token(new_token_handle);

  if (lockdown_default_dacl_) {
    // Don't add Restricted sid and also remove logon sid access.
    if (!RevokeLogonSidFromDefaultDacl(new_token.Get()))
      return ::GetLastError();
  } else {
    // Modify the default dacl on the token to contain Restricted.
    if (!AddSidToDefaultDacl(new_token.Get(), WinRestrictedCodeSid,
                             GRANT_ACCESS, GENERIC_ALL)) {
      return ::GetLastError();
    }
  }

  // Add user to default dacl.
  if (!AddUserSidToDefaultDacl(new_token.Get(), GENERIC_ALL))
    return ::GetLastError();

  // 对IntegrityLevel的处理在这儿，实际上这个已经看过了，就是到SID的转换
  // 然后通过SetTokenInformation API来设置属性为SE_GROUP_INTEGRITY的SID
  // 换句话说，IL也不过是SID的一个子集，而SID是token的子集
  DWORD error = SetTokenIntegrityLevel(new_token.Get(), integrity_level_);
  if (ERROR_SUCCESS != error)
    return error;

  HANDLE token_handle;
  if (!::DuplicateHandle(::GetCurrentProcess(), new_token.Get(),
                         ::GetCurrentProcess(), &token_handle, TOKEN_ALL_ACCESS,
                         false,  // Don't inherit.
                         0)) {
    return ::GetLastError();
  }

  token->Set(token_handle);
  return ERROR_SUCCESS;
}

DWORD RestrictedToken::GetRestrictedTokenForImpersonation(
    base::win::ScopedHandle* token) const {
  DCHECK(init_);
  if (!init_)
    return ERROR_NO_TOKEN;

  base::win::ScopedHandle restricted_token;
  DWORD err_code = GetRestrictedToken(&restricted_token);
  if (ERROR_SUCCESS != err_code)
    return err_code;

  HANDLE impersonation_token_handle;
  if (!::DuplicateToken(restricted_token.Get(), SecurityImpersonation,
                        &impersonation_token_handle)) {
    return ::GetLastError();
  }
  base::win::ScopedHandle impersonation_token(impersonation_token_handle);

  HANDLE token_handle;
  if (!::DuplicateHandle(::GetCurrentProcess(), impersonation_token.Get(),
                         ::GetCurrentProcess(), &token_handle, TOKEN_ALL_ACCESS,
                         false,  // Don't inherit.
                         0)) {
    return ::GetLastError();
  }

  token->Set(token_handle);
  return ERROR_SUCCESS;
}
```

## `CreateRestrictedToken`

```cpp
// Creates a restricted token based on the effective token of the current
// process. The parameter security_level determines how much the token is
// restricted. The token_type determines if the token will be used as a primary
// token or impersonation token. The integrity level of the token is set to
// |integrity level| on Vista only.
// |token| is the output value containing the handle of the newly created
// restricted token.
// |lockdown_default_dacl| indicates the token's default DACL should be locked
// down to restrict what other process can open kernel resources created while
// running under the token.
// If the function succeeds, the return value is ERROR_SUCCESS. If the
// function fails, the return value is the win32 error code corresponding to
// the error.
// 根据注释多少能明白一些，传入的这些参数都是PolicyBase对象在调用前就设置好了的成员变量
DWORD CreateRestrictedToken(TokenLevel security_level,
                            IntegrityLevel integrity_level,
                            TokenType token_type,
                            bool lockdown_default_dacl,
                            base::win::ScopedHandle* token) {
  RestrictedToken restricted_token;	// 构造接Init素质二连
  restricted_token.Init(nullptr);  // Initialized with the current process token
  // 如果有默认DACL，就设置下去
  if (lockdown_default_dacl)
    restricted_token.SetLockdownDefaultDacl();

  // 这两个是用于开绿灯的白名单，一个是不要DenyOnly的SID，一个是不要删除的特权
  // 这就看出了chrome的狠毒，除了白名单的指派其他一无所有。
  std::vector<base::string16> privilege_exceptions;
  std::vector<Sid> sid_exceptions;

  bool deny_sids = true;
  bool remove_privileges = true;

  // TokenLevel是chrome封装的，到这里再会头看它定义处的注释，是否就豁然开朗了？
  // 下面的分支处理，不过是对不同level的定义，部署3个代表（restricting SID/Deny Only SID/Privileges）
  switch (security_level) {
    // 无需保护，关闭sid deny和privilege remove
    case USER_UNPROTECTED: {
      deny_sids = false;
      remove_privileges = false;
      break;
    }
    case USER_RESTRICTED_SAME_ACCESS: {
      deny_sids = false;
      remove_privileges = false;

      unsigned err_code = restricted_token.AddRestrictingSidAllSids();
      if (ERROR_SUCCESS != err_code)
        return err_code;

      break;
    }
    case USER_NON_ADMIN: {
      sid_exceptions.push_back(WinBuiltinUsersSid);
      sid_exceptions.push_back(WinWorldSid);
      sid_exceptions.push_back(WinInteractiveSid);
      sid_exceptions.push_back(WinAuthenticatedUserSid);
      privilege_exceptions.push_back(SE_CHANGE_NOTIFY_NAME);
      break;
    }
    case USER_INTERACTIVE: {
      sid_exceptions.push_back(WinBuiltinUsersSid);
      sid_exceptions.push_back(WinWorldSid);
      sid_exceptions.push_back(WinInteractiveSid);
      sid_exceptions.push_back(WinAuthenticatedUserSid);
      privilege_exceptions.push_back(SE_CHANGE_NOTIFY_NAME);
      restricted_token.AddRestrictingSid(WinBuiltinUsersSid);
      restricted_token.AddRestrictingSid(WinWorldSid);
      restricted_token.AddRestrictingSid(WinRestrictedCodeSid);
      restricted_token.AddRestrictingSidCurrentUser();
      restricted_token.AddRestrictingSidLogonSession();
      break;
    }
    case USER_LIMITED: {
      sid_exceptions.push_back(WinBuiltinUsersSid);
      sid_exceptions.push_back(WinWorldSid);
      sid_exceptions.push_back(WinInteractiveSid);
      privilege_exceptions.push_back(SE_CHANGE_NOTIFY_NAME);
      restricted_token.AddRestrictingSid(WinBuiltinUsersSid);
      restricted_token.AddRestrictingSid(WinWorldSid);
      restricted_token.AddRestrictingSid(WinRestrictedCodeSid);

      // This token has to be able to create objects in BNO.
      // Unfortunately, on Vista+, it needs the current logon sid
      // in the token to achieve this. You should also set the process to be
      // low integrity level so it can't access object created by other
      // processes.
      restricted_token.AddRestrictingSidLogonSession();
      break;
    }
    case USER_RESTRICTED: {
      privilege_exceptions.push_back(SE_CHANGE_NOTIFY_NAME);
      restricted_token.AddUserSidForDenyOnly();
      restricted_token.AddRestrictingSid(WinRestrictedCodeSid);
      break;
    }
    case USER_LOCKDOWN: {
      restricted_token.AddUserSidForDenyOnly();
      restricted_token.AddRestrictingSid(WinNullSid);
      break;
    }
    default: { return ERROR_BAD_ARGUMENTS; }
  }

  DWORD err_code = ERROR_SUCCESS;
  if (deny_sids) {
    err_code = restricted_token.AddAllSidsForDenyOnly(&sid_exceptions);
    if (ERROR_SUCCESS != err_code)
      return err_code;
  }

  if (remove_privileges) {
    err_code = restricted_token.DeleteAllPrivileges(&privilege_exceptions);
    if (ERROR_SUCCESS != err_code)
      return err_code;
  }

  // 这个只是填充个成员变量罢了
  restricted_token.SetIntegrityLevel(integrity_level);

  // lockdown token是PRIMARY而initial token是IMPERSONATION
  // IMPERSONATION是个临时的token
  switch (token_type) {
    case PRIMARY: {
      err_code = restricted_token.GetRestrictedToken(token);
      break;
    }
    case IMPERSONATION: {
      err_code = restricted_token.GetRestrictedTokenForImpersonation(token);
      break;
    }
    default: {
      err_code = ERROR_BAD_ARGUMENTS;
      break;
    }
  }

  return err_code;
}
```

## `SpawnTarget related`

回到`BrokerServicesBase::SpawnTarget`中，通过`policy_base->MakeToken`做出了三个token后，将initial和lockdown传给了`TargetProcess`构造器，new出target进程对象。

```cpp
TargetProcess* target = new TargetProcess(
      std::move(initial_token), std::move(lockdown_token), job.Get(),
      thread_pool_.get(),
      profile ? profile->GetImpersonationCapabilities() : std::vector<Sid>());
```

而在构造器中，仅仅只是将两个token的所有权移给了成员`lockdown_token_`和`initial_token_`。此后在`TargetProcess::Create`中调用`CreateProcessAsUserW` API时，使用了lockdown。该API的调用也是Windows使用token的定式。

所以target进程在创建的时候，使用的是lockdown token作为process的token。

此后，`TargetProcess::Create`继续处理initial token：

```cpp
if (initial_token_.IsValid()) {
    HANDLE impersonation_token = initial_token_.Get();
    base::win::ScopedHandle app_container_token;
  	// 这里是判断是否应用了AC，如果是的话，就用AC的app_container_token取代initial token
    if (GetAppContainerImpersonationToken(
            process_info.process_handle(), impersonation_token,
            impersonation_capabilities_, &app_container_token)) {
      impersonation_token = app_container_token.Get();
    }

    // Change the token of the main thread of the new process for the
    // impersonation token with more rights. This allows the target to start;
    // otherwise it will crash too early for us to help.
  	// 主线程临时使用这个impersonation token
    HANDLE temp_thread = process_info.thread_handle();
    if (!::SetThreadToken(&temp_thread, impersonation_token)) {
      *win_error = ::GetLastError();
      ::TerminateProcess(process_info.process_handle(), 0);
      return SBOX_ERROR_SET_THREAD_TOKEN;
    }
  	// 只能用一次，这个东西此后就没用了，关闭掉
    initial_token_.Close();
  }
```

返回到`SpawnTarget`，此后会继续判断lowbox token的合法性，如果有就`target->AssignLowBoxToken(lowbox_token);`。关于这个lowbox的使用意义，我暂时不深究。

此时，target进程已经做出来了，token也配置好了，但target是挂起的状态。

`SpawnTarget`在`target->Create`之后_section, access, false, 0)) {
    *win_error = ::GetLastError();
    return SBOX_ERROR_DUPLICATE_SHARED_SECTION;
  }

  void* shared_memory = ::MapViewOfFile(
      shared_section_.Get(), FILE_MAP_WRITE | FILE_MAP_READ, 0, 0, 0);
  if (!shared_memory) {
    *win_error = ::GetLastError();
    return SBOX_ERROR_MAP_VIEW_OF_SHARED_SECTION;
  }

  // 通过MapViewOfFile的共享内存方式，把policy传给target
  CopyPolicyToTarget(policy, shared_policy_size,
                     reinterpret_cast<char*>(shared_memory) + shared_IPC_size);

  ResultCode ret;
  // Set the global variables in the target. These are not used on the broker.
  g_shared_section = target_shared_section;
  ret = TransferVariable("g_shared_section", &g_shared_section,
                         sizeof(g_shared_section));
  g_shared_section = nullptr;
  if (SBOX_ALL_OK != ret) {
    *win_error = ::GetLastError();
    return ret;
  }
  g_shared_IPC_size = shared_IPC_size;
  ret = TransferVariable("g_shared_IPC_size", &g_shared_IPC_size,
                         sizeof(g_shared_IPC_size));
  g_shared_IPC_size = 0;
  if (SBOX_ALL_OK != ret) {
    *win_error = ::GetLastError();
    return ret;
  }
  g_shared_policy_size = shared_policy_size;
  ret = TransferVariable("g_shared_policy_size", &g_shared_policy_size,
                         sizeof(g_shared_policy_size));
  g_shared_policy_size = 0;
  if (SBOX_ALL_OK != ret) {
    *win_error = ::GetLastError();
    return ret;
  }

  ipc_server_.reset(new SharedMemIPCServer(
      sandbox_process_info_.process_handle(),
      sandbox_process_info_.process_id(), thread_pool_, ipc_dispatcher));

  if (!ipc_server_->Init(shared_memory, shared_IPC_size, kIPCChannelSize))
    return SBOX_ERROR_NO_SPACE;

  // After this point we cannot use this handle anymore.
  ::CloseHandle(sandbox_process_info_.TakeThreadHandle());

  return SBOX_ALL_OK;
}
```

到这里，broker能对target进程做的就都做完了。而控制target运行的runner则又是另一片天地了。

回头看`TargetServicesBase::LowerToken`:

​```cpp
// Failure here is a breach of security so the process is terminated.
void TargetServicesBase::LowerToken() {
  if (ERROR_SUCCESS !=
      SetProcessIntegrityLevel(g_shared_delayed_integrity_level))
    ::TerminateProcess(::GetCurrentProcess(), SBOX_FATAL_INTEGRITY);
  process_state_.SetRevertedToSelf();
 
  // If the client code as called RegOpenKey, advapi32.dll has cached some
  // handles. The following code gets rid of them.
  // 这个API名称虽然不起眼，但却大有作为，实际上他完成了从撤销impersonate到恢复primary token的过程
  if (!::RevertToSelf())
    ::TerminateProcess(::GetCurrentProcess(), SBOX_FATAL_DROPTOKEN);
  // 这些先不关心，处理cache handle
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
  // 处理mitigation
  // Enabling mitigations must happen last otherwise handle closing breaks
  if (g_shared_delayed_mitigations &&
      !ApplyProcessMitigationsToCurrentProcess(g_shared_delayed_mitigations))
    ::TerminateProcess(::GetCurrentProcess(), SBOX_FATAL_MITIGATION);
}

// 这个就一目了然了，使用新的integrity_level
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

  base::win::ScopedHandle token(token_handle);

  return SetTokenIntegrityLevel(token.Get(), integrity_level);
}

// 这些早先已经看过了
DWORD SetTokenIntegrityLevel(HANDLE token, IntegrityLevel integrity_level) {
  const wchar_t* integrity_level_str = GetIntegrityLevelString(integrity_level);
  if (!integrity_level_str) {
    // No mandatory level specified, we don't change it.
    return ERROR_SUCCESS;
  }

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

说白了，除了token有两个，initial和lockdown，mitigation和IL也有两个，后者更为严格且作为delayed项。当`LowerToken`时一并设置下去，而它们都是由broker的policy_base一早就设置好了的，并通过IPC传给了target。

## `PolicyBase` related

到此，broker对target的所作所为，涉及到token的地方，就全都清楚了。而在`LowerToken`处我们发现还有个delayed IL设置了下去，这个IL并非由broker控制，而是传入到`SpawnTarget`时，policy_base已经设置好了。

而`PolicyBase`的Owner就尤为关键了，policy的创建是由`BrokerServices::CreatePolicy`控制的，那么policy对象应该是独立的，在`SpawnTarget`之前，policy会调用成员函数来设置各种组件，比如我们研究的token。

在sandbox的源码中，可以参考一些test单元测试文件来验证猜想(policy_target_test.cc)。

随便贴一个：

```cpp
// Launches the app in the sandbox and ask it to wait in an
// infinite loop. Waits for 2 seconds and then check if the
// desktop associated with the app thread is not the same as the
// current desktop.
TEST(PolicyTargetTest, DesktopPolicy) {
  BrokerServices* broker = GetBroker();

  // Precreate the desktop.
  scoped_refptr<TargetPolicy> temp_policy = broker->CreatePolicy();
  temp_policy->CreateAlternateDesktop(false);
  temp_policy = nullptr;

  ASSERT_TRUE(broker);

  // Get the path to the sandboxed app.
  wchar_t prog_name[MAX_PATH];
  GetModuleFileNameW(nullptr, prog_name, MAX_PATH);

  base::string16 arguments(L"\"");
  arguments += prog_name;
  arguments += L"\" -child 0 wait";  // Don't care about the "state" argument.

  // Launch the app.
  ResultCode result = SBOX_ALL_OK;
  ResultCode warning_result = SBOX_ALL_OK;
  DWORD last_error = ERROR_SUCCESS;
  base::win::ScopedProcessInformation target;

  scoped_refptr<TargetPolicy> policy = broker->CreatePolicy();
  policy->SetAlternateDesktop(false);
  policy->SetTokenLevel(USER_INTERACTIVE, USER_LOCKDOWN);
  PROCESS_INFORMATION temp_process_info = {};
  result =
      broker->SpawnTarget(prog_name, arguments.c_str(), policy, &warning_result,
                          &last_error, &temp_process_info);
  base::string16 desktop_name = policy->GetAlternateDesktop();
  policy = nullptr;

  EXPECT_EQ(SBOX_ALL_OK, result);
  if (result == SBOX_ALL_OK)
    target.Set(temp_process_info);

  EXPECT_EQ(1u, ::ResumeThread(target.thread_handle()));

  EXPECT_EQ(static_cast<DWORD>(WAIT_TIMEOUT),
            ::WaitForSingleObject(target.process_handle(), 2000));

  EXPECT_NE(::GetThreadDesktop(target.thread_id()),
            ::GetThreadDesktop(::GetCurrentThreadId()));

  HDESK desk = ::OpenDesktop(desktop_name.c_str(), 0, false, DESKTOP_ENUMERATE);
  EXPECT_TRUE(desk);
  EXPECT_TRUE(::CloseDesktop(desk));
  EXPECT_TRUE(::TerminateProcess(target.process_handle(), 0));

  ::WaitForSingleObject(target.process_handle(), INFINITE);

  // Close the desktop handle.
  temp_policy = broker->CreatePolicy();
  temp_policy->DestroyAlternateDesktop();
  temp_policy = nullptr;

  // Make sure the desktop does not exist anymore.
  desk = ::OpenDesktop(desktop_name.c_str(), 0, false, DESKTOP_ENUMERATE);
  EXPECT_FALSE(desk);
}
```

暂不用看懂所有逻辑，理解整个运营的方式即可。这个TEST是检查alternative desktop的，在设置后，理应和用户当前的desktop不一致。

> Alternative desktop也是policy三大组件之一，它比较简单，目的在于防止对用户desktop的其他window的攻击，比如全局hook，监听等等。本质上就是把target窗口隔离。

关于Token的事儿暂时就说到这儿，chrome仅仅只是根据设计（TokenLevel和IntegrityLevel）按等级权限使用了Windows的restricted token。其中还用到了impersonate机制。

想要理解这些设置是如何生效的，对象的过审检查是如何进行的，首先得了解Windows的对象安全机制。文首已经给出了非常nice的参考书，尽管《Windows Internals》因为作者职位的敏感，文中的描述都是高度抽象且没有代码参照，但凭借google在手，还是能够有一定收获的。
