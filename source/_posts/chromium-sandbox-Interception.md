---
title: Chromium-sandbox-Interception-analysis
date: 2018-05-26 10:25:11
categories: 源码剖析
tags:
	- chromium
	- chromium-sandbox
---

本篇是sandbox源码剖析的第十篇，主要分析了windows平台下，Chromium sandbox中子系统三大组件构成中的第二大组件——Interception。阅读本篇前，请先阅读前四篇。

想要流程的阅读本系列你需要以下几个条件：
1. 较高水平的C++编码能力（至少通读C++ Primer 5th，刷过课后题，有一定编码量）。
2. 熟悉Windows API编程，尤其是安全相关的内容。
3. 对二进制安全有一定了解，熟悉各类型安全漏洞成因、漏洞缓解措施及bypass手法。

<!--more-->

# chromium-sandbox-Interception-analysis

此前已经看过了子系统的第一大组件`Dispatcher`，`Dispatcher`把IPC请求联系了起来，broker使用`TopLevelDispatcher`来分发给子系统具体的`xxxDispatcher`，然后进行server端处理的那些事儿。

今天来分析子系统三大组件中第二个——`Interception`。我们已经不止一次的在前面的分析之旅中见过`InterceptionManager`这个用来管理`Interception`的类了。

## `InterceptionManager`

`InterceptionManager`类负责沙盒进程（target）各种拦截器的安装。

看一下类头定义：

```cpp
// The InterceptionManager executes on the parent application, and it is in
// charge of setting up the desired interceptions, and placing the Interception
// Agent into the child application.
// 这是个泛用说法，对应chrome，parent就是broker，child主要是renderer(target)。
// broker上运行的InterceptionManager实例负责安装Interception Agent到renderer上
//
// The exposed API consists of two methods: AddToPatchedFunctions to set up a
// particular interception, and InitializeInterceptions to actually go ahead and
// perform all interceptions and transfer data to the child application.
//
// The typical usage is something like this:
//
// InterceptionManager interception_manager(child);
// if (!interception_manager.AddToPatchedFunctions(
//         L"ntdll.dll", "NtCreateFile",
//         sandbox::INTERCEPTION_SERVICE_CALL, &MyNtCreateFile, MY_ID_1))
//   return false;
//
// if (!interception_manager.AddToPatchedFunctions(
//         L"kernel32.dll", "CreateDirectoryW",
//         sandbox::INTERCEPTION_EAT, L"MyCreateDirectoryW@12", MY_ID_2))
//   return false;
//
// sandbox::ResultCode rc = interception_manager.InitializeInterceptions();
// if (rc != sandbox::SBOX_ALL_OK) {
//   DWORD error = ::GetLastError();
//   return rc;
// }
// Add->Add->Add->...->Init一波流
//
// Any required syncronization must be performed outside this class. Also, it is
// not possible to perform further interceptions after InitializeInterceptions
// is called.
// 一旦InitializeInterceptions后，就不能再添加interceptions了
class InterceptionManager {
  // The unit test will access private members.
  // Allow tests to be marked DISABLED_. Note that FLAKY_ and FAILS_ prefixes
  // do not work with sandbox tests.
  // 单元测试的批量友元类，为了访问private，常见套路，这里不关心
  FRIEND_TEST_ALL_PREFIXES(InterceptionManagerTest, BufferLayout1);
  FRIEND_TEST_ALL_PREFIXES(InterceptionManagerTest, BufferLayout2);

 public:
  // An interception manager performs interceptions on a given child process.
  // If we are allowed to intercept functions that have been patched by somebody
  // else, relaxed should be set to true.
  // Note: We increase the child's reference count internally.
  // 构造器关联了TargetProcess对象
  // 如果relaxed置true则表示可以覆盖别人的patch
  InterceptionManager(TargetProcess* child_process, bool relaxed);
  ~InterceptionManager();

  // Patches function_name inside dll_name to point to replacement_code_address.
  // function_name has to be an exported symbol of dll_name.
  // Returns true on success.
  //
  // The new function should match the prototype and calling convention of the
  // function to intercept except for one extra argument (the first one) that
  // contains a pointer to the original function, to simplify the development
  // of interceptors (for IA32). In x64, there is no extra argument to the
  // interceptor, so the provided InterceptorId is used to keep a table of
  // intercepted functions so that the interceptor can index that table to get
  // the pointer that would have been the first argument (g_originals[id]).
  //
  // For example, to intercept NtClose, the following code could be used:
  // x86和x64对hook函数的不同处理
  // 下面给了一个对NtCloseFunction进行hook的方式，增加了一个原始函数指针的参数
  // typedef NTSTATUS (WINAPI *NtCloseFunction) (IN HANDLE Handle);
  // NTSTATUS WINAPI MyNtCose(IN NtCloseFunction OriginalClose,
  //                          IN HANDLE Handle) {
  //   // do something
  //   // call the original function
  //   return OriginalClose(Handle);
  // }
  //
  // And in x64:
  // x64没有传参，而是使用了g_originals全局函数指针表
  // typedef NTSTATUS (WINAPI *NtCloseFunction) (IN HANDLE Handle);
  // NTSTATUS WINAPI MyNtCose64(IN HANDLE Handle) {
  //   // do something
  //   // call the original function
  //   NtCloseFunction OriginalClose = g_originals[NT_CLOSE_ID];
  //   return OriginalClose(Handle);
  // }
  // add方法的几种重载
  // 这个是把function_name的函数指针替换成replacement_code_address起始的地址
  bool AddToPatchedFunctions(const wchar_t* dll_name,
                             const char* function_name,
                             InterceptionType interception_type,
                             const void* replacement_code_address,
                             InterceptorId id);

  // Patches function_name inside dll_name to point to
  // replacement_function_name.
  // 用replacement_function_name来替换function_name
  bool AddToPatchedFunctions(const wchar_t* dll_name,
                             const char* function_name,
                             InterceptionType interception_type,
                             const char* replacement_function_name,
                             InterceptorId id);

  // The interception agent will unload the dll with dll_name.
  // unload dll
  bool AddToUnloadModules(const wchar_t* dll_name);

  // Initializes all interceptions on the client.
  // Returns SBOX_ALL_OK on success, or an appropriate error code.
  //
  // The child process must be created suspended, and cannot be resumed until
  // after this method returns. In addition, no action should be performed on
  // the child that may cause it to resume momentarily, such as injecting
  // threads or APCs.
  //
  // This function must be called only once, after all interceptions have been
  // set up using AddToPatchedFunctions.
  // 这个就是最后的fire
  ResultCode InitializeInterceptions();

 private:
  // Used to store the interception information until the actual set-up.
  struct InterceptionData {
    InterceptionData();
    InterceptionData(const InterceptionData& other);
    ~InterceptionData();

    //这两个都是枚举量，id主要是用于划分给不同Dispatcher，type是某一种拦截类型
    //这个结构体把所有有用的信息都封装好了
    InterceptionType type;            // Interception type.
    InterceptorId id;                 // Interceptor id.
    base::string16 dll;               // Name of dll to intercept.
    std::string function;             // Name of function to intercept.
    std::string interceptor;          // Name of interceptor function.
    const void* interceptor_address;  // Interceptor's entry point.
  };

  // 计算config buffer的尺寸，InterceptionManager也需要一个buffer，这个buffer用于
  // 承载安装的Interception，然后发给InterceptionAgent
  // Calculates the size of the required configuration buffer.
  size_t GetBufferSize() const;

  // Rounds up the size of a given buffer, considering alignment (padding).
  // value is the current size of the buffer, and alignment is specified in
  // bytes.
  // 向上对齐
  static inline size_t RoundUpToMultiple(size_t value, size_t alignment) {
    return ((value + alignment - 1) / alignment) * alignment;
  }

  // Sets up a given buffer with all the information that has to be transfered
  // to the child.
  // Returns true on success.
  //
  // The buffer size should be at least the value returned by GetBufferSize
  // 部署config buffer
  bool SetupConfigBuffer(void* buffer, size_t buffer_bytes);

  // Fills up the part of the transfer buffer that corresponds to information
  // about one dll to patch.
  // data is the first recorded interception for this dll.
  // Returns true on success.
  //
  // On successful return, buffer will be advanced from it's current position
  // to the point where the next block of configuration data should be written
  // (the actual interception info), and the current size of the buffer will
  // decrease to account the space used by this method.
  // 部署某个被patch的dll的信息到buffer中，这里可以看出buffer中实际上是InterceptionData
  bool SetupDllInfo(const InterceptionData& data,
                    void** buffer,
                    size_t* buffer_bytes) const;

  // Fills up the part of the transfer buffer that corresponds to a single
  // function to patch.
  // dll_info points to the dll being updated with the interception stored on
  // data. The buffer pointer and remaining size are updated by this call.
  // Returns true on success.
  // 部署某个单一函数patch的信息到buffer
  bool SetupInterceptionInfo(const InterceptionData& data,
                             void** buffer,
                             size_t* buffer_bytes,
                             DllPatchInfo* dll_info) const;

  // Returns true if this interception is to be performed by the child
  // as opposed to from the parent.
  bool IsInterceptionPerformedByChild(const InterceptionData& data) const;

  // Allocates a buffer on the child's address space (returned on
  // remote_buffer), and fills it with the contents of a local buffer.
  // Returns SBOX_ALL_OK on success.
  // 这个应该就是跨进程写数据的素质3连
  ResultCode CopyDataToChild(const void* local_buffer,
                             size_t buffer_bytes,
                             void** remote_buffer) const;

  // Performs the cold patch (from the parent) of ntdll.
  // Returns SBOX_ALL_OK on success.
  //
  // This method will insert additional interceptions to launch the interceptor
  // agent on the child process, if there are additional interceptions to do.
  ResultCode PatchNtdll(bool hot_patch_needed);

  // Peforms the actual interceptions on ntdll.
  // thunks is the memory to store all the thunks for this dll (on the child),
  // and dll_data is a local buffer to hold global dll interception info.
  // Returns SBOX_ALL_OK on success.
  ResultCode PatchClientFunctions(DllInterceptionData* thunks,
                                  size_t thunk_bytes,
                                  DllInterceptionData* dll_data);

  // The process to intercept.
  // 看起来一个Manager管理一个target进程
  TargetProcess* child_;
  // Holds all interception info until the call to initialize (perform the
  // actual patch).
  // target进程所有待安装的拦截器
  std::list<InterceptionData> interceptions_;

  // Keep track of patches added by name.
  bool names_used_;

  // true if we are allowed to patch already-patched functions.
  bool relaxed_;

  DISALLOW_COPY_AND_ASSIGN(InterceptionManager);
};
```

 ### 构造/析构器

