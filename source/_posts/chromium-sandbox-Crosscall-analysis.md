---
title: Chromium-sandbox-crosscall-analysis
date: 2018-05-19 10:31:11
categories: 源码剖析
tags:
	- chromium
	- chromium-sandbox
---

本篇是sandbox源码剖析的第七篇，主要分析了windows平台下，Chromium sandbox IPC通信中参数返回值的封装以及IPC Channel Buffer的结构设计。本篇相对独立，可以直接阅读。

想要流程的阅读本系列你需要以下几个条件：
1. 较高水平的C++编码能力（至少通读C++ Primer 5th，刷过课后题，有一定编码量）。
2. 熟悉Windows API编程，尤其是安全相关的内容。
3. 对二进制安全有一定了解，熟悉各类型安全漏洞成因、漏洞缓解措施及bypass手法。

<!--more-->

# chromium-sandbox-crosscall-analysis

## Common gargets

CrossCall是沙盒IPC实现的灵魂。想要了解CrossCall代码，势必先了解它的设计。从crosscall_params.h的注释头可以获取大量的有用信息：

```cpp
// This header is part of CrossCall: the sandbox inter-process communication.
// This header defines the basic types used both in the client IPC and in the
// server IPC code. CrossCallParams and ActualCallParams model the input
// parameters of an IPC call and CrossCallReturn models the output params and
// the return value.
//
// An IPC call is defined by its 'tag' which is a (uint32_t) unique identifier
// that is used to route the IPC call to the proper server. Every tag implies
// a complete call signature including the order and type of each parameter.
//
// Like most IPC systems. CrossCall is designed to take as inputs 'simple'
// types such as integers and strings. Classes, generic arrays or pointers to
// them are not supported.
//
// Another limitation of CrossCall is that the return value and output
// parameters can only be uint32_t integers. Returning complex structures or
// strings is not supported.
```

简单梳理一下：

1. client和server端通过IPC通信，其中`CrossCallParams`和`ActualCallParams`封装了client端发起IPC调用的输入型参数，而`CrossCallReturn`封装了输出型参数和返回值。
2. 每种IPC调用都以它的tag区分，tag是个`uint32_t`类型值，不同的值对应不同的IPC调用。tag用以将IPC调用转发给正确的server处理，每种tag类型暗示了一套完整的调用签名，这包括参数的类型和顺序。
3. CrossCall的输入参数只能使用整型、字符串等简单类型。类、数组或指针是不行的。
4. CrossCall的另一个限制在于返回值和输出参数只能是`uint32_t`类型，不能返回复杂的结构体或字符串。

> 其实`uint32_t`已经绰绰有余了，x86平台指针也是32位的，可以做类型转换。

### Infrastructure

```cpp
// max number of extended return parameters. See CrossCallReturn
// 输出型参数最多只能有8个，不清楚是否包括返回值，等到看CrossCallReturn时再说
const size_t kExtendedReturnCount = 8;

// Union of multiple types to be used as extended results
// in the CrossCallReturn.
// 前面已经看到了，返回值和输出型参数只能是uint32_t
// 实际上这种说法不严谨，编码意义上的本质在于只能使用32位的值。
// 32位的值可以表示一个任意类型x86指针，可以表示句柄等等。
// 下面的union就整理了CrossCallReturn可能会用到的几种类型
union MultiType {
  uint32_t unsigned_int;
  void* pointer;
  HANDLE handle;
  ULONG_PTR ulong_ptr;
};

// Maximum number of IPC parameters currently supported.
// To increase this value, we have to:
//  - Add another Callback typedef to Dispatcher.
//  - Add another case to the switch on SharedMemIPCServer::InvokeCallback.
//  - Add another case to the switch in GetActualAndMaxBufferSize
// 这个应该是IPC参数的最大数量
// 看起来如果扩展参数数量会相当麻烦：
//		- Dispatcher的Callback需要增加
//		- SharedMemIPCServer::InvokeCallback的switch需要加个case
//		- GetActualAndMaxBufferSize的switch需要加个case
const int kMaxIpcParams = 9;
```

再看IPC buffer的info封装：

```cpp
// Contains the information about a parameter in the ipc buffer.
// IPC通信无论如何折腾，最本质的client用buffer承载数据，将其发给server
// ParamInfo则抽象了buffer中的某个参数
struct ParamInfo {
  ArgType type_;	// 看下面
  uint32_t offset_;	// 真实数据偏移
  uint32_t size_;	// 真实数据尺寸
};
// 这种offset+size的组合通常来说都是真实数据游离于ParamInfo结构之外
// 而真实数据append到ParamInfo后面的某个地址处，offset和size用于定位真实数据的起始

// Defines the supported C++ types encoding to numeric id. Like a simplified
// RTTI. Note that true C++ RTTI will not work because the types are not
// polymorphic anyway.
// 这个就表示参数（上面提到的真实数据）是哪一种C++类型，用枚举来表示
// 这就很像一个简化的RTTI（runtime type identify），当然了这些类型没有多态所以RTTI是没戏的。
enum ArgType {
  INVALID_TYPE = 0,
  WCHAR_TYPE,
  UINT32_TYPE,
  UNISTR_TYPE,
  VOIDPTR_TYPE,
  INPTR_TYPE,
  INOUTPTR_TYPE,
  LAST_TYPE
};
```

### `CrossCallReturn`

```cpp
// Models the return value and the return parameters of an IPC call
// currently limited to one status code and eight generic return values
// which cannot be pointers to other data. For x64 ports this structure
// might have to use other integer types.
// 封装了IPC调用的输出型参数和返回值
struct CrossCallReturn {
  // the IPC tag. It should match the original IPC tag.
  uint32_t tag;	// 每种IPC调用独有的tag
  // The result of the IPC operation itself.
  ResultCode call_outcome;	// 保存IPC操作本身的状态码结果
  // the result of the IPC call as executed in the server. The interpretation
  // of this value depends on the specific service.
  // 这个就是server上处理IPC call的结果状态值，具体意义取决于特定的服务
  union {
    NTSTATUS nt_status;
    DWORD win32_result;
  };
  // Number of extended return values.
  uint32_t extended_count;	// 应该是输出型参数的个数，暂不清楚是否包含返回值
  // for calls that should return a windows handle. It is found here.
  HANDLE handle;	// 如果有需要返回windows句柄的，可以存在这里
  // The array of extended values.
  // extended values数组，每一种都是MultiType这个union
  MultiType extended[kExtendedReturnCount];//kExtendedReturnCount是8
};
```

`CrossCallReturn`看起来的确封装了输出参数和返回值，在client发起IPC调用时，应该是直接或者间接通过某种结构嵌套传到server，而server在把结果和输出参数填充好，client再读出来。

### `CrossCallParams`

再看看输入型参数的封装结构：

