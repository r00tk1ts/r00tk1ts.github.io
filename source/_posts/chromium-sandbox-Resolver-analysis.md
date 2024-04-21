---
title: Chromium-sandbox-Resolver-analysis
date: 2018-05-26 10:27:11
categories: 源码剖析
tags:
	- chromium
	- chromium-sandbox
---

本篇是sandbox源码剖析的第十一篇，承接了上一篇Interception的分析。Resolver是负责操纵Interceptions的相关模块。阅读本篇前，请先阅读前四篇及第十篇。

想要流程的阅读本系列你需要以下几个条件：
1. 较高水平的C++编码能力（至少通读C++ Primer 5th，刷过课后题，有一定编码量）。
2. 熟悉Windows API编程，尤其是安全相关的内容。
3. 对二进制安全有一定了解，熟悉各类型安全漏洞成因、漏洞缓解措施及bypass手法。

<!--more-->

# chromium-sandbox-Resolver-analysis

本节分析负责执行Interceptions的相关类。

不跟你多BB，直接从上一节的疑问看起。也就是`ResolverThunk`类。

## `ResolverThunk`

```cpp
// A resolver is the object in charge of performing the actual interception of
// a function. There should be a concrete implementation of a resolver roughly
// per type of interception.
// resolver负责处理某个函数具体的拦截，每种类型的interception都应该有一个具体的resolver对象
// 这也就是上一节看到的，resolver对应每个的3个类型都有着一个派生类
class ResolverThunk {
 public:
  ResolverThunk() {}
  virtual ~ResolverThunk() {}

  // Performs the actual interception of a function.
  // target_name is an exported function from the module loaded at
  // target_module, and must be replaced by interceptor_name, exported from
  // interceptor_module. interceptor_entry_point can be provided instead of
  // interceptor_name / interceptor_module.
  // thunk_storage must point to a buffer on the child's address space, to hold
  // the patch thunk, and related data. If provided, storage_used will receive
  // the number of bytes used from thunk_storage.
  //
  // Example: (without error checking)
  //
  // size_t size = resolver.GetThunkSize();
  // char* buffer = ::VirtualAllocEx(child_process, nullptr, size,
  //                                 MEM_COMMIT, PAGE_READWRITE);
  // resolver.Setup(ntdll_module, nullptr, L"NtCreateFile", nullptr,
  //                &MyReplacementFunction, buffer, size, nullptr);
  //
  // In general, the idea is to allocate a single big buffer for all
  // interceptions on the same dll, and call Setup n times.
  // WARNING: This means that any data member that is specific to a single
  // interception must be reset within this method.
  virtual NTSTATUS Setup(const void* target_module,//函数所在dll
                         const void* interceptor_module,//hook函数所在dll
                         const char* target_name,//original函数名称
                         const char* interceptor_name,//hook函数名称
                         const void* interceptor_entry_point,//hook入口地址
                         void* thunk_storage,//存储thunk的buffer
                         size_t storage_bytes,//buffer的大小
                         size_t* storage_used) = 0;

  // 下面两个函数用于确定hook和original函数的地址
  // Gets the address of function_name inside module (main exe).
  virtual NTSTATUS ResolveInterceptor(const void* module,
                                      const char* function_name,
                                      const void** address);

  // Gets the address of an exported function_name inside module.
  virtual NTSTATUS ResolveTarget(const void* module,
                                 const char* function_name,
                                 void** address);

  // Gets the required buffer size for this type of thunk.
  // 计算该thunk需要的buffer大小
  virtual size_t GetThunkSize() const = 0;

 protected:
  // Performs basic initialization on behalf of a concrete instance of a
  // resolver. That is, parameter validation and resolution of the target
  // and the interceptor into the member variables.
  //
  // target_name is an exported function from the module loaded at
  // target_module, and must be replaced by interceptor_name, exported from
  // interceptor_module. interceptor_entry_point can be provided instead of
  // interceptor_name / interceptor_module.
  // thunk_storage must point to a buffer on the child's address space, to hold
  // the patch thunk, and related data.
  // 常规套路init
  virtual NTSTATUS Init(const void* target_module,
                        const void* interceptor_module,
                        const char* target_name,
                        const char* interceptor_name,
                        const void* interceptor_entry_point,
                        void* thunk_storage,
                        size_t storage_bytes);

  // Gets the required buffer size for the internal part of the thunk.
  size_t GetInternalThunkSize() const;

  // Initializes the internal part of the thunk.
  // interceptor is the function to be called instead of original_function.
  // 调用此接口来用interceptor替换original的调用
  bool SetInternalThunk(void* storage, size_t storage_bytes,
                        const void* original_function, const void* interceptor);

  // Holds the resolved interception target.
  void* target_;	//这个对应original function地址
  // Holds the resolved interception interceptor.
  const void* interceptor_;	//对应hook函数地址

  DISALLOW_COPY_AND_ASSIGN(ResolverThunk);
};
```

### `Init`

```cpp
NTSTATUS ResolverThunk::Init(const void* target_module,
                             const void* interceptor_module,
                             const char* target_name,
                             const char* interceptor_name,
                             const void* interceptor_entry_point,
                             void* thunk_storage,
                             size_t storage_bytes) {
  // 传入的这些值必须得有效
  if (!thunk_storage || 0 == storage_bytes || !target_module || !target_name)
    return STATUS_INVALID_PARAMETER;

  if (storage_bytes < GetThunkSize())
    return STATUS_BUFFER_TOO_SMALL;

  NTSTATUS ret = STATUS_SUCCESS;
  // InterceptionManager中看到的两个Add方法，其中一个写入的是interceptor_entry_point，没有interceptor_name。
  // 这种情况刚刚好，我们要的就是地址（毕竟不是非要用已知的函数来hook另一个函数，大可以diy）
  // 而另一个Add传入的是interceptor_name，此时也必然有interceptor_module。
  // 这时用interceptor_module和interceptor_name解析出interceptor_entry_point
  if (!interceptor_entry_point) {
    ret = ResolveInterceptor(interceptor_module, interceptor_name,
                             &interceptor_entry_point);
    if (!NT_SUCCESS(ret))
      return ret;
  }

  // 解析出original函数的地址
  // original函数地址给到target_成员
  // interceptor地址给到interceptor_成员
  ret = ResolveTarget(target_module, target_name, &target_);
  if (!NT_SUCCESS(ret))
    return ret;

  interceptor_ = interceptor_entry_point;

  return ret;
}
```

展开看看：

```cpp
NTSTATUS ResolverThunk::ResolveInterceptor(const void* interceptor_module,
                                           const char* interceptor_name,
                                           const void** address) {
  DCHECK_NT(address);
  if (!interceptor_module)
    return STATUS_INVALID_PARAMETER;

  // 基础代码中对PE文件结构的封装
  base::win::PEImage pe(interceptor_module);
  // 验证是否是有效PE
  if (!pe.VerifyMagic())
    return STATUS_INVALID_IMAGE_FORMAT;

  // 获取interceptor_name的地址
  *address = reinterpret_cast<void*>(pe.GetProcAddress(interceptor_name));

  if (!(*address))
    return STATUS_PROCEDURE_NOT_FOUND;

  return STATUS_SUCCESS;
}

NTSTATUS ResolverThunk::ResolveTarget(const void* module,
                                      const char* function_name,
                                      void** address) {
  const void** casted = const_cast<const void**>(address);
  return ResolverThunk::ResolveInterceptor(module, function_name, casted);
}

```

可以看到基类的两个Resolve方法实际上都是`PEImage`中`GetProcAddress`。

### x86: `InternalThunk`

再继续分析前，我们需要看一些基础设施，防止后续分析时一脸懵逼。

在resolver_32.cc中有一个非常漂亮的内联汇编payload：

```cpp
#pragma pack(push, 1)
struct InternalThunk {
  // This struct contains roughly the following code:
  // sub esp, 8                             // Create working space
  // push edx                               // Save register
  // mov edx, [esp + 0xc]                   // Get return adddress
  // mov [esp + 8], edx                     // Store return address
  // mov dword ptr [esp + 0xc], 0x7c401200  // Store extra argument
  // mov dword ptr [esp + 4], 0x40010203    // Store address to jump to
  // pop edx                                // Restore register
  // ret                                    // Jump to interceptor
  //
  // This code only modifies esp and eip so it must work with to normal calling
  // convention. It is assembled as:
  //
  // 00 83ec08           sub     esp,8
  // 03 52               push    edx
  // 04 8b54240c         mov     edx,dword ptr [esp + 0Ch]
  // 08 89542408         mov     dword ptr [esp + 8], edx
  // 0c c744240c0012407c mov     dword ptr [esp + 0Ch], 7C401200h
  // 14 c744240403020140 mov     dword ptr [esp + 4], 40010203h
  // 1c 5a               pop     edx
  // 1d c3               ret
  InternalThunk() {
    opcodes_1 = 0x5208ec83;
    opcodes_2 = 0x0c24548b;
    opcodes_3 = 0x08245489;
    opcodes_4 = 0x0c2444c7; 
    opcodes_5 = 0x042444c7;
    opcodes_6 = 0xc35a;
    extra_argument = 0;
    interceptor_function = 0;
  };
  ULONG opcodes_1;  // = 0x5208ec83
  ULONG opcodes_2;  // = 0x0c24548b
  ULONG opcodes_3;  // = 0x08245489
  ULONG opcodes_4;  // = 0x0c2444c7
  ULONG extra_argument;
  ULONG opcodes_5;  // = 0x042444c7
  ULONG interceptor_function;
  USHORT opcodes_6;  // = 0xc35a
};
#pragma pack(pop)
```