```cpp
InterceptionManager::InterceptionManager(TargetProcess* child_process,
                                         bool relaxed)
    : child_(child_process), names_used_(false), relaxed_(relaxed) {
  child_->AddRef();	// 引用计数增加，因为InterceptionManager用到了TargetProcess对象
}

InterceptionManager::~InterceptionManager() {
  child_->Release();	// 引用计数减少
}
```

### `AddToPatchedFunctions`

构造和析构毫无营养，我们来看几个重要的接口，先看看重载的两个`AddToPatchedFunctions`:

```cpp
bool InterceptionManager::AddToPatchedFunctions(
    const wchar_t* dll_name,
    const char* function_name,
    InterceptionType interception_type,
    const void* replacement_code_address,
    InterceptorId id) {
  // 只是单纯的填充InterceptionData，注意interceptions_中存的不是指针，是InterceptionData对象
  InterceptionData function;
  function.type = interception_type;
  function.id = id;
  function.dll = dll_name;
  function.function = function_name;
  function.interceptor_address = replacement_code_address;
	
  // 唯独没有填充interceptor，即拦截函数的名称，实际上也用不到
  interceptions_.push_back(function);	//填充然后push back
  return true;
}

bool InterceptionManager::AddToPatchedFunctions(
    const wchar_t* dll_name,
    const char* function_name,
    InterceptionType interception_type,
    const char* replacement_function_name,
    InterceptorId id) {
  InterceptionData function;
  function.type = interception_type;
  function.id = id;
  function.dll = dll_name;
  function.function = function_name;
  // 这里赋值了拦截函数名称，而interceptor_address置为了nullptr
  function.interceptor = replacement_function_name;
  function.interceptor_address = nullptr;

  interceptions_.push_back(function);
  names_used_ = true;
  return true;
}
// unload实际上也是一种interception，它的type为固定的INTERCEPTION_UNLOAD_MODULE
bool InterceptionManager::AddToUnloadModules(const wchar_t* dll_name) {
  InterceptionData module_to_unload;
  module_to_unload.type = INTERCEPTION_UNLOAD_MODULE;
  module_to_unload.dll = dll_name;
  // The next two are dummy values that make the structures regular, instead
  // of having special cases. They should not be used.
  // 这两个成员对于该对象来说没有实际意义，填充dummy数据
  module_to_unload.function = kUnloadDLLDummyFunction;
  module_to_unload.interceptor_address = reinterpret_cast<void*>(1);

  interceptions_.push_back(module_to_unload);
  return true;
}
```

先看看都有哪几种type：

```cpp
enum InterceptionType {
  INTERCEPTION_INVALID = 0,
  INTERCEPTION_SERVICE_CALL,  // Trampoline of an NT native call
  INTERCEPTION_EAT,
  INTERCEPTION_SIDESTEP,        // Preamble patch
  INTERCEPTION_SMART_SIDESTEP,  // Preamble patch but bypass internal calls
  INTERCEPTION_UNLOAD_MODULE,   // Unload the module (don't patch)
  INTERCEPTION_LAST             // Placeholder for last item in the enumeration
};
```

