---
title: Chromium-sandbox-HandleCloser-analysis
date: 2018-05-21 20:13:11
categories: 源码剖析
tags:
	- chromium
	- chromium-sandbox
---

本篇是sandbox源码剖析的第九篇，主要分析了windows平台下，Chromium sandbox中broker制造并传输target需要关闭的句柄的实现机制。阅读本篇前，请先阅读前四篇。

想要流程的阅读本系列你需要以下几个条件：
1. 较高水平的C++编码能力（至少通读C++ Primer 5th，刷过课后题，有一定编码量）。
2. 熟悉Windows API编程，尤其是安全相关的内容。
3. 对二进制安全有一定了解，熟悉各类型安全漏洞成因、漏洞缓解措施及bypass手法。

<!--more-->

# chromium-sandbox-HandleCloser-analysis

在分析了`PolicyBase`的三大组件后，转去分析了一波不太相关的`SharedMemIPC`机制。实际上`PolicyBase`内部还有很多东西我们没有拆解，比如AppContainer、`HandleClose`、Mitigations等等。今天来分析一个比较简单的`HandleCloser`类。该类模块维护了一个句柄类型到名称的映射，用于在target进程中关闭句柄。而联系此前在`PolicyBase`中所观察到的对`handle_closer_`成员的使用以及句柄列表的传输，这个模块的功用也就呼之欲出了。

## `PolicyBase` related

从`PolicyBase`的操纵开始分析，在`BrokerServicesBase::SpawnTarget`中创建`TargetProcess`并将target挂到`PolicyBase`对象时，调用了`PolicyBase::AddTarget`，而它的内部进行了这样几个操作：

1. low-level-policy的收工（这个low-level-policy是个狭义的policy，和`PolicyBase`的这个policy不是一个层面的意思，我们之后会说到，windows子系统行为鉴权的三大构成：policy+dispatcher+interception）
2. mitigation对target进程的应用（当前还是挂起态）
3. 部署Interceptions
4. 部署`HandleCloser`
5. 创建`SharedMemIPC`的Server端，实际上创建的是IPC server和`Dispatcher`，用于处理target进程的IPC请求。
6. 对target进程的全局变量`g_shared_delayed_integrity_level`和`g_shared_delayed_mitigations`赋值

这里关于第五点和第六点，我们都已经清楚了它的实现机理与意义，前三个我们暂时不关心，看看第四点对`HandleCloser`的部署：

```cpp
bool PolicyBase::SetupHandleCloser(TargetProcess* target) {
  // 调用了handle_closer_的InitializeTargetHandles方法
  // handle_closer_是PolicyBase内部的HandleCloser对象
  return handle_closer_.InitializeTargetHandles(target);
}
```

## `HandleCloser`

在展开`HandleCloser`类头定义前，先看看该类成员类型`HandleMap`的定义。`HandleMap`是这样的一种类型：`typedef std::map<const base::string16, std::set<base::string16>> HandleMap;`

从`const string16`(也就是`const std::wstring`)到`set<base::string16>`的一个map。

除了`HandleMap`以外，先不关心其他的辅助结构，看看类头：

```cpp
// Adds handles to close after lockdown.
// 在target进程被LowerToken()降权时，关闭加入的所有句柄
class HandleCloser {
 public:
  HandleCloser();
  ~HandleCloser();

  // Adds a handle that will be closed in the target process after lockdown.
  // A nullptr value for handle_name indicates all handles of the specified
  // type. An empty string for handle_name indicates the handle is unnamed.
  // 可以看到这个接口需要被提供句柄类型和名称
  // 如果name是nullptr，那就表示所有的该类型句柄都要加入
  // 如果name是空串，就表示该句柄是匿名的
  ResultCode AddHandle(const base::char16* handle_type,
                       const base::char16* handle_name);

  // Serializes and copies the closer table into the target process.
  // 看起来是序列化待关闭句柄表，copy给target进程
  // 这个函数就是在PolicyBase::SetupHandleCloser中调用的
  bool InitializeTargetHandles(TargetProcess* target);

 private:
  // Calculates the memory needed to copy the serialized handles list (rounded
  // to the nearest machine-word size).
  // copy序列化句柄列表时，用于计算buffer尺寸
  size_t GetBufferSize();

  // Serializes the handle list into the target process.
  // 序列化句柄列表到target进程
  bool SetupHandleList(void* buffer, size_t buffer_bytes);

  HandleMap handles_to_close_;	//这个HandleMap就是维护句柄类型到名称的成员

  DISALLOW_COPY_AND_ASSIGN(HandleCloser);
};
```

