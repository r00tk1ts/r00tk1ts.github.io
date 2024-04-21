---
title: Chromium-sandbox-Filesystem-analysis
date: 2018-06-03 11:36:11
categories: 源码剖析
tags:
	- chromium
	- chromium-sandbox
---

本篇是sandbox源码剖析的第十四篇，主要分析了子系统Filesystem的三大组件。

Target在执行5个文件系统相关的API时，会因为Interceptions的部署（该子系统由broker远程部署(service call)）而调用hook函数，hook函数通过SharedMem IPC与broker通信，将调用参数传给broker，broker此后通过dispatcher分发找到对应子系统的dispatcher，子系统dispatcher会匹配调用参数并调用一早便注册好的callback，callback会进行Low-level policy的Evaluate鉴权，并进一步调用low-level policy的业务处理函数，根据鉴权结果来执行API，并在CrossCallResult回执结果。

阅读本篇前，请先阅读前面所有章节。

想要流程的阅读本系列你需要以下几个条件：
1. 较高水平的C++编码能力（至少通读C++ Primer 5th，刷过课后题，有一定编码量）。
2. 熟悉Windows API编程，尤其是安全相关的内容。
3. 对二进制安全有一定了解，熟悉各类型安全漏洞成因、漏洞缓解措施及bypass手法。

<!--more-->

# chromium-sandbox-Filesystem-analysis

Low-level policy、Dispatcher、Interceptions作为三个基本组件，目前都已经分析完了。sandbox中对每个子系统使用了这三大组件。

## low-level policy

类头定义：

```cpp
// This class centralizes most of the knowledge related to file system policy
class FileSystemPolicy {
 public:
  // Creates the required low-level policy rules to evaluate a high-level
  // policy rule for File IO, in particular open or create actions.
  // 'name' is the file or directory name.
  // 'semantics' is the desired semantics for the open or create.
  // 'policy' is the policy generator to which the rules are going to be added.
  // 使用low-level policy来evaluate一个high-level FILE IO policy rule
  static bool GenerateRules(const wchar_t* name,
                            TargetPolicy::Semantics semantics,
                            LowLevelPolicy* policy);	

  // Add basic file system rules.
  // 基础的文件系统rule添加接口
  static bool SetInitialRules(LowLevelPolicy* policy);

  // Performs the desired policy action on a create request with an
  // API that is compatible with the IPC-received parameters.
  // 'client_info' : the target process that is making the request.
  // 'eval_result' : The desired policy action to accomplish.
  // 'file' : The target file or directory.
  // 通过IPC收到的create request
  static bool CreateFileAction(EvalResult eval_result,
                               const ClientInfo& client_info,
                               const base::string16& file,
                               uint32_t attributes,
                               uint32_t desired_access,
                               uint32_t file_attributes,
                               uint32_t share_access,
                               uint32_t create_disposition,
                               uint32_t create_options,
                               HANDLE* handle,
                               NTSTATUS* nt_status,
                               ULONG_PTR* io_information);

  // Performs the desired policy action on an open request with an
  // API that is compatible with the IPC-received parameters.
  // 'client_info' : the target process that is making the request.
  // 'eval_result' : The desired policy action to accomplish.
  // 'file' : The target file or directory.
  // IPC收到的open request
  static bool OpenFileAction(EvalResult eval_result,
                             const ClientInfo& client_info,
                             const base::string16& file,
                             uint32_t attributes,
                             uint32_t desired_access,
                             uint32_t share_access,
                             uint32_t open_options,
                             HANDLE* handle,
                             NTSTATUS* nt_status,
                             ULONG_PTR* io_information);

  // Performs the desired policy action on a query request with an
  // API that is compatible with the IPC-received parameters.
  // IPC收到的query request
  static bool QueryAttributesFileAction(EvalResult eval_result,
                                        const ClientInfo& client_info,
                                        const base::string16& file,
                                        uint32_t attributes,
                                        FILE_BASIC_INFORMATION* file_info,
                                        NTSTATUS* nt_status);

  // Performs the desired policy action on a query request with an
  // API that is compatible with the IPC-received parameters.
  // IPC收到的query full request
  static bool QueryFullAttributesFileAction(
      EvalResult eval_result,
      const ClientInfo& client_info,
      const base::string16& file,
      uint32_t attributes,
      FILE_NETWORK_OPEN_INFORMATION* file_info,
      NTSTATUS* nt_status);

  // Performs the desired policy action on a set_info request with an
  // API that is compatible with the IPC-received parameters.
  // IPC收到的set_info request
  static bool SetInformationFileAction(EvalResult eval_result,
                                       const ClientInfo& client_info,
                                       HANDLE target_file_handle,
                                       void* file_info,
                                       uint32_t length,
                                       uint32_t info_class,
                                       IO_STATUS_BLOCK* io_block,
                                       NTSTATUS* nt_status);
};
```

实际上就是文件系统中，几种对文件操作的broker端审判，而请求由target发起。二者通过IPC通信。

注意到所有的成员都是static函数，类的定义仅仅是为了结构上的封装。这些函数全部由外部对象调用。

### `GenerateRules`

在分析该函数前，我们先看看这个函数在哪里被调用，通过交叉引用，我们发现，在`PolicyBase::AddRuleInternal`中存在调用关系。而这个函数实际上是`PolicyBase::AddRule`的helper函数。而`AddRule`这个函数我们清楚，它是`PolicyBase`对broker提供的调用接口，用于添加一个规则。