暂时只有`INTERCEPTION_UNLOAD_MODULE`的意义是明确的。

其他的几个我们给个猜想：

1. SERVICE_CALL是对系统调用的hook，为什么说成是蹦床，暂不清楚。
2. EAT应该是取代本体
3. SIDESTEP应该是inline hook，执行序列后最终会跳回原始地址
4. SMART_SIDESTEP也是inline hook，但不会跳回

根据以往对hook的理解，暂时给出这些猜想。

再看看id的分类：

```cpp
enum InterceptorId {
  // Internal use:
  MAP_VIEW_OF_SECTION_ID = 0,
  UNMAP_VIEW_OF_SECTION_ID,
  // Policy broker:
  SET_INFORMATION_THREAD_ID,
  OPEN_THREAD_TOKEN_ID,
  OPEN_THREAD_TOKEN_EX_ID,
  OPEN_THREAD_ID,
  OPEN_PROCESS_ID,
  OPEN_PROCESS_TOKEN_ID,
  OPEN_PROCESS_TOKEN_EX_ID,
  // Filesystem dispatcher:
  CREATE_FILE_ID,
  OPEN_FILE_ID,
  QUERY_ATTRIB_FILE_ID,
  QUERY_FULL_ATTRIB_FILE_ID,
  SET_INFO_FILE_ID,
  // Named pipe dispatcher:
  CREATE_NAMED_PIPE_ID,
  // Process-thread dispatcher:
  CREATE_PROCESSW_ID,
  CREATE_PROCESSA_ID,
  CREATE_THREAD_ID,
  // Registry dispatcher:
  CREATE_KEY_ID,
  OPEN_KEY_ID,
  OPEN_KEY_EX_ID,
  // Sync dispatcher:
  CREATE_EVENT_ID,
  OPEN_EVENT_ID,
  // Process mitigations Win32k dispatcher:
  GDIINITIALIZE_ID,
  GETSTOCKOBJECT_ID,
  REGISTERCLASSW_ID,
  ENUMDISPLAYMONITORS_ID,
  ENUMDISPLAYDEVICESA_ID,
  GETMONITORINFOA_ID,
  GETMONITORINFOW_ID,
  CREATEOPMPROTECTEDOUTPUTS_ID,
  GETCERTIFICATE_ID,
  GETCERTIFICATESIZE_ID,
  GETCERTIFICATEBYHANDLE_ID,
  GETCERTIFICATESIZEBYHANDLE_ID,
  DESTROYOPMPROTECTEDOUTPUT_ID,
  CONFIGUREOPMPROTECTEDOUTPUT_ID,
  GETOPMINFORMATION_ID,
  GETOPMRANDOMNUMBER_ID,
  GETSUGGESTEDOPMPROTECTEDOUTPUTARRAYSIZE_ID,
  SETOPMSIGNINGKEYANDSEQUENCENUMBERS_ID,
  INTERCEPTOR_MAX_ID
};
```

可以看出来是给每个子系统分派的。

### `InitializeInterceptions`

这个就是fire函数了，broker会把target自己需要按照的interceptions发过去。

```cpp
ResultCode InterceptionManager::InitializeInterceptions() {
  // 如果没有拦截器，什么都不用做
  if (interceptions_.empty())
    return SBOX_ALL_OK;  // Nothing to do here

  // 老套路，用GetBufferSize计算尺寸，new出一片天
  size_t buffer_bytes = GetBufferSize();
  std::unique_ptr<char[]> local_buffer(new char[buffer_bytes]);

  // 部署buffer
  if (!SetupConfigBuffer(local_buffer.get(), buffer_bytes))
    return SBOX_ERROR_CANNOT_SETUP_INTERCEPTION_CONFIG_BUFFER;

  void* remote_buffer;
  // 关键Call，buffer是如何通过broker发给target进程的呢？
  // 应该还是那个老套路，VirtualAlloc+WriteProcessMemory
  // 然后接全局变量的TransferVariable为target索引内存空间
  ResultCode rc =
      CopyDataToChild(local_buffer.get(), buffer_bytes, &remote_buffer);

  if (rc != SBOX_ALL_OK)
    return rc;

  // 如果buffer不为空，就要打Ntdll 热补丁，这个函数暂时不清楚意义何在
  bool hot_patch_needed = (0 != buffer_bytes);
  rc = PatchNtdll(hot_patch_needed);

  if (rc != SBOX_ALL_OK)
    return rc;

  // 果然有全局变量的值传递
  g_interceptions = reinterpret_cast<SharedMemory*>(remote_buffer);
  rc = child_->TransferVariable("g_interceptions", &g_interceptions,
                                sizeof(g_interceptions));
  return rc;
}
```

函数内部实际上是各种其他成员函数的组合技，逐一审视：

```cpp
size_t InterceptionManager::GetBufferSize() const {
  std::set<base::string16> dlls;
  size_t buffer_bytes = 0;

  // 迭代interception，interceptions_实际上是std::list<InterceptionData>
  for (const auto& interception : interceptions_) {
    // skip interceptions that are performed from the parent
    // 设计上拦截器分两种，一种由broker来给target执行，另一种由target自己执行
    // 前者就不需要传递给target进程了，所以这里做了skip
    // 那么怎么分类呢？实际上是根据type来判定
    if (!IsInterceptionPerformedByChild(interception))
      continue;

    // 如果本次interception的dll没有出现过，就把该dll的名称插入到dlls
    // 同一个dll尽管出现多次，也只有一个DllPatchInfo结构
    if (!dlls.count(interception.dll)) {
      // NULL terminate the dll name on the structure
      size_t dll_name_bytes = (interception.dll.size() + 1) * sizeof(wchar_t);

      // include the dll related size
      // dll_name是DLLPatchInfo的最后一个成员，是个flexible数组，这里是计算出整个结构体真实的长度，且单位要对齐
      // 我们mark一下DllPatchInfo结构体
      buffer_bytes += RoundUpToMultiple(
          offsetof(DllPatchInfo, dll_name) + dll_name_bytes, sizeof(size_t));
      dlls.insert(interception.dll);
    }

    // we have to NULL terminate the strings on the structure
    // 被拦截函数的名称尺寸以及拦截函数的尺寸，2表示两个终结符
    // 但有个疑问在于我们观察两个重载的Add方法时，发现其中一个并没有设置interceptor
    // 如果interceptor在这种情况下为空串，那么还好，但Add时并没有设置，所以可能存在隐患
    size_t strings_chars =
        interception.function.size() + interception.interceptor.size() + 2;

    // a new FunctionInfo is required per function
    // FunctionInfo也类似DllPatchInfo，function是最后一个flexible数组成员，这个function是两个字符串，一个function，一个interception
    size_t record_bytes = offsetof(FunctionInfo, function) + strings_chars;
    record_bytes = RoundUpToMultiple(record_bytes, sizeof(size_t));
    buffer_bytes += record_bytes;
  }

  // SharedMemory也是个类似的结构体，dll_list是最后一个DllPatchInfo flexible数组成员
  if (0 != buffer_bytes)
    // add the part of SharedMemory that we have not counted yet
    buffer_bytes += offsetof(SharedMemory, dll_list);

  return buffer_bytes;
}
```