此外，有个全局的`g_handle_closer_info`指针指向`HandleCloserInfo`结构。

```cpp
// Global parameters and a pointer to the list of entries.
// 看起来是个全局量，内部维护了一个到handle列表的指针
struct HandleCloserInfo {
  size_t record_bytes;  // Rounded to sizeof(size_t) bytes.
  size_t num_handle_types;	//有多少种类型
  struct HandleListEntry handle_entries[1];	//应该是flexible
};

// Type and set of corresponding handle names to close.
// 某种类型句柄的所有handle
struct HandleListEntry {
  size_t record_bytes;     // Rounded to sizeof(size_t) bytes.
  size_t offset_to_names;  // Nul terminated strings of name_count names.
  size_t name_count;	// 有多少个name
  base::char16 handle_type[1];	// 这个应该不是flexible，表示type类型
};
```

这里的设计很像`SharedMemIPC`中buffer的设计，它也分为三级结构：

1. `HandleCloserInfo`的`num_handle_types`表明了有多少个`HandleListEntry`结构
2. 每个`HandleListEntry`包含了一个`offset_to_names`成员，指向了存储该type的句柄所有名称的位置
3. 指向的真实数据就是一堆句柄的名称了，它们应该是紧凑的附加在`HandleListEntry`数组的正后方的。

当然，这里仅仅是大胆的猜测，是否真的如此，还得继续看代码逻辑处理。

###`InitializeTargetHandles`

由`PolicyBase::SetupHandleCloser`跟入：

```cpp
bool HandleCloser::InitializeTargetHandles(TargetProcess* target) {
  // Do nothing on an empty list (global pointer already initialized to
  // nullptr).
  // 如果没什么需要target关闭的，返回就好了
  if (handles_to_close_.empty())
    return true;

  size_t bytes_needed = GetBufferSize();//先不管他怎么算出来的
  // new出一片空间，划分成size_t大小的块，local_buffer指向首块
  std::unique_ptr<size_t[]> local_buffer(
      new size_t[bytes_needed / sizeof(size_t)]);

  // 切成size_t数组的缓冲区丢给SetupHandleList，内部应该就是对buffer结构的部署
  if (!SetupHandleList(local_buffer.get(), bytes_needed))
    return false;

  // 拿到target进程的进程句柄
  HANDLE child = target->Process();

  // Windows跨进程写入数据的常见套路VirtualAllocEx->WriteProcessMemory
  // 然而第二个参数使用的是nullptr，这就表示broker在这里并不是把数据写入到某个固定的内存空间
  // Allocate memory in the target process without specifying the address
  void* remote_data = ::VirtualAllocEx(child, nullptr, bytes_needed, MEM_COMMIT,
                                       PAGE_READWRITE);
  if (!remote_data)
    return false;

  // Copy the handle buffer over.
  SIZE_T bytes_written;
  bool result = ::WriteProcessMemory(child, remote_data, local_buffer.get(),
                                     bytes_needed, &bytes_written);
  if (!result || bytes_written != bytes_needed) {
    ::VirtualFreeEx(child, remote_data, 0, MEM_RELEASE);
    return false;
  }
	
  // 为了让target自己能够找到这段内存空间，broker把一个指向这段内存起始的remote_data也传了过去
  // 依然是通过老方法TransferVariable，对target自己的全局变量g_handles_to_close赋值
  // 这里也可以看到，内存空间已经布局成了HandleCloserInfo，暂时和我们的猜想一致
  g_handles_to_close = reinterpret_cast<HandleCloserInfo*>(remote_data);
  ResultCode rc = target->TransferVariable(
      "g_handles_to_close", &g_handles_to_close, sizeof(g_handles_to_close));

  return (SBOX_ALL_OK == rc);
}
```