```cpp
ResultCode PolicyBase::AddRule(SubSystem subsystem,	// 哪个子系统的规则
                               Semantics semantics,	// rule匹配时允许的语义
                               const wchar_t* pattern) {
  ResultCode result = AddRuleInternal(subsystem, semantics, pattern);
  LOG_IF(ERROR, result != SBOX_ALL_OK)
      << "Failed to add sandbox rule."
      << " error = " << result << ", subsystem = " << subsystem
      << ", semantics = " << semantics << ", pattern = '" << pattern << "'";
  return result;
}
```

对于我们本次分析的filesystem子系统来说，`SubSystem`就是`SUBSYS_FILES`，而`Semantics`有这几个值：

```cpp
FILES_ALLOW_ANY,       // Allows open or create for any kind of access that
					 // the file system supports.
FILES_ALLOW_READONLY,  // Allows open or create with read access only.
FILES_ALLOW_QUERY,     // Allows access to query the attributes of a file.
FILES_ALLOW_DIR_ANY,   // Allows open or create with directory semantics
					 // only.
```

再看helper函数：

```cpp
ResultCode PolicyBase::AddRuleInternal(SubSystem subsystem,
                                       Semantics semantics,
                                       const wchar_t* pattern) {
  // policy_是PolicyGlobal成员，我们现在知道了它是存放所有PolicyBuffer的buffer
  if (!policy_) {
    // 如果还没准备，就先make一个4096*14尺寸的PolicyGlobal
    policy_ = MakeBrokerPolicyMemory();
    DCHECK(policy_);
    // 使用该PolicyGlobal new出一个LowLevelPolicy
    policy_maker_ = new LowLevelPolicy(policy_);
    DCHECK(policy_maker_);
  }

  switch (subsystem) {
    case SUBSYS_FILES: {
      // 文件子系统会进到这里，一开始file_system_init_是false
      if (!file_system_init_) {
        // 执行完SetInitialRules后，file_system_init_置true，即SetInitialRules只执行一次
        if (!FileSystemPolicy::SetInitialRules(policy_maker_))
          return SBOX_ERROR_BAD_PARAMS;
        file_system_init_ = true;
      }
      // 根据传入的pattern,semantics，用LowLevelPolicy生成一个rule
      if (!FileSystemPolicy::GenerateRules(pattern, semantics, policy_maker_)) {
        NOTREACHED();
        return SBOX_ERROR_BAD_PARAMS;
      }
      break;
    }
    case SUBSYS_SYNC: {
      if (!SyncPolicy::GenerateRules(pattern, semantics, policy_maker_)) {
        NOTREACHED();
        return SBOX_ERROR_BAD_PARAMS;
      }
      break;
    }
    case SUBSYS_PROCESS: {
      if (lockdown_level_ < USER_INTERACTIVE &&
          TargetPolicy::PROCESS_ALL_EXEC == semantics) {
        // This is unsupported. This is a huge security risk to give full access
        // to a process handle.
        return SBOX_ERROR_UNSUPPORTED;
      }
      if (!ProcessPolicy::GenerateRules(pattern, semantics, policy_maker_)) {
        NOTREACHED();
        return SBOX_ERROR_BAD_PARAMS;
      }
      break;
    }
    case SUBSYS_NAMED_PIPES: {
      if (!NamedPipePolicy::GenerateRules(pattern, semantics, policy_maker_)) {
        NOTREACHED();
        return SBOX_ERROR_BAD_PARAMS;
      }
      break;
    }
    case SUBSYS_REGISTRY: {
      if (!RegistryPolicy::GenerateRules(pattern, semantics, policy_maker_)) {
        NOTREACHED();
        return SBOX_ERROR_BAD_PARAMS;
      }
      break;
    }
    case SUBSYS_WIN32K_LOCKDOWN: {
      if (!ProcessMitigationsWin32KLockdownPolicy::GenerateRules(
              pattern, semantics, policy_maker_)) {
        NOTREACHED();
        return SBOX_ERROR_BAD_PARAMS;
      }
      break;
    }

    default: { return SBOX_ERROR_UNSUPPORTED; }
  }

  return SBOX_ALL_OK;
}
```

不仅可以看到`GenerateRules`，还可以看到`SetInitialRules`调用的时机，我们先分析前者：