关于它的作用，一图胜千言：

![](/images/internal_thunk.png)

通过这一thunk，就实现了先调用interceptor，interceptor返回后再回到ret addr，interceptor调用期间，多了一个参数，这个参数实际上就是original function。

`extra_argument`和`interceptor_function`是外部传入的，在构造`InternalThunk`对象时，填充即可。此时`InternalThunk`的payload就正确了。典型的把Data看成Code的范例。

`InternalThunk`对象在`SetInternalThunk`中使用：

```cpp
bool ResolverThunk::SetInternalThunk(void* storage,
                                     size_t storage_bytes,
                                     const void* original_function,
                                     const void* interceptor) {
  // 存储thunk的空间是否放得下InternalThunk
  if (storage_bytes < sizeof(InternalThunk))
    return false;

  // 在thunk buffer上部署InternalThunk
  InternalThunk* thunk = new (storage) InternalThunk;

#pragma warning(push)
#pragma warning(disable : 4311)
  // These casts generate warnings because they are 32 bit specific.
  // 这两个一填充，就达成了interceptor(original_function, xxx) -> ret addr的效果
  // 那么关键的就在于interceptor内部要如何处理这个extra参数了
  thunk->interceptor_function = reinterpret_cast<ULONG>(interceptor);
  thunk->extra_argument = reinterpret_cast<ULONG>(original_function);
#pragma warning(pop)

  return true;
}
```

其他接口：

```cpp
size_t ResolverThunk::GetInternalThunkSize() const {
  return sizeof(InternalThunk);
}
```

### x64: `InternalThunk`

x64因为全盘使用的`g_originals`而非将original function地址作为hook的第一个参数，所以要简单得多。只需要把interceptor地址传入即可，thunk所做的仅仅是jmp到该地址，至于interceptor function如何处理original function，那就是interceptor的事儿了。

这里的thunk没有动调用栈帧。

```cpp
struct InternalThunk {
  // This struct contains roughly the following code:
  // 01 48b8f0debc9a78563412  mov   rax,123456789ABCDEF0h
  // ff e0                    jmp   rax
  //
  // The code modifies rax, but that's fine for x64 ABI.

  InternalThunk() {
    mov_rax = kMovRax;
    jmp_rax = kJmpRax;
    interceptor_function = 0;
  };
  USHORT mov_rax;  // = 48 B8
  ULONG_PTR interceptor_function;
  USHORT jmp_rax;  // = ff e0
};
```

与x86接口一致：

```cpp
bool ResolverThunk::SetInternalThunk(void* storage,
                                     size_t storage_bytes,
                                     const void* original_function,	// 没有用到
                                     const void* interceptor) {
  if (storage_bytes < sizeof(InternalThunk))
    return false;

  // 在thunk buffer处部署了payload，关联interceptor
  InternalThunk* thunk = new (storage) InternalThunk;
  thunk->interceptor_function = reinterpret_cast<ULONG_PTR>(interceptor);

  return true;
}

size_t ResolverThunk::GetInternalThunkSize() const {
  return sizeof(InternalThunk);
}
```

有趣的是x64的`ResolveTarget`:

```cpp
NTSTATUS ResolverThunk::ResolveTarget(const void* module,
                                      const char* function_name,
                                      void** address) {
  // We don't support sidestep & co.
  // 看起来基类的这个函数会被sidestep类型的interception用到，而x64不支持这种类型，所以也就不实现
  // 那么其他类型呢，x64总有Eat类型吧。可能Eat类型有自己的ResolveTarget override。
  return STATUS_NOT_IMPLEMENTED;
}
```

到此，基类相关的内容都已经过了一遍，想要把Interception和Resolver挂钩，还需要逐个分析各种类型的派生类。

## `EatResolverThunk`

此前我们在`InterceptionAgent::PatchDll`中看到了resolver的使用，根据前面的分析，我们知道`PatchDll`是在target进程加载感兴趣dll的时候执行的，执行期间，会根据broker传过来的`DllPatchInfo`，对该dll的所有感兴趣的function进行处理，而此时借助的就是对应类型的resolver来处理，通过Setup方法传入了相关的`FunctionInfo`信息和thunk data在dll上的buffer（实际上就是`base_address`）。

待加载dll的`base_address`处布置了一个`DllInterceptionData`对象，它内部的`ThunkData`数组即对应所有感兴趣的函数的thunk data，而data的实装应该是由`xxxResolver::Setup`来完成的。

我们暂且假定target要加载一个感兴趣的dll，这个dll中的感兴趣的函数既有eat类型，也有sidestep类型（target端不会有service call类型，这个在broker中处理）。

我们先分析Eat类型的执行流程。

```cpp
// This is the concrete resolver used to perform exports table interceptions.
// Eat类型是指对导出表函数的拦截？看来此前的猜想有误。
class EatResolverThunk : public ResolverThunk {
 public:
  EatResolverThunk() : eat_entry_(nullptr) {}
  ~EatResolverThunk() override {}

  // Implementation of Resolver::Setup.
  // 纯虚函数的override，父类显然不清楚子类是如何把控thunk和相关info的
  NTSTATUS Setup(const void* target_module,
                 const void* interceptor_module,
                 const char* target_name,
                 const char* interceptor_name,
                 const void* interceptor_entry_point,
                 void* thunk_storage,
                 size_t storage_bytes,
                 size_t* storage_used) override;

  // Implementation of Resolver::ResolveTarget.
  // eat类型需要这个函数，这里还要override父类函数
  // 我们已经知道了父类的这个函数实际上是间接调用ResolveInterceptor
  NTSTATUS ResolveTarget(const void* module,
                         const char* function_name,
                         void** address) override;

  // Implementation of Resolver::GetThunkSize.
  // 父类的这个函数是纯虚函数，这说明不同类型的interception，其thunk尺寸的计算各不相同
  // 父类本身是不可能知道派生类对thunk的设计需求的，同Setup
  size_t GetThunkSize() const override;

 private:
  // The entry to patch.
  DWORD* eat_entry_;	// patch入口

  DISALLOW_COPY_AND_ASSIGN(EatResolverThunk);
};
```

`eat_entry_`是唯一的一个扩展成员，我们在下面要盯住它的使用。

### `Setup`

看看最为关键的Setup：

```cpp
NTSTATUS EatResolverThunk::Setup(const void* target_module,
                                 const void* interceptor_module,
                                 const char* target_name,
                                 const char* interceptor_name,
                                 const void* interceptor_entry_point,
                                 void* thunk_storage,
                                 size_t storage_bytes,
                                 size_t* storage_used) {
  // 先进行Init，调用父类的方法
  // 回忆一下父类的流程，先是判断ThunkData这个64字节的坑够不够放下thunk data
  // 对于Eat类型来说，它所需要的thunk size，对于x86就是GetInternalThunkSize()
  // x64要乘2（尽管乘2也比x86小），为什么需要两次jmp呢？我们继续往下看。
  // 而InternalThunk我们已经很清楚了，就是那段payload，它作为eat类型的thunk data
  // 在Init内部，对Eat类型来说，ThunkData[i]这个坑位显然放得下，于是Init会继续解析出
  // Interceptor的地址（如果传入的是名称的话）和Target的地址
  // Interceptor没什么好说的，Eat类型的ResolveTarget做了override，使用的是导出表的那套机制
  NTSTATUS ret =
      Init(target_module, interceptor_module, target_name, interceptor_name,
           interceptor_entry_point, thunk_storage, storage_bytes);
  if (!NT_SUCCESS(ret))
    return ret;

  if (!eat_entry_)
    return STATUS_INVALID_PARAMETER;

#if defined(_WIN64)
  // We have two thunks, in order: the return path and the forward path.
  // x64有两个thunk，一来一回
  // x64用不到第三个参数，在x86的设计中它表示original地址
  // 这里在ThunkData[i]处先放置了第一个跳转到original地址的thunk data
  if (!SetInternalThunk(thunk_storage, storage_bytes, nullptr, target_))
    return STATUS_BUFFER_TOO_SMALL;

  // thunk_storage向后挪动，指向需要部署第二个InternalThunk的位置
  size_t thunk_bytes = GetInternalThunkSize();
  storage_bytes -= thunk_bytes;
  thunk_storage = reinterpret_cast<char*>(thunk_storage) + thunk_bytes;
#endif

  // 对x86来说，这里实现了interceptor_(target_, xxx)替换target(xxx)的效果
  // 当外部call ThunkData[i]位置的时候，通过执行InternalThunk的指令，最后会调用interceptor_(target_, xxx)
  // 当然这种机制依赖两个外部条件：
  // 1. interceptor要外部提供，利用上origin参数
  // 2. 在调用一个函数时，不直接call target而是call ThunkData[i]
  //
  // 对x64来说，这里相当于第二次设置，第三个参数是没意义的，仅仅是把jmp interceptor_放在了
  // 第一个InternalThunk(jmp original)的正后方
  // 这样看来x64的thunk比较奇怪，应该是call ThunkData[i]+sizeof(InternalThunk)去调用
  // interceptor，interceptor内部再去call ThunkData[i]来调用original
  // 
  if (!SetInternalThunk(thunk_storage, storage_bytes, target_, interceptor_))
    return STATUS_BUFFER_TOO_SMALL;

  AutoProtectMemory memory;
  ret = memory.ChangeProtection(eat_entry_, sizeof(DWORD), PAGE_READWRITE);
  if (!NT_SUCCESS(ret))
    return ret;

  // Perform the patch.
  // 看到这里就明白了为什么x64要把jmp original放在前面
  // 无论是x64还是x86，此时eat_entry是original函数的导出表地址，我们只需要把这个导出表地址
  // 替换成Interceptor的地址就行了。
  // 这里就解决了上面的一个外部条件的达成：调用dll导出表函数时，此后不会call target，而是
  // call ThunkData[i]，此外之所以要减去基地址是因为导出表存的是RVA
  *eat_entry_ = static_cast<DWORD>(reinterpret_cast<uintptr_t>(thunk_storage)) -
                static_cast<DWORD>(reinterpret_cast<uintptr_t>(target_module));

  // OUT型参数回置使用的thunk data大小
  if (storage_used)
    *storage_used = GetThunkSize();

  return ret;
}
```