展开`GetBufferSize`看看：

```cpp
size_t HandleCloser::GetBufferSize() {
  // 先算出HandleCloserInfo第一层数据的尺寸
  size_t bytes_total = offsetof(HandleCloserInfo, handle_entries);

  // 迭代每种type的handles，
  for (HandleMap::iterator i = handles_to_close_.begin();
       i != handles_to_close_.end(); ++i) {
    // bytes_entry表示HandleListEntry二层结构的尺寸加上handle_type本身这个字符串的尺寸
    size_t bytes_entry = offsetof(HandleListEntry, handle_type) +
                         (i->first.size() + 1) * sizeof(base::char16);
    // 迭代该类型handles的set，里面存的都是名称
    for (HandleMap::mapped_type::iterator j = i->second.begin();
         j != i->second.end(); ++j) {
      // 这里又对bytes_entry累加了所有名称的长度
      bytes_entry += ((*j).size() + 1) * sizeof(base::char16);
    }

    // Round up to the nearest multiple of word size.
    // 因为字符串的单位不规则，为了解构时方便索引下一个HandleListEntry，做了个对齐操作
    bytes_entry = RoundUpToWordSize(bytes_entry);
    // 加到总的尺寸上
    bytes_total += bytes_entry;
  }

  return bytes_total;
}
```

好吧，看到这个`GetBuffer`感觉和此前的猜想有些出入，至少`HandleListEntry`的`handle_type`是flexible的，但它的长度是不定的，而紧随在一个`HandleListEntry`之后的并不是下一个`HandleListEntry`，应该是该种类型handles的名称字符串集合，`HandleListEntry`的`offset_to_names`指向了第一个name起始。

为了确认这一布局，我们在`SetupHandleList`中一探究竟：

```cpp
bool HandleCloser::SetupHandleList(void* buffer, size_t buffer_bytes) {
  // buffer_bytes就是GetBuffer算出来的尺寸，buffer就是new出来的空间
  ::ZeroMemory(buffer, buffer_bytes);
  HandleCloserInfo* handle_info = reinterpret_cast<HandleCloserInfo*>(buffer);
  // 总的字节数，有多少种句柄
  handle_info->record_bytes = buffer_bytes;
  handle_info->num_handle_types = handles_to_close_.size();

  // output指向第一个HandleListEntry起始位置
  // end指向整个buffer的末尾
  base::char16* output =
      reinterpret_cast<base::char16*>(&handle_info->handle_entries[0]);
  base::char16* end = reinterpret_cast<base::char16*>(
      reinterpret_cast<char*>(buffer) + buffer_bytes);
  // 这里面包含了各种不定长的HandleListEntry对nameset的嵌套
  // 还是迭代每种类型
  for (HandleMap::iterator i = handles_to_close_.begin();
       i != handles_to_close_.end(); ++i) {
    // 这说明算出来的buffer有问题
    if (output >= end)
      return false;
    // 找到本次迭代的HandleListEntry
    // 尽管HandleListEntry在HandleClientInfo中定义成了flexible array，但因为不定长的原因
    // 不能直接使用HandleListEntry[index]来索引
    HandleListEntry* list_entry = reinterpret_cast<HandleListEntry*>(output);
    //调整output到handle_type，起始处是个type name
    output = &list_entry->handle_type[0];

    // Copy the typename and set the offset and count.
    i->first.copy(output, i->first.size());
    *(output += i->first.size()) = L'\0';
    output++;
    // skip typename后，就是一串name了，偏移也就可以算出来了
    list_entry->offset_to_names =
        reinterpret_cast<char*>(output) - reinterpret_cast<char*>(list_entry);
    list_entry->name_count = i->second.size();

    // Copy the handle names.
    // 拷贝所有的handle name到buffer中，output相应做位置调整
    for (HandleMap::mapped_type::iterator j = i->second.begin();
         j != i->second.end(); ++j) {
      output = std::copy((*j).begin(), (*j).end(), output) + 1;
    }

    // Round up to the nearest multiple of sizeof(size_t).
    // 向上取整，按size_t对齐，这个和GetBuffer的对齐方法必须要一致，否则这里的迭代会有问题
    output = RoundUpToWordSize(output);
    list_entry->record_bytes =
        reinterpret_cast<char*>(output) - reinterpret_cast<char*>(list_entry);
    //处理下一个type
  }

  DCHECK_EQ(reinterpret_cast<size_t>(output), reinterpret_cast<size_t>(end));
  return output <= end;
}
```