所以到此可以看出，整个buffer的组成实际上是`SharedMemory`结构体起始，它的末尾会有多个`DllPatchInfo`，而每个`DllPatchInfo`的末尾都有一个`dll_name`接不定个数的`FunctionInfo`，每个`FunctionInfo`的末尾function由两个函数名称字符串组成。

每一个上层结构都有成员来维护不定个数的下层结构，这和我们之前看到的`HandleCloser`结构很相似。

我们看一下这几个结构体：

```cpp
// All interceptions:
struct SharedMemory {
  int num_intercepted_dlls;
  void* interceptor_base;
  DllPatchInfo dll_list[1];  // placeholder for the list of dlls
};

// A single dll:
struct DllPatchInfo {
  size_t record_bytes;  // rounded to sizeof(size_t) bytes
  size_t offset_to_functions;
  int num_functions;
  bool unload_module;
  wchar_t dll_name[1];  // placeholder for null terminated name
  // FunctionInfo function_info[] // followed by the functions to intercept
};

// Structures for the shared memory that contains patching information
// for the InterceptionAgent.
// A single interception:
struct FunctionInfo {
  size_t record_bytes;  // rounded to sizeof(size_t) bytes
  InterceptionType type;
  InterceptorId id;
  const void* interceptor_address;
  char function[1];  // placeholder for null terminated name
  // char interceptor[]           // followed by the interceptor function
};
```

成员里的两个注释成员也是很有灵性，注意他们对语法是不可见的。

计算出长度以后，在`InitializeInterceptions`的内部new出了这个buffer，此后进入到`SetupConfigBuffer`部署：

```cpp
// Basically, walk the list of interceptions moving them to the config buffer,
// but keeping together all interceptions that belong to the same dll.
// The config buffer is a local buffer, not the one allocated on the child.
bool InterceptionManager::SetupConfigBuffer(void* buffer, size_t buffer_bytes) {
  if (0 == buffer_bytes)
    return true;

  DCHECK(buffer_bytes > sizeof(SharedMemory));

  // 开始逐一解构
  SharedMemory* shared_memory = reinterpret_cast<SharedMemory*>(buffer);
  DllPatchInfo* dll_info = shared_memory->dll_list;
  int num_dlls = 0;

  // 如果names_used_为真，就把target进程的入口地址作为拦截器的基址
  shared_memory->interceptor_base =
      names_used_ ? child_->MainModule() : nullptr;

  buffer_bytes -= offsetof(SharedMemory, dll_list);
  buffer = dll_info;

  //开始了，抽离出每个dll，SetupDllInfo
  std::list<InterceptionData>::iterator it = interceptions_.begin();
  for (; it != interceptions_.end();) {
    // skip interceptions that are performed from the parent
    if (!IsInterceptionPerformedByChild(*it)) {
      ++it;
      continue;
    }

    // 部署本次的dll
    const base::string16 dll = it->dll;
    if (!SetupDllInfo(*it, &buffer, &buffer_bytes))
      return false;

    // walk the interceptions from this point, saving the ones that are
    // performed on this dll, and removing the entry from the list.
    // advance the iterator before removing the element from the list
    // 找到后面相同dll的interception，一并处理然后移除
    std::list<InterceptionData>::iterator rest = it;
    for (; rest != interceptions_.end();) {
      if (rest->dll == dll) {
        if (!SetupInterceptionInfo(*rest, &buffer, &buffer_bytes, dll_info))
          return false;
        if (it == rest)
          ++it;
        rest = interceptions_.erase(rest);
      } else {
        ++rest;
      }
    }
    dll_info = reinterpret_cast<DllPatchInfo*>(buffer);
    ++num_dlls;
  }

  shared_memory->num_intercepted_dlls = num_dlls;	//此时信息就修正了
  return true;
}
```

可以看到`SetupConfigBuffer`旨在修正buffer中的各个成员。对`DllPatchInfo`的处理由两个函数协助完成：

```cpp
// Fills up just the part that depends on the dll, not the info that depends on
// the actual interception.
bool InterceptionManager::SetupDllInfo(const InterceptionData& data,
                                       void** buffer,
                                       size_t* buffer_bytes) const {
  DCHECK(buffer_bytes);
  DCHECK(buffer);
  DCHECK(*buffer);

  DllPatchInfo* dll_info = reinterpret_cast<DllPatchInfo*>(*buffer);

  // the strings have to be zero terminated
  size_t required = offsetof(DllPatchInfo, dll_name) +
                    (data.dll.size() + 1) * sizeof(wchar_t);
  required = RoundUpToMultiple(required, sizeof(size_t));
  if (*buffer_bytes < required)
    return false;

  *buffer_bytes -= required;
  *buffer = reinterpret_cast<char*>(*buffer) + required;

  // set up the dll info to be what we know about it at this time
  // 首次处理一个dll时，仅仅填充了dll相关的信息，FunctionInfo还没有拉进来
  dll_info->unload_module = (data.type == INTERCEPTION_UNLOAD_MODULE);	// 是不是要unload的dll
  dll_info->record_bytes = required;
  dll_info->offset_to_functions = required;
  dll_info->num_functions = 0;
  data.dll.copy(dll_info->dll_name, data.dll.size());
  dll_info->dll_name[data.dll.size()] = L'\0';

  return true;
}

bool InterceptionManager::SetupInterceptionInfo(const InterceptionData& data,
                                                void** buffer,
                                                size_t* buffer_bytes,
                                                DllPatchInfo* dll_info) const {
  DCHECK(buffer_bytes);
  DCHECK(buffer);
  DCHECK(*buffer);

  // 都要卸载了，还patch个毛，检查一下有没有这种矛盾的情况
  if ((dll_info->unload_module) && (data.function != kUnloadDLLDummyFunction)) {
    // Can't specify a dll for both patch and unload.
    NOTREACHED();
  }

  // 常规操作，把InterceptorData的信息全部倒出来，给FunctionInfo
  FunctionInfo* function = reinterpret_cast<FunctionInfo*>(*buffer);

  size_t name_bytes = data.function.size();
  size_t interceptor_bytes = data.interceptor.size();

  // the strings at the end of the structure are zero terminated
  size_t required =
      offsetof(FunctionInfo, function) + name_bytes + interceptor_bytes + 2;
  required = RoundUpToMultiple(required, sizeof(size_t));
  if (*buffer_bytes < required)
    return false;

  // update the caller's values
  *buffer_bytes -= required;
  *buffer = reinterpret_cast<char*>(*buffer) + required;

  function->record_bytes = required;
  function->type = data.type;
  function->id = data.id;
  // FunctionInfo实际上有interceptor_address
  function->interceptor_address = data.interceptor_address;
  char* names = function->function;

  data.function.copy(names, name_bytes);
  names += name_bytes;
  *names++ = '\0';

  // interceptor follows the function_name
  // 所以对于第一种Add方法，这个interceptor_bytes应该就是0
  data.interceptor.copy(names, interceptor_bytes);
  names += interceptor_bytes;
  *names++ = '\0';

  // update the dll table
  // 这两个结构才是最为重要的，有他们才能正确定位尾随的FunctionInfo
  dll_info->num_functions++;		
  dll_info->record_bytes += required;

  return true;
}
```