### `ResolveTarget`

```cpp
NTSTATUS EatResolverThunk::ResolveTarget(const void* module,
                                         const char* function_name,
                                         void** address) {
  DCHECK_NT(address);
  if (!module)
    return STATUS_INVALID_PARAMETER;

  base::win::PEImage pe(module);
  if (!pe.VerifyMagic())
    return STATUS_INVALID_IMAGE_FORMAT;

  // 直接从导出表拿了，所以eat_entry是导出表中的地址，是个RVA
  // 而该函数是在Init时调用的，这说明eat_entry_已经有值了
  eat_entry_ = pe.GetExportEntry(function_name);

  if (!eat_entry_)
    return STATUS_PROCEDURE_NOT_FOUND;

  // RVA到addr，给回address，实际上address就是original函数地址
  *address = pe.RVAToAddr(*eat_entry_);

  return STATUS_SUCCESS;
}
```

**所以，我们此前对Eat类型的猜想是错误的。Eat类型实际上是用于处理Dll导出表函数的resolver，它的做法是用`InternalThunk`来部署`interceptor`，并且把它的地址RVA替换导出表的函数地址，以实现interception。**

## `SidestepResolverThunk`

在target中，x86还会用到两种Resolver，分别是`SidestepResolverThunk`和`SmartSidestepResolverThunk`。

先看一下头：

```cpp
// This is the concrete resolver used to perform sidestep interceptions.
class SidestepResolverThunk : public ResolverThunk {
 public:
  SidestepResolverThunk() {}
  ~SidestepResolverThunk() override {}

  // Setup和GetThunkSize作为两大核心接口，是子类必须override的纯虚函数
  // Implementation of Resolver::Setup.
  NTSTATUS Setup(const void* target_module,
                 const void* interceptor_module,
                 const char* target_name,
                 const char* interceptor_name,
                 const void* interceptor_entry_point,
                 void* thunk_storage,
                 size_t storage_bytes,
                 size_t* storage_used) override;

  // Implementation of Resolver::GetThunkSize.
  size_t GetThunkSize() const override;

 private:
  DISALLOW_COPY_AND_ASSIGN(SidestepResolverThunk);
};
```

### `GetThunkSize`

在分析之前，我们还是得先看看sidestep使用的Thunk是什么。

```cpp
size_t SidestepResolverThunk::GetThunkSize() const {
  // 除了InternalThunk以外，还有一个额外的kSizeOfSidestepStub，这个值是32
  return GetInternalThunkSize() + kSizeOfSidestepStub;
}
```

那么额外的32个字节是用来干什么的呢？根据注释：

```cpp
// Maximum size of the preamble stub. We overwrite at least the first 5
// bytes of the function. Considering the worst case scenario, we need 4
// bytes + the max instruction size + 5 more bytes for our jump back to
// the original code. With that in mind, 32 is a good number :)
// 所以看起来sidestep是inline hook，它会覆盖original头至少5个字节。
// 最坏的情况下，我们需要4个字节 + 最大指令长度 + 用于跳回到original代码的5个额外字节
// 为了对齐就用了32.
const size_t kMaxPreambleStubSize = 32;
```

### `Setup`

```cpp
NTSTATUS SidestepResolverThunk::Setup(const void* target_module,
                                      const void* interceptor_module,
                                      const char* target_name,
                                      const char* interceptor_name,
                                      const void* interceptor_entry_point,
                                      void* thunk_storage,
                                      size_t storage_bytes,
                                      size_t* storage_used) {
  // 常规init，注意sidestep类型没有自己override ResolveTarget，使用的是父类的函数
  // 而这个函数和ResolveInterception是一样的，通过GetProcAddress取得
  NTSTATUS ret =
      Init(target_module, interceptor_module, target_name, interceptor_name,
           interceptor_entry_point, thunk_storage, storage_bytes);
  if (!NT_SUCCESS(ret))
    return ret;

  // sidestep类型的resolver，把传入的ThunkData[i]存储空间看成SidestepThunk
  // 为什么要设计个类型呢？从GetThunkSize中我们就可以看到，sidestep的ThunkData由两部分构成
  // 前32个字节是inline stub，ThunkData[i]的32个字节之后存储InternalThunk
  /*
  	struct SidestepThunk {
      char sidestep[kSizeOfSidestepStub];  // Storage for the sidestep stub.
      int internal_thunk;  // Dummy member to the beginning of the internal thunk.
    };
  */
  SidestepThunk* thunk = reinterpret_cast<SidestepThunk*>(thunk_storage);

  size_t internal_bytes = storage_bytes - kSizeOfSidestepStub;
  // 别担心，SetInternalThunk内部会检查剩余的空间够不够存放InternalThunk，这里64-32=32是够放的
  // 调用后，32个字节之后的InternalThunk就安排好了
  if (!SetInternalThunk(&thunk->internal_thunk, internal_bytes, thunk_storage,
                        interceptor_))
    return STATUS_BUFFER_TOO_SMALL;

  AutoProtectMemory memory;
  ret = memory.ChangeProtection(target_, kSizeOfSidestepStub, PAGE_READWRITE);
  if (!NT_SUCCESS(ret))
    return ret;

  // 前面32个字节的处理借助了PreamblePatcher这个对象的Patch方法
  // 我们下面展开来看他的用途
  sidestep::SideStepError rv = sidestep::PreamblePatcher::Patch(
      target_, reinterpret_cast<void*>(&thunk->internal_thunk), thunk_storage,
      kSizeOfSidestepStub);

  if (sidestep::SIDESTEP_INSUFFICIENT_BUFFER == rv)
    return STATUS_BUFFER_TOO_SMALL;

  if (sidestep::SIDESTEP_SUCCESS != rv)
    return STATUS_UNSUCCESSFUL;

  if (storage_used)
    *storage_used = GetThunkSize();

  return ret;
}
```

### `PreamblePatcher`

命名空间是`sidestep`，很明显是供sidestep resolver自身使用的helper接口。先看一下头部定义：

```cpp
// Implements a patching mechanism that overwrites the first few bytes of
// a function preamble with a jump to our hook function, which is then
// able to call the original function via a specially-made preamble-stub
// that imitates the action of the original preamble.
// 
// inline hook的局限性与危险性说明：
// Note that there are a number of ways that this method of patching can
// fail.  The most common are:
//    - If there is a jump (jxx) instruction in the first 5 bytes of
//    the function being patched, we cannot patch it because in the
//    current implementation we do not know how to rewrite relative
//    jumps after relocating them to the preamble-stub.  Note that
//    if you really really need to patch a function like this, it
//    would be possible to add this functionality (but at some cost).
//    - If there is a return (ret) instruction in the first 5 bytes
//    we cannot patch the function because it may not be long enough
//    for the jmp instruction we use to inject our patch.
//    - If there is another thread currently executing within the bytes
//    that are copied to the preamble stub, it will crash in an undefined
//    way.
//
// If you get any other error than the above, you're either pointing the
// patcher at an invalid instruction (e.g. into the middle of a multi-
// byte instruction, or not at memory containing executable instructions)
// or, there may be a bug in the disassembler we use to find
// instruction boundaries.
class PreamblePatcher {
 public:
  // Patches target_function to point to replacement_function using a provided
  // preamble_stub of stub_size bytes.
  // Returns An error code indicating the result of patching.
  // 函数模板，其实target_function和replacement_function都设void *应该就可以
  // 我没有get到为什么要设一个模板在这里，为了便于扩展？
  template <class T>
  static SideStepError Patch(T target_function,
                             T replacement_function,
                             void* preamble_stub,
                             size_t stub_size) {
    // 而且这个函数不是模板，是明确的参数void *
    return RawPatchWithStub(target_function, replacement_function,
                            reinterpret_cast<unsigned char*>(preamble_stub),
                            stub_size, nullptr);
  }

 private:
  // Patches a function by overwriting its first few bytes with
  // a jump to a different function.  This is similar to the RawPatch
  // function except that it uses the stub allocated by the caller
  // instead of allocating it.
  //
  // To use this function, you first have to call VirtualProtect to make the
  // target function writable at least for the duration of the call.
  //
  // target_function: A pointer to the function that should be
  // patched.
  //
  // replacement_function: A pointer to the function that should
  // replace the target function.  The replacement function must have
  // exactly the same calling convention and parameters as the original
  // function.
  //
  // preamble_stub: A pointer to a buffer where the preamble stub
  // should be copied. The size of the buffer should be sufficient to
  // hold the preamble bytes.
  //
  // stub_size: Size in bytes of the buffer allocated for the
  // preamble_stub
  //
  // bytes_needed: Pointer to a variable that receives the minimum
  // number of bytes required for the stub.  Can be set to nullptr if you're
  // not interested.
  //
  // Returns An error code indicating the result of patching.
  static SideStepError RawPatchWithStub(void* target_function,
                                        void* replacement_function,
                                        unsigned char* preamble_stub,
                                        size_t stub_size,
                                        size_t* bytes_needed);
};
```