看了这个布局函数，也就彻底明白了`HandleCloserInfo`的嵌套结构。所以，`HandleCloserInfo`和`SharedMemIPC`的设计不同，前者只有两级结构，二级结构的flexible数组是不定长的，而后者则是三级结构，二级结构flexible数组定长。

此外，我们在`InitializeTargetHandles`中看到，这些数组都被拷贝给了target进程中，由`g_handles_to_close`全局变量来定位，且不管target自己是怎么关闭这些handle的（还能怎么关，不过是同种方式解包然后`CloseHandle`），至少我们知道了在调用`InitializeTargetHandles`之前，`handles_to_close_`成员就应该已经填充好了需要关闭的句柄（这就和target可继承句柄列表的设置很像，先填充成员，再`CreateProcess`）。

那么添加句柄的接口是谁呢？是还未分析的`AddHandle`。操纵`AddHandle`的又是谁呢？现在还不知道。

### `AddHandle`

先看看`AddHandle`：

```cpp
ResultCode HandleCloser::AddHandle(const base::char16* handle_type,
                                   const base::char16* handle_name) {
  if (!handle_type)	// 这个必须有
    return SBOX_ERROR_BAD_PARAMS;

  base::string16 resolved_name;
  if (handle_name) {	// 这个可以有
    resolved_name = handle_name;
    if (handle_type == base::string16(L"Key"))	// 如果handle是Key类型，那么名称要修剪一下,ResolveRegistryName其实是一个Windows注册表相关操作简单的封装
      // 这里就不详细研究名称是什么了，想要了解可以用调试器下断点分析
      if (!ResolveRegistryName(resolved_name, &resolved_name))
        return SBOX_ERROR_BAD_PARAMS;
  }

  // 先在handles_to_close_找一下是否有该类型的句柄，map中type为键
  HandleMap::iterator names = handles_to_close_.find(handle_type);
  if (names == handles_to_close_.end()) {  // We have no entries for this type.
    // 如果没有的话，就插入handle_type, HandleMap::mapped_type()这样的一个entry
    std::pair<HandleMap::iterator, bool> result = handles_to_close_.insert(
        HandleMap::value_type(handle_type, HandleMap::mapped_type()));
    names = result.first;// names索引插入的entry
    if (handle_name)
      	names->second.insert(resolved_name);// 如果有名字，那就在该type对应的空set中插入该名字
  } else if (!handle_name) {  // Now we need to close all handles of this type.
    // 如果handle_name是nullptr，会清掉当前该type对应set的所有名字
    // 原来是clear，看前面注释还以为是把所有该类型句柄都加进来。。。想想也不太可能
    names->second.clear();
  } else if (!names->second.empty()) {  // Add another name for this type.
    // 如果当前type对应的set不为空，则插入另一个名字
    names->second.insert(resolved_name);
  }  // If we're already closing all handles of type then we're done.
	// 如果此时已经全都清掉了，那再来的就不管了。
  return SBOX_ALL_OK;
}
```

> 逻辑很紧凑，也可以看出它的工作机理和外部的操纵息息相关，外部如果出现第一次插入某个匿名的不存在其他该type的句柄，然后后续又插入一个该type的命名句柄，并不会顺利插入。
>
> 感觉这个函数挖了坑，说不定以后会有错误的使用者出现。

再来找找操纵者，除了test测试代码，仅在这里找到：

```cpp
ResultCode PolicyBase::AddKernelObjectToClose(const base::char16* handle_type,
                                              const base::char16* handle_name) {
  return handle_closer_.AddHandle(handle_type, handle_name);
}
```

再次查找`AddKernelObjectToClose`的caller，除了test，只有`PolicyBase::SetDisconnectCsrss`中有身影：