```cpp
bool FileSystemPolicy::GenerateRules(const wchar_t* name,
                                     TargetPolicy::Semantics semantics,
                                     LowLevelPolicy* policy) {
  // 对文件子系统来说，传过来的pattern也就是name实际上是文件名称
  base::string16 mod_name(name);
  if (mod_name.empty()) {
    return false;
  }

  // 处理reparse(..)，就是把当前路径'.'，上一路径'..'这些先处理一下
  // 内部调用的是ConvertToLongPath接IsReparsePoint，我们此前见过
  if (!PreProcessName(&mod_name)) {
    // The path to be added might contain a reparse point.
    NOTREACHED();
    return false;
  }

  // TODO(cpu) bug 32224: This prefix add is a hack because we don't have the
  // infrastructure to normalize names. In any case we need to escape the
  // question marks.
  // 对"\\Device\\"前缀的一个patch，具体原因得参考bug 32224，这里不是我们研究的主题，先pass
  if (_wcsnicmp(mod_name.c_str(), kNTDevicePrefix, kNTDevicePrefixLen)) {
    mod_name = FixNTPrefixForMatch(mod_name);
    name = mod_name.c_str();
  }

  // 生成的rule的action opcode是ASK_BROKER
  EvalResult result = ASK_BROKER;

  // List of supported calls for the filesystem.
  const unsigned kCallNtCreateFile = 0x1;
  const unsigned kCallNtOpenFile = 0x2;
  const unsigned kCallNtQueryAttributesFile = 0x4;
  const unsigned kCallNtQueryFullAttributesFile = 0x8;
  const unsigned kCallNtSetInfoRename = 0x10;

  DWORD rule_to_add = kCallNtOpenFile | kCallNtCreateFile |
                      kCallNtQueryAttributesFile |
                      kCallNtQueryFullAttributesFile | kCallNtSetInfoRename;

  // 新建了5个PolicyRule，分别管制5种请求，action为ASK_BROKER
  PolicyRule create(result);
  PolicyRule open(result);
  PolicyRule query(result);
  PolicyRule query_full(result);
  PolicyRule rename(result);

  //根据semantics处理分支
  switch (semantics) {
    case TargetPolicy::FILES_ALLOW_DIR_ANY: {
      // 只能打开目录，这种情况就对open和create两套规则的OpenFile::OPTIONS parameter_
      // 设置值为FILE_DIRECTORY_FILE的按位与匹配
      open.AddNumberMatch(IF, OpenFile::OPTIONS, FILE_DIRECTORY_FILE, AND);
      create.AddNumberMatch(IF, OpenFile::OPTIONS, FILE_DIRECTORY_FILE, AND);
      break;
    }
    case TargetPolicy::FILES_ALLOW_READONLY: {
      // We consider all flags that are not known to be readonly as potentially
      // used for write.
      // 只读方式打开，所以对OpenFile::ACCESS来说，allowed_flags以外的任何flag都不能设置
      // 而OpenFile::DISPOSITION则必须得是FILE_OPEN
      DWORD allowed_flags = FILE_READ_DATA | FILE_READ_ATTRIBUTES |
                            FILE_READ_EA | SYNCHRONIZE | FILE_EXECUTE |
                            GENERIC_READ | GENERIC_EXECUTE | READ_CONTROL;
      DWORD restricted_flags = ~allowed_flags;
      open.AddNumberMatch(IF_NOT, OpenFile::ACCESS, restricted_flags, AND);
      open.AddNumberMatch(IF, OpenFile::DISPOSITION, FILE_OPEN, EQUAL);
      create.AddNumberMatch(IF_NOT, OpenFile::ACCESS, restricted_flags, AND);
      create.AddNumberMatch(IF, OpenFile::DISPOSITION, FILE_OPEN, EQUAL);

      // Read only access don't work for rename.
      // 移除Rename的rule，本次不设置
      rule_to_add &= ~kCallNtSetInfoRename;
      break;
    }
    case TargetPolicy::FILES_ALLOW_QUERY: {
      // Here we don't want to add policy for the open or the create.
      // 允许对某个文件属性的query，本次open/create/rename也就无需设置了
      rule_to_add &=
          ~(kCallNtOpenFile | kCallNtCreateFile | kCallNtSetInfoRename);
      break;
    }
    case TargetPolicy::FILES_ALLOW_ANY: {
      break;
    }
    default: {
      NOTREACHED();
      return false;
    }
  }

  // 根据当前rule_to_add的状态，把OpenFile::NAME这一参数进行设置
  // PolicyRule add好以后，就通过policy->AddRule添加到LowLevelPolicy中
  // 注意AddRule时会绑定service id和PolicyRule
  // filesystem子系统占用了5个service id，分别对应open, create, rename, query, queryFull
  if ((rule_to_add & kCallNtCreateFile) &&
      (!create.AddStringMatch(IF, OpenFile::NAME, name, CASE_INSENSITIVE) ||
       !policy->AddRule(IPC_NTCREATEFILE_TAG, &create))) {
    return false;
  }

  if ((rule_to_add & kCallNtOpenFile) &&
      (!open.AddStringMatch(IF, OpenFile::NAME, name, CASE_INSENSITIVE) ||
       !policy->AddRule(IPC_NTOPENFILE_TAG, &open))) {
    return false;
  }

  if ((rule_to_add & kCallNtQueryAttributesFile) &&
      (!query.AddStringMatch(IF, FileName::NAME, name, CASE_INSENSITIVE) ||
       !policy->AddRule(IPC_NTQUERYATTRIBUTESFILE_TAG, &query))) {
    return false;
  }

  if ((rule_to_add & kCallNtQueryFullAttributesFile) &&
      (!query_full.AddStringMatch(IF, FileName::NAME, name, CASE_INSENSITIVE) ||
       !policy->AddRule(IPC_NTQUERYFULLATTRIBUTESFILE_TAG, &query_full))) {
    return false;
  }

  if ((rule_to_add & kCallNtSetInfoRename) &&
      (!rename.AddStringMatch(IF, FileName::NAME, name, CASE_INSENSITIVE) ||
       !policy->AddRule(IPC_NTSETINFO_RENAME_TAG, &rename))) {
    return false;
  }

  return true;
}
```

### `SetInitialRules`

初始化基本Rule的`SetInitialRules`在第一次`PolicyBase::AddRuleInternal`时处理filesystem类别时调用。