展开看看`RawPatchWithStub`：

```cpp
SideStepError PreamblePatcher::RawPatchWithStub(
    void* target_function,
    void* replacement_function,
    unsigned char* preamble_stub,
    size_t stub_size,
    size_t* bytes_needed) {
  if ((NULL == target_function) ||
      (NULL == replacement_function) ||
      (NULL == preamble_stub)) {
    ASSERT(false, (L"Invalid parameters - either pTargetFunction or "
                   L"pReplacementFunction or pPreambleStub were NULL."));
    return SIDESTEP_INVALID_PARAMETER;
  }

  // TODO(V7:joi) Siggi and I just had a discussion and decided that both
  // patching and unpatching are actually unsafe.  We also discussed a
  // method of making it safe, which is to freeze all other threads in the
  // process, check their thread context to see if their eip is currently
  // inside the block of instructions we need to copy to the stub, and if so
  // wait a bit and try again, then unfreeze all threads once we've patched.
  // Not implementing this for now since we're only using SideStep for unit
  // testing, but if we ever use it for production code this is what we
  // should do.
  // 这个sidestep目前还没有投入到产品，所以线程的安全性处理还没做，开发者还是很谨小慎微的
  // 说不定这个东西问世的时候会带来安全隐患
  // 
  // NOTE: Stoyan suggests we can write 8 or even 10 bytes atomically using
  // FPU instructions, and on newer processors we could use cmpxchg8b or
  // cmpxchg16b. So it might be possible to do the patching/unpatching
  // atomically and avoid having to freeze other threads.  Note though, that
  // doing it atomically does not help if one of the other threads happens
  // to have its eip in the middle of the bytes you change while you change
  // them.
  // original函数地址
  unsigned char* target = reinterpret_cast<unsigned char*>(target_function);

  // Let's disassemble the preamble of the target function to see if we can
  // patch, and to see how much of the preamble we need to take.  We need 5
  // bytes for our jmp instruction, so let's find the minimum number of
  // instructions to get 5 bytes.
  // 通过反汇编original函数来看看是否能够patch，需要patch多少个字节。
  // 注意jmp指令需要5个字节
  MiniDisassembler disassembler;
  unsigned int preamble_bytes = 0;
  while (preamble_bytes < 5) {
    InstructionType instruction_type =
      disassembler.Disassemble(target + preamble_bytes, &preamble_bytes);
    if (IT_JUMP == instruction_type) {
      // 如果前5个字节有jmp系列指令，是没办法hook的
      ASSERT(false, (L"Unable to patch because there is a jump instruction "
                     L"in the first 5 bytes."));
      return SIDESTEP_JUMP_INSTRUCTION;
    } else if (IT_RETURN == instruction_type) {
      // 如果前5个字节有ret系列指令，那么函数太短了，不够patch一个jmp
      ASSERT(false, (L"Unable to patch because function is too short"));
      return SIDESTEP_FUNCTION_TOO_SMALL;
    } else if (IT_GENERIC != instruction_type) {
      // 这种是异端的情况，指令不认识。。。
      ASSERT(false, (L"Disassembler encountered unsupported instruction "
                     L"(either unused or unknown"));
      return SIDESTEP_UNSUPPORTED_INSTRUCTION;
    }
  }

  // 这个作为OUT型参数返回需要用到的字节数，5表示额外的jmp指令字节数
  if (NULL != bytes_needed)
    *bytes_needed = preamble_bytes + 5;

  // Inv: preamble_bytes is the number of bytes (at least 5) that we need to
  // take from the preamble to have whole instructions that are 5 bytes or more
  // in size total. The size of the stub required is cbPreamble + size of
  // jmp (5)
  if (preamble_bytes + 5 > stub_size) {
    NOTREACHED_NT();
    return SIDESTEP_INSUFFICIENT_BUFFER;
  }

  // First, copy the preamble that we will overwrite.
  // 把要被覆盖的首字节序列copy出来，这个RawMemcpy是逐字节copy，没有用crt
  RawMemcpy(reinterpret_cast<void*>(preamble_stub),
            reinterpret_cast<void*>(target), preamble_bytes);

  // Now, make a jmp instruction to the rest of the target function (minus the
  // preamble bytes we moved into the stub) and copy it into our preamble-stub.
  // find address to jump to, relative to next address after jmp instruction
#pragma warning(push)
#pragma warning(disable:4244)
  // This assignment generates a warning because it is 32 bit specific.
  // 计算original剩下的指令相对preamble_stub+preamble_bytes+5的偏移
  int relative_offset_to_target_rest
    = ((reinterpret_cast<unsigned char*>(target) + preamble_bytes) -
        (preamble_stub + preamble_bytes + 5));
#pragma warning(pop)
  // jmp (Jump near, relative, displacement relative to next instruction)
  // 这里放上jmp
  preamble_stub[preamble_bytes] = ASM_JMP32REL;
  // copy the address
  // jmp后跟上4个字节的相对跳转地址，此时jmp就跳到了overwrite之后的rest code
  RawMemcpy(reinterpret_cast<void*>(preamble_stub + preamble_bytes + 1),
            reinterpret_cast<void*>(&relative_offset_to_target_rest), 4);

  // Inv: preamble_stub points to assembly code that will execute the
  // original function by first executing the first cbPreamble bytes of the
  // preamble, then jumping to the rest of the function.

  // Overwrite the first 5 bytes of the target function with a jump to our
  // replacement function.
  // (Jump near, relative, displacement relative to next instruction)
  // 修改original起始地址，做出一个jmp
  target[0] = ASM_JMP32REL;

  // Find offset from instruction after jmp, to the replacement function.
#pragma warning(push)
#pragma warning(disable:4244)
  int offset_to_replacement_function =
    reinterpret_cast<unsigned char*>(replacement_function) -
    reinterpret_cast<unsigned char*>(target) - 5;
#pragma warning(pop)
  // complete the jmp instruction
  RawMemcpy(reinterpret_cast<void*>(target + 1),
            reinterpret_cast<void*>(&offset_to_replacement_function), 4);
  // 此时original的入口就变成了jmp到InternalThunk处
  // Set any remaining bytes that were moved to the preamble-stub to INT3 so
  // as not to cause confusion (otherwise you might see some strange
  // instructions if you look at the disassembly, or even invalid
  // instructions). Also, by doing this, we will break into the debugger if
  // some code calls into this portion of the code.  If this happens, it
  // means that this function cannot be patched using this patcher without
  // further thought.
  // 如果overwrite的字节数比5要多，就用0xcc填充剩下的部分
  if (preamble_bytes > 5) {
    RawMemset(reinterpret_cast<void*>(target + 5), ASM_INT3,
              preamble_bytes - 5);
  }

  // Inv: The memory pointed to by target_function now points to a relative
  // jump instruction that jumps over to the preamble_stub.  The preamble
  // stub contains the first stub_size bytes of the original target
  // function's preamble code, followed by a relative jump back to the next
  // instruction after the first cbPreamble bytes.

  return SIDESTEP_SUCCESS;
}
```

至于`MiniDisassembler`就不展开分析了，起始看到这里，已经明白了这个sidestep类型的resolver是如何patch的了。实际上，就是对原始的函数进行了inline hook，此后，当call original时，会jmp到`InternalThunk`，而`InternalThunk`中的`extra_argument`实际上是`ThunkData[i]`前面的32个字节，也就是copy出来的`preamble_stub`+`jmp original_rest`，当`interceptor_`执行完毕后再调用original时，就会找到这里执行copy出来的original原本的首指令序列并jmp到rest部分。

可以看出这个sidestep实现的inline hook非常的复杂。

## `SmartSidestepResolverThunk`

还有个以sidestep类型为基础的smart_sidestep类型。我们一并看了吧，找找不同。