```cpp
// CrossCallParams base class that models the input params all packed in a
// single compact memory blob. The representation can vary but in general a
// given child of this class is meant to represent all input parameters
// necessary to make a IPC call.
// 把输入型参数紧凑的捏成一团，用于发起IPC调用
// 
// This class cannot have virtual members because its assumed the IPC
// parameters start from the 'this' pointer to the end, which is defined by
// one of the subclasses
// 该类无法拥有虚成员函数，因为设计上会假定从this指针起始到结尾（并不是对象内存空间尾）这部分内存空间
// 要作为IPC参数的buffer，这由它的子类定义
//（含有虚函数的对象会有虚表，虚表在this处即对象头部位置）
// 
// Objects of this class cannot be constructed directly. Only derived
// classes have the proper knowledge to construct it.
// 限于这种复杂的使用方式，该类不能简单的直接构造，而是必须通过子类以特殊方式构造。
class CrossCallParams {
 public:
  // Returns the tag (ipc unique id) associated with this IPC.
  // 获取该IPC调用的tag，实际上存储在tag_成员
  uint32_t GetTag() const { return tag_; }

  // Returns the beggining of the buffer where the IPC params can be stored.
  // prior to an IPC call
  // 这里就看出类头注释的意义了，该对象实体的整个内存空间都是IPC参数存储的buffer
  const void* GetBuffer() const { return this; }

  // Returns how many parameter this IPC call should have.
  // 返回该IPC call有多少个参数
  uint32_t GetParamsCount() const { return params_count_; }

  // Returns a pointer to the CrossCallReturn structure.
  // 返回CrossCallReturn结构指针，这里可以看出该对象内部封装了一个用于承载返回值和
  // 输出型参数的结构
  CrossCallReturn* GetCallReturn() { return &call_return; }

  // Returns true if this call contains InOut parameters.
  // 是否有InOut型即输入输出型参数
  bool IsInOut() const { return (1 == is_in_out_); }

  // Tells the CrossCall object if it contains InOut parameters.
  void SetIsInOut(bool value) {
    if (value)
      is_in_out_ = 1;
    else
      is_in_out_ = 0;
  }

 protected:
  // constructs the IPC call params. Called only from the derived classes
  // 构造器是protected，表明该对象的实例化需要借助派生类。
  CrossCallParams(uint32_t tag, uint32_t params_count)
      : tag_(tag), is_in_out_(0), params_count_(params_count) {}

 private:
  uint32_t tag_;
  uint32_t is_in_out_;
  CrossCallReturn call_return;
  const uint32_t params_count_;	
  DISALLOW_COPY_AND_ASSIGN(CrossCallParams);
};
```

能够嗅到一点设计上熟悉的味道，下一步就是找到它的派生类，看看整个buffer到底是什么，`CrossCallParams`内存空间后又贴了哪些数据。

### `ActualCallParams`

```cpp
// ActualCallParams models an specific IPC call parameters with respect to the
// storage allocation that the packed parameters should need.
// NUMBER_PARAMS: the number of parameters, valid from 1 to N
// BLOCK_SIZE: the total storage that the NUMBER_PARAMS parameters can take,
// typically the block size is defined by the channel size of the underlying
// ipc mechanism.
// In practice this class is used to levergage C++ capacity to properly
// calculate sizes and displacements given the possibility of the packed params
// blob to be complex.
//
// As is, this class assumes that the layout of the blob is as follows. Assume
// that NUMBER_PARAMS = 2 and a 32-bit build:
// 这就是关键之处了，刻画了2个参数情形下的buffer布局
// 
// [ tag                4 bytes]
// [ IsOnOut            4 bytes]	// 这个应该是IsInOut吧。。.
// [ call return       52 bytes]
// [ params count       4 bytes]	// 上面这些就是父类CrossCallParam的64B内存空间
// [ parameter 0 type   4 bytes]	// 三元组就是struct ParamInfo，2表示有3个
// [ parameter 0 offset 4 bytes] ---delta to ---\
// [ parameter 0 size   4 bytes]                |
// [ parameter 1 type   4 bytes]                |
// [ parameter 1 offset 4 bytes] ---------------|--\
// [ parameter 1 size   4 bytes]                |  |
// [ parameter 2 type   4 bytes]                |  |
// [ parameter 2 offset 4 bytes] ----------------------\
// [ parameter 2 size   4 bytes]                |  |   |
// |---------------------------|                |  |   |
// | value 0     (x bytes)     | <--------------/  |   |// 这部分对应真实数据
// | value 1     (y bytes)     | <-----------------/   |
// |                           |                       |
// | end of buffer             | <---------------------/// 最后一个offset表示结尾
// |---------------------------|
//
// Note that the actual number of params is NUMBER_PARAMS + 1
// so that the size of each actual param can be computed from the difference
// between one parameter and the next down. The offset of the last param
// points to the end of the buffer and the type and size are undefined.
// 描述了最后一个ParamInfo的说明
//
// 类模板，常数型，用以在定义ActualCallParams<xxx,xxx>类时指定参数个数与buffer尺寸
template <size_t NUMBER_PARAMS, size_t BLOCK_SIZE>
class ActualCallParams : public CrossCallParams {
 public:
  // constructor. Pass the ipc unique tag as input
  // 以传入的tag和模板常量NUMBER_PARAMS调用父类构造器
  // 像这种单基本型参数的构造器都要声明explicit，防止编译器自作聪明的在某些情况把uint32_t自动转化成ActualCallParams<xxx,xxx>对象
  explicit ActualCallParams(uint32_t tag)
      : CrossCallParams(tag, NUMBER_PARAMS) {
    // 第一个ParamInfo的offset是已知的
    // parameters_数组存储真实参数，它的首地址减去param_info_数组地址就是param_info_[0]的offset
    param_info_[0].offset_ =
        static_cast<uint32_t>(parameters_ - reinterpret_cast<char*>(this));
  }

  // Testing-only constructor. Allows setting the |number_params| to a
  // wrong value.
  // 测试用的构造器，这里没有用模板常量而是使用了传入的参数个数
  ActualCallParams(uint32_t tag, uint32_t number_params)
      : CrossCallParams(tag, number_params) {
    param_info_[0].offset_ =
        static_cast<uint32_t>(parameters_ - reinterpret_cast<char*>(this));
  }

  // Testing-only method. Allows setting the apparent size to a wrong value.
  // returns the previous size.
  // 测试用，修改一个param_info_[someone]的offset
  uint32_t OverrideSize(uint32_t new_size) {
    uint32_t previous_size = param_info_[NUMBER_PARAMS].offset_;
    param_info_[NUMBER_PARAMS].offset_ = new_size;
    return previous_size;
  }

  // Copies each paramter into the internal buffer. For each you must supply:
  // index: 0 for the first param, 1 for the next an so on
  // 把参数拷贝到正确的buffer位置，index表示第几个参数
  bool CopyParamIn(uint32_t index,
                   const void* parameter_address,
                   uint32_t size,
                   bool is_in_out,
                   ArgType type) {
    if (index >= NUMBER_PARAMS) {	// sanity check
      return false;
    }

    if (UINT32_MAX == size) {
      // Memory error while getting the size.
      return false;
    }

    if (size && !parameter_address) {
      return false;
    }

    if ((size > sizeof(*this)) ||
        (param_info_[index].offset_ > (sizeof(*this) - size))) {
      // It does not fit, abort copy.
      return false;
    }

	// 安检通过，找到坑位
    char* dest = reinterpret_cast<char*>(this) + param_info_[index].offset_;

    // We might be touching user memory, this has to be done from inside a try
    // except.
    // copy参数过去
    __try {
      memcpy(dest, parameter_address, size);
    } __except (EXCEPTION_EXECUTE_HANDLER) {
      return false;
    }

    // Set the flag to tell the broker to update the buffer once the call is
    // made.
    // 如果这个参数是输入输出型参数，enable is_in_out_
    if (is_in_out)
      SetIsInOut(true);

	// 常规offset计算与type,size填充
    param_info_[index + 1].offset_ = Align(param_info_[index].offset_ + size);
    param_info_[index].size_ = size;
    param_info_[index].type_ = type;
    return true;
  }

  // Returns a pointer to a parameter in the memory section.
  // get特定参数，返回的是buffer中真实参数的指针
  void* GetParamPtr(size_t index) {
    return reinterpret_cast<char*>(this) + param_info_[index].offset_;
  }

  // Returns the total size of the buffer. Only valid once all the paramters
  // have been copied in with CopyParamIn.
  // 只有当所有参数都通过CopyParamIn拷贝过来后，这个函数返回的才是正确的值。
  // 所以如果我并不copy NUMBER_PARAMS个参数的话，是否会引入某些漏洞？
  uint32_t GetSize() const { return param_info_[NUMBER_PARAMS].offset_; }

 protected:
  // tag为0的构造，这货是protected，对外隐藏
  ActualCallParams() : CrossCallParams(0, NUMBER_PARAMS) {}

 private:
  ParamInfo param_info_[NUMBER_PARAMS + 1];//这个+1说明最后一个边界块不占用参数个数
  char parameters_[BLOCK_SIZE - sizeof(CrossCallParams) -
                   sizeof(ParamInfo) * (NUMBER_PARAMS + 1)];
  DISALLOW_COPY_AND_ASSIGN(ActualCallParams);	//用不到就禁了，以绝后患
};
```