```cpp
// Right now we insert two rules, to be evaluated before any user supplied rule:
// - go to the broker if the path doesn't look like the paths that we push on
//    the policy (namely \??\something).
// - go to the broker if it looks like this is a short-name path.
//
// It is possible to add a rule to go to the broker in any case; it would look
// something like:
//    rule = new PolicyRule(ASK_BROKER);
//    rule->AddNumberMatch(IF_NOT, FileName::BROKER, true, AND);
//    policy->AddRule(service, rule);
bool FileSystemPolicy::SetInitialRules(LowLevelPolicy* policy) {
  // 两个ASK_BROKER的action rule
  PolicyRule format(ASK_BROKER);
  PolicyRule short_name(ASK_BROKER);

  // format按位与匹配FileName::BROKER没有BROKER_TRUE标志位
  bool rv = format.AddNumberMatch(IF_NOT, FileName::BROKER, BROKER_TRUE, AND);
  // 匹配FileName::NAME不能为L"\\/?/?\\*"
  rv &= format.AddStringMatch(IF_NOT, FileName::NAME, L"\\/?/?\\*",
                              CASE_SENSITIVE);

  // shortname按位与匹配FileName::BROKER没有BROKER_TRUE标志位
  rv &= short_name.AddNumberMatch(IF_NOT, FileName::BROKER, BROKER_TRUE, AND);
  // 匹配FileName::NAME为L"*~*"
  rv &= short_name.AddStringMatch(IF, FileName::NAME, L"*~*", CASE_SENSITIVE);

  // 为5个service id，各添加这两个rule到LowLevelPolicy
  if (!rv || !policy->AddRule(IPC_NTCREATEFILE_TAG, &format))
    return false;

  if (!policy->AddRule(IPC_NTCREATEFILE_TAG, &short_name))
    return false;

  if (!policy->AddRule(IPC_NTOPENFILE_TAG, &format))
    return false;

  if (!policy->AddRule(IPC_NTOPENFILE_TAG, &short_name))
    return false;

  if (!policy->AddRule(IPC_NTQUERYATTRIBUTESFILE_TAG, &format))
    return false;

  if (!policy->AddRule(IPC_NTQUERYATTRIBUTESFILE_TAG, &short_name))
    return false;

  if (!policy->AddRule(IPC_NTQUERYFULLATTRIBUTESFILE_TAG, &format))
    return false;

  if (!policy->AddRule(IPC_NTQUERYFULLATTRIBUTESFILE_TAG, &short_name))
    return false;

  if (!policy->AddRule(IPC_NTSETINFO_RENAME_TAG, &format))
    return false;

  if (!policy->AddRule(IPC_NTSETINFO_RENAME_TAG, &short_name))
    return false;

  return true;
}
```

实际上是对`L"\\/?/?\\*`"和`L"*~*"`的额外处理，这两个在windows下都是特殊地址标识符。

###5个请求处理函数

剩下的5个对应create, open, rename, query, queryFull五种请求的处理函数，展开看看：

```cpp
bool FileSystemPolicy::CreateFileAction(EvalResult eval_result,
                                        const ClientInfo& client_info,
                                        const base::string16& file,
                                        uint32_t attributes,
                                        uint32_t desired_access,
                                        uint32_t file_attributes,
                                        uint32_t share_access,
                                        uint32_t create_disposition,
                                        uint32_t create_options,
                                        HANDLE* handle,
                                        NTSTATUS* nt_status,
                                        ULONG_PTR* io_information) {
  *handle = nullptr;
  // The only action supported is ASK_BROKER which means create the requested
  // file as specified.
  // 对open来说，eval_result只可能是ASK_BROKER
  if (ASK_BROKER != eval_result) {
    *nt_status = STATUS_ACCESS_DENIED;
    return false;
  }
  IO_STATUS_BLOCK io_block = {};
  UNICODE_STRING uni_name = {};
  OBJECT_ATTRIBUTES obj_attributes = {};
  SECURITY_QUALITY_OF_SERVICE security_qos = GetAnonymousQOS();

  // 填充obj_attributes
  InitObjectAttribs(file, attributes, nullptr, &obj_attributes, &uni_name,
                    IsPipe(file) ? &security_qos : nullptr);
  *nt_status =
      NtCreateFileInTarget(handle, desired_access, &obj_attributes, &io_block,
                           file_attributes, share_access, create_disposition,
                           create_options, nullptr, 0, client_info.process);

  *io_information = io_block.Information;
  return true;
}
```

展开NtCreateFileInTarget看看吧：

```cpp
NTSTATUS NtCreateFileInTarget(HANDLE* target_file_handle,
                              ACCESS_MASK desired_access,
                              OBJECT_ATTRIBUTES* obj_attributes,
                              IO_STATUS_BLOCK* io_status_block,
                              ULONG file_attributes,
                              ULONG share_access,
                              ULONG create_disposition,
                              ULONG create_options,
                              PVOID ea_buffer,
                              ULONG ea_length,
                              HANDLE target_process) {
  // 获取到NtCreateFile函数地址
  NtCreateFileFunction NtCreateFile = nullptr;
  ResolveNTFunctionPtr("NtCreateFile", &NtCreateFile);// 实际上是GetProcAddress

  // 调用NtCreateFile
  HANDLE local_handle = INVALID_HANDLE_VALUE;
  NTSTATUS status =
      NtCreateFile(&local_handle, desired_access, obj_attributes,
                   io_status_block, nullptr, file_attributes, share_access,
                   create_disposition, create_options, ea_buffer, ea_length);
  if (!NT_SUCCESS(status)) {
    return status;
  }

  if (!sandbox::SameObject(local_handle, obj_attributes->ObjectName->Buffer)) {
    // The handle points somewhere else. Fail the operation.
    ::CloseHandle(local_handle);
    return STATUS_ACCESS_DENIED;
  }

  // 拷贝handle给target_process的target_file_handle
  // 这个target_file_handle的值在返回到FilesystemDispatcher::NtCreateFile时
  // 会拷贝到ipc的CrossCallReturn中，传递给target进程，我们下面分析dispatcher时会看到
  if (!::DuplicateHandle(::GetCurrentProcess(), local_handle, target_process,
                         target_file_handle, 0, false,
                         DUPLICATE_CLOSE_SOURCE | DUPLICATE_SAME_ACCESS)) {
    return STATUS_ACCESS_DENIED;
  }
  return STATUS_SUCCESS;
}
```