```cpp
// This is the concrete resolver used to perform smart sidestep interceptions.
// This means basically a sidestep interception that skips the interceptor when
// the caller resides on the same dll being intercepted. It is intended as
// a helper only, because that determination is not infallible.
// SidestepResolverThunk的派生类
// 看起来是当call的发起者与被拦截的dll是同一个时，会跳过interceptor的执行
// 也就是说dll本体上的调用不会触发hook
// 好吧，我们此前猜测的完全不对。。。
class SmartSidestepResolverThunk : public SidestepResolverThunk {
 public:
  SmartSidestepResolverThunk() {}
  ~SmartSidestepResolverThunk() override {}

  // Implementation of Resolver::Setup.
  NTSTATUS Setup(const void* target_module,
                 const void* interceptor_module,
                 const char* target_name,
                 const char* interceptor_name,
                 const void* interceptor_entry_point,
                 void* thunk_storage,
                 size_t storage_bytes,
                 size_t* storage_used) override;

  // Implementation of Resolver::GetThunkSize.
  size_t GetThunkSize() const override;

 private:
  // Performs the actual call to the interceptor if the conditions are correct
  // (as determined by IsInternalCall).
  static void SmartStub();

  // Returns true if return_address is inside the module loaded at base.
  static bool IsInternalCall(const void* base, void* return_address);

  DISALLOW_COPY_AND_ASSIGN(SmartSidestepResolverThunk);
};
```

可以看出来这个类多了两个private函数用于判断caller是内部还是外部以及stub的处理。

先看看两个private：

```cpp
bool SmartSidestepResolverThunk::IsInternalCall(const void* base,
                                                void* return_address) {
  DCHECK_NT(base);
  DCHECK_NT(return_address);

  // 其实很简单，找找address是否在该PE上就知道了
  base::win::PEImage pe(base);
  if (pe.GetImageSectionFromAddr(return_address))
    return true;
  return false;
}

// This code must basically either call the intended interceptor or skip the
// call and invoke instead the original function. In any case, we are saving
// the registers that may be trashed by our c++ code.
//
// This function is called with a first parameter inserted by us, that points
// to our SmartThunk. When we call the interceptor we have to replace this
// parameter with the one expected by that function (stored inside our
// structure); on the other hand, when we skip the interceptor we have to remove
// that extra argument before calling the original function.
//
// When we skip the interceptor, the transformation of the stack looks like:
//  On Entry:                         On Use:                     On Exit:
//  [param 2] = first real argument   [param 2] (esp+1c)          [param 2]
//  [param 1] = our SmartThunk        [param 1] (esp+18)          [ret address]
//  [ret address] = real caller       [ret address] (esp+14)      [xxx]
//  [xxx]                             [addr to jump to] (esp+10)  [xxx]
//  [xxx]                             [saved eax]                 [xxx]
//  [xxx]                             [saved ebx]                 [xxx]
//  [xxx]                             [saved ecx]                 [xxx]
//  [xxx]                             [saved edx]                 [xxx]
__declspec(naked)
void SmartSidestepResolverThunk::SmartStub() {
  __asm {
    push eax                  // Space for the jump.
    push eax                  // Save registers.
    push ebx
    push ecx
    push edx
    mov ebx, [esp + 0x18]     // First parameter = SmartThunk.
    mov edx, [esp + 0x14]     // Get the return address.
    mov eax, [ebx]SmartThunk.module_base
    push edx
    push eax
    call SmartSidestepResolverThunk::IsInternalCall	// 这里判断一下是否是internal
    add esp, 8

    test eax, eax					  // 如果是的话，就直接call original就行了
    lea edx, [ebx]SmartThunk.sidestep   // The original function. 盯住这个edx
    jz call_interceptor				   // 如果不是internal，就得部署interceptor

    // Skip this call
    mov ecx, [esp + 0x14]               // Return address.
    mov [esp + 0x18], ecx               // Remove first parameter.
    mov [esp + 0x10], edx			   // edx是original function，这个位置作为ret时的返回地址
    pop edx                             // Restore registers.
    pop ecx
    pop ebx
    pop eax
    ret 4                               // Jump to original function.

  call_interceptor:
    mov ecx, [ebx]SmartThunk.interceptor
    mov [esp + 0x18], edx               // Replace first parameter. orignal地址成了第一个参数
    mov [esp + 0x10], ecx				// interceptor地址变成了ret addr
    pop edx                             // Restore registers.
    pop ecx
    pop ebx
    pop eax
    ret                                 // Jump to original function. 这个理应是jmp interceptor
  }
}
```

其实看了这些就基本理清了，展开`GetThunkSize`和`Setup`印证一下想法：

### `GetThunkSize`

```cpp
size_t SmartSidestepResolverThunk::GetThunkSize() const {
  return GetInternalThunkSize() + kSizeOfSidestepStub +
         offsetof(SmartThunk, sidestep);// SmartThunk包含了SidestepThunk
}

struct SmartThunk {
  const void* module_base;  // Target module's base.
  const void* interceptor;  // Real interceptor.
  SidestepThunk sidestep;   // Standard sidestep thunk.
};
```

### `Setup`

```cpp
// This is basically a wrapper around the normal sidestep patch that extends
// the thunk to use a chained interceptor. It uses the fact that
// SetInternalThunk generates the code to pass as the first parameter whatever
// it receives as original_function; we let SidestepResolverThunk set this value
// to its saved code, and then we change it to our thunk data.
NTSTATUS SmartSidestepResolverThunk::Setup(const void* target_module,
                                           const void* interceptor_module,
                                           const char* target_name,
                                           const char* interceptor_name,
                                           const void* interceptor_entry_point,
                                           void* thunk_storage,
                                           size_t storage_bytes,
                                           size_t* storage_used) {
  if (storage_bytes < GetThunkSize())
    return STATUS_BUFFER_TOO_SMALL;

  // 看出SmartThunk，填充module_base，SmartStub中会用到
  SmartThunk* thunk = reinterpret_cast<SmartThunk*>(thunk_storage);
  thunk->module_base = target_module;

  NTSTATUS ret;
  // 填充interceptor，SmartStub中会用到
  if (interceptor_entry_point) {
    thunk->interceptor = interceptor_entry_point;
  } else {
    ret = ResolveInterceptor(interceptor_module, interceptor_name,
                             &thunk->interceptor);
    if (!NT_SUCCESS(ret))
      return ret;
  }

  // Perform a standard sidestep patch on the last part of the thunk, but point
  // to our internal smart interceptor.
  size_t standard_bytes = storage_bytes - offsetof(SmartThunk, sidestep);
  // SmartThunk的SodestepThunk的填充和基类SidestepResolverThunk一致
  // 但注意interceptor_entry_point此时不再是interceptor的地址，而是SmartStub地址
  // 相当于在interceptor的基础上用SmartStub又拦了一次，而SmartStub会根据internal caller
  // 的判断跳转到interceptor的分支或original分支，interceptor在thunk_storage中记录了
  ret = SidestepResolverThunk::Setup(target_module, interceptor_module,
                                     target_name, nullptr,
                                     reinterpret_cast<void*>(&SmartStub),
                                     &thunk->sidestep, standard_bytes, nullptr);
  if (!NT_SUCCESS(ret))
    return ret;

  // Fix the internal thunk to pass the whole buffer to the interceptor.
  // 这里SmartStub地址作为了拦截器，而原本的一大坨thunk_storage作为original，它相当于一个代理
  SetInternalThunk(&thunk->sidestep.internal_thunk, GetInternalThunkSize(),
                   thunk_storage, reinterpret_cast<void*>(&SmartStub));

  if (storage_used)
    *storage_used = GetThunkSize();

  return ret;
}
```

## `ServiceResolverThunk`

我们分析过了三种target进程使用的Resolver，还有一种broker使用的Resolver类型——系统调用。

```cpp
// This is the concrete resolver used to perform service-call type functions
// inside ntdll.dll.
// 用于执行ntdll中的系统调用类型函数的拦截
class ServiceResolverThunk : public ResolverThunk {
 public:
  // The service resolver needs a child process to write to.
  ServiceResolverThunk(HANDLE process, bool relaxed)
      : ntdll_base_(nullptr),
        process_(process),
        relaxed_(relaxed),
        relative_jump_(0) {}
  ~ServiceResolverThunk() override {}

  // Implementation of Resolver::Setup.
  NTSTATUS Setup(const void* target_module,
                 const void* interceptor_module,
                 const char* target_name,
                 const char* interceptor_name,
                 const void* interceptor_entry_point,
                 void* thunk_storage,
                 size_t storage_bytes,
                 size_t* storage_used) override;

  // 可以看到两个resolve都override了
  // Implementation of Resolver::ResolveInterceptor.
  NTSTATUS ResolveInterceptor(const void* module,
                              const char* function_name,
                              const void** address) override;

  // Implementation of Resolver::ResolveTarget.
  NTSTATUS ResolveTarget(const void* module,
                         const char* function_name,
                         void** address) override;

  // Implementation of Resolver::GetThunkSize.
  size_t GetThunkSize() const override;

  // 这几个居然还是virtual，看来还有派生类
  // Call this to set up ntdll_base_ which will allow for local patches.
  virtual void AllowLocalPatches();

  // Verifies that the function specified by |target_name| in |target_module| is
  // a service and copies the data from that function into |thunk_storage|. If
  // |storage_bytes| is too small, then the method fails.
  virtual NTSTATUS CopyThunk(const void* target_module,
                             const char* target_name,
                             BYTE* thunk_storage,
                             size_t storage_bytes,
                             size_t* storage_used);

 protected:
  // The unit test will use this member to allow local patch on a buffer.
  HMODULE ntdll_base_;

  // Handle of the child process.
  HANDLE process_;

 private:
  // Returns true if the code pointer by target_ corresponds to the expected
  // type of function. Saves that code on the first part of the thunk pointed
  // by local_thunk (should be directly accessible from the parent).
  virtual bool IsFunctionAService(void* local_thunk) const;

  // Performs the actual patch of target_.
  // local_thunk must be already fully initialized, and the first part must
  // contain the original code. The real type of this buffer is ServiceFullThunk
  // (yes, private). remote_thunk (real type ServiceFullThunk), must be
  // allocated on the child, and will contain the thunk data, after this call.
  // Returns the apropriate status code.
  virtual NTSTATUS PerformPatch(void* local_thunk, void* remote_thunk);

  // Provides basically the same functionality as IsFunctionAService but it
  // continues even if it does not recognize the function code. remote_thunk
  // is the address of our memory on the child.
  bool SaveOriginalFunction(void* local_thunk, void* remote_thunk);

  // true if we are allowed to patch already-patched functions.
  bool relaxed_;
  ULONG relative_jump_;

  DISALLOW_COPY_AND_ASSIGN(ServiceResolverThunk);
};
```