```cpp
ResultCode PolicyBase::SetDisconnectCsrss() {
// Does not work on 32-bit, and the ASAN runtime falls over with the
// CreateThread EAT patch used when this is enabled.
// See https://crbug.com/783296#c27.
// 未开启ASAN的win10 x64所用，看起来和某个bug有关，以后有空看看
#if defined(_WIN64) && !defined(ADDRESS_SANITIZER)
  if (base::win::GetVersion() >= base::win::VERSION_WIN10) {
    is_csrss_connected_ = false;
    return AddKernelObjectToClose(L"ALPC Port", nullptr);//但这里是把ALPC Port都clear了
  }
#endif  // !defined(_WIN64)
  return SBOX_ALL_OK;
}
```

这里是个clear操作，显然是为了让win10 x64的target可以使用ALPC Port句柄。sandbox找不到其他的调用处，应该是启动器外部通过`PolicyBase`调用了`AddKernelObjectToClose`。

毕竟`PolicyBase`是传进来的，外部通过`BrokerServicesBase::CreatePolicy`创建policy，然后对policy一顿操作，然后`BrokerServicesBase::SpawnTarget`中传入了policy。

## `HandleCloserAgent`

Broker这边的流程都已了解了，下面我们找找target自身是如何关闭这些句柄的。在handle_closer_agent.h中找到了target用到的封装类。

```cpp
// Target process code to close the handle list copied over from the broker.
// 这个是target进程的代码，它关闭broker传过来的句柄列表
class HandleCloserAgent {
 public:
  HandleCloserAgent();
  ~HandleCloserAgent();

  // Reads the serialized list from the broker and creates the lookup map.
  // Updates is_csrss_connected based on type of handles closed.
  // 从broker读取序列化列表，创建一个lookup map
  // 根据关闭的句柄类型，更新is_csrss_connected
  void InitializeHandlesToClose(bool* is_csrss_connected);

  // Closes any handles matching those in the lookup map.
  // 关闭lookup map中匹配的句柄
  bool CloseHandles();

  // True if we have handles waiting to be closed.
  // 这是个static方法，用于判断当前是否在等待句柄被关闭
  static bool NeedsHandlesClosed();

 private:
  // Attempt to stuff a closed handle with a dummy Event.
  bool AttemptToStuffHandleSlot(HANDLE closed_handle,
                                const base::string16& type);

  HandleMap handles_to_close_;
  base::win::ScopedHandle dummy_handle_;

  DISALLOW_COPY_AND_ASSIGN(HandleCloserAgent);
};
```

### `InitializeHandlesToClose`

这个函数应该就是从传过来的g_handles_to_close读操作。

```cpp
// Reads g_handles_to_close and creates the lookup map.
void HandleCloserAgent::InitializeHandlesToClose(bool* is_csrss_connected) {
  CHECK(g_handles_to_close);
  //target进也定义了一个g_handles_to_close指针，但实际上是由broker进程来写入

  // Default to connected state
  // 默认情况是连接到运行时子系统的
  *is_csrss_connected = true;

  // Grab the header.
  // 开始了，对HandleCloserInfo开始parse
  HandleListEntry* entry = g_handles_to_close->handle_entries;
  for (size_t i = 0; i < g_handles_to_close->num_handle_types; ++i) {
    // Set the type name.
    base::char16* input = entry->handle_type;
    // 如果关闭的句柄有ALPC Port类型，就会断开与csrss的连接（csrss就是通过这个ALPC端口连接的）
    // 还记得前面探索AddTarget时发现的win10 x64特殊处理吗？
    if (!wcscmp(input, L"ALPC Port")) {
      *is_csrss_connected = false;
    }
    //handle_names是句柄名的set
    HandleMap::mapped_type& handle_names = handles_to_close_[input];
    input = reinterpret_cast<base::char16*>(reinterpret_cast<char*>(entry) +
                                            entry->offset_to_names);
    // Grab all the handle names.
    // 这就导入到成员handles_to_close_中了
    for (size_t j = 0; j < entry->name_count; ++j) {
      std::pair<HandleMap::mapped_type::iterator, bool> name =
          handle_names.insert(input);
      CHECK(name.second);
      input += name.first->size() + 1;
    }

    // Move on to the next entry.
    // 继续下一个
    entry = reinterpret_cast<HandleListEntry*>(reinterpret_cast<char*>(entry) +
                                               entry->record_bytes);

    DCHECK(reinterpret_cast<base::char16*>(entry) >= input);
    DCHECK(reinterpret_cast<base::char16*>(entry) - input <
           static_cast<ptrdiff_t>(sizeof(size_t) / sizeof(base::char16)));
  }

  // Clean up the memory we copied over.
  // 拷贝到handles_to_close_完毕，可以清了g_handles_to_close
  ::VirtualFree(g_handles_to_close, 0, MEM_RELEASE);
  g_handles_to_close = nullptr;
}
```