此后，buffer拷贝给child:

```cpp
ResultCode InterceptionManager::CopyDataToChild(const void* local_buffer,
                                                size_t buffer_bytes,
                                                void** remote_buffer) const {
  DCHECK(remote_buffer);
  if (0 == buffer_bytes) {
    *remote_buffer = nullptr;
    return SBOX_ALL_OK;
  }

  // 借助TargetProcess拿到target进程的句柄
  HANDLE child = child_->Process();

  // Allocate memory on the target process without specifying the address
  // 经典的素质二连：VirtualAllocEx + WriteProcessMemory
  void* remote_data = ::VirtualAllocEx(child, nullptr, buffer_bytes, MEM_COMMIT,
                                       PAGE_READWRITE);
  if (!remote_data)
    return SBOX_ERROR_NO_SPACE;

  SIZE_T bytes_written;
  bool success = ::WriteProcessMemory(child, remote_data, local_buffer,
                                      buffer_bytes, &bytes_written);
  if (!success || bytes_written != buffer_bytes) {
    ::VirtualFreeEx(child, remote_data, 0, MEM_RELEASE);
    return SBOX_ERROR_CANNOT_COPY_DATA_TO_CHILD;
  }

  //注意remote_buffer参数是个二级指针，如此通过OUT型参数返回了分配地址
  *remote_buffer = remote_data;

  return SBOX_ALL_OK;
}
```

此后`g_interceptions`值也传递（`TransferVariable`）过去，target就可以用它来索引这块`remote_buffer`。

分析至此，剩余三个疑点：

1. 某个Interception是由target自己执行还是由broker执行，判断的具体条件是什么？
2. broker的fire函数仅仅只是把target自己要执行的Interceptions传递了过去，但具体是怎么使用的？broker端执行的Interceptions又是怎样使用的？
3. `PatchNtdll`究竟是什么鬼？

第一个问题实际上很简单，我们展开看看判断函数：

```cpp
// Only return true if the child should be able to perform this interception.
bool InterceptionManager::IsInterceptionPerformedByChild(
    const InterceptionData& data) const {
  // 心智健全？
  if (INTERCEPTION_INVALID == data.type)
    return false;

  // 系统调用类型的Interception不能由target自己执行，要broker去执行
  if (INTERCEPTION_SERVICE_CALL == data.type)
    return false;

  // 心智健全？
  if (data.type >= INTERCEPTION_LAST)
    return false;

  // ntdll相关的Interception，都只能由broker执行
  base::string16 ntdll(kNtdllName);
  if (ntdll == data.dll)
    return false;  // ntdll has to be intercepted from the parent

  return true;
}
```

第二个问题涉及到了target进程使用的Agent对端，很快就会看到。

第三个我们展开`PatchNtdll`看看：

```cpp
ResultCode InterceptionManager::PatchNtdll(bool hot_patch_needed) {
  // Maybe there is nothing to do
  // 如果不需要热补丁且interception空空如也，就什么都不用做
  if (!hot_patch_needed && interceptions_.empty())
    return SBOX_ALL_OK;

  // 如果需要热补丁
  if (hot_patch_needed) {
#if defined(SANDBOX_EXPORTS)
// Make sure the functions are not excluded by the linker.
#if defined(_WIN64)
#pragma comment(linker, "/include:TargetNtMapViewOfSection64")
#pragma comment(linker, "/include:TargetNtUnmapViewOfSection64")
#else
#pragma comment(linker, "/include:_TargetNtMapViewOfSection@44")
#pragma comment(linker, "/include:_TargetNtUnmapViewOfSection@12")
#endif
#endif  // defined(SANDBOX_EXPORTS)
    // 这个宏有点意思，第二个参数的两个InterceptorId是内部使用的前两个成员
    ADD_NT_INTERCEPTION(NtMapViewOfSection, MAP_VIEW_OF_SECTION_ID, 44);
    ADD_NT_INTERCEPTION(NtUnmapViewOfSection, UNMAP_VIEW_OF_SECTION_ID, 12);
  }

  // Reserve a full 64k memory range in the child process.
  // 在target进程中储备一个64k大小的内存
  HANDLE child = child_->Process();
  BYTE* thunk_base = reinterpret_cast<BYTE*>(::VirtualAllocEx(
      child, nullptr, kAllocGranularity, MEM_RESERVE, PAGE_NOACCESS));

  // Find an aligned, random location within the reserved range.
  // 每一个interceptions都占用一个ThunkData，ThunkData实际上是个char[64]包装
  // DllInterceptionData是ThunkData flexible数组的头部
  size_t thunk_bytes =
      interceptions_.size() * sizeof(ThunkData) + sizeof(DllInterceptionData);
  // 在64k内存中找到一个随机的偏移起始，当然偏移必须要合适，即64k-offset的尺寸比thunk_bytes大
  size_t thunk_offset = internal::GetGranularAlignedRandomOffset(thunk_bytes);

  // Split the base and offset along page boundaries.
  thunk_base += thunk_offset & ~(kPageSize - 1);
  thunk_offset &= kPageSize - 1;

  // Make an aligned, padded allocation, and move the pointer to our chunk.
  size_t thunk_bytes_padded = (thunk_bytes + kPageSize - 1) & ~(kPageSize - 1);
  // 分配padded部分内存空间
  thunk_base = reinterpret_cast<BYTE*>(
      ::VirtualAllocEx(child, thunk_base, thunk_bytes_padded, MEM_COMMIT,
                       PAGE_EXECUTE_READWRITE));
  CHECK(thunk_base);  // If this fails we'd crash anyway on an invalid access.
  // 找到存储DllInterceptionData的起始位置
  DllInterceptionData* thunks =
      reinterpret_cast<DllInterceptionData*>(thunk_base + thunk_offset);

  DllInterceptionData dll_data;
  dll_data.data_bytes = thunk_bytes;
  dll_data.num_thunks = 0;
  dll_data.used_bytes = offsetof(DllInterceptionData, thunks);

  // Reset all helpers for a new child.
  // 清空g_originals全局函数指针数组
  memset(g_originals, 0, sizeof(g_originals));

  // this should write all the individual thunks to the child's memory
  // 对target进程的内存空间写入数据，这个函数看起来很关键
  ResultCode rc = PatchClientFunctions(thunks, thunk_bytes, &dll_data);

  if (rc != SBOX_ALL_OK)
    return rc;

  // and now write the first part of the table to the child's memory
  // 这里向target进程的内存空间写入了DllInterceptionData头部数据
  // 看起来PatchClientFunctions写的是ThunkData[]，并更新了dll_data的成员
  SIZE_T written;
  bool ok =
      !!::WriteProcessMemory(child, thunks, &dll_data,
                             offsetof(DllInterceptionData, thunks), &written);

  if (!ok || (offsetof(DllInterceptionData, thunks) != written))
    return SBOX_ERROR_CANNOT_WRITE_INTERCEPTION_THUNK;

  // Attempt to protect all the thunks, but ignore failure
  // 对target进程这段空间设置为只读，不允许更改
  DWORD old_protection;
  ::VirtualProtectEx(child, thunks, thunk_bytes, PAGE_EXECUTE_READ,
                     &old_protection);

  // 这里传过去了g_originals全局变量，显然这货肯定在PatchClientFunctions中调整了值
  ResultCode ret =
      child_->TransferVariable("g_originals", g_originals, sizeof(g_originals));
  return ret;
}
```

看看`ADD_NT_INTERCEPTION`：

```cpp
#define ADD_NT_INTERCEPTION(service, id, num_params)        \
  AddToPatchedFunctions(kNtdllName, #service,               \
                        sandbox::INTERCEPTION_SERVICE_CALL, \
                        MAKE_SERVICE_NAME(service, num_params), id)

// This macro simply calls interception_manager.AddToPatchedFunctions with
// the given service to intercept (INTERCEPTION_SERVICE_CALL), and assumes that
// the interceptor is called "TargetXXX", where XXX is the name of the service.
// Note that num_params is the number of bytes to pop out of the stack for
// the exported interceptor, following the calling convention of a service call
// (WINAPI = with the "C" underscore).
#if SANDBOX_EXPORTS
#if defined(_WIN64)
#define MAKE_SERVICE_NAME(service, params) "Target" #service "64"
#else
#define MAKE_SERVICE_NAME(service, params) "_Target" #service "@" #params
#endif
```

其实就是拦截ntdll中的某个系统调用，拦截的函数x64上叫TargetXXX64，x86叫_TargetXXX@YYY。

XXX是系统调用名称，YYY是拦截器在栈上所用的字节数。有个问题在于，这个拦截函数是在哪儿定义的呢？

再看关键Call：

```cpp
ResultCode InterceptionManager::PatchClientFunctions(
    DllInterceptionData* thunks,
    size_t thunk_bytes,
    DllInterceptionData* dll_data) {
  DCHECK(thunks);
  DCHECK(dll_data);

  // 拿到ntdll基址
  HMODULE ntdll_base = ::GetModuleHandle(kNtdllName);
  if (!ntdll_base)
    return SBOX_ERROR_NO_HANDLE;

  char* interceptor_base = nullptr;

#if defined(SANDBOX_EXPORTS)
  interceptor_base = reinterpret_cast<char*>(child_->MainModule());
  base::ScopedNativeLibrary local_interceptor(::LoadLibrary(child_->Name()));
#endif  // defined(SANDBOX_EXPORTS)

  // 这个ServiceResolverThunk是个什么东西？
  std::unique_ptr<ServiceResolverThunk> thunk;
#if defined(_WIN64)
  thunk.reset(new ServiceResolverThunk(child_->Process(), relaxed_));
#else
  base::win::OSInfo* os_info = base::win::OSInfo::GetInstance();
  if (os_info->wow64_status() == base::win::OSInfo::WOW64_ENABLED) {
    if (os_info->version() >= base::win::VERSION_WIN10)
      thunk.reset(new Wow64W10ResolverThunk(child_->Process(), relaxed_));
    else if (os_info->version() >= base::win::VERSION_WIN8)
      thunk.reset(new Wow64W8ResolverThunk(child_->Process(), relaxed_));
    else
      thunk.reset(new Wow64ResolverThunk(child_->Process(), relaxed_));
  } else if (os_info->version() >= base::win::VERSION_WIN8) {
    thunk.reset(new Win8ResolverThunk(child_->Process(), relaxed_));
  } else {
    thunk.reset(new ServiceResolverThunk(child_->Process(), relaxed_));
  }
#endif

  for (auto interception : interceptions_) {
    const base::string16 ntdll(kNtdllName);
    // 必须的是ntdll的系统调用类型拦截器
    if (interception.dll != ntdll)
      return SBOX_ERROR_BAD_PARAMS;

    if (INTERCEPTION_SERVICE_CALL != interception.type)
      return SBOX_ERROR_BAD_PARAMS;

#if defined(SANDBOX_EXPORTS)
    // We may be trying to patch by function name.
    if (!interception.interceptor_address) {
      const char* address;
      NTSTATUS ret = thunk->ResolveInterceptor(
          local_interceptor.get(), interception.interceptor.c_str(),
          reinterpret_cast<const void**>(&address));
      if (!NT_SUCCESS(ret)) {
        ::SetLastError(GetLastErrorFromNtStatus(ret));
        return SBOX_ERROR_CANNOT_RESOLVE_INTERCEPTION_THUNK;
      }

      // Translate the local address to an address on the child.
      interception.interceptor_address =
          interceptor_base +
          (address - reinterpret_cast<char*>(local_interceptor.get()));
    }
#endif  // defined(SANDBOX_EXPORTS)
    NTSTATUS ret = thunk->Setup(
        ntdll_base, interceptor_base, interception.function.c_str(),
        interception.interceptor.c_str(), interception.interceptor_address,
        &thunks->thunks[dll_data->num_thunks],	// 这里把thunk发在了thunks中，thunks实际上是child的内存空间
        thunk_bytes - dll_data->used_bytes, nullptr);
    if (!NT_SUCCESS(ret)) {
      ::SetLastError(GetLastErrorFromNtStatus(ret));
      return SBOX_ERROR_CANNOT_SETUP_INTERCEPTION_THUNK;
    }

    // 看起来都是围绕着ServiceResolverThunk对象的操作，暂时不关心他具体做了什么
    // 但这里可以看到g_originals的ntdll service call的id索引到了正确的thunks
    // 实际上就是Interceptor id中第一个类别：internal
    DCHECK(!g_originals[interception.id]);
    g_originals[interception.id] = &thunks->thunks[dll_data->num_thunks];

    dll_data->num_thunks++;
    dll_data->used_bytes += sizeof(ThunkData);
  }

  return SBOX_ALL_OK;
}
```