显然，和前面分析的3个resolver不是一套体系的，这个要复杂得多。

service_resolver.h中也可以看到根据不同系统，以及位数进行了分类，定义了`ServiceResolverThunk`的众多子类。

```cpp
// This is the concrete resolver used to perform service-call type functions
// inside ntdll.dll on WOW64 (32 bit ntdll on 64 bit Vista).
class Wow64ResolverThunk : public ServiceResolverThunk {
 public:
  // The service resolver needs a child process to write to.
  Wow64ResolverThunk(HANDLE process, bool relaxed)
      : ServiceResolverThunk(process, relaxed) {}
  ~Wow64ResolverThunk() override {}

 private:
  bool IsFunctionAService(void* local_thunk) const override;

  DISALLOW_COPY_AND_ASSIGN(Wow64ResolverThunk);
};

// This is the concrete resolver used to perform service-call type functions
// inside ntdll.dll on WOW64 for Windows 8.
class Wow64W8ResolverThunk : public ServiceResolverThunk {
 public:
  // The service resolver needs a child process to write to.
  Wow64W8ResolverThunk(HANDLE process, bool relaxed)
      : ServiceResolverThunk(process, relaxed) {}
  ~Wow64W8ResolverThunk() override {}

 private:
  bool IsFunctionAService(void* local_thunk) const override;

  DISALLOW_COPY_AND_ASSIGN(Wow64W8ResolverThunk);
};

// This is the concrete resolver used to perform service-call type functions
// inside ntdll.dll on Windows 8.
class Win8ResolverThunk : public ServiceResolverThunk {
 public:
  // The service resolver needs a child process to write to.
  Win8ResolverThunk(HANDLE process, bool relaxed)
      : ServiceResolverThunk(process, relaxed) {}
  ~Win8ResolverThunk() override {}

 private:
  bool IsFunctionAService(void* local_thunk) const override;

  DISALLOW_COPY_AND_ASSIGN(Win8ResolverThunk);
};

// This is the concrete resolver used to perform service-call type functions
// inside ntdll.dll on WOW64 for Windows 10.
class Wow64W10ResolverThunk : public ServiceResolverThunk {
 public:
  // The service resolver needs a child process to write to.
  Wow64W10ResolverThunk(HANDLE process, bool relaxed)
      : ServiceResolverThunk(process, relaxed) {}
  ~Wow64W10ResolverThunk() override {}

 private:
  bool IsFunctionAService(void* local_thunk) const override;

  DISALLOW_COPY_AND_ASSIGN(Wow64W10ResolverThunk);
};
```

这些子类都override了`IsFunctionAService`，这是因为不同系统版本位数对系统调用的判断条件是不同的。

但是父类的另外几个虚函数，没有找到身影。

### 两个resolve

```cpp
NTSTATUS ServiceResolverThunk::ResolveInterceptor(
    const void* interceptor_module,
    const char* interceptor_name,
    const void** address) {
  // After all, we are using a locally mapped version of the exe, so the
  // action is the same as for a target function.
  return ResolveTarget(interceptor_module, interceptor_name,
                       const_cast<void**>(address));
}

// In this case all the work is done from the parent, so resolve is
// just a simple GetProcAddress.
NTSTATUS ServiceResolverThunk::ResolveTarget(const void* module,
                                             const char* function_name,
                                             void** address) {
  if (!module)
    return STATUS_UNSUCCESSFUL;

  // 还是老套路，没啥好说的
  base::win::PEImage module_image(module);
  *address =
      reinterpret_cast<void*>(module_image.GetProcAddress(function_name));

  if (!*address) {
    NOTREACHED_NT();
    return STATUS_UNSUCCESSFUL;
  }

  return STATUS_SUCCESS;
}
```

跟基类的原理是一样的，但这里和基类的两个resolve却是反过来的，基类是target->interceptor，而这里却是interceptor->target。

不太清楚是否有什么禁忌在其中。

### `x86 ServiceFullThunk`

在研究`Setup`之前，还是按照惯例先研究Thunk：

```cpp
struct ServiceFullThunk {
  union {
    ServiceEntry original;
    ServiceEntryW8 original_w8;
    Wow64Entry wow_64;
    Wow64EntryW8 wow_64_w8;
  };
  int internal_thunk;  // Dummy member to the beginning of the internal thunk.
};
```

内部是个union，说明有4种不同的Thunk，对应到不同的系统版本和位数。我们先研究一下各版本的Entry。

```cpp
// Service code for 32 bit systems.
// NOTE: on win2003 "call dword ptr [edx]" is "call edx".
// 这个是32位系统调用样例
struct ServiceEntry {
  // This struct contains roughly the following code:
  // 00 mov     eax,25h
  // 05 mov     edx,offset SharedUserData!SystemCallStub (7ffe0300)
  // 0a call    dword ptr [edx]
  // 0c ret     2Ch
  // 0f nop
  BYTE mov_eax;         // = B8
  ULONG service_id;
  BYTE mov_edx;         // = BA
  ULONG stub;
  USHORT call_ptr_edx;  // = FF 12
  BYTE ret;             // = C2
  USHORT num_params;
  BYTE nop;
};

// Service code for 32 bit Windows 8.
// 32位win8的service code入口
struct ServiceEntryW8 {
  // This struct contains the following code:
  // 00 b825000000      mov     eax,25h
  // 05 e803000000      call    eip+3
  // 0a c22c00          ret     2Ch
  // 0d 8bd4            mov     edx,esp
  // 0f 0f34            sysenter
  // 11 c3              ret
  // 12 8bff            mov     edi,edi
  BYTE mov_eax;         // = B8
  ULONG service_id;
  BYTE call_eip;        // = E8
  ULONG call_offset;
  BYTE ret_p;           // = C2
  USHORT num_params;
  USHORT mov_edx_esp;   // = BD D4
  USHORT sysenter;      // = 0F 34
  BYTE ret;             // = C3
  USHORT nop;
};

// Service code for a 32 bit process running on a 64 bit os.
// 32位进程在64位os上运行的service code
struct Wow64Entry {
  // This struct may contain one of two versions of code:
  // 1. For XP, Vista and 2K3:
  // 00 b825000000      mov     eax, 25h
  // 05 33c9            xor     ecx, ecx
  // 07 8d542404        lea     edx, [esp + 4]
  // 0b 64ff15c0000000  call    dword ptr fs:[0C0h]
  // 12 c22c00          ret     2Ch
  //
  // 2. For Windows 7:
  // 00 b825000000      mov     eax, 25h
  // 05 33c9            xor     ecx, ecx
  // 07 8d542404        lea     edx, [esp + 4]
  // 0b 64ff15c0000000  call    dword ptr fs:[0C0h]
  // 12 83c404          add     esp, 4
  // 15 c22c00          ret     2Ch
  //
  // So we base the structure on the bigger one:
  BYTE mov_eax;         // = B8
  ULONG service_id;
  USHORT xor_ecx;       // = 33 C9
  ULONG lea_edx;        // = 8D 54 24 04
  ULONG call_fs1;       // = 64 FF 15 C0
  USHORT call_fs2;      // = 00 00
  BYTE call_fs3;        // = 00
  BYTE add_esp1;        // = 83             or ret
  USHORT add_esp2;      // = C4 04          or num_params
  BYTE ret;             // = C2
  USHORT num_params;	// 这个得算，根据当前是哪种情况
};

// Service code for a 32 bit process running on 64 bit Windows 8.
struct Wow64EntryW8 {
  // 00 b825000000      mov     eax, 25h
  // 05 64ff15c0000000  call    dword ptr fs:[0C0h]
  // 0b c22c00          ret     2Ch
  // 0f 90              nop
  BYTE mov_eax;         // = B8
  ULONG service_id;
  ULONG call_fs1;       // = 64 FF 15 C0
  USHORT call_fs2;      // = 00 00
  BYTE call_fs3;        // = 00
  BYTE ret;             // = C2
  USHORT num_params;
  BYTE nop;
};
```

### `x64 ServiceFullThunk`