实际上create的操作是在broker端得到了执行，而将返回的句柄复制给了target进程传过来的target_file_handle。

其他的action函数也类似，都是对API的一层封装，它们并没有做检查处理，因为policy rule对请求的检查要优先于此（实际上是`EvalPolicy`操作）。这些请求处理函数的调用位置在`FilesystemDispatcher`中。

其他的就不一一枚举了。

**小结**

**FileSystem的low-level policy部分做了两件事：**

1. **为`PolicyBase::AddRule`提供了该子系统5个service或者叫action的Rule制定接口。**
2. **编写了broker端5个请求的处理，在broker端调用了对应的API函数。**

## Dispatcher

Dispatcher处理的是target发起的IPC请求。我们此前分析`Dispatcher`类时就发现，它是个基类，而对应每一个子系统，都有一个具体的派生类。对于filesystem来说，这个派生类就是`FilesystemDispatcher`。

```cpp
// This class handles file system-related IPC calls.
class FilesystemDispatcher : public Dispatcher {
 public:
  // 构造器关联PolicyBase对象
  explicit FilesystemDispatcher(PolicyBase* policy_base);
  ~FilesystemDispatcher() override {}

  // Dispatcher interface.
  // 与InterceptionManager对接之处
  bool SetupService(InterceptionManager* manager, int service) override;

 private:
  // target的各种IPC请求，target在call这5个API时会发起IPC请求
  // Processes IPC requests coming from calls to NtCreateFile in the target.
  bool NtCreateFile(IPCInfo* ipc,
                    base::string16* name,
                    uint32_t attributes,
                    uint32_t desired_access,
                    uint32_t file_attributes,
                    uint32_t share_access,
                    uint32_t create_disposition,
                    uint32_t create_options);

  // Processes IPC requests coming from calls to NtOpenFile in the target.
  bool NtOpenFile(IPCInfo* ipc,
                  base::string16* name,
                  uint32_t attributes,
                  uint32_t desired_access,
                  uint32_t share_access,
                  uint32_t create_options);

  // Processes IPC requests coming from calls to NtQueryAttributesFile in the
  // target.
  bool NtQueryAttributesFile(IPCInfo* ipc,
                             base::string16* name,
                             uint32_t attributes,
                             CountedBuffer* info);

  // Processes IPC requests coming from calls to NtQueryFullAttributesFile in
  // the target.
  bool NtQueryFullAttributesFile(IPCInfo* ipc,
                                 base::string16* name,
                                 uint32_t attributes,
                                 CountedBuffer* info);

  // Processes IPC requests coming from calls to NtSetInformationFile with the
  // rename information class.
  bool NtSetInformationFile(IPCInfo* ipc,
                            HANDLE handle,
                            CountedBuffer* status,
                            CountedBuffer* info,
                            uint32_t length,
                            uint32_t info_class);

  PolicyBase* policy_base_;
  DISALLOW_COPY_AND_ASSIGN(FilesystemDispatcher);
};
```

###构造器

```cpp
FilesystemDispatcher::FilesystemDispatcher(PolicyBase* policy_base)
    : policy_base_(policy_base) {
  // 把相关的几个IPCCall全部实体化，这里完成各个service id与callback的绑定
  // 绑定的函数就是类内部定义的几个接口处理函数
  static const IPCCall create_params = {
      {IPC_NTCREATEFILE_TAG,
       {WCHAR_TYPE, UINT32_TYPE, UINT32_TYPE, UINT32_TYPE, UINT32_TYPE,
        UINT32_TYPE, UINT32_TYPE}},
      reinterpret_cast<CallbackGeneric>(&FilesystemDispatcher::NtCreateFile)};

  static const IPCCall open_file = {
      {IPC_NTOPENFILE_TAG,
       {WCHAR_TYPE, UINT32_TYPE, UINT32_TYPE, UINT32_TYPE, UINT32_TYPE}},
      reinterpret_cast<CallbackGeneric>(&FilesystemDispatcher::NtOpenFile)};

  static const IPCCall attribs = {
      {IPC_NTQUERYATTRIBUTESFILE_TAG, {WCHAR_TYPE, UINT32_TYPE, INOUTPTR_TYPE}},
      reinterpret_cast<CallbackGeneric>(
          &FilesystemDispatcher::NtQueryAttributesFile)};

  static const IPCCall full_attribs = {
      {IPC_NTQUERYFULLATTRIBUTESFILE_TAG,
       {WCHAR_TYPE, UINT32_TYPE, INOUTPTR_TYPE}},
      reinterpret_cast<CallbackGeneric>(
          &FilesystemDispatcher::NtQueryFullAttributesFile)};

  static const IPCCall set_info = {
      {IPC_NTSETINFO_RENAME_TAG,
       {VOIDPTR_TYPE, INOUTPTR_TYPE, INOUTPTR_TYPE, UINT32_TYPE, UINT32_TYPE}},
      reinterpret_cast<CallbackGeneric>(
          &FilesystemDispatcher::NtSetInformationFile)};

  // 全部置入ipc_calls_中维护
  ipc_calls_.push_back(create_params);
  ipc_calls_.push_back(open_file);
  ipc_calls_.push_back(attribs);
  ipc_calls_.push_back(full_attribs);
  ipc_calls_.push_back(set_info);
}
```