`PatchNtdll`看得云里雾里，看起来`InterceptionManager`确实是个名副其实的管理者，负责的工作有限，我们到此还没有看到完整的功能拼图。

```cpp
// Dummy single thunk:
struct ThunkData {
  char data[kMaxThunkDataBytes];
};

// In-memory representation of the interceptions for a given dll:
struct DllInterceptionData {
  size_t data_bytes;
  size_t used_bytes;
  void* base;
  int num_thunks;
#if defined(_WIN64)
  int dummy;  // Improve alignment.
#endif
  ThunkData thunks[1];
};
```

## `InterceptionAgent`

暂时不理`ServiceResolverThunk`，去看看target进程的收端Agent：

```cpp
// of setting up the desired interceptions or indicating what module needs to
// be unloaded.
// target进程上部署具体的拦截器，broker的manager把interceptions发了过来
// 
// The exposed API consists of three methods: GetInterceptionAgent to retrieve
// the single class instance, OnDllLoad and OnDllUnload to process a dll being
// loaded and unloaded respectively.
//
// This class assumes that it will get called for every dll being loaded,
// starting with kernel32, so the singleton will be instantiated from within the
// loader lock.
class InterceptionAgent {
 public:
  // Returns the single InterceptionAgent object for this process.
  // InterceptionAgent是单例模式
  static InterceptionAgent* GetInterceptionAgent();

  // This method should be invoked whenever a new dll is loaded to perform the
  // required patches. If the return value is false, this dll should not be
  // allowed to load.
  //
  // full_path is the (optional) full name of the module being loaded and name
  // is the internal module name. If full_path is provided, it will be used
  // before the internal name to determine if we care about this dll.
  bool OnDllLoad(const UNICODE_STRING* full_path, const UNICODE_STRING* name,
                 void* base_address);

  // Performs cleanup when a dll is unloaded.
  void OnDllUnload(void* base_address);

 private:
  ~InterceptionAgent() {}	// 限定到static成员使用

  // Performs initialization of the singleton.
  bool Init(SharedMemory* shared_memory);

  // Returns true if we are interested on this dll. dll_info is an entry of the
  // list of intercepted dlls.
  bool DllMatch(const UNICODE_STRING* full_path, const UNICODE_STRING* name,
                const DllPatchInfo* dll_info);

  // Performs the patching of the dll loaded at base_address.
  // The patches to perform are described on dll_info, and thunks is the thunk
  // storage for the whole dll.
  // Returns true on success.
  bool PatchDll(const DllPatchInfo* dll_info, DllInterceptionData* thunks);

  // 这里又出现了一个ResolverThunk
  // Returns a resolver for a given interception type.
  ResolverThunk* GetResolver(InterceptionType type);

  // 这个是SharedMemory结构
  // 里面描述了有哪些dll，每个dll的哪些函数被hook了
  // Shared memory containing the list of functions to intercept.
  SharedMemory* interceptions_;

  // Array of thunk data buffers for the intercepted dlls. This object singleton
  // is allocated with a placement new with enough space to hold the complete
  // array of pointers, not just the first element.
  // 这个是DllInterceptionData flexible数组结构，每个dll对应一个成员，这里面承载的是thunk数据
  DllInterceptionData* dlls_[1];

  DISALLOW_IMPLICIT_CONSTRUCTORS(InterceptionAgent);
};
```

### `GetInterceptionAgent`

先看看单例模式的静态Get方法：

```cpp
// Memory buffer mapped from the parent, with the list of interceptions.
// 这个已经认识了，存储数据的内存空间由broker开辟，g_interceptions会指向那段空间
SANDBOX_INTERCEPT SharedMemory* g_interceptions = nullptr;

InterceptionAgent* InterceptionAgent::GetInterceptionAgent() {
  static InterceptionAgent* s_singleton = nullptr;
  if (!s_singleton) {
    if (!g_interceptions)	// broker得把信息发过来先
      return nullptr;

    // 注意这里并不是直接new InterceptionAgent对象，而是附加了array_bytes尺寸
    // 附加的尺寸实际上就是flexible成员dlls_这个DllInterceptionData指针数组的尺寸
    // 每有一个dll，就分配一个DllInterceptionData指针出来
    size_t array_bytes = g_interceptions->num_intercepted_dlls * sizeof(void*);
    s_singleton = reinterpret_cast<InterceptionAgent*>(
        new (NT_ALLOC) char[array_bytes + sizeof(InterceptionAgent)]);

    // 实际上是个“零化”函数，全局g_interceptions会在broker中设置，然后调整interceptions_成员
    // 指向这片内存空间，此外，所有的dlls_[i]（每个对应一个DllInterceptionData结构）先指向null
    bool success = s_singleton->Init(g_interceptions);
    if (!success) {
      operator delete(s_singleton, NT_ALLOC);
      s_singleton = nullptr;
    }
  }
  return s_singleton;
}
```

### `OnDllLoad`

```cpp
bool InterceptionAgent::OnDllLoad(const UNICODE_STRING* full_path,
                                  const UNICODE_STRING* name,
                                  void* base_address) {
  // 解包，拿到DllPatchInfo结构
  DllPatchInfo* dll_info = interceptions_->dll_list;
  int i = 0;
  // 当dll加载时，先看看想要加载的这个dll是否在拦截列表里
  for (; i < interceptions_->num_intercepted_dlls; i++) {
    if (DllMatch(full_path, name, dll_info))
      break;

    // 定位下一个DllPatchInfo
    dll_info = reinterpret_cast<DllPatchInfo*>(
        reinterpret_cast<char*>(dll_info) + dll_info->record_bytes);
  }

  // Return now if the dll is not in our list of interest.
  // 如果不在，说明拦截器对它没兴趣，那么直接load就行了
  if (i == interceptions_->num_intercepted_dlls)
    return true;

  // The dll must be unloaded.
  // 到此说明拦截器有这个dll，如果它unload_module的true，那么就表示不允许load
  if (dll_info->unload_module)
    return false;

  // Purify causes this condition to trigger.
  // 看看是否已经部署了Dll的DllInterceptionData，也就是拦截器已实装。如果已经实装了，就let it go
  if (dlls_[i])
    return true;

  // 还没有的话就要new出来这个dlls_[i]
  // 每个拦截的函数都有一个ThunkData，为这个拦截dll分配出DllInterceptionData
  // 在dll加载的base_address处分配，这个base_address是参数，究竟把这些thunk data写到了哪里呢？
  size_t buffer_bytes = offsetof(DllInterceptionData, thunks) +
                        dll_info->num_functions * sizeof(ThunkData);
  dlls_[i] = reinterpret_cast<DllInterceptionData*>(
      new (NT_PAGE, base_address) char[buffer_bytes]);

  DCHECK_NT(dlls_[i]);
  // 这种情况居然返回true？我很好奇如何fall through
  if (!dlls_[i])
    return true;

  // 这个时候还仅仅只是一片初始化的空间，ThunkData未填充，num_thunks待更新
  dlls_[i]->data_bytes = buffer_bytes;
  dlls_[i]->num_thunks = 0;
  dlls_[i]->base = base_address;
  dlls_[i]->used_bytes = offsetof(DllInterceptionData, thunks);

  // 这个PatchDll看起来很关键，从传入的两个参数就能看出来，应该是真正的thunk实装
  VERIFY(PatchDll(dll_info, dlls_[i]));

  // dlls_[i]指向的这段内存标记为RE，防止Write
  ULONG old_protect;
  SIZE_T real_size = buffer_bytes;
  void* to_protect = dlls_[i];
  VERIFY_SUCCESS(g_nt.ProtectVirtualMemory(NtCurrentProcess, &to_protect,
                                           &real_size, PAGE_EXECUTE_READ,
                                           &old_protect));
  return true;
}
```