### `CloseHandles`

导入操作已经知晓了，此时`handles_to_close_`已装载。下一个最为重要的就是close操作了：

```cpp
bool HandleCloserAgent::CloseHandles() {
  DWORD handle_count = UINT_MAX;
  const int kInvalidHandleThreshold = 100;
  // 句柄都是4的倍数，可以参考wrk的句柄表设计，PID的设计也复用了句柄表设计，所以都是4的倍数
  const size_t kHandleOffset = 4;  // Handles are always a multiple of 4.

  if (!::GetProcessHandleCount(::GetCurrentProcess(), &handle_count))
    return false;

  // Set up buffers for the type info and the name.
  std::vector<BYTE> type_info_buffer(sizeof(OBJECT_TYPE_INFORMATION) +
                                     32 * sizeof(wchar_t));
  OBJECT_TYPE_INFORMATION* type_info =
      reinterpret_cast<OBJECT_TYPE_INFORMATION*>(&(type_info_buffer[0]));
  base::string16 handle_name;
  HANDLE handle = nullptr;
  int invalid_count = 0;

  // Keep incrementing until we hit the number of handles reported by
  // GetProcessHandleCount(). If we hit a very long sequence of invalid
  // handles we assume that we've run past the end of the table.
  // 以句柄号从下限值到上限值迭代，通过NtQueryObject来判断某个句柄值是否有效，然后再匹配
  while (handle_count && invalid_count < kInvalidHandleThreshold) {
    reinterpret_cast<size_t&>(handle) += kHandleOffset;
    NTSTATUS rc;

    // Get the type name, reusing the buffer.
    // 又见二次调用的操作，只不过做了二层封装
    ULONG size = static_cast<ULONG>(type_info_buffer.size());
    rc = QueryObjectTypeInformation(handle, type_info, &size);//封装了NtQueryObject的一些操作
    while (rc == STATUS_INFO_LENGTH_MISMATCH || rc == STATUS_BUFFER_OVERFLOW) {
      type_info_buffer.resize(size + sizeof(wchar_t));
      type_info =
          reinterpret_cast<OBJECT_TYPE_INFORMATION*>(&(type_info_buffer[0]));
      rc = QueryObjectTypeInformation(handle, type_info, &size);
      // Leave padding for the nul terminator.
      if (NT_SUCCESS(rc) && size == type_info_buffer.size())
        rc = STATUS_INFO_LENGTH_MISMATCH;
    }
    // 如果这个句柄号是有效的，那么就说明确实有这个句柄，没有的话就不浪费感情lookup匹配了
    if (!NT_SUCCESS(rc) || !type_info->Name.Buffer) {
      ++invalid_count;
      continue;
    }

    --handle_count;
    type_info->Name.Buffer[type_info->Name.Length / sizeof(wchar_t)] = L'\0';
    // 这段是查找传过来的handles_to_close_，看看是否需要关闭该句柄
    // 如果需要关闭，那就CloseHandle
    // Check if we're looking for this type of handle.
    HandleMap::iterator result = handles_to_close_.find(type_info->Name.Buffer);
    if (result != handles_to_close_.end()) {
      HandleMap::mapped_type& names = result->second;
      // Empty set means close all handles of this type; otherwise check name.
      if (!names.empty()) {
        // Move on to the next handle if this name doesn't match.
        // 这个GetHandleName在handle_closer.cc中实现，实际上依然是NtQueryObject的封装
        if (!GetHandleName(handle, &handle_name) || !names.count(handle_name))
          continue;
      }

      if (!::SetHandleInformation(handle, HANDLE_FLAG_PROTECT_FROM_CLOSE, 0))
        return false;
      if (!::CloseHandle(handle))
        return false;
      // Attempt to stuff this handle with a new dummy Event.
      // 尝试填满Handle槽？不懂什么意思，传入了handle和type
      AttemptToStuffHandleSlot(handle, result->first);
    }
  }

  return true;
}

// Returns the object manager's name associated with a handle
// 通过handle反查name，实际上用NtQueryObject这个万用API可以做到，查询的class为
// ObjectNameInformation
// NtQueryObject很强大，根据传入的OBJECT_INFORMATION_CLASS值可以查询不同的信息
bool GetHandleName(HANDLE handle, base::string16* handle_name) {
  static NtQueryObject QueryObject = nullptr;
  if (!QueryObject)
    ResolveNTFunctionPtr("NtQueryObject", &QueryObject);

  ULONG size = MAX_PATH;
  std::unique_ptr<UNICODE_STRING, base::FreeDeleter> name;
  NTSTATUS result;

  do {
    name.reset(static_cast<UNICODE_STRING*>(malloc(size)));
    DCHECK(name.get());
    result =
        QueryObject(handle, ObjectNameInformation, name.get(), size, &size);
  } while (result == STATUS_INFO_LENGTH_MISMATCH ||
           result == STATUS_BUFFER_OVERFLOW);

  if (NT_SUCCESS(result) && name->Buffer && name->Length)
    handle_name->assign(name->Buffer, name->Length / sizeof(wchar_t));
  else
    handle_name->clear();

  return NT_SUCCESS(result);
}
```