到此，IPC调用传输所用的IN/OUT参数、返回值是如何被安排在buffer上，已经非常清楚了。下面就是对Client和Server两端驱动者的分析。

## Client

老规矩先看看crosscall_client.h头注释的说明：

```cpp
// This header defines the CrossCall(..) family of templated functions
// Their purpose is to simulate the syntax of regular call but to generate
// and IPC from the client-side.
//
// The basic pattern is to
//   1) use template argument deduction to compute the size of each
//      parameter and the appropriate copy method
//   2) pack the parameters in the appropriate ActualCallParams< > object
//   3) call the IPC interface IPCProvider::DoCall( )
//
// The general interface of CrossCall is:
//  ResultCode CrossCall(IPCProvider& ipc_provider,
//                       uint32_t tag,
//                       const Par1& p1, const Par2& p2,...pn
//                       CrossCallReturn* answer)
//
//  where:
//    ipc_provider: is a specific implementation of the ipc transport see
//                  sharedmem_ipc_server.h for an example.
//    tag : is the unique id for this IPC call. Is used to route the call to
//          the appropriate service.
//    p1, p2,.. pn : The input parameters of the IPC. Use only simple types
//                   and wide strings (can add support for others).
//    answer : If the IPC was successful. The server-side answer is here. The
//             interpretation of the answer is private to client and server.
//
// The return value is ALL_OK if the IPC was delivered to the server, other
// return codes indicate that the IPC transport failed to deliver it.
```

归纳一下大概就是：

1. 定义了一套`CrossCall`函数模板，每一个函数模板都对应不同个数的参数。
2. `CrossCall`的使用方式:
   1. 利用模板参数来推断每个参数尺寸的计算以及合适的copy方法
   2. 把参数打包到`ActualCallParams< >`对象
   3. 发起IPC调用`IPCProvider::DoCall()`

### `CrossCall`

可以说是安排的明明白白，那么我们先看一下`CrossCall`全家桶。

```cpp
// CrossCall template with one input parameter
// 一个参数的CrossCall模板
// 模板参数IPCProvider实际上才是IPC机制真正的操纵者，至于它是什么，sandbox用的是哪个Provider，以后再说
template <typename IPCProvider, typename Par1>
ResultCode CrossCall(IPCProvider& ipc_provider,
                     uint32_t tag,
                     const Par1& p1,
                     CrossCallReturn* answer) {
  // 这两个宏有点复杂，一会儿展开
  // 相当于
  // ActualCallParams<1,1024>* call_params = new (ipc_provider.GetBuffer())ActualCallParams<1,1024>(tag);
  // ActualCallParams<>对象是buffer上的抽象，负责具体的参数布局
  // 但buffer内存空间的来源是ipc_provider提供的
  XCALL_GEN_PARAMS_OBJ(1, call_params);
  // 相当于
  // CopyHelper<Par1> ch1(p1); 
  // call_params->CopyParamIn(0,ch1.GetStart(),ch1.GetSize(),ch1.IsInOut(),ch1.GetType())
  // 至于CopyHelper是什么，一会儿再分析，功能上推测是个控制参数拷贝的类模板
  XCALL_GEN_COPY_PARAM(1, call_params);

  // 实际上还是借用了ipc_provider的DoCall，call_params是个ActualCallParam没问题
  // ActualCallParam已经包含了一个CrossCallReturn，这里又传入了一个CrossCallReturn
  // 是传参的answer，call_params在new的时候并没有对内部的CrossCallReturn做什么
  // 所以answer和call_params中的CrossCallReturn的关系现在还不明朗
  ResultCode result = ipc_provider.DoCall(call_params, answer);

  if (SBOX_ERROR_CHANNEL_ERROR != result) {
    // if(!ch1.Update(call_params->GetParamPtr(num - 1)))
    // {ipc_provider.FreeBuffer(ipc_provider.GetBuffer());}
    XCALL_GEN_UPDATE_PARAM(1, call_params);
    // ipc_provider.FreeBuffer(ipc_provider.GetBuffer());
    XCALL_GEN_FREE_CHANNEL();
  }

  return result;
}
```

这几个宏的展开：