```cpp
// We don't have an internal thunk for x64.
// x64看来不需要internal thunk
struct ServiceFullThunk {
  union {
    ServiceEntry original;
    ServiceEntryW8 original_w8;
    ServiceEntryWithInt2E original_int2e_fallback;
  };
};

// Service code for 64 bit systems.
struct ServiceEntry {
  // This struct contains roughly the following code:
  // 00 mov     r10,rcx
  // 03 mov     eax,52h
  // 08 syscall
  // 0a ret
  // 0b xchg    ax,ax
  // 0e xchg    ax,ax

  ULONG mov_r10_rcx_mov_eax;  // = 4C 8B D1 B8
  ULONG service_id;
  USHORT syscall;             // = 0F 05
  BYTE ret;                   // = C3
  BYTE pad;                   // = 66
  USHORT xchg_ax_ax1;         // = 66 90
  USHORT xchg_ax_ax2;         // = 66 90
};

// Service code for 64 bit Windows 8.
struct ServiceEntryW8 {
  // This struct contains the following code:
  // 00 48894c2408      mov     [rsp+8], rcx
  // 05 4889542410      mov     [rsp+10], rdx
  // 0a 4c89442418      mov     [rsp+18], r8
  // 0f 4c894c2420      mov     [rsp+20], r9
  // 14 4c8bd1          mov     r10,rcx
  // 17 b825000000      mov     eax,25h
  // 1c 0f05            syscall
  // 1e c3              ret
  // 1f 90              nop

  ULONG64 mov_1;              // = 48 89 4C 24 08 48 89 54
  ULONG64 mov_2;              // = 24 10 4C 89 44 24 18 4C
  ULONG mov_3;                // = 89 4C 24 20
  ULONG mov_r10_rcx_mov_eax;  // = 4C 8B D1 B8
  ULONG service_id;
  USHORT syscall;             // = 0F 05
  BYTE ret;                   // = C3
  BYTE nop;                   // = 90
};

// Service code for 64 bit systems with int 2e fallback.
struct ServiceEntryWithInt2E {
  // This struct contains roughly the following code:
  // 00 4c8bd1           mov     r10,rcx
  // 03 b855000000       mov     eax,52h
  // 08 f604250803fe7f01 test byte ptr SharedUserData!308, 1
  // 10 7503             jne [over syscall]
  // 12 0f05             syscall
  // 14 c3               ret
  // 15 cd2e             int 2e
  // 17 c3               ret

  ULONG mov_r10_rcx_mov_eax;  // = 4C 8B D1 B8
  ULONG service_id;
  USHORT test_byte;           // = F6 04
  BYTE ptr;                   // = 25
  ULONG user_shared_data_ptr;
  BYTE one;                   // = 01
  USHORT jne_over_syscall;    // = 75 03
  USHORT syscall;             // = 0F 05
  BYTE ret;                   // = C3
  USHORT int2e;               // = CD 2E
  BYTE ret2;                  // = C3
};
```

### `GetThunkSize`

```cpp
// x86的结构是ServiceFullThunk接一个InternalThunk
size_t ServiceResolverThunk::GetThunkSize() const {
  return offsetof(ServiceFullThunk, internal_thunk) + GetInternalThunkSize();
}

// x64结构只有一个ServiceFullThunk，没用InternalThunk
size_t ServiceResolverThunk::GetThunkSize() const {
  return sizeof(ServiceFullThunk);
}
```

### `x86 Setup`

```cpp
// x86的Setup
NTSTATUS ServiceResolverThunk::Setup(const void* target_module,
                                     const void* interceptor_module,
                                     const char* target_name,
                                     const char* interceptor_name,
                                     const void* interceptor_entry_point,
                                     void* thunk_storage,
                                     size_t storage_bytes,
                                     size_t* storage_used) {
  NTSTATUS ret =
      Init(target_module, interceptor_module, target_name, interceptor_name,
           interceptor_entry_point, thunk_storage, storage_bytes);
  if (!NT_SUCCESS(ret))
    return ret;

  relative_jump_ = 0;
  size_t thunk_bytes = GetThunkSize();
  // thunk buffer是内部new出来的
  std::unique_ptr<char[]> thunk_buffer(new char[thunk_bytes]);
  // 部署成ServiceFullThunk结构
  ServiceFullThunk* thunk =
      reinterpret_cast<ServiceFullThunk*>(thunk_buffer.get());

  // 检查一下想要被hook的函数是否是系统调用
  // SaveOriginalFunction是什么？暂时不清楚
  if (!IsFunctionAService(&thunk->original) &&
      (!relaxed_ || !SaveOriginalFunction(&thunk->original, thunk_storage))) {
    return STATUS_OBJECT_NAME_COLLISION;
  }

  // 关键call，这里进行了patch实装
  ret = PerformPatch(thunk, thunk_storage);

  if (storage_used)
    *storage_used = thunk_bytes;

  return ret;
}
```

`IsFunctionAService`顾名思义，就是读取target进程的original函数的字节码与`ServiceEntry`进行比较（对于`ServiceResolverThunk`来说用的是`ServiceEntry`）。

```cpp
bool ServiceResolverThunk::IsFunctionAService(void* local_thunk) const {
  ServiceEntry function_code;
  SIZE_T read;
  // 读取target进程上的original函数字节码，长度为ServiceEntry的尺寸
  if (!::ReadProcessMemory(process_, target_, &function_code,
                           sizeof(function_code), &read)) {
    return false;
  }

  if (sizeof(function_code) != read)
    return false;

  // 开始逐字节比较，固定的几处字节码如果有误的话，说明不是一个系统调用
  if (kMovEax != function_code.mov_eax || kMovEdx != function_code.mov_edx ||
      (kCallPtrEdx != function_code.call_ptr_edx &&	// 这里的&&是因为win2003上是call edx
       kCallEdx != function_code.call_ptr_edx) ||
      kRet != function_code.ret) {
    return false;
  }

  // Find the system call pointer if we don't already have it.
  // 如果是call dword ptr [edx]的话，要读取出[edx]
  if (kCallEdx != function_code.call_ptr_edx) {
    DWORD ki_system_call;
    if (!::ReadProcessMemory(process_,
                             bit_cast<const void*>(function_code.stub),
                             &ki_system_call, sizeof(ki_system_call), &read)) {
      return false;
    }

    if (sizeof(ki_system_call) != read)
      return false;

    HMODULE module_1, module_2;
    // 检查一下读取到的ki_system_call的module和original函数的module是否一致
    // last check, call_stub should point to a KiXXSystemCall function on ntdll
    if (!GetModuleHandleEx(GET_MODULE_HANDLE_EX_FLAG_FROM_ADDRESS |
                               GET_MODULE_HANDLE_EX_FLAG_UNCHANGED_REFCOUNT,
                           bit_cast<const wchar_t*>(ki_system_call),
                           &module_1)) {
      return false;
    }

    if (ntdll_base_) {
      // This path is only taken when running the unit tests. We want to be
      // able to patch a buffer in memory, so target_ is not inside ntdll.
      module_2 = ntdll_base_;
    } else {
      if (!GetModuleHandleEx(GET_MODULE_HANDLE_EX_FLAG_FROM_ADDRESS |
                                 GET_MODULE_HANDLE_EX_FLAG_UNCHANGED_REFCOUNT,
                             reinterpret_cast<const wchar_t*>(target_),
                             &module_2))
        return false;
    }

    if (module_1 != module_2)
      return false;
  }

  // Save the verified code
  // 此时Setup中的ServiceFullThunk的ServiceEntry成员就正确填充了
  memcpy(local_thunk, &function_code, sizeof(function_code));

  return true;
}
```

可以看到`IsFunctionAService`不仅判断了是否是service call，还拷贝了original function的字节码给了Setup传递进来的`ServiceEntry`。继续展开`SaveOriginalFunction`看看本地填充好的`ServiceEntry`是如何与外部传进来的`thunk_storage`建立联系的。

```cpp
bool ServiceResolverThunk::SaveOriginalFunction(void* local_thunk,
                                                void* remote_thunk) {
  ServiceEntry function_code;
  SIZE_T read;
  // 又去读了一遍
  if (!::ReadProcessMemory(process_, target_, &function_code,
                           sizeof(function_code), &read)) {
    return false;
  }

  if (sizeof(function_code) != read)
    return false;

  // 进到这个函数就表示，Resolver是允许repatch的，那么第一次patch之后其后再次的patch
  // 首字节就不再是mov eax,xx而是jmp xxx了
  if (kJmp32 == function_code.mov_eax) {
    // Plain old entry point patch. The relative jump address follows it.
    // 读出来上一次patch的jmp到的地址
    ULONG relative = function_code.service_id;

    // First, fix our copy of their patch.
    // 修正地址
    relative += bit_cast<ULONG>(target_) - bit_cast<ULONG>(remote_thunk);

    function_code.service_id = relative;

    // And now, remember how to re-patch it.
    // 这里处理repatch，remote_thunk是个ServiceFullThunk结构
    ServiceFullThunk* full_thunk =
        reinterpret_cast<ServiceFullThunk*>(remote_thunk);

    const ULONG kJmp32Size = 5;

    // internal_thunk指向的是InternalThunk，它的地址减去original的地址再扣除5个jmp的尺寸
    // 就是最终的跳转的地址偏移量，更新对象的成员relative_jump_，它用于从InternalThunk调用
    // original时跳转到original
    relative_jump_ = bit_cast<ULONG>(&full_thunk->internal_thunk) -
                     bit_cast<ULONG>(target_) - kJmp32Size;
  }

  // Save the verified code
  // 如果不是首次进入，就把更新了jmp地址的ServiceEntry拷贝给local_thunk
  // 如果是首次的话，实际上是copy出了original的字节码
  memcpy(local_thunk, &function_code, sizeof(function_code));

  return true;
}
```