关键的`PatchDll`：

```cpp
// TODO(rvargas): We have to deal with prebinded dlls. I see two options: change
// the timestamp of the patched dll, or modify the info on the prebinded dll.
// the first approach messes matching of debug symbols, the second one is more
// complicated.
bool InterceptionAgent::PatchDll(const DllPatchInfo* dll_info,
                                 DllInterceptionData* thunks) {
  DCHECK_NT(thunks);
  DCHECK_NT(dll_info);

  // 定位到DllPatchInfo中的第一个FunctionInfo
  const FunctionInfo* function = reinterpret_cast<const FunctionInfo*>(
      reinterpret_cast<const char*>(dll_info) + dll_info->offset_to_functions);

  // 对每个需要interception的函数进行处理
  for (int i = 0; i < dll_info->num_functions; i++) {
    // 尺寸校验
    if (!IsWithinRange(dll_info, dll_info->record_bytes, function->function)) {
      NOTREACHED_NT();
      return false;
    }

    // 这里用到了resolver，function->type指定了这是哪一类别的interception，所以就找到具体的resolver
    // 怎么看resolver都是个父类指针指向子类对象，而猜测每一种type的interception都会有一个resolver
    ResolverThunk* resolver = GetResolver(function->type);
    if (!resolver)
      return false;

    // interceptor的名称紧随function的名称其后
    const char* interceptor =
        function->function + g_nt.strlen(function->function) + 1;

    if (!IsWithinRange(function, function->record_bytes, interceptor) ||
        !IsWithinRange(dll_info, dll_info->record_bytes, interceptor)) {
      NOTREACHED_NT();
      return false;
    }
	// resolver的Setup方法看起来很关键，thunk data和DllPatchInfo的关联就在此处
    NTSTATUS ret = resolver->Setup(
        thunks->base, interceptions_->interceptor_base, function->function,
        interceptor, function->interceptor_address, &thunks->thunks[i],
        sizeof(ThunkData), nullptr);
    if (!NT_SUCCESS(ret)) {
      NOTREACHED_NT();
      return false;
    }

    DCHECK_NT(!g_originals[function->id]);
    // 这里调整了g_originals的数组成员，根据被拦截函数的id，索引指针指向正确的thunk data
    // 看起来DllPatchInfo中的FunctionInfo和DllInterceptionData中的ThunkData一一对应
    // FunctionInfo[i]->ThunkData[i]
    g_originals[function->id] = &thunks->thunks[i];

    // 更新DllInterceptionData的数据成员
    thunks->num_thunks++;
    thunks->used_bytes += sizeof(ThunkData);

    // 但FunctionInfo毕竟是不定长的，不能用FunctionInfo[i]，所以要通过record_bytes来step
    function = reinterpret_cast<const FunctionInfo*>(
        reinterpret_cast<const char*>(function) + function->record_bytes);
  }

  return true;
}
```

具体resolver是如何利用的：

```cpp
// This method is called from within the loader lock
ResolverThunk* InterceptionAgent::GetResolver(InterceptionType type) {
  // 这几个都是ResolverThunk的派生类，根据名称可以发现和type息息相关
  // 看起来此前的推断是正确的
  static EatResolverThunk* eat_resolver = nullptr;
  static SidestepResolverThunk* sidestep_resolver = nullptr;
  static SmartSidestepResolverThunk* smart_sidestep_resolver = nullptr;

  // static成员，在第一次进入时new出单例
  if (!eat_resolver)
    eat_resolver = new (NT_ALLOC) EatResolverThunk;

#if !defined(_WIN64)
  // Sidestep is not supported for x64.
  // sidestep和smart_sidestep不能在x64上使用
  if (!sidestep_resolver)
    sidestep_resolver = new (NT_ALLOC) SidestepResolverThunk;

  if (!smart_sidestep_resolver)
    smart_sidestep_resolver = new (NT_ALLOC) SmartSidestepResolverThunk;
#endif

  // 根据type返回对应type的resolver
  // 注意到实际上只有3种类型，少了service call和unload module
  // unload module可以理解，毕竟不涉及到function的interception，但service call为什么用不到呢？
  // 根据此前对两种interception的了解，service call应该都是由broker来执行的，target并不会用到
  switch (type) {
    case INTERCEPTION_EAT:
      return eat_resolver;
    case INTERCEPTION_SIDESTEP:
      return sidestep_resolver;
    case INTERCEPTION_SMART_SIDESTEP:
      return smart_sidestep_resolver;
    default:
      NOTREACHED_NT();
  }

  return nullptr;
}
```

另一个最为关键的就是`Resolver`的`Setup`方法了，是他把`ThunkData`和`FunctionInfo`关联了起来，实装了补丁，那么具体是如何操纵的呢？我们下一篇抽离`Resolver`相关内容单独分析。

### `OnDllUnload`

```cpp
void InterceptionAgent::OnDllUnload(void* base_address) {
  for (int i = 0; i < interceptions_->num_intercepted_dlls; i++) {
    // 这种逻辑处理方式，如果有两个感兴趣的dll在load的时候，base_address一致怎么办？
    // 这样就可能在unload时找错了目标。
    // 但仔细想想，如果base_address是dll加载基址的话，不同dll也不可能具有相同的base_address
    // 所以这个base_address的意义就很重要了
    if (dlls_[i] && dlls_[i]->base == base_address) {
      operator delete(dlls_[i], NT_PAGE);
      dlls_[i] = nullptr;
      break;
    }
  }
}
```

这个就相当简单了，此前安装的dll如果部署了interception，那就需要delete掉在`base_address`分配的内存空间。这部分内存空间此前用来承载thunk data。

到此，`Interception`相关的Manager和Agent就分析完了，显然缺少了`Resolver`的介入，我们对整个流程还是不甚清楚，下一节我们分析`Resolver`，直捣黄龙。