而SetupService则是该对象的灵魂了：

```cpp
bool FilesystemDispatcher::SetupService(InterceptionManager* manager,
                                        int service) {
  // 根据service id，找到是哪个IPCCall
  switch (service) {
    case IPC_NTCREATEFILE_TAG:
      return INTERCEPT_NT(manager, NtCreateFile, CREATE_FILE_ID, 48);

    case IPC_NTOPENFILE_TAG:
      return INTERCEPT_NT(manager, NtOpenFile, OPEN_FILE_ID, 28);

    case IPC_NTQUERYATTRIBUTESFILE_TAG:
      return INTERCEPT_NT(manager, NtQueryAttributesFile, QUERY_ATTRIB_FILE_ID,
                          12);

    case IPC_NTQUERYFULLATTRIBUTESFILE_TAG:
      return INTERCEPT_NT(manager, NtQueryFullAttributesFile,
                          QUERY_FULL_ATTRIB_FILE_ID, 12);

    case IPC_NTSETINFO_RENAME_TAG:
      return INTERCEPT_NT(manager, NtSetInformationFile, SET_INFO_FILE_ID, 24);

    default:
      return false;
  }
}
```

将宏展开：

```cpp
#define INTERCEPT_NT(manager, service, id, num_params) \
  manager->ADD_NT_INTERCEPTION(service, id, num_params)

#define ADD_NT_INTERCEPTION(service, id, num_params)            \
  AddToPatchedFunctions(                                        \
      kNtdllName, #service, sandbox::INTERCEPTION_SERVICE_CALL, \
      reinterpret_cast<void*>(MAKE_SERVICE_NAME(service)), id)

#if defined(_WIN64)
#define MAKE_SERVICE_NAME(service) &Target##service##64
#else
#define MAKE_SERVICE_NAME(service) &Target##service
#endif
```

以`NtCreateFile`为例，实际上最终使用`AddToPatchedFunctions`将`NtCreateFile`替换成了`TargetNtCreateFile`（32位），`SetupService`这一操作，完成了这5个系统调用的Interception的部署。

另外，系统调用类型的函数是在broker端处理的。

那么Target开头的函数从哪儿来，总不能凭空捏造吧，其实他们在filesystem的Interceptions中定义。

> 我只找到了32位的几个Target函数，64位的没有找到，貌似对这几个service call的Interceptions并没有在x64上实现，尽管service call的Interceptions提供了这种机制。

还记得此前resolver的处理流程吗，`TopLevelDispatcher`中有一个`Dispatcher*`数组，每个IPC请求对应一个成员，其中有5个和filesystem子系统相关的成员指向了同一个构造器new出来的`FilesystemDispatcher`。

```cpp
Dispatcher* dispatcher;

dispatcher = new FilesystemDispatcher(policy_);
ipc_targets_[IPC_NTCREATEFILE_TAG] = dispatcher;
ipc_targets_[IPC_NTOPENFILE_TAG] = dispatcher;
ipc_targets_[IPC_NTSETINFO_RENAME_TAG] = dispatcher;
ipc_targets_[IPC_NTQUERYATTRIBUTESFILE_TAG] = dispatcher;
ipc_targets_[IPC_NTQUERYFULLATTRIBUTESFILE_TAG] = dispatcher;
// filesystem_dispatcher是TopLevelDispatcher的成员，从此就有了FilesystemDispatcher对象
filesystem_dispatcher_.reset(dispatcher);
```

而broker作为IPC的server端，处理IPC请求的过程是这样的：

1. `SharedMemIPCServer::Init`中会用`ThreadProvider`注册一个等待ping事件的`ThreadPingEventReady`，注册时使用的参数的`dispatcher`绑定了构造器中关联的`call_dispatcher_`。
2. client发起IPC请求时，会signaled ping，此后broker进入`ThreadPingEventReady`
3. `ThreadPingEventReady`中会调用`InvokeCallback`匹配callback
4. `InvokeCallback`内部调用dispatcher的`OnMessageReady`。遍历该dispatcher中的`IPCParam`来找到绑定的callback。

到此，我们就把filesystem的整个过程联系了起来，对于`IPC_NTCREATEFILE_TAG`等5种IPC请求，最终会找到`filesystem_dispatcher`这个具体的dispatcher，而它早在构造器中将`IPCParam`和5个callback函数绑定了。

所以下一步，就会调用某个callback，比如`NtCreateFile`。我们就展开看看这个callback `NtCreateFile`干了什么：