然后是最后的patch实装：

```cpp
NTSTATUS ServiceResolverThunk::PerformPatch(void* local_thunk,
                                            void* remote_thunk) {
  ServiceEntry intercepted_code;
  size_t bytes_to_write = sizeof(intercepted_code);
  ServiceFullThunk* full_local_thunk =
      reinterpret_cast<ServiceFullThunk*>(local_thunk);
  ServiceFullThunk* full_remote_thunk =
      reinterpret_cast<ServiceFullThunk*>(remote_thunk);

  // patch the original code
  // original字节码拷贝给local buffer
  memcpy(&intercepted_code, &full_local_thunk->original,
         sizeof(intercepted_code));
  intercepted_code.mov_eax = kMovEax;
  intercepted_code.service_id = full_local_thunk->original.service_id;
  intercepted_code.mov_edx = kMovEdx;
  intercepted_code.stub = bit_cast<ULONG>(&full_remote_thunk->internal_thunk);
  intercepted_code.call_ptr_edx = kJmpEdx;
  bytes_to_write = kMinServiceSize;

  // 如果relative_jump_有效，说明不是首次patch的实装，此时就要修正local buffer中的字节码为正确
  // 的jmp relative_jump_，这个偏移量是remote buffer中InternalThunk的地址减去original函数的地址
  // 再扣除5个字节的jmp
  if (relative_jump_) {
    intercepted_code.mov_eax = kJmp32;
    intercepted_code.service_id = relative_jump_;
    bytes_to_write = offsetof(ServiceEntry, mov_edx);
  }

  // setup the thunk
  // 部署InternalThunk，InternalThunk的效果是interceptor_(remote_thunk,xxx)
  // 也就是跳到remote_thunk的起始ServiceEntry处，而不是original
  SetInternalThunk(&full_local_thunk->internal_thunk, GetInternalThunkSize(),
                   remote_thunk, interceptor_);

  size_t thunk_size = GetThunkSize();

  // copy the local thunk buffer to the child
  // local buffer给了remote_thunk的ServiceEntry
  // 所以当interceptor_跳到这里时会执行jmp xxx（存在repatch）或者执行original原本的代码序列（没有repatch）
  SIZE_T written;
  if (!::WriteProcessMemory(process_, remote_thunk, local_thunk, thunk_size,
                            &written)) {
    return STATUS_UNSUCCESSFUL;
  }

  if (thunk_size != written)
    return STATUS_UNSUCCESSFUL;

  // and now change the function to intercept, on the child
  // 现在要处理的就是把原本的original函数开头的那一段内容修改成interceptor_
  // 实现call original时实际上是call interceptor_，以此完成了整个hook链
  if (ntdll_base_) {
    // running a unit test
    if (!::WriteProcessMemory(process_, target_, &intercepted_code,
                              bytes_to_write, &written))
      return STATUS_UNSUCCESSFUL;
  } else {
    if (!WriteProtectedChildMemory(process_, target_, &intercepted_code,
                                   bytes_to_write))
      return STATUS_UNSUCCESSFUL;
  }

  return STATUS_SUCCESS;
}
```

### `x64 Setup`

至于x64就比较简单了，因为它的设计并不是interceptor_(original,xxx)，所以也就没有这么复杂的Thunk实装过程，它的`InternalThunk`仅仅是:

```cpp
struct InternalThunk {
  // This struct contains roughly the following code:
  // 01 48b8f0debc9a78563412  mov   rax,123456789ABCDEF0h
  // ff e0                    jmp   rax
  //
  // The code modifies rax, but that's fine for x64 ABI.

  InternalThunk() {
    mov_rax = kMovRax;
    jmp_rax = kJmpRax;
    interceptor_function = 0;
  };
  USHORT mov_rax;  // = 48 B8
  ULONG_PTR interceptor_function;
  USHORT jmp_rax;  // = ff e0
};
```

也是通过`WriteChildProcessMemory`把original的入口改写成Thunk的地址。

```cpp
NTSTATUS ServiceResolverThunk::Setup(const void* target_module,
                                     const void* interceptor_module,
                                     const char* target_name,
                                     const char* interceptor_name,
                                     const void* interceptor_entry_point,
                                     void* thunk_storage,
                                     size_t storage_bytes,
                                     size_t* storage_used) {
  NTSTATUS ret =
      Init(target_module, interceptor_module, target_name, interceptor_name,
           interceptor_entry_point, thunk_storage, storage_bytes);
  if (!NT_SUCCESS(ret))
    return ret;

  size_t thunk_bytes = GetThunkSize();
  std::unique_ptr<char[]> thunk_buffer(new char[thunk_bytes]);
  ServiceFullThunk* thunk =
      reinterpret_cast<ServiceFullThunk*>(thunk_buffer.get());

  if (!IsFunctionAService(&thunk->original))
    return STATUS_OBJECT_NAME_COLLISION;

  ret = PerformPatch(thunk, thunk_storage);

  if (storage_used)
    *storage_used = thunk_bytes;

  return ret;
}

NTSTATUS ServiceResolverThunk::PerformPatch(void* local_thunk,
                                            void* remote_thunk) {
  // Patch the original code.
  ServiceEntry local_service;
  DCHECK_NT(GetInternalThunkSize() <= sizeof(local_service));
  if (!SetInternalThunk(&local_service, sizeof(local_service), nullptr,
                        interceptor_))
    return STATUS_UNSUCCESSFUL;

  // Copy the local thunk buffer to the child.
  // local thunk buffer存储的是original函数原本的指令序列，拷贝到了远端的remote_thunk
  SIZE_T actual;
  if (!::WriteProcessMemory(process_, remote_thunk, local_thunk,
                            sizeof(ServiceFullThunk), &actual))
    return STATUS_UNSUCCESSFUL;

  if (sizeof(ServiceFullThunk) != actual)
    return STATUS_UNSUCCESSFUL;

  // And now change the function to intercept, on the child.
  if (ntdll_base_) {
    // Running a unit test.
    if (!::WriteProcessMemory(process_, target_, &local_service,
                              sizeof(local_service), &actual))
      return STATUS_UNSUCCESSFUL;
  } else {
    // 此时original函数面目全非，变成了InternalThunk的mov rax,interceptor;jmp rax;
    // 至于如何从interceptor内部跳回remote_thunk，那就是interceptor的事了（g_orignal）
    // 还记得InterceptionManager::PatchClientFunctions吗？
    if (!WriteProtectedChildMemory(process_, target_, &local_service,
                                   sizeof(local_service)))
      return STATUS_UNSUCCESSFUL;
  }

  return STATUS_SUCCESS;
}
```

到此，我们终于可以明白`InterceptionManager::PatchClientFunctions`的意义了：

```cpp
ResultCode InterceptionManager::PatchClientFunctions(
    DllInterceptionData* thunks,
    size_t thunk_bytes,
    DllInterceptionData* dll_data) {
  DCHECK(thunks);
  DCHECK(dll_data);

  HMODULE ntdll_base = ::GetModuleHandle(kNtdllName);
  if (!ntdll_base)
    return SBOX_ERROR_NO_HANDLE;

  char* interceptor_base = nullptr;

#if defined(SANDBOX_EXPORTS)
  interceptor_base = reinterpret_cast<char*>(child_->MainModule());
  base::ScopedNativeLibrary local_interceptor(::LoadLibrary(child_->Name()));
#endif  // defined(SANDBOX_EXPORTS)

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
    if (interception.dll != ntdll)
      return SBOX_ERROR_BAD_PARAMS;

    if (INTERCEPTION_SERVICE_CALL != interception.type)
      return SBOX_ERROR_BAD_PARAMS;

    // 对每个service call类型的ntdll的拦截函数进行处理
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
    // 这里对&thunks->thunks[dll_data->num_thunks]进行了实装
    // 它是broker在target进程中开辟的内存空间，空间起始是DllInterceptionData
    // 这里是某一个拦截函数的ThunkData
    NTSTATUS ret = thunk->Setup(
        ntdll_base, interceptor_base, interception.function.c_str(),
        interception.interceptor.c_str(), interception.interceptor_address,
        &thunks->thunks[dll_data->num_thunks],
        thunk_bytes - dll_data->used_bytes, nullptr);
    if (!NT_SUCCESS(ret)) {
      ::SetLastError(GetLastErrorFromNtStatus(ret));
      return SBOX_ERROR_CANNOT_SETUP_INTERCEPTION_THUNK;
    }

    // 这里更新了g_originals，使某个类型的interception指向正确的remote端内存地址
    // 另一方面，g_originals的设计也暴露出来了拦截的函数实际上是固定的那些，凡是
    // 使用到的拦截函数，在InterceptorId这个enum结构中都对应一个值
    DCHECK(!g_originals[interception.id]);
    g_originals[interception.id] = &thunks->thunks[dll_data->num_thunks];

    dll_data->num_thunks++;
    dll_data->used_bytes += sizeof(ThunkData);
  }

  return SBOX_ALL_OK;
}
```

Interception和Resolver是目前分析sandbox中最为复杂的部分，尽管一路上磕磕绊绊，但总算是缕清了分分毫毫。

子系统的三大组件中，我们已经分析过了用于分发IPC请求的dispatcher，也分析过了安装Hook的Interception和Resolver，下一节我们分析最后一个——Policy。