另一个`QueryObjectTypeInformation`也类似：

```cpp
// Returns type infomation for an NT object. This routine is expected to be
// called for invalid handles so it catches STATUS_INVALID_HANDLE exceptions
// that can be generated when handle tracing is enabled.
// 返回的是handle对应的type而非name
NTSTATUS QueryObjectTypeInformation(HANDLE handle, void* buffer, ULONG* size) {
  static NtQueryObject QueryObject = nullptr;
  // 还是用NtQueryObject
  if (!QueryObject)
    ResolveNTFunctionPtr("NtQueryObject", &QueryObject);

  NTSTATUS status = STATUS_UNSUCCESSFUL;
  __try {
    // 这一次用的class是ObjectTypeInformation
    status = QueryObject(handle, ObjectTypeInformation, buffer, *size, size);
  } __except (GetExceptionCode() == STATUS_INVALID_HANDLE
                  ? EXCEPTION_EXECUTE_HANDLER
                  : EXCEPTION_CONTINUE_SEARCH) {
    status = STATUS_INVALID_HANDLE;
  }
  return status;
}
```

`CloseHandles`最后调用了`AttemptToStuffHandleSlot`：

```cpp
// Attempts to stuff |closed_handle| with a duplicated handle for a dummy Event
// with no access. This should allow the handle to be closed, to avoid
// generating EXCEPTION_INVALID_HANDLE on shutdown, but nothing else. For now
// the only supported |type| is Event or File.
// 关闭句柄时防止产生EXCEPTION_INVALID_HANDLE，仅仅支持Event或File类型句柄
// 这东西用途我还不理解，看起来是对于Event和File句柄做的额外操作，复制一个再CloseHandle
// 不懂有什么用
bool HandleCloserAgent::AttemptToStuffHandleSlot(HANDLE closed_handle,
                                                 const base::string16& type) {
  // Only attempt to stuff Files and Events at the moment.
  if (type != L"Event" && type != L"File") {
    return true;
  }

  if (!dummy_handle_.IsValid())
    return false;

  // This should never happen, as g_dummy is created before closing to_stuff.
  DCHECK(dummy_handle_.Get() != closed_handle);

  std::vector<HANDLE> to_close;
  HANDLE dup_dummy = nullptr;
  size_t count = 16;

  do {
    if (!::DuplicateHandle(::GetCurrentProcess(), dummy_handle_.Get(),
                           ::GetCurrentProcess(), &dup_dummy, 0, false, 0))
      break;
    if (dup_dummy != closed_handle)
      to_close.push_back(dup_dummy);
  } while (count-- && reinterpret_cast<uintptr_t>(dup_dummy) <
                          reinterpret_cast<uintptr_t>(closed_handle));

  for (HANDLE h : to_close)
    ::CloseHandle(h);

  // TODO(wfh): Investigate why stuffing handles sometimes fails.
  // http://crbug.com/649904
  return dup_dummy == closed_handle;
}
```

这里主要是不懂dummy_handle_成员的意义，不知道为什么要频繁尝试复制，且再次Close掉。这里面应该涉及到了某种技术，而这个函数只是抛开基础架设的一个应用。

如果以后我对它有了正确的理解，再来补充吧，留个坑。

下面感兴趣的就是target进程如何操纵。

## `TargetServicesBase` related

`TargetServicesBase`是给target进程提供的服务面。在`TargetServicesBase::LowerToken`中找到了这样一个操作：

```cpp
// Failure here is a breach of security so the process is terminated.
void TargetServicesBase::LowerToken() {
  if (ERROR_SUCCESS !=
      SetProcessIntegrityLevel(g_shared_delayed_integrity_level))
    ::TerminateProcess(::GetCurrentProcess(), SBOX_FATAL_INTEGRITY);
  process_state_.SetRevertedToSelf();
  // If the client code as called RegOpenKey, advapi32.dll has cached some
  // handles. The following code gets rid of them.
  // 最不起眼的其实是最关键的call，不过本次我们就不理它了
  if (!::RevertToSelf())
    ::TerminateProcess(::GetCurrentProcess(), SBOX_FATAL_DROPTOKEN);
  if (!FlushCachedRegHandles())
    ::TerminateProcess(::GetCurrentProcess(), SBOX_FATAL_FLUSHANDLES);
  if (ERROR_SUCCESS != ::RegDisablePredefinedCache())
    ::TerminateProcess(::GetCurrentProcess(), SBOX_FATAL_CACHEDISABLE);
  if (!WarmupWindowsLocales())
    ::TerminateProcess(::GetCurrentProcess(), SBOX_FATAL_WARMUP);
  bool is_csrss_connected = true;
  // 就是它，关闭了句柄，并根据ALPC port是否被关闭来设置csrss的连接状态
  if (!CloseOpenHandles(&is_csrss_connected))
    ::TerminateProcess(::GetCurrentProcess(), SBOX_FATAL_CLOSEHANDLES);
  process_state_.SetCsrssConnected(is_csrss_connected);
  // Enabling mitigations must happen last otherwise handle closing breaks
  if (g_shared_delayed_mitigations &&
      !ApplyProcessMitigationsToCurrentProcess(g_shared_delayed_mitigations))
    ::TerminateProcess(::GetCurrentProcess(), SBOX_FATAL_MITIGATION);
}
```

 这里还可以看到`g_shared_delayed_integrity_level`和`g_shared_delayed_mitigations`的使用。

展开`CloseOpenHandles`:

```cpp
// Checks if we have handle entries pending and runs the closer.
// Updates is_csrss_connected based on which handle types are closed.
bool CloseOpenHandles(bool* is_csrss_connected) {
  if (HandleCloserAgent::NeedsHandlesClosed()) {//其实就是!!g_handles_to_close，转bool值判断
    // 此时broker已经写好了g_handles_to_close，开始吧
    HandleCloserAgent handle_closer;
    // 第一步，InitializeHandlesToClose
    handle_closer.InitializeHandlesToClose(is_csrss_connected);
    if (!*is_csrss_connected) {
      if (!CsrssDisconnectCleanup()) {
        return false;
      }
    }
    // 第二步，CloseHandles
    if (!handle_closer.CloseHandles())
      return false;
  }
  return true;
}
```

到此，关于`HandleCloser`和`HandleCloserAgent`的分析就结束了，相对简单的一部分。