```cpp
bool FilesystemDispatcher::NtCreateFile(IPCInfo* ipc,
                                        base::string16* name,
                                        uint32_t attributes,
                                        uint32_t desired_access,
                                        uint32_t file_attributes,
                                        uint32_t share_access,
                                        uint32_t create_disposition,
                                        uint32_t create_options) {
  if (!PreProcessName(name)) {
    // The path requested might contain a reparse point.
    ipc->return_info.nt_status = STATUS_ACCESS_DENIED;
    return true;
  }

  const wchar_t* filename = name->c_str();

  // 把target传过来的参数值全部置入到CountedParameterSet中
  uint32_t broker = BROKER_TRUE;
  CountedParameterSet<OpenFile> params;
  params[OpenFile::NAME] = ParamPickerMake(filename);
  params[OpenFile::ACCESS] = ParamPickerMake(desired_access);
  params[OpenFile::DISPOSITION] = ParamPickerMake(create_disposition);
  params[OpenFile::OPTIONS] = ParamPickerMake(create_options);
  params[OpenFile::BROKER] = ParamPickerMake(broker);

  // To evaluate the policy we need to call back to the policy object. We
  // are just middlemen in the operation since is the FileSystemPolicy which
  // knows what to do.
  // EvalPolicy检查各种rules是否通过，这些rules此前已通过PolicyBase::AddRules设置好了
  // 然后去找Policy模块的CreateFileAction来干实事，也就是前面分析的
  // FileSystemPolicy::CreateFileAction
  EvalResult result =
      policy_base_->EvalPolicy(IPC_NTCREATEFILE_TAG, params.GetBase());
  HANDLE handle;
  ULONG_PTR io_information = 0;
  NTSTATUS nt_status;
  // 在这内部，会判断result是否是ASK_BROKER，如果是说明匹配到了rule，那就执行API
  // 如果不是说明凉凉
  if (!FileSystemPolicy::CreateFileAction(
          result, *ipc->client_info, *name, attributes, desired_access,
          file_attributes, share_access, create_disposition, create_options,
          &handle, &nt_status, &io_information)) {
    ipc->return_info.nt_status = STATUS_ACCESS_DENIED;
    return true;
  }
  // 把返回的结果给IPC调用的CrossCallReturn结构，用于指示target client端
  // Return operation status on the IPC.
  ipc->return_info.extended[0].ulong_ptr = io_information;
  ipc->return_info.nt_status = nt_status;
  ipc->return_info.handle = handle;
  return true;
}
```

到此，filesystem相关的5个IPC请求在broker端处理的整个链条就清楚了。

```cpp
EvalResult PolicyBase::EvalPolicy(int service,
                                  CountedParameterSetBase* params) {
  if (policy_) {
    // 找到PolicyGlobal中对应的entry[service]一项
    if (!policy_->entry[service]) {
      // There is no policy for this particular service. This is not a big
      // deal.
      return DENY_ACCESS;
    }
    for (size_t i = 0; i < params->count; i++) {
      if (!params->parameters[i].IsValid()) {
        NOTREACHED();
        return SIGNAL_ALARM;
      }
    }
    // 基于policy_->entry[service]这个PolicyBuffer建一个PolicyProcessor
    PolicyProcessor pol_evaluator(policy_->entry[service]);
    // 逐group进行evaluate
    PolicyResult result =
        pol_evaluator.Evaluate(kShortEval, params->parameters, params->count);
    // 如果匹配到了，就获取这一group的action
    if (POLICY_MATCH == result) {
      return pol_evaluator.GetAction();
    }
    DCHECK(POLICY_ERROR != result);
  }

  return DENY_ACCESS;
}
```

**小结**

**filesystem的`FilesystemDispatcher`负责三件事：**

1. **受`TopLevelDispatcher`的驱使，处理5个filesystem相关的IPC请求**
2. **绑定5个callback函数，每个函数内部执行`EvalPolicy`审判，然后将参数和结果通通丢给policy的5个处理函数。**
3. **负责5个ntdll.dll中对应的系统调用的拦截器安装。**

## Interception

那么这个拦截函数就有点意思了，实际上这5个API函数在target端都不能直接调用，target想要调用`CreateFile`时，要先起一个对应id的IPC请求给broker，broker通过前面的重重处理，最终在`FileSystemDispatcher::NtCreateFile`中执行`EvalPolicy`审判，然后调用`FileSystemPolicy::CreateFileAction`。

而在具体的action处理内部，审判结果需要是`ASK_BROKER`才会调用`NtCreateFile`这个系统调用。

而对target端，broker已经通过`FilesystemDispatcher::SetupService`为target部署好了5个系统调用的拦截器，当target进程调用`NtCreateFile`时，实际上调用到的是`TargetNtCreateFile`函数：