```cpp
#define XCALL_GEN_PARAMS_OBJ(num, params)                      \
  typedef ActualCallParams<num, kIPCChannelSize> ActualParams; \
  void* raw_mem = ipc_provider.GetBuffer();                    \
  if (!raw_mem)                                                \
    return SBOX_ERROR_NO_SPACE;                                \
  ActualParams* params = new (raw_mem) ActualParams(tag);

#define XCALL_GEN_COPY_PARAM(num, params)                                  \
  static_assert(kMaxIpcParams >= num, "too many parameters");              \
  CopyHelper<Par##num> ch##num(p##num);                                    \
  if (!params->CopyParamIn(num - 1, ch##num.GetStart(), ch##num.GetSize(), \
                           ch##num.IsInOut(), ch##num.GetType()))          \
    return SBOX_ERROR_NO_SPACE;

#define XCALL_GEN_UPDATE_PARAM(num, params)            \
  if (!ch##num.Update(params->GetParamPtr(num - 1))) { \
    ipc_provider.FreeBuffer(raw_mem);                  \
    return SBOX_ERROR_BAD_PARAMS;                      \
  }

#define XCALL_GEN_FREE_CHANNEL() ipc_provider.FreeBuffer(raw_mem);
```

到此，一个参数的`CrossCall`类模板就清楚了，那么按照` static_assert(kMaxIpcParams >= num, "too many parameters"); `的指示，应该还有2个参数到9个参数的版本。

但观察了一下crosscall_client.h的定义，只发现了2到7个参数的模板，我也不清楚为何定义和最大值常量有出入。

展开看看7个参数的版本吧，实际上只是简单的叠加：

```cpp
// CrossCall template with seven input parameters.
template <typename IPCProvider,
          typename Par1,
          typename Par2,
          typename Par3,
          typename Par4,
          typename Par5,
          typename Par6,
          typename Par7>
ResultCode CrossCall(IPCProvider& ipc_provider,
                     uint32_t tag,
                     const Par1& p1,
                     const Par2& p2,
                     const Par3& p3,
                     const Par4& p4,
                     const Par5& p5,
                     const Par6& p6,
                     const Par7& p7,
                     CrossCallReturn* answer) {
  XCALL_GEN_PARAMS_OBJ(7, call_params);
  XCALL_GEN_COPY_PARAM(1, call_params);
  XCALL_GEN_COPY_PARAM(2, call_params);
  XCALL_GEN_COPY_PARAM(3, call_params);
  XCALL_GEN_COPY_PARAM(4, call_params);
  XCALL_GEN_COPY_PARAM(5, call_params);
  XCALL_GEN_COPY_PARAM(6, call_params);
  XCALL_GEN_COPY_PARAM(7, call_params);

  ResultCode result = ipc_provider.DoCall(call_params, answer);

  if (SBOX_ERROR_CHANNEL_ERROR != result) {
    XCALL_GEN_UPDATE_PARAM(1, call_params);
    XCALL_GEN_UPDATE_PARAM(2, call_params);
    XCALL_GEN_UPDATE_PARAM(3, call_params);
    XCALL_GEN_UPDATE_PARAM(4, call_params);
    XCALL_GEN_UPDATE_PARAM(5, call_params);
    XCALL_GEN_UPDATE_PARAM(6, call_params);
    XCALL_GEN_UPDATE_PARAM(7, call_params);
    XCALL_GEN_FREE_CHANNEL();
  }
  return result;
}
```

### `CopyHelper`

一个类模板，用以推断合适的copy函数来把输入参数拷贝到buffer。

```cpp
// The copy helper uses templates to deduce the appropriate copy function to
// copy the input parameters in the buffer that is going to be send across the
// IPC. These template facility can be made more sophisticated as need arises.

// The default copy helper. It catches the general case where no other
// specialized template matches better. We set the type to UINT32_TYPE, so this
// only works with objects whose size is 32 bits.
// 这个是default模板，除了下面对明确类型的模板类定义以外，其他的都匹配到这里
template <typename T>
class CopyHelper {
 public:
  CopyHelper(const T& t) : t_(t) {}

  // Returns the pointer to the start of the input.
  const void* GetStart() const { return &t_; }

  // Update the stored value with the value in the buffer. This is not
  // supported for this type.
  bool Update(void* buffer) {
    // Not supported;
    // 这个显然对未知类型很危险，不能瞎j8赋值
    return true;
  }

  // Returns the size of the input in bytes.
  uint32_t GetSize() const { return sizeof(T); }

  // Returns true if the current type is used as an In or InOut parameter.
  bool IsInOut() { return false; }

  // Returns this object's type.
  // 这里的处理强制了UINT32_TYPE
  ArgType GetType() {
    static_assert(sizeof(T) == sizeof(uint32_t), "specialization needed");
    return UINT32_TYPE;
  }

 private:
  const T& t_;
};
```

再看几个具体的包装（实际上就是对`ArgType`的每种类型都包装一个），这些才是有实际实用意义的：

```cpp
// This copy helper template specialization if for the void pointer
// case both 32 and 64 bit.
// 这个是T为void*的情景，实际上没做什么实际内容
template <>
class CopyHelper<void*> {
 public:
  CopyHelper(void* t) : t_(t) {}

  // Returns the pointer to the start of the input.
  const void* GetStart() const { return &t_; }

  // Update the stored value with the value in the buffer. This is not
  // supported for this type.
  bool Update(void* buffer) {
    // Not supported;
    return true;
  }

  // Returns the size of the input in bytes.
  uint32_t GetSize() const { return sizeof(t_); }

  // Returns true if the current type is used as an In or InOut parameter.
  bool IsInOut() { return false; }

  // Returns this object's type.
  ArgType GetType() { return VOIDPTR_TYPE; }

 private:
  const void* t_;
};

// This copy helper template specialization catches the cases where the
// parameter is a pointer to a string.
// 这个是const宽字符指针，实际上是字符串
template <>
class CopyHelper<const wchar_t*> {
 public:
  CopyHelper(const wchar_t* t) : t_(t) {}

  // Returns the pointer to the start of the string.
  const void* GetStart() const { return t_; }

  // Update the stored value with the value in the buffer. This is not
  // supported for this type.
  // 都const了，改个毛线
  bool Update(void* buffer) {
    // Not supported;
    return true;
  }

  // Returns the size of the string in bytes. We define a nullptr string to
  // be of zero length.
  // 获取尺寸，这里并不是类型的尺寸，而是宽字符串的整体大小
  uint32_t GetSize() const {
    __try {
      return (!t_) ? 0
                   : static_cast<uint32_t>(StringLength(t_) * sizeof(t_[0]));
    } __except (EXCEPTION_EXECUTE_HANDLER) {
      return UINT32_MAX;
    }
  }

  // Returns true if the current type is used as an In or InOut parameter.
  bool IsInOut() { return false; }

  ArgType GetType() { return WCHAR_TYPE; }

 private:
  // We provide our not very optimized version of wcslen(), since we don't
  // want to risk having the linker use the version in the CRT since the CRT
  // might not be present when we do an early IPC call.
  // 起始就是简单的count，\0结束，包含了\0
  static size_t __cdecl StringLength(const wchar_t* wcs) {
    const wchar_t* eos = wcs;
    while (*eos++)
      ;
    return static_cast<size_t>(eos - wcs - 1);
  }

  const wchar_t* t_;
};

// Specialization for non-const strings. We just reuse the implementation of the
// const string specialization.
// 这个是non-const字符串
template <>
class CopyHelper<wchar_t*> : public CopyHelper<const wchar_t*> {
 public:
  typedef CopyHelper<const wchar_t*> Base;//定义这货是怕编译器混淆，用了自生成类模板类吗？
  CopyHelper(wchar_t* t) : Base(t) {}

  const void* GetStart() const { return Base::GetStart(); }

  bool Update(void* buffer) { return Base::Update(buffer); }

  uint32_t GetSize() const { return Base::GetSize(); }

  bool IsInOut() { return Base::IsInOut(); }

  ArgType GetType() { return Base::GetType(); }
};

// Specialization for wchar_t arrays strings. We just reuse the implementation
// of the const string specialization.
template <size_t n>
class CopyHelper<const wchar_t[n]> : public CopyHelper<const wchar_t*> {
 public:
  typedef const wchar_t array[n];
  typedef CopyHelper<const wchar_t*> Base;
  CopyHelper(array t) : Base(t) {}

  const void* GetStart() const { return Base::GetStart(); }

  bool Update(void* buffer) { return Base::Update(buffer); }

  uint32_t GetSize() const { return Base::GetSize(); }

  bool IsInOut() { return Base::IsInOut(); }

  ArgType GetType() { return Base::GetType(); }
};

// This copy helper template specialization catches the cases where the
// parameter is a an input/output buffer.
// 输入输出型参数比较特殊，用InOutCountedBuffer结构，这个结构一会儿再分析
// 对应ArgType为INOUTPTR_TYPE
template <>
class CopyHelper<InOutCountedBuffer> {
 public:
  CopyHelper(const InOutCountedBuffer t) : t_(t) {}

  // Returns the pointer to the start of the string.
  const void* GetStart() const { return t_.Buffer(); }

  // Updates the buffer with the value from the new buffer in parameter.
  bool Update(void* buffer) {
    // We are touching user memory, this has to be done from inside a try
    // except.
    __try {
      memcpy(t_.Buffer(), buffer, t_.Size());
    } __except (EXCEPTION_EXECUTE_HANDLER) {
      return false;
    }
    return true;
  }

  // Returns the size of the string in bytes. We define a nullptr string to
  // be of zero length.
  uint32_t GetSize() const { return t_.Size(); }

  // Returns true if the current type is used as an In or InOut parameter.
  bool IsInOut() { return true; }

  ArgType GetType() { return INOUTPTR_TYPE; }

 private:
  const InOutCountedBuffer t_;
};
```

展开`InOutCountedBuffer`，实际上是一个对指针的包装类，描述了指针和指向buffer的尺寸：

```cpp
// Generic encapsulation class containing a pointer to a buffer and the
// size of the buffer. It is used by the IPC to be able to pass in/out
// parameters.
class InOutCountedBuffer : public CountedBuffer {
 public:
  InOutCountedBuffer(void* buffer, uint32_t size)
      : CountedBuffer(buffer, size) {}
};

// Encapsulates a pointer to a buffer and the size of the buffer.
class CountedBuffer {
 public:
  CountedBuffer(void* buffer, uint32_t size) : size_(size), buffer_(buffer) {}

  uint32_t Size() const { return size_; }

  void* Buffer() const { return buffer_; }

 private:
  uint32_t size_;
  void* buffer_;
};
```

## Server

再看看server的描述：

```cpp
// This is the IPC server interface for CrossCall: The  IPC for the Sandbox
// On the server, CrossCall needs two things:
// 1) threads: Or better said, someone to provide them, that is what the
//             ThreadProvider interface is defined for. These thread(s) are
//             the ones that will actually execute the  IPC data retrieval.
//
// 2) a dispatcher: This interface represents the way to route and process
//                  an  IPC call given the  IPC tag.
//
// The other class included here CrossCallParamsEx is the server side version
// of the CrossCallParams class of /sandbox/crosscall_params.h The difference
// is that the sever version is paranoid about the correctness of the IPC
// message and will do all sorts of verifications.
//
// A general diagram of the interaction is as follows:
//
//                                 ------------
//                                 |          |
//  ThreadProvider <--(1)Register--|  IPC     |
//      |                          | Implemen |
//      |                          | -tation  |
//     (2)                         |          |  OnMessage
//     IPC fired --callback ------>|          |--(3)---> Dispatcher
//                                 |          |
//                                 ------------
//
//  The  IPC implementation sits as a middleman between the handling of the
//  specifics of scheduling a thread to service the  IPC and the multiple
//  entities that can potentially serve each particular IPC.
```

大致梳理一下：

1. IPC的server端需要完成两件事：
   1. 需要一个provide IPC调用的线程池，provider每次收到IPC call就assign到一个thread来处理
   2. 一个分发器：基于IPC tag来引导、处理每一种IPC call
2. 图示已经描摹出了整个状态机，看起来Server借助了两个外部组件：”线程池供给者ThreadProvider“+"消息处理器Dispatcher"

说白了就是每来一个消息就用一个线程来处理，实现并发性，而消息根据tag在Dispatcher的OnMessage中分门别类，找到它自己的回调处理Handler。

### ThreadProvider

```cpp
// ThreadProvider models a thread factory. The idea is to decouple thread
// creation and lifetime from the inner guts of the IPC. The contract is
// simple:
//   - the IPC implementation calls RegisterWait with a waitable object that
//     becomes signaled when an IPC arrives and needs to be serviced.
//   - when the waitable object becomes signaled, the thread provider conjures
//     a thread that calls the callback (CrossCallIPCCallback) function
//   - the callback function tries its best not to block and return quickly
//     and should not assume that the next callback will use the same thread
//   - when the callback returns the ThreadProvider owns again the thread
//     and can destroy it or keep it around.
// simple已经描述的很清楚了，IPC实现体会利用RegisterWait来注册一个可等待对象，一旦IPC
// 请求到来，那么该对象signaled，ThreadProvider这个线程工厂会生成一个线程来调用CrossCallIPCCallback
// CrossCallIPCCallback很快回来，ThreadProvider再次控制该线程，可以销毁也可以保持
// 前后两次执行callback的不一定是同一个线程
// 那么CrossCallIPCCallback是个什么样的callback呢？实际上是个函数指针类型定义：

// This function signature is required as the callback when an  IPC call fires.
// context: a user-defined pointer that was set using  ThreadProvider
// reason: 0 if the callback was fired because of a timeout.
//         1 if the callback was fired because of an event.
typedef void(__stdcall* CrossCallIPCCallback)(void* context,
                                              unsigned char reason);
class ThreadProvider {
 public:
  // Registers a waitable object with the thread provider.
  // client: A number to associate with all the RegisterWait calls, typically
  //         this is the address of the caller object. This parameter cannot
  //         be zero.
  // waitable_object : a kernel object that can be waited on
  // callback: a function pointer which is the function that will be called
  //           when the waitable object fires
  // context: a user-provider pointer that is passed back to the callback
  //          when its called
  // client作为标志把waitable_object与callback绑定，当waitable_object signaled（IPC arrive），调用callback
  virtual bool RegisterWait(const void* client,
                            HANDLE waitable_object,
                            CrossCallIPCCallback callback,
                            void* context) = 0;

  // Removes all the registrations done with the same cookie parameter.
  // This frees internal thread pool resources.
  virtual bool UnRegisterWaits(void* cookie) = 0;
  virtual ~ThreadProvider() {}
};
```

但这只是个抽象基类，实际上windows应该是靠`Win2kThreadPool`这个内存池类来操作的。而线程池本身是个非常复杂的东西，本身也不属于sandbox的范畴，日后有空的时候再分析一下。

### `CrossCallParamsEx`

```cpp
// Models the server-side of the original input parameters.
// Provides IPC buffer validation and it is capable of reading the parameters
// out of the IPC buffer.
// CrossCallParams的另一个子类，用在server
// 模拟server端原始输入参数的处理，提供了IPC buffer的检查并copy到另一个对象结构
class CrossCallParamsEx : public CrossCallParams {
 public:
  // Factory constructor. Pass an IPCbuffer (and buffer size) that contains a
  // pending IPCcall. This constructor will:
  // 1) validate the IPC buffer. returns nullptr is the IPCbuffer is malformed.
  // 2) make a copy of the IPCbuffer (parameter capture)
  // 颇为关键的static工厂方法，验证IPC buffer并拷贝参数
  static CrossCallParamsEx* CreateFromBuffer(void* buffer_base,
                                             uint32_t buffer_size,
                                             uint32_t* output_size);

  // Provides IPCinput parameter raw access:
  // index : the parameter to read; 0 is the first parameter
  // returns nullptr if the parameter is non-existent. If it exists it also
  // returns the size in *size
  // IPCinput参数的各种形态原生访问方法
  void* GetRawParameter(uint32_t index, uint32_t* size, ArgType* type);

  // Gets a parameter that is four bytes in size.
  // Returns false if the parameter does not exist or is not 32 bits wide.
  bool GetParameter32(uint32_t index, uint32_t* param);

  // Gets a parameter that is void pointer in size.
  // Returns false if the parameter does not exist or is not void pointer sized.
  bool GetParameterVoidPtr(uint32_t index, void** param);

  // Gets a parameter that is a string. Returns false if the parameter does not
  // exist.
  bool GetParameterStr(uint32_t index, base::string16* string);

  // Gets a parameter that is an in/out buffer. Returns false is the parameter
  // does not exist or if the size of the actual parameter is not equal to the
  // expected size.
  bool GetParameterPtr(uint32_t index, uint32_t expected_size, void** pointer);

  // Frees the memory associated with the IPC parameters.
  static void operator delete(void* raw_memory) throw();

 private:
  // Only the factory method CreateFromBuffer can construct these objects.
  CrossCallParamsEx();//CrossCallParamsEx对象必须从static工厂方法中make

  ParamInfo param_info_[1];	//ParamInfo是通用的(type,offset,size)三元组，熟悉的味道
  DISALLOW_COPY_AND_ASSIGN(CrossCallParamsEx);//用不到就禁了，防呆
};
```

这个类就是解包client IPC请求的，工厂方法相当关键：

```cpp
// This function uses a SEH try block so cannot use C++ objects that
// have destructors or else you get Compiler Error C2712. So no DCHECKs
// inside this function.
CrossCallParamsEx* CrossCallParamsEx::CreateFromBuffer(void* buffer_base,
                                                       uint32_t buffer_size,
                                                       uint32_t* output_size) {
  // IMPORTANT: Everything inside buffer_base and derived from it such
  // as param_count and declared_size is untrusted.
  // 心智检查
  if (!buffer_base)
    return nullptr;
  if (buffer_size < sizeof(CrossCallParams))
    return nullptr;
  if (buffer_size > kMaxBufferSize)	//就是1024，IPC Channel实现体目前的硬编码最大尺寸
    return nullptr;

  char* backing_mem = nullptr;
  uint32_t param_count = 0;
  uint32_t declared_size;
  uint32_t min_declared_size;
  CrossCallParamsEx* copied_params = nullptr;

  // Touching the untrusted buffer is done under a SEH try block. This
  // will catch memory access violations so we don't crash.
  __try {
    CrossCallParams* call_params =
        reinterpret_cast<CrossCallParams*>(buffer_base);
    //传入的应该是client的ActualCallParams，call_params父类指针指向子类对象

    // 过安检，CrossCallParams+((param_count + 1) * sizeof(ParamInfo))是除了真实参数数据以外至少需要的空间
    // Check against the minimum size given the number of stated params
    // if too small we bail out.
    param_count = call_params->GetParamsCount();
    min_declared_size =
        sizeof(CrossCallParams) + ((param_count + 1) * sizeof(ParamInfo));

    // Initial check for the buffer being big enough to determine the actual
    // buffer size.
    // 如果buffer_size比min_declared_size还小，说明这段数据是有问题的，显然不能继续解析了
    if (buffer_size < min_declared_size)
      return nullptr;

    // Retrieve the declared size which if it fails returns 0.
    // 不管参数个数有几个，参数总尺寸是可以计算的
    // 参数总尺寸的计算其实就是ParamInfo[]的parser，一会儿展开看
    declared_size = GetActualBufferSize(param_count, buffer_base);

    // 判断一下buffer_size，buffer_size理应>=declared_size而declared_size理应>=min_declared_size
    // 这个函数很简单
    if (!IsSizeWithinRange(buffer_size, min_declared_size, declared_size))
      return nullptr;

    // 这里就进行解包了，移花接木给copied_params，CrossCallParamsEx对象是在这里new的
    // Now we copy the actual amount of the message.
    *output_size = declared_size;
    backing_mem = new char[declared_size];
    // 依然是这种间接new操作，因为buffer尺寸是已知的，但CrossCallParamsEx并没有包含所有的
    // buffer数据（真实数据由ParamInfo定位，附加在类的正后方），所以要用间接的方式把这段
    // 内存空间看成CrossCallParamsEx + 额外的真实参数数据
    copied_params = reinterpret_cast<CrossCallParamsEx*>(backing_mem);
    // 这个memcpy实际上非常讲究，把ActualCallParams + 额外真实参数数据直接copy到了另外
    // 一个对象CrossCallParamsEx+额外真实参数数据
    // 这意味着CrossCallParamsEx和ActualCallParams有着相同的内存布局，事实上也确实如此
    // 都是CrossCallParams基类+ParamInfo[] flexible数据的结构
    // 在server和client各封装这样一个成员变量布局相同的类，主要是因为布局理应相同，但
    // 要用到的接口函数是互逆的
    memcpy(backing_mem, call_params, declared_size);

    // Avoid compiler optimizations across this point. Any value stored in
    // memory should be stored for real, and values previously read from memory
    // should be actually read.
    // 内存屏障，防止此处的编译优化，所有内存的值必须用存储的真实值，此前从memory中读出的高速缓存不能在寄存器中直接复用
    base::subtle::MemoryBarrier();

    // 内存屏障是为这一句准备的我懂，但我不明白为啥要重算
    // 有大神了解的话还请解惑
    min_declared_size =
        sizeof(CrossCallParams) + ((param_count + 1) * sizeof(ParamInfo));

    // Check that the copied buffer is still valid.
    // 可以看出server对input param的检查近乎严苛
    // 还要检查copy过去后buffer是否符合预期
    if (copied_params->GetParamsCount() != param_count ||
        GetActualBufferSize(param_count, backing_mem) != declared_size ||
        !IsSizeWithinRange(buffer_size, min_declared_size, declared_size)) {
      delete[] backing_mem;
      return nullptr;
    }

  } __except (EXCEPTION_EXECUTE_HANDLER) {
    // In case of a windows exception we know it occurred while touching the
    // untrusted buffer so we bail out as is.
    delete[] backing_mem;
    return nullptr;
  } // 内存的访问很可能因畸形或恶意构造IPCInput出现access violations，这里套上了try块，windows对应SEH

  const char* last_byte = &backing_mem[declared_size];
  const char* first_byte = &backing_mem[min_declared_size];

  // last_byte和first_byte之间就是真实参数数据区间了
  // Verify here that all and each parameters make sense. This is done in the
  // local copy.
  // 检查每个parameter都有意义，可以说是相当的严格
  for (uint32_t ix = 0; ix != param_count; ++ix) {
    uint32_t size = 0;
    ArgType type;
    char* address = reinterpret_cast<char*>(
        //一会儿展开看这货
        copied_params->GetRawParameter(ix, &size, &type));
    if ((!address) ||                                     // No null params.
        (INVALID_TYPE >= type) || (LAST_TYPE <= type) ||  // Unknown type.
        (address < backing_mem) ||         // Start cannot point before buffer.
        (address < first_byte) ||          // Start cannot point too low.
        (address > last_byte) ||           // Start cannot point past buffer.
        ((address + size) < address) ||    // Invalid size.
        ((address + size) > last_byte)) {  // End cannot point past buffer.
      // Malformed.
      delete[] backing_mem;
      return nullptr;
    }
  }
  // The parameter buffer looks good.
  return copied_params;
}
```

工厂只是做了移花接木的操作。

看看几个关键的函数：

```cpp
// Returns the actual size for the parameters in an IPC buffer. Returns
// zero if the |param_count| is zero or too big.
// 又见呆逼操作，不定参数个数的匹配处理在这里完成，不管是几个参数，返回的就是参数总尺寸
// 这里又变成1-9个了，怕是client端少写了两个，然后一直用不到？
uint32_t GetActualBufferSize(uint32_t param_count, void* buffer_base) {
  // The template types are used to calculate the maximum expected size.
  // kMaxBufferSize是硬编码的sandbox::kIPCChannelSize，也就是1024
  // 把9种模板类都做一下typedef
  typedef ActualCallParams<1, kMaxBufferSize> ActualCP1;
  typedef ActualCallParams<2, kMaxBufferSize> ActualCP2;
  typedef ActualCallParams<3, kMaxBufferSize> ActualCP3;
  typedef ActualCallParams<4, kMaxBufferSize> ActualCP4;
  typedef ActualCallParams<5, kMaxBufferSize> ActualCP5;
  typedef ActualCallParams<6, kMaxBufferSize> ActualCP6;
  typedef ActualCallParams<7, kMaxBufferSize> ActualCP7;
  typedef ActualCallParams<8, kMaxBufferSize> ActualCP8;
  typedef ActualCallParams<9, kMaxBufferSize> ActualCP9;

  // Retrieve the actual size and the maximum size of the params buffer.
  // 根据参数个数，可以判断出是哪一个模板类
  switch (param_count) {
    case 0:
      return 0;
    case 1:
      return reinterpret_cast<ActualCP1*>(buffer_base)->GetSize();
    case 2:
      return reinterpret_cast<ActualCP2*>(buffer_base)->GetSize();
    case 3:
      return reinterpret_cast<ActualCP3*>(buffer_base)->GetSize();
    case 4:
      return reinterpret_cast<ActualCP4*>(buffer_base)->GetSize();
    case 5:
      return reinterpret_cast<ActualCP5*>(buffer_base)->GetSize();
    case 6:
      return reinterpret_cast<ActualCP6*>(buffer_base)->GetSize();
    case 7:
      return reinterpret_cast<ActualCP7*>(buffer_base)->GetSize();
    case 8:
      return reinterpret_cast<ActualCP8*>(buffer_base)->GetSize();
    case 9:
      return reinterpret_cast<ActualCP9*>(buffer_base)->GetSize();
    default:
      return 0;
  }
}
```

`GetSize`实际上就是client端`ActualCallParams<>`的接口，我们再次看一下：

```cpp
uint32_t GetSize() const { return param_info_[NUMBER_PARAMS].offset_; }
// 所以想要计算尺寸，只需要获取最后一个ParamInfo的offset即可，这个就是最后一个end struct
```

再看简单的值范围检查：

```cpp
// Verifies that the declared sizes of an IPC buffer are within range.
bool IsSizeWithinRange(uint32_t buffer_size,
                       uint32_t min_declared_size,
                       uint32_t declared_size) {
  if ((buffer_size < min_declared_size) ||
      (sizeof(CrossCallParamsEx) > min_declared_size)) {
    // Minimal computed size bigger than existing buffer or param_count
    // integer overflow.
    return false;
  }

  if ((declared_size > buffer_size) || (declared_size < min_declared_size)) {
    // Declared size is bigger than buffer or smaller than computed size
    // or param_count is equal to 0 or bigger than 9.
    return false;
  }

  return true;
}
```

最后一个关键的`GetRawParameter`：

```cpp
// Accessors to the parameters in the raw buffer.
void* CrossCallParamsEx::GetRawParameter(uint32_t index,
                                         uint32_t* size,
                                         ArgType* type) {
  // 不能越界
  if (index >= GetParamsCount())
    return nullptr;
  // The size is always computed from the parameter minus the next
  // parameter, this works because the message has an extra parameter slot
  // 拿到该参数的尺寸和类型
  *size = param_info_[index].size_;
  *type = param_info_[index].type_;

  // 返回该参数的真实数据所在的位置
  return param_info_[index].offset_ + reinterpret_cast<char*>(this);
}
```

也是非常简单的函数。

### `Diapatcher`

`CrossCallParamsEx`不过是个承载buffer的容器，真正的驱动者另有其人。在crosscall_server.h中可以找到`Dispatcher`类，根据头的注释可以知道它时IPC消息的操纵者。

```cpp
// Models an entity that can process an IPC message or it can route to another
// one that could handle it. When an IPC arrives the IPC implementation will:
// 1) call OnMessageReady() with the tag of the pending IPC. If the dispatcher
//    returns nullptr it means that it cannot handle this IPC but if it returns
//    non-null, it must be the pointer to a dispatcher that can handle it.
// 2) When the  IPC finally obtains a valid Dispatcher the IPC
//    implementation creates a CrossCallParamsEx from the raw IPC buffer.
// 3) It calls the returned callback, with the IPC info and arguments.
// 所以IPC请求给到IPC实现体时，会先使用tag调用Dispatcher::OnMessageReady()。
// Dispatcher维护了一组callback，如果该IPC与其中某一个匹配的话，就表示该Dispatcher
// 可以处理该IPC调用
// 找到以后，IPC实现体会创建一个CrossCallParamsEx，从buffer拷贝数据
// 然后，以IPC info和args调用对应的callback
class Dispatcher {
 public:
  // Called from the  IPC implementation to handle a specific IPC message.
  // 又是这种笨拙的函数指针定义，根据参数个数的多少，定义callback多种形态
  typedef bool (Dispatcher::*CallbackGeneric)();
  typedef bool (Dispatcher::*Callback0)(IPCInfo* ipc);
  typedef bool (Dispatcher::*Callback1)(IPCInfo* ipc, void* p1);
  typedef bool (Dispatcher::*Callback2)(IPCInfo* ipc, void* p1, void* p2);
  typedef bool (Dispatcher::*Callback3)(IPCInfo* ipc,
                                        void* p1,
                                        void* p2,
                                        void* p3);
  typedef bool (Dispatcher::*Callback4)(IPCInfo* ipc,
                                        void* p1,
                                        void* p2,
                                        void* p3,
                                        void* p4);
  typedef bool (Dispatcher::*Callback5)(IPCInfo* ipc,
                                        void* p1,
                                        void* p2,
                                        void* p3,
                                        void* p4,
                                        void* p5);
  typedef bool (Dispatcher::*Callback6)(IPCInfo* ipc,
                                        void* p1,
                                        void* p2,
                                        void* p3,
                                        void* p4,
                                        void* p5,
                                        void* p6);
  typedef bool (Dispatcher::*Callback7)(IPCInfo* ipc,
                                        void* p1,
                                        void* p2,
                                        void* p3,
                                        void* p4,
                                        void* p5,
                                        void* p6,
                                        void* p7);
  typedef bool (Dispatcher::*Callback8)(IPCInfo* ipc,
                                        void* p1,
                                        void* p2,
                                        void* p3,
                                        void* p4,
                                        void* p5,
                                        void* p6,
                                        void* p7,
                                        void* p8);
  typedef bool (Dispatcher::*Callback9)(IPCInfo* ipc,
                                        void* p1,
                                        void* p2,
                                        void* p3,
                                        void* p4,
                                        void* p5,
                                        void* p6,
                                        void* p7,
                                        void* p8,
                                        void* p9);

  // Called from the  IPC implementation when an  IPC message is ready override
  // on a derived class to handle a set of  IPC messages. Return nullptr if your
  // subclass does not handle the message or return the pointer to the subclass
  // that can handle it.
  // IPC消息到来时，IPC实现体先调用这个MessageReady事件响应，看看是否有能力handle
  virtual Dispatcher* OnMessageReady(IPCParams* ipc, CallbackGeneric* callback);

  // Called when a target proces is created, to setup the interceptions related
  // with the given service (IPC).
  // 当target进程创建时，部署与给定服务（IPC）相关的interceptions
  // 这个暂时不关心，涉及到它的上层组件Interception机制
  virtual bool SetupService(InterceptionManager* manager, int service) = 0;

  Dispatcher();
  virtual ~Dispatcher();

 protected:
  // Structure that defines an IPC Call with all the parameters and the handler.
  struct IPCCall {
    IPCParams params;
    CallbackGeneric callback;
  };

  // List of IPC Calls supported by the class.
  // 一个Dispatcher所支持的IPC调用列表，这个结构在内部定义使用
  // 封装了IPCParams和CallbackGeneric
  std::vector<IPCCall> ipc_calls_;
};
```

看看这几个结构体：

```cpp
// Represents the client process that initiated the IPC which boils down to the
// process handle and the job object handle that contains the client process.
struct ClientInfo {
  HANDLE process;
  DWORD process_id;
};

// All IPC-related information to be passed to the IPC handler.
// 一组callback的第一个参数，分组了tag，进程相关信息以及一个CrossCallReturn
struct IPCInfo {
  int ipc_tag;
  const ClientInfo* client_info;
  CrossCallReturn return_info;
};

// This structure identifies IPC signatures.
struct IPCParams {
  int ipc_tag;
  ArgType args[kMaxIpcParams];

  bool Matches(IPCParams* other) const {
    return !memcmp(this, other, sizeof(*other));//这个IPCParams的对比有点粗暴
  }
};
```

看一下`OnMessageReady`的实现：

```cpp
Dispatcher* Dispatcher::OnMessageReady(IPCParams* ipc,
                                       CallbackGeneric* callback) {
  DCHECK(callback);
  std::vector<IPCCall>::iterator it = ipc_calls_.begin();
  for (; it != ipc_calls_.end(); ++it) {
    //IPCCall的第一个成员是IPCParams，所以可以直接调用IPCParams的Matches
    if (it->params.Matches(ipc)) {
      *callback = it->callback;	//如果可以处理该ipc，就填充callback，这种callback是个CallbackGeneric类型
      return this;
    }
  }
  return nullptr;
}
```

很明显，`Dispatcher`也只是个基类。`ipc_calls_`应该是派生的子类中某个setup方法中填充的，而父类的构造和析构都是空函数。除此之外，基类定义的不过是一组typedef，原材料都已准备好了，但依然找不到使用者。

显然`Dispatcher`也不过是架在`CrossCallParamsEx`之上的adapter。那么`Dispatcher`又是由谁来操纵呢？我们下回继续求索。