```cpp
NTSTATUS WINAPI TargetNtCreateFile(NtCreateFileFunction orig_CreateFile,
                                   PHANDLE file,
                                   ACCESS_MASK desired_access,
                                   POBJECT_ATTRIBUTES object_attributes,
                                   PIO_STATUS_BLOCK io_status,
                                   PLARGE_INTEGER allocation_size,
                                   ULONG file_attributes,
                                   ULONG sharing,
                                   ULONG disposition,
                                   ULONG options,
                                   PVOID ea_buffer,
                                   ULONG ea_length) {
  // Check if the process can open it first.
  // x86的第一个参数是original function地址，先试试能不能打开这个文件，如果不是STATUS_ACCESS_DENIED
  // 就直接调用返回status就行了，无需请求IPC
  NTSTATUS status = orig_CreateFile(
      file, desired_access, object_attributes, io_status, allocation_size,
      file_attributes, sharing, disposition, options, ea_buffer, ea_length);
  if (STATUS_ACCESS_DENIED != status)
    return status;

  // 确保IPC已经可用
  // We don't trust that the IPC can work this early.
  if (!SandboxFactory::GetTargetServices()->GetState()->InitCalled())
    return status;

  wchar_t* name = nullptr;
  do {
    if (!ValidParameter(file, sizeof(HANDLE), WRITE))
      break;
    if (!ValidParameter(io_status, sizeof(IO_STATUS_BLOCK), WRITE))
      break;

    // 获取SharedMemory IPC的全局内存空间指针g_shared_IPC_memory
    void* memory = GetGlobalIPCMemory();
    if (!memory)
      break;

    // 设置文件属性
    uint32_t attributes = 0;
    NTSTATUS ret =
        AllocAndCopyName(object_attributes, &name, &attributes, nullptr);
    if (!NT_SUCCESS(ret) || !name)
      break;

    // 设置其他参数
    uint32_t desired_access_uint32 = desired_access;
    uint32_t options_uint32 = options;
    uint32_t disposition_uint32 = disposition;
    uint32_t broker = BROKER_FALSE;

    // 做出一个CountedParameterSet<OpenFile>，按索引调用ParamPickerMake填充参数
    CountedParameterSet<OpenFile> params;
    params[OpenFile::NAME] = ParamPickerMake(name);
    params[OpenFile::ACCESS] = ParamPickerMake(desired_access_uint32);
    params[OpenFile::DISPOSITION] = ParamPickerMake(disposition_uint32);
    params[OpenFile::OPTIONS] = ParamPickerMake(options_uint32);
    params[OpenFile::BROKER] = ParamPickerMake(broker);

    // 内部会获取承载PolicyBuffer的g_shared_policy_memory
    // 找到PolicyBuffer并架设一个PolicyProcessor来对CountedParameterSet<OpenFile>进行Evaluate
    // 如果匹配rule且action为ASK_BROKER，才继续发起IPC请求
    if (!QueryBroker(IPC_NTCREATEFILE_TAG, params.GetBase()))
      break;

    SharedMemIPCClient ipc(memory);
    CrossCallReturn answer = {0};
    // The following call must match in the parameters with
    // FilesystemDispatcher::ProcessNtCreateFile.
    // 发起IPC请求，填充这些参数
    ResultCode code = CrossCall(ipc, IPC_NTCREATEFILE_TAG, name, attributes,
                                desired_access_uint32, file_attributes, sharing,
                                disposition, options_uint32, &answer);
    if (SBOX_ALL_OK != code)
      break;

    status = answer.nt_status;

    if (!NT_SUCCESS(answer.nt_status))
      break;

    __try {
      *file = answer.handle;
      io_status->Status = answer.nt_status;
      io_status->Information = answer.extended[0].ulong_ptr;
    } __except (EXCEPTION_EXECUTE_HANDLER) {
      break;
    }
  } while (false);

  if (name)
    operator delete(name, NT_ALLOC);

  return status;
}

bool QueryBroker(int ipc_id, CountedParameterSetBase* params) {
  DCHECK_NT(static_cast<size_t>(ipc_id) < kMaxServiceCount);
  DCHECK_NT(g_shared_policy_memory);
  DCHECK_NT(g_shared_policy_size > 0);

  if (static_cast<size_t>(ipc_id) >= kMaxServiceCount)
    return false;

  // g_shared_policy_memory我们知道是怎么设置共享的
  // 但是broker的PolicyGlobal是内部在堆上new的，和这里的PolicyGlobal应该不是一片内存空间
  // 那么应该有某种同步的机制
  PolicyGlobal* global_policy =
      reinterpret_cast<PolicyGlobal*>(g_shared_policy_memory);

  if (!global_policy->entry[ipc_id])
    return false;

  PolicyBuffer* policy = reinterpret_cast<PolicyBuffer*>(
      reinterpret_cast<char*>(g_shared_policy_memory) +
      reinterpret_cast<size_t>(global_policy->entry[ipc_id]));

  if ((reinterpret_cast<size_t>(global_policy->entry[ipc_id]) >
       global_policy->data_size) ||
      (g_shared_policy_size < global_policy->data_size)) {
    NOTREACHED_NT();
    return false;
  }

  for (size_t i = 0; i < params->count; i++) {
    if (!params->parameters[i].IsValid()) {
      NOTREACHED_NT();
      return false;
    }
  }

  PolicyProcessor processor(policy);
  PolicyResult result =
      processor.Evaluate(kShortEval, params->parameters, params->count);
  DCHECK_NT(POLICY_ERROR != result);

  return POLICY_MATCH == result && ASK_BROKER == processor.GetAction();
}
```

搜索`g_shared_policy_memory`相关，在`CopyPolicyToTarget`中发现端倪：

```cpp
void CopyPolicyToTarget(const void* source, size_t size, void* dest) {
  if (!source || !size)
    return;
  memcpy(dest, source, size);
  sandbox::PolicyGlobal* policy =
      reinterpret_cast<sandbox::PolicyGlobal*>(dest);

  size_t offset = reinterpret_cast<size_t>(source);

  for (size_t i = 0; i < sandbox::kMaxServiceCount; i++) {
    // PolicyBuffer要修正offset
    size_t buffer = reinterpret_cast<size_t>(policy->entry[i]);
    if (buffer) {
      buffer -= offset;
      policy->entry[i] = reinterpret_cast<sandbox::PolicyBuffer*>(buffer);
    }
  }
}
```

该函数由`IPC_Leak`调用:

```cpp
SBOX_TESTS_COMMAND int IPC_Leak(int argc, wchar_t** argv) {
  if (argc != 1)
    return SBOX_TEST_FAILED;

  // Replace current target policy with one that forwards all interceptions to
  // broker.
  PolicyGlobal* policy = GenerateBlankPolicy();
  PolicyGlobal* current_policy =
      (PolicyGlobal*)sandbox::GetGlobalPolicyMemory();
  CopyPolicyToTarget(policy, policy->data_size + sizeof(PolicyGlobal),
                     current_policy);
  ...
```

回到target发起IPC请求的地方，这里可以看到target自身也进行了一次对参数的审判。如果不过审干脆不回发起IPC调用。而broker端的审判是必须的，因为它不能依赖target的自审，无论target是否自审，broker端都需要进行一次审判，这也是一种合乎安全性的设计。

