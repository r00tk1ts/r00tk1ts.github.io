---
title: Chromium-sandbox-PolicyEngine-analysis
date: 2018-05-26 11:48:11
categories: 源码剖析
tags:
	- chromium
	- chromium-sandbox
---

本篇是sandbox源码剖析的第十二篇，主要分析了windows平台下，Chromium sandbox中子系统三大组件构成中的第三大组件——low-level policy的基础设施PolicyEngine。阅读本篇前，请先阅读前四篇。

想要流程的阅读本系列你需要以下几个条件：
1. 较高水平的C++编码能力（至少通读C++ Primer 5th，刷过课后题，有一定编码量）。
2. 熟悉Windows API编程，尤其是安全相关的内容。
3. 对二进制安全有一定了解，熟悉各类型安全漏洞成因、漏洞缓解措施及bypass手法。

<!--more-->

# chromium-sandbox-PolicyEngine-analysis

子系统共有三大组件：负责IPC请求分发的Dispatcher、对敏感函数拦截的Interception+Resolver、low-level-policy。此前已经分析过前两个组件，本篇开始分析low-level-policy，来看看low-level-policy究竟是什么。

定位源代码时，发现low-level-policy这套机制是建立在policy_engine之上的，所以分析low-level-policy之前，首先要搞懂policy_engine的方方面面。

## opcodes

policy_engine由两部分组成：opcodes和processor。先研究一下opcodes。

何为opcodes？实际上头文件给出的功能描述已经相当详尽了。

```cpp
// The low-level policy is implemented using the concept of policy 'opcodes'.
// An opcode is a structure that contains enough information to perform one
// comparison against one single input parameter. For example, an opcode can
// encode just one of the following comparison:
// low-level policy是基于opcodes概念实现的
// opcode是一种包含了充分信息的结构用来与单一输入参数进行比较
// 下面给出了几个比较的范例：
// 
// - Is input parameter 3 not equal to nullptr?
// - Does input parameter 2 start with L"c:\\"?
// - Is input parameter 5, bit 3 is equal 1?
// 看起来opcode可以裁决某个输入参数的值是否是理想值
// 
// Each opcode is in fact equivalent to a function invocation where all
// the parameters are known by the opcode except one. So say you have a
// function of this form:
//      bool fn(a, b, c, d)  with 4 arguments
//
// Then an opcode is:
//      op(fn, b, c, d)
// Which stores the function to call and its 3 last arguments
//
// Then and opcode evaluation is:
//      op.eval(a)  ------------------------> fn(a,b,c,d)
//                        internally calls
//
// The idea is that complex policy rules can be split into streams of
// opcodes which are evaluated in sequence. The evaluation is done in
// groups of opcodes that have N comparison opcodes plus 1 action opcode:
//
// [comparison 1][comparison 2]...[comparison N][action][comparison 1]...
//    ----- evaluation order----------->
//
// Each opcode group encodes one high-level policy rule. The rule applies
// only if all the conditions on the group evaluate to true. The action
// opcode contains the policy outcome for that particular rule.
//
// Note that this header contains the main building blocks of low-level policy
// but not the low level policy class.
```

尽管描述详尽，但仅凭现有的信息和猜想，还无法理解如此复杂的设计，我们继续往下看。

头文件中定义了几个关键的结构体和常量，以及两个非常重要的类：`OpcodePolicy`和`OpcodeFactory`。

### structure & const

```cpp
// These are the possible policy outcomes. Note that some of them might
// not apply and can be removed. Also note that The following values only
// specify what to do, not how to do it and it is acceptable given specific
// cases to ignore the policy outcome.
// 仅仅表示应该怎样处理，但并未规范如何处理。对于特殊情景，可以不采纳输出的结果。
enum EvalResult {
  // Comparison opcode values:
  // 参考头注释的说明，group是以多个comparison+一个action组成
  // 这三个result是comparison的结果
  EVAL_TRUE,   // Opcode condition evaluated true.
  EVAL_FALSE,  // Opcode condition evaluated false.
  EVAL_ERROR,  // Opcode condition generated an error while evaluating.
  // Action opcode values:
  // 这些都是action的值，表示裁决的行动
  ASK_BROKER,   // The target must generate an IPC to the broker. On the broker
                // side, this means grant access to the resource.
  DENY_ACCESS,  // No access granted to the resource.
  GIVE_READONLY,   // Give readonly access to the resource.
  GIVE_ALLACCESS,  // Give full access to the resource.
  GIVE_CACHED,     // IPC is not required. Target can return a cached handle.
  GIVE_FIRST,      // TODO(cpu)
  SIGNAL_ALARM,    // Unusual activity. Generate an alarm.
  FAKE_SUCCESS,    // Do not call original function. Just return 'success'.
  FAKE_ACCESS_DENIED,  // Do not call original function. Just return 'denied'
                       // and do not do IPC.
  TERMINATE_PROCESS,   // Destroy target process. Do IPC as well.
};
```

根据`EvalResult`的枚举值，能够猜想到comparison是一次比较，但comparison本身是怎样的结构，还不清楚。而action则是最终的审判。

```cpp
// The following are the implemented opcodes.
enum OpcodeID {
  // 前6个都是具体的比较方式
  OP_ALWAYS_FALSE,        // Evaluates to false (EVAL_FALSE).
  OP_ALWAYS_TRUE,         // Evaluates to true (EVAL_TRUE).
  OP_NUMBER_MATCH,        // Match a 32-bit integer as n == a.
  OP_NUMBER_MATCH_RANGE,  // Match a 32-bit integer as a <= n <= b.
  OP_NUMBER_AND_MATCH,    // Match using bitwise AND; as in: n & a != 0.
  OP_WSTRING_MATCH,       // Match a string for equality.
  // 后1个应该是标志action所用
  OP_ACTION               // Evaluates to an action opcode.
};
```

comparison看起来会是一种具体的比较类别，各不相同，要么返回FALSE/TRUE，要么比较`int`、`string`值等等。

```cpp
// 下面的几个应该都是标志位，从某个角度来影响裁决结果或过程
// Options that apply to every opcode. They are specified when creating
// each opcode using OpcodeFactory::MakeOpXXXXX() family of functions
// Do nothing special.
const uint32_t kPolNone = 0;

// Convert EVAL_TRUE into EVAL_FALSE and vice-versa. This allows to express
// negated conditions such as if ( a && !b).
const uint32_t kPolNegateEval = 1;

// Zero the MatchContext context structure. This happens after the opcode
// is evaluated.
const uint32_t kPolClearContext = 2;

// Use OR when evaluating this set of opcodes. The policy evaluator by default
// uses AND when evaluating. Very helpful when
// used with kPolNegateEval. For example if you have a condition best expressed
// as if(! (a && b && c)), the use of this flags allows it to be expressed as
// if ((!a) || (!b) || (!c)).
const uint32_t kPolUseOREval = 4;
```

一个专门为连续字符串匹配定制的专属结构体：

```cpp
// Keeps the evaluation state between opcode evaluations. This is used
// for string matching where the next opcode needs to continue matching
// from the last character position from the current opcode. The match
// context is preserved across opcode evaluation unless an opcode specifies
// as an option kPolClearContext.
struct MatchContext {
  size_t position;
  uint32_t options;

  MatchContext() { Clear(); }

  void Clear() {
    position = 0;
    options = 0;
  }
};
```

看起来`kPolClearContext`这个标记和连续匹配字符串有关。

### `PolicyOpcode`

```cpp
// Models a policy opcode; that is a condition evaluation were all the
// arguments but one are stored in objects of this class. Use OpcodeFactory
// to create objects of this type.
// policy opcode是基于此运营的，除了第一个参数都存储在此对象中进行裁决。
// OpcodeFactory工厂类负责创建该类型对象。
// 
// This class is just an implementation artifact and not exposed to the
// API clients or visible in the intercepted service. Internally, an
// opcode is just:
//  - An integer that identifies the actual opcode.
//  - An index to indicate which one is the input argument
//  - An array of arguments.
// 该类仅仅是抽象体，不对外暴露（client/service）。
// 操作码的三个组成
// - opcode的id
// - 一个index索引表示哪个是输入参数
// - 一个参数数组
// 
// While an OO hierarchy of objects would have been a natural choice, the fact
// that 1) this code can execute before the CRT is loaded, presents serious
// problems in terms of guarantees about the actual state of the vtables and
// 2) because the opcode objects are generated in the broker process, we need to
// use plain objects. To preserve some minimal type safety templates are used
// when possible.
class PolicyOpcode {
  friend class OpcodeFactory;

 public:
  // Evaluates the opcode. For a typical comparison opcode the return value
  // is EVAL_TRUE or EVAL_FALSE. If there was an error in the evaluation the
  // the return is EVAL_ERROR. If the opcode is an action opcode then the
  // return can take other values such as ASK_BROKER.
  // 这个函数就是裁决的执行，根据opcode是comparison还是action，返回两类结果
  //
  // parameters: An array of all input parameters. This argument is normally
  // created by the macros POLPARAMS_BEGIN() POLPARAMS_END.
  // count: The number of parameters passed as first argument.
  // match: The match context that is persisted across the opcode evaluation
  // sequence.
  // 可以看到参数数组是个ParameterSet结构，这个结构在policy_engine_params.h中定义
  // 一会儿展开看看。count表示参数个数，match是一个辅助结构，用于支持连续字符串匹配用的
  EvalResult Evaluate(const ParameterSet* parameters,
                      size_t count,
                      MatchContext* match);
	
  // 一目了然的三个set/get方法
  // Retrieves a stored argument by index. Valid index values are
  // from 0 to < kArgumentCount.
  template <typename T>
  void GetArgument(size_t index, T* argument) const {
    static_assert(sizeof(T) <= sizeof(arguments_[0]), "invalid size");
    *argument = *reinterpret_cast<const T*>(&arguments_[index].mem);
  }

  // Sets a stored argument by index. Valid index values are
  // from 0 to < kArgumentCount.
  template <typename T>
  void SetArgument(size_t index, const T& argument) {
    static_assert(sizeof(T) <= sizeof(arguments_[0]), "invalid size");
    *reinterpret_cast<T*>(&arguments_[index].mem) = argument;
  }

  // Retrieves the actual address of an string argument. When using
  // GetArgument() to retrieve an index that contains a string, the returned
  // value is just an offset to the actual string.
  // index: the stored string index. Valid values are from 0
  // to < kArgumentCount.
  const wchar_t* GetRelativeString(size_t index) const {
    ptrdiff_t str_delta = 0;
    GetArgument(index, &str_delta);
    // 字符串的GetArgument返回的是一个offset
    const char* delta = reinterpret_cast<const char*>(this) + str_delta;
    return reinterpret_cast<const wchar_t*>(delta);
  }

  // OpcodeID相关
  // Returns true if this opcode is an action opcode without actually
  // evaluating it. Used to do a quick scan forward to the next opcode group.
  bool IsAction() const { return (OP_ACTION == opcode_id_); }

  // Returns the opcode type.
  OpcodeID GetID() const { return opcode_id_; }

  // Returns the stored options such as kPolNegateEval and others.
  uint32_t GetOptions() const { return options_; }

  // Sets the stored options such as kPolNegateEval.
  void SetOptions(uint32_t options) {
    options_ = base::checked_cast<uint16_t>(options);
  }

 private:
  static const size_t kArgumentCount = 4;  // The number of supported argument.

  struct OpcodeArgument {
    UINT_PTR mem;
  };

  // Better define placement new in the class instead of relying on the
  // global definition which seems to be fubared.
  void* operator new(size_t, void* location) { return location; }

  // Helper function to evaluate the opcode. The parameters have the same
  // meaning that in Evaluate().
  // 这个应该是真正的裁决处理helper函数
  EvalResult EvaluateHelper(const ParameterSet* parameters,
                            MatchContext* match);
  OpcodeID opcode_id_;
  int16_t parameter_;	
  // TODO(cpu): Making |options_| a uint32_t would avoid casting, but causes
  // test failures.  Somewhere code is relying on the size of this struct.
  // http://crbug.com/420296
  uint16_t options_;
  OpcodeArgument arguments_[PolicyOpcode::kArgumentCount]; // 其实就是4个指针，供参数get/set用
};
```

除了`Evaluate`以外，其他基本都在类头定义了。暂且不管`ParamterSet`参数，看看处理的逻辑：

```cpp
//////////////////////////////////////////////////////////////////////////////
// Opcode evaluation dispatchers.

// This function is the one and only entry for evaluating any opcode. It is
// in charge of applying any relevant opcode options and calling EvaluateInner
// were the actual dispatch-by-id is made. It would seem at first glance that
// the dispatch should be done by virtual function (vtable) calls but you have
// to remember that the opcodes are made in the broker process and copied as
// raw memory to the target process.

EvalResult PolicyOpcode::Evaluate(const ParameterSet* call_params,
                                  size_t param_count,
                                  MatchContext* match) {
  if (!call_params)
    return EVAL_ERROR;
  const ParameterSet* selected_param = nullptr;
  if (parameter_ >= 0) {
    if (static_cast<size_t>(parameter_) >= param_count) {
      return EVAL_ERROR;
    }
    // 看来PolicyOpcode的parameter_成员是个索引，call_params是个ParameterSet数组
    // selected_param借助parameter_的值找到本次想要处理的ParameterSet
    selected_param = &call_params[parameter_];
  }
  // 这里干大事
  EvalResult result = EvaluateHelper(selected_param, match);

  // Apply the general options regardless of the particular type of opcode.
  // 如果本PolicyOpcode的标记为kPolNone，直接返回结果，什么也不做
  if (kPolNone == options_) {
    return result;
  }

  // 如果本PolicyOpcode的标记了kPolNegateEval位，那么就要对结果取反（ERROR不管）
  if (options_ & kPolNegateEval) {
    if (EVAL_TRUE == result) {
      result = EVAL_FALSE;
    } else if (EVAL_FALSE == result) {
      result = EVAL_TRUE;
    } else if (EVAL_ERROR != result) {
      result = EVAL_ERROR;
    }
  }
  if (match) {
    // 如果标记了kPolClearContext位，那么要对辅助结构MatchContext进行清理工作
    if (options_ & kPolClearContext)
      match->Clear();
    // 如果标记了kPolUseOREval，那么就对辅助结构MatchContext打上标记
    if (options_ & kPolUseOREval)
      match->options = kPolUseOREval;	//默认是用AND来裁决，该标记表示用OR
  }
  return result;
}


#define OPCODE_EVAL(op, x, y, z) \
  case op:                       \
    return OpcodeEval<op>(x, y, z)

EvalResult PolicyOpcode::EvaluateHelper(const ParameterSet* parameters,
                                        MatchContext* match) {
  switch (opcode_id_) {
    OPCODE_EVAL(OP_ALWAYS_FALSE, this, parameters, match);
    OPCODE_EVAL(OP_ALWAYS_TRUE, this, parameters, match);
    OPCODE_EVAL(OP_NUMBER_MATCH, this, parameters, match);
    OPCODE_EVAL(OP_NUMBER_MATCH_RANGE, this, parameters, match);
    OPCODE_EVAL(OP_NUMBER_AND_MATCH, this, parameters, match);
    OPCODE_EVAL(OP_WSTRING_MATCH, this, parameters, match);
    OPCODE_EVAL(OP_ACTION, this, parameters, match);
    default:
      return EVAL_ERROR;
  }
}
```

根据`opcode_id_`分类，helper函数内部做了类别分发。这里`OpcodeEval`是个函数模板：

```cpp
template <int>
EvalResult OpcodeEval(PolicyOpcode* opcode,
                      const ParameterSet* pp,
                      MatchContext* match);
```

对应每种opcode_id_，都有一个具体的实现：

```cpp
template <>
EvalResult OpcodeEval<OP_ALWAYS_FALSE>(PolicyOpcode* opcode,
                                       const ParameterSet* param,
                                       MatchContext* context) {
  return EVAL_FALSE;
}

template <>
EvalResult OpcodeEval<OP_ALWAYS_TRUE>(PolicyOpcode* opcode,
                                      const ParameterSet* param,
                                      MatchContext* context) {
  return EVAL_TRUE;
}
// 前两个没啥好说的，根本不用比，对于id为OP_ALWAYS_FALSE或OP_ALWAYS_TRUE的PolicyOpcode
// 直接返回值结果就行了
// opcode id为action的比较特别，它的值在argumetnts_[0].mem保存
template <>
EvalResult OpcodeEval<OP_ACTION>(PolicyOpcode* opcode,
                                 const ParameterSet* param,
                                 MatchContext* context) {
  int action = 0;
  opcode->GetArgument(0, &action);
  return static_cast<EvalResult>(action);
}

// uint32比较有两种情况，param可以是一个uint32值，也可以是一个指针
// 无论哪一种，都是把输入的值和该PolicyOpcode中的arguments_[0].mem进行比较
template <>
EvalResult OpcodeEval<OP_NUMBER_MATCH>(PolicyOpcode* opcode,
                                       const ParameterSet* param,
                                       MatchContext* context) {
  uint32_t value_uint32 = 0;
  if (param->Get(&value_uint32)) {
    uint32_t match_uint32 = 0;
    opcode->GetArgument(0, &match_uint32);
    return (match_uint32 != value_uint32) ? EVAL_FALSE : EVAL_TRUE;
  } else {
    const void* value_ptr = nullptr;
    if (param->Get(&value_ptr)) {
      const void* match_ptr = nullptr;
      opcode->GetArgument(0, &match_ptr);
      return (match_ptr != value_ptr) ? EVAL_FALSE : EVAL_TRUE;
    }
  }
  return EVAL_ERROR;
}

// 和uint32类似，但此时该PolicyOpcode的arguments_中0成员是下限，1成员是上限值
// 此时的param中承载的是值，而不是指针
template <>
EvalResult OpcodeEval<OP_NUMBER_MATCH_RANGE>(PolicyOpcode* opcode,
                                             const ParameterSet* param,
                                             MatchContext* context) {
  uint32_t value = 0;
  if (!param->Get(&value))
    return EVAL_ERROR;

  uint32_t lower_bound = 0;
  uint32_t upper_bound = 0;
  opcode->GetArgument(0, &lower_bound);
  opcode->GetArgument(1, &upper_bound);
  return ((lower_bound <= value) && (upper_bound >= value)) ? EVAL_TRUE
                                                            : EVAL_FALSE;
}

// 这个是按位与的结果，param中的也是值
template <>
EvalResult OpcodeEval<OP_NUMBER_AND_MATCH>(PolicyOpcode* opcode,
                                           const ParameterSet* param,
                                           MatchContext* context) {
  uint32_t value = 0;
  if (!param->Get(&value))
    return EVAL_ERROR;

  uint32_t number = 0;
  opcode->GetArgument(0, &number);
  return (number & value) ? EVAL_TRUE : EVAL_FALSE;
}

// 这个复杂些
template <>
EvalResult OpcodeEval<OP_WSTRING_MATCH>(PolicyOpcode* opcode,
                                        const ParameterSet* param,
                                        MatchContext* context) {
  // 对于字符串的比较，需要用到传入的辅助结构MatchContext
  if (!context) {
    return EVAL_ERROR;
  }
  const wchar_t* source_str = nullptr;
  if (!param->Get(&source_str))
    return EVAL_ERROR;

  int start_position = 0;
  int match_len = 0;
  unsigned int match_opts = 0;
  // PolicyOpcode中的arguments_[1,2,3]成员分别是比较的长度、起始位置和比较时的选项
  opcode->GetArgument(1, &match_len);
  opcode->GetArgument(2, &start_position);
  opcode->GetArgument(3, &match_opts);

  // PolicyOpcode中的arguments_[0]是字符串的offset，通过GetRelativeString才能转成真实地址
  const wchar_t* match_str = opcode->GetRelativeString(0);
  // Advance the source string to the last successfully evaluated position
  // according to the match context.
  // 如果是首次匹配字符串，context->position应该是0，如果是连续匹配source字符串，那就表示下一个串的位置
  source_str = &source_str[context->position];
  // 每个串都有终结符，计算出本次待比较的长度
  int source_len = static_cast<int>(g_nt.wcslen(source_str));

  if (0 == source_len) {
    // If we reached the end of the source string there is nothing we can
    // match against.
    return EVAL_FALSE;
  }
  if (match_len > source_len) {
    // There can't be a positive match when the target string is bigger than
    // the source string
    return EVAL_FALSE;
  }

  // 看看match_opts里是否指明了大小写敏感
  BOOLEAN case_sensitive = (match_opts & CASE_INSENSITIVE) ? TRUE : FALSE;

  // We have three cases, depending on the value of start_pos:
  // Case 1. We skip N characters and compare once.
  // Case 2: We skip to the end and compare once.
  // Case 3: We match the first substring (if we find any).
  // 具体的子串匹配，基操勿6
  if (start_position >= 0) {
    if (kSeekToEnd == start_position) {
      start_position = source_len - match_len;
    } else if (match_opts & EXACT_LENGTH) {
      // A sub-case of case 3 is when the EXACT_LENGTH flag is on
      // the match needs to be not just substring but full match.
      if ((match_len + start_position) != source_len) {
        return EVAL_FALSE;
      }
    }

    // Advance start_pos characters. Warning! this does not consider
    // utf16 encodings (surrogate pairs) or other Unicode 'features'.
    source_str += start_position;

    // Since we skipped, lets reevaluate just the lengths again.
    if ((match_len + start_position) > source_len) {
      return EVAL_FALSE;
    }

    UNICODE_STRING match_ustr;
    UNICODE_STRING source_ustr;
    if (!InitStringUnicode(match_str, match_len, &match_ustr) ||
        !InitStringUnicode(source_str, match_len, &source_ustr))
      return EVAL_ERROR;

    if (0 == g_nt.RtlCompareUnicodeString(&match_ustr, &source_ustr,
                                          case_sensitive)) {
      // Match! update the match context.
      // 更新context的索引
      context->position += start_position + match_len;
      return EVAL_TRUE;
    } else {
      return EVAL_FALSE;
    }
  } else if (start_position < 0) {
    UNICODE_STRING match_ustr;
    UNICODE_STRING source_ustr;
    if (!InitStringUnicode(match_str, match_len, &match_ustr) ||
        !InitStringUnicode(source_str, match_len, &source_ustr))
      return EVAL_ERROR;

    do {
      if (0 == g_nt.RtlCompareUnicodeString(&match_ustr, &source_ustr,
                                            case_sensitive)) {
        // Match! update the match context.
        // 更新context的索引
        context->position += (source_ustr.Buffer - source_str) + match_len;
        return EVAL_TRUE;
      }
      ++source_ustr.Buffer;
      --source_len;
    } while (source_len >= match_len);
  }
  return EVAL_FALSE;
}
```

无论是哪种比较，本质上都是`param`和`opcode`中Argument的比较，而`opcode`是在`OpcodeFactory`中配置的。

另一方面也可以看出，`PolicyOpcode::Evaluate`一次仅仅是比较了参数集中的一个参数，它是一个下层的接口，整个参数集的裁决需要依赖使用者的正确性。

### `OpcodeFactory`

研究一下它的工厂类。

```cpp
// Opcodes that do string comparisons take a parameter that is the starting
// position to perform the comparison so we can do substring matching. There
// are two special values:
//
// Start from the current position and compare strings advancing forward until
// a match is found if any. Similar to CRT strstr().
const int kSeekForward = -1;
// Perform a match with the end of the string. It only does a single comparison.
const int kSeekToEnd = 0xfffff;

// A PolicyBuffer is a variable size structure that contains all the opcodes
// that are to be created or evaluated in sequence.
struct PolicyBuffer {
  size_t opcode_count;
  PolicyOpcode opcodes[1];	// 又见flexible
};

// Helper class to create any opcode sequence. This class is normally invoked
// only by the high level policy module or when you need to handcraft a special
// policy.
// The factory works by creating the opcodes using a chunk of memory given
// in the constructor. The opcodes themselves are allocated from the beginning
// (top) of the memory, while any string that an opcode needs is allocated from
// the end (bottom) of the memory.
//
// In essence:
//
//   low address ---> [opcode 1]
//                    [opcode 2]
//                    [opcode 3]
//                    |        | <--- memory_top_
//                    | free   |
//                    |        |
//                    |        | <--- memory_bottom_
//                    [string 1]
//   high address --> [string 2]
//
// Note that this class does not keep track of the number of opcodes made and
// it is designed to be a building block for low-level policy.
//
// Note that any of the MakeOpXXXXX member functions below can return nullptr on
// failure. When that happens opcode sequence creation must be aborted.
class OpcodeFactory {
 public:
  // memory: base pointer to a chunk of memory where the opcodes are created.
  // memory_size: the size in bytes of the memory chunk.
  // OpcodeFactory维护了一堆组织好的PolicyOpcode，所以需要一个buffer来存储
  // 构造器先对buffer的顶和底进行初始化，memory_top_和memory_bottom_
  // 根据头注释，这个buffer应该是双向增长的，共用中间的free空间
  OpcodeFactory(char* memory, size_t memory_size) : memory_top_(memory) {
    memory_bottom_ = &memory_top_[memory_size];
  }

  // policy: contains the raw memory where the opcodes are created.
  // memory_size: contains the actual size of the policy argument.
  // 传PolicyBuffer进来，与上面不同的地方在于，memory_top_一开始跳过了前4个opcode_count字节
  OpcodeFactory(PolicyBuffer* policy, size_t memory_size) {
    memory_top_ = reinterpret_cast<char*>(&policy->opcodes[0]);
    memory_bottom_ = &memory_top_[memory_size];
  }

  // Returns the available memory to make opcodes.
  // 看看free还有多大
  size_t memory_size() const {
    DCHECK_GE(memory_bottom_, memory_top_);
    return memory_bottom_ - memory_top_;
  }

  // 荷枪实弹
  // Creates an OpAlwaysFalse opcode.
  PolicyOpcode* MakeOpAlwaysFalse(uint32_t options);

  // Creates an OpAlwaysFalse opcode.
  PolicyOpcode* MakeOpAlwaysTrue(uint32_t options);

  // Creates an OpAction opcode.
  // action: The action to return when Evaluate() is called.
  PolicyOpcode* MakeOpAction(EvalResult action, uint32_t options);

  // Creates an OpNumberMatch opcode.
  // selected_param: index of the input argument. It must be a uint32_t or the
  // evaluation result will generate a EVAL_ERROR.
  // match: the number to compare against the selected_param.
  PolicyOpcode* MakeOpNumberMatch(int16_t selected_param,
                                  uint32_t match,
                                  uint32_t options);

  // Creates an OpNumberMatch opcode (void pointers are cast to numbers).
  // selected_param: index of the input argument. It must be an void* or the
  // evaluation result will generate a EVAL_ERROR.
  // match: the pointer numeric value to compare against selected_param.
  PolicyOpcode* MakeOpVoidPtrMatch(int16_t selected_param,
                                   const void* match,
                                   uint32_t options);

  // Creates an OpNumberMatchRange opcode using the memory passed in the ctor.
  // selected_param: index of the input argument. It must be a uint32_t or the
  // evaluation result will generate a EVAL_ERROR.
  // lower_bound, upper_bound: the range to compare against selected_param.
  PolicyOpcode* MakeOpNumberMatchRange(int16_t selected_param,
                                       uint32_t lower_bound,
                                       uint32_t upper_bound,
                                       uint32_t options);

  // Creates an OpWStringMatch opcode using the raw memory passed in the ctor.
  // selected_param: index of the input argument. It must be a wide string
  // pointer or the evaluation result will generate a EVAL_ERROR.
  // match_str: string to compare against selected_param.
  // start_position: when its value is from 0 to < 0x7fff it indicates an
  // offset from the selected_param string where to perform the comparison. If
  // the value is SeekForward  then a substring search is performed. If the
  // value is SeekToEnd the comparison is performed against the last part of
  // the selected_param string.
  // Note that the range in the position (0 to 0x7fff) is dictated by the
  // current implementation.
  // match_opts: Indicates additional matching flags. Currently CaseInsensitive
  // is supported.
  PolicyOpcode* MakeOpWStringMatch(int16_t selected_param,
                                   const wchar_t* match_str,
                                   int start_position,
                                   StringMatchOptions match_opts,
                                   uint32_t options);

  // Creates an OpNumberAndMatch opcode using the raw memory passed in the ctor.
  // selected_param: index of the input argument. It must be uint32_t or the
  // evaluation result will generate a EVAL_ERROR.
  // match: the value to bitwise AND against selected_param.
  PolicyOpcode* MakeOpNumberAndMatch(int16_t selected_param,
                                     uint32_t match,
                                     uint32_t options);

 private:
  // Constructs the common part of every opcode. selected_param is the index
  // of the input param to use when evaluating the opcode. Pass -1 in
  // selected_param to indicate that no input parameter is required.
  PolicyOpcode* MakeBase(OpcodeID opcode_id,
                         uint32_t options,
                         int16_t selected_param);

  // Allocates (and copies) a string (of size length) inside the buffer and
  // returns the displacement with respect to start.
  ptrdiff_t AllocRelative(void* start, const wchar_t* str, size_t length);

  // Points to the lowest currently available address of the memory
  // used to make the opcodes. This pointer increments as opcodes are made.
  char* memory_top_;

  // Points to the highest currently available address of the memory
  // used to make the opcodes. This pointer decrements as opcode strings are
  // allocated.
  char* memory_bottom_;

  DISALLOW_COPY_AND_ASSIGN(OpcodeFactory);
};
```

荷枪实弹的几个实装方法：

```cpp
//////////////////////////////////////////////////////////////////////////////
// Opcode OpAlwaysFalse:
// Does not require input parameter.

PolicyOpcode* OpcodeFactory::MakeOpAlwaysFalse(uint32_t options) {
  return MakeBase(OP_ALWAYS_FALSE, options, -1);
}

//////////////////////////////////////////////////////////////////////////////
// Opcode OpAlwaysTrue:
// Does not require input parameter.

PolicyOpcode* OpcodeFactory::MakeOpAlwaysTrue(uint32_t options) {
  return MakeBase(OP_ALWAYS_TRUE, options, -1);
}

//////////////////////////////////////////////////////////////////////////////
// Opcode OpAction:
// Does not require input parameter.
// Argument 0 contains the actual action to return.

PolicyOpcode* OpcodeFactory::MakeOpAction(EvalResult action, uint32_t options) {
  PolicyOpcode* opcode = MakeBase(OP_ACTION, options, 0);
  if (!opcode)
    return nullptr;
  // 这就与OpcodeEval<OP_ACTION>的处理对上了，action的确是放在arguments_[0]的
  opcode->SetArgument(0, action);
  return opcode;
}

// 剩下的这些也都和OpcodeEval<xxx>一一对上，就不多说了
//////////////////////////////////////////////////////////////////////////////
// Opcode OpNumberMatch:
// Requires a uint32_t or void* in selected_param
// Argument 0 is the stored number to match.
// Argument 1 is the C++ type of the 0th argument.

PolicyOpcode* OpcodeFactory::MakeOpNumberMatch(int16_t selected_param,
                                               uint32_t match,
                                               uint32_t options) {
  PolicyOpcode* opcode = MakeBase(OP_NUMBER_MATCH, options, selected_param);
  if (!opcode)
    return nullptr;
  opcode->SetArgument(0, match);
  opcode->SetArgument(1, UINT32_TYPE);
  return opcode;
}

PolicyOpcode* OpcodeFactory::MakeOpVoidPtrMatch(int16_t selected_param,
                                                const void* match,
                                                uint32_t options) {
  PolicyOpcode* opcode = MakeBase(OP_NUMBER_MATCH, options, selected_param);
  if (!opcode)
    return nullptr;
  opcode->SetArgument(0, match);
  opcode->SetArgument(1, VOIDPTR_TYPE);
  return opcode;
}

//////////////////////////////////////////////////////////////////////////////
// Opcode OpNumberMatchRange
// Requires a uint32_t in selected_param.
// Argument 0 is the stored lower bound to match.
// Argument 1 is the stored upper bound to match.

PolicyOpcode* OpcodeFactory::MakeOpNumberMatchRange(int16_t selected_param,
                                                    uint32_t lower_bound,
                                                    uint32_t upper_bound,
                                                    uint32_t options) {
  if (lower_bound > upper_bound) {
    return nullptr;
  }
  PolicyOpcode* opcode =
      MakeBase(OP_NUMBER_MATCH_RANGE, options, selected_param);
  if (!opcode)
    return nullptr;
  opcode->SetArgument(0, lower_bound);
  opcode->SetArgument(1, upper_bound);
  return opcode;
}
//////////////////////////////////////////////////////////////////////////////
// Opcode OpNumberAndMatch:
// Requires a uint32_t in selected_param.
// Argument 0 is the stored number to match.

PolicyOpcode* OpcodeFactory::MakeOpNumberAndMatch(int16_t selected_param,
                                                  uint32_t match,
                                                  uint32_t options) {
  PolicyOpcode* opcode = MakeBase(OP_NUMBER_AND_MATCH, options, selected_param);
  if (!opcode)
    return nullptr;
  opcode->SetArgument(0, match);
  return opcode;
}
//////////////////////////////////////////////////////////////////////////////
// Opcode OpWStringMatch:
// Requires a wchar_t* in selected_param.
// Argument 0 is the byte displacement of the stored string.
// Argument 1 is the length in chars of the stored string.
// Argument 2 is the offset to apply on the input string. It has special values.
// as noted in the header file.
// Argument 3 is the string matching options.

PolicyOpcode* OpcodeFactory::MakeOpWStringMatch(int16_t selected_param,
                                                const wchar_t* match_str,
                                                int start_position,
                                                StringMatchOptions match_opts,
                                                uint32_t options) {
  if (!match_str)
    return nullptr;
  if ('\0' == match_str[0])
    return nullptr;

  int length = lstrlenW(match_str);

  PolicyOpcode* opcode = MakeBase(OP_WSTRING_MATCH, options, selected_param);
  if (!opcode)
    return nullptr;
  // 字符串分配在buffer尾部，从bottom向上抬
  ptrdiff_t delta_str = AllocRelative(opcode, match_str, wcslen(match_str) + 1);
  if (0 == delta_str)
    return nullptr;
  // 注意这个delta_str只是个偏移，所以OpcodeEval<OP_WSTRING_MATCH>时使用的是GetRelativeString
  // 而不是GetArgument
  opcode->SetArgument(0, delta_str);
  opcode->SetArgument(1, length);
  opcode->SetArgument(2, start_position);
  opcode->SetArgument(3, match_opts);
  return opcode;
}
```

每个Make方法都是new了一个`PolicyOpcode`对象，把实装的信息填充到`PolicyOpcode`对象中。在new出一个`PolicyOpcode`对象时，都使用了一个相同的基准方法`MakeBase`：

```cpp
//////////////////////////////////////////////////////////////////////////////
// OpcodeMaker (other member functions).

PolicyOpcode* OpcodeFactory::MakeBase(OpcodeID opcode_id,
                                      uint32_t options,
                                      int16_t selected_param) {
  if (memory_size() < sizeof(PolicyOpcode))
    return nullptr;

  // opcode从top开始向下占用buffer
  // Create opcode using placement-new on the buffer memory.
  PolicyOpcode* opcode = new (memory_top_) PolicyOpcode();

  // Fill in the standard fields, that every opcode has.
  memory_top_ += sizeof(PolicyOpcode);
  opcode->opcode_id_ = opcode_id;	// 哪种opcode
  opcode->SetOptions(options);		// 哪些标记
  // 传入的selected_param表示用于和该PolicyOpcode比较的参数在ParameterSet中是第几个，也就是索引
  // 看到这里也就明白了Evaluate中为什么直接读用parameter_做索引了，因为早在Make的时候已经设置好了
  opcode->parameter_ = selected_param;	
  return opcode;
}
```

字符串的匹配比较特殊，除了要保存一个`PolicyOpcode`对象以外，还要额外存储一个字符串pattern。这个字符串在尾端分配：

```cpp
ptrdiff_t OpcodeFactory::AllocRelative(void* start,
                                       const wchar_t* str,
                                       size_t length) {
  size_t bytes = length * sizeof(wchar_t);
  if (memory_size() < bytes)
    return 0;
  memory_bottom_ -= bytes;
  if (reinterpret_cast<UINT_PTR>(memory_bottom_) & 1) {
    // TODO(cpu) replace this for something better.
    ::DebugBreak();
  }
  memcpy(memory_bottom_, str, bytes);
  ptrdiff_t delta = memory_bottom_ - reinterpret_cast<char*>(start);
  return delta;
}
```

工厂类的分析可以对接上`PolicyOpcode`本身的一些接口，到此我们也十分清楚工厂类的使用者在为某个函数设置policy时，会为每个参数调用一系列的Make函数，设置理想的裁决值。

但究竟是谁在用`OpcodeFactory`，又是如何操纵另一个重要的`ParameterSet`呢？现在还不得而知。猜想应该就是engine_processor。

### `ParameterSet`

移步processor前，我们先分析被比较的`ParameterSet`都提供了哪些接口，至少我们已经不止一次的看到了它的Get。

```cpp
// Models the set of interesting parameters of an intercepted system call
// normally you don't create objects of this class directly, instead you
// use the POLPARAMS_XXX macros.
// For example, if an intercepted function has the following signature:
//
// NTSTATUS NtOpenFileFunction (PHANDLE FileHandle,
//                              ACCESS_MASK DesiredAccess,
//                              POBJECT_ATTRIBUTES ObjectAttributes,
//                              PIO_STATUS_BLOCK IoStatusBlock,
//                              ULONG ShareAccess,
//                              ULONG OpenOptions);
//
// You could say that the following parameters are of interest to policy:
//
//   POLPARAMS_BEGIN(open_params)
//      POLPARAM(DESIRED_ACCESS)
//      POLPARAM(OBJECT_NAME)
//      POLPARAM(SECURITY_DESCRIPTOR)
//      POLPARAM(IO_STATUS)
//      POLPARAM(OPEN_OPTIONS)
//   POLPARAMS_END;
//
// and the actual code will use this for defining the parameters:
//
//   CountedParameterSet<open_params> p;
//   p[open_params::DESIRED_ACCESS] = ParamPickerMake(DesiredAccess);
//   p[open_params::OBJECT_NAME] =
//       ParamPickerMake(ObjectAttributes->ObjectName);
//   p[open_params::SECURITY_DESCRIPTOR] =
//       ParamPickerMake(ObjectAttributes->SecurityDescriptor);
//   p[open_params::IO_STATUS] = ParamPickerMake(IoStatusBlock);
//   p[open_params::OPEN_OPTIONS] = ParamPickerMake(OpenOptions);
//
//  These will create an stack-allocated array of ParameterSet objects which
//  have each 1) the address of the parameter 2) a numeric id that encodes the
//  original C++ type. This allows the policy to treat any set of supported
//  argument types uniformily and with some type safety.
//
//  TODO(cpu): support not fully implemented yet for unicode string and will
//  probably add other types as well.
class ParameterSet {
 public:
  ParameterSet() : real_type_(INVALID_TYPE), address_(nullptr) {}

  // real_type的3种类型对应的重载Get方法，real_type_来判定
  // Retrieve the stored parameter. If the type does not match ulong fail.
  bool Get(uint32_t* destination) const {
    if (real_type_ != UINT32_TYPE) {
      return false;
    }
    *destination = Void2TypePointerCopy<uint32_t>();//返回的实际上是address_成员的地址
    return true;
  }

  // Retrieve the stored parameter. If the type does not match void* fail.
  bool Get(const void** destination) const {
    if (real_type_ != VOIDPTR_TYPE) {
      return false;
    }
    *destination = Void2TypePointerCopy<void*>();
    return true;
  }

  // Retrieve the stored parameter. If the type does not match wchar_t* fail.
  bool Get(const wchar_t** destination) const {
    if (real_type_ != WCHAR_TYPE) {
      return false;
    }
    *destination = Void2TypePointerCopy<const wchar_t*>();
    return true;
  }

  // False if the parameter is not properly initialized.
  bool IsValid() const { return real_type_ != INVALID_TYPE; }

 protected:
  // The constructor can only be called by derived types, which should
  // safely provide the real_type and the address of the argument.
  // 所以他才是关键，真正有用的构造函数，以address赋给address_，实际上参数的buffer
  // 可以看出由外部维护，ParameterSet只是架在上面的操纵者罢了
  // protected权限又表示外部使用该对象时，必须得间接用派生对象
  ParameterSet(ArgType real_type, const void* address)
      : real_type_(real_type), address_(address) {}

 private:
  // This template provides the same functionality as bits_cast but
  // it works with pointer while the former works only with references.
  template <typename T>
  T Void2TypePointerCopy() const {
    return *(reinterpret_cast<const T*>(address_));//类型转换模板
  }

  ArgType real_type_;
  const void* address_;
};
```

### `ParameterSetEx`

protected权限的构造器也就表明了派生类的存在意义：

```cpp
// To safely infer the type, we use a set of template specializations
// in ParameterSetEx with a template function ParamPickerMake to do the
// parameter type deduction.

// Base template class. Not implemented so using unsupported types should
// fail to compile.
// 使用ParameterSetEx类模板
template <typename T>
class ParameterSetEx : public ParameterSet {
 public:
  ParameterSetEx(const void* address);
};

// 一串不同typename对应的实现体
// 实际上差别仅在于设置不同的ArgType
template <>
class ParameterSetEx<void const*> : public ParameterSet {
 public:
  ParameterSetEx(const void* address) : ParameterSet(VOIDPTR_TYPE, address) {}
};

template <>
class ParameterSetEx<void*> : public ParameterSet {
 public:
  ParameterSetEx(const void* address) : ParameterSet(VOIDPTR_TYPE, address) {}
};

template <>
class ParameterSetEx<wchar_t*> : public ParameterSet {
 public:
  ParameterSetEx(const void* address) : ParameterSet(WCHAR_TYPE, address) {}
};

template <>
class ParameterSetEx<wchar_t const*> : public ParameterSet {
 public:
  ParameterSetEx(const void* address) : ParameterSet(WCHAR_TYPE, address) {}
};

template <>
class ParameterSetEx<uint32_t> : public ParameterSet {
 public:
  ParameterSetEx(const void* address) : ParameterSet(UINT32_TYPE, address) {}
};

template <>
class ParameterSetEx<UNICODE_STRING> : public ParameterSet {
 public:
  ParameterSetEx(const void* address) : ParameterSet(UNISTR_TYPE, address) {}
};
```

根据父类的`ArgType`型参数，派生类的每个实现体指明了具体的类型，此时`real_type_`就有效了。

当然，`ParameterSetEx`实现体太多了，为了使用方便，又定义了：

```cpp
template <typename T>
ParameterSet ParamPickerMake(T& parameter) {
  return ParameterSetEx<T>(&parameter);
};
```

非常简单的一个转换适配器函数模板，各找各妈。

### `CountedParameterSet`

一个`ParameterSet`的集合，每一种interception对应一个这样的policy parameters串

```cpp
// This template defines the actual list of policy parameters for a given
// interception.
// Warning: This template stores the address to the actual variables, in
// other words, the values are not copied.
template <typename T>
struct CountedParameterSet {
  CountedParameterSet() : count(T::PolParamLast) {}

  // 重载了[]操作符，对此操作时就可以直接返回选择的ParameterSet对象
  ParameterSet& operator[](typename T::Args n) { return parameters[n]; }

  // 其实就是count的起始地址
  CountedParameterSetBase* GetBase() {
    return reinterpret_cast<CountedParameterSetBase*>(this);
  }

  size_t count;
  ParameterSet parameters[T::PolParamLast];//这个T::PolParamLast源于T
};
```

`ParameterSet`相关的定义能够暗示一些操纵者的用法，但还不够明朗。我们接下来就研究一下processor这个操纵者是如何驾驭`ParameterSet`和`OpcodeFactory`的。

## processor

文件头的描述已经相当详细了。

```cpp
// This header contains the core policy evaluator. In its simplest form
// it evaluates a stream of opcodes assuming that they are laid out in
// memory as opcode groups.
//
// An opcode group has N comparison opcodes plus 1 action opcode. For
// example here we have 3 opcode groups (A, B,C):
// 一个opcode组由N个comparison opcodes加上一个action opcode组成
// 这里是个3组的示范：
//
// [comparison 1]  <-- group A start
// [comparison 2]
// [comparison 3]
// [action A    ]
// [comparison 1]  <-- group B start
// [action B    ]
// [comparison 1]  <-- group C start
// [comparison 2]
// [action C    ]
//
// The opcode evaluator proceeds from the top, evaluating each opcode in
// sequence. An opcode group is evaluated until the first comparison that
// returns false. At that point the rest of the group is skipped and evaluation
// resumes with the first comparison of the next group. When all the comparisons
// in a group have evaluated to true and the action is reached. The group is
// considered a matching group.
// 裁决从top开始，逐个opcode进行evaluate。
// 一组opcode设定的条件只有全部满足时，才继续下一组的匹配，否则就不必继续审判了
// 
// In the 'ShortEval' mode evaluation stops when it reaches the end or the first
// matching group. The action opcode from this group is the resulting policy
// action.
//
// In the 'RankedEval' mode evaluation stops only when it reaches the end of the
// the opcode stream. In the process all matching groups are saved and at the
// end the 'best' group is selected (what makes the best is TBD) and the action
// from this group is the resulting policy action.
//
// As explained above, the policy evaluation of a group is a logical AND of
// the evaluation of each opcode. However an opcode can request kPolUseOREval
// which makes the evaluation to use logical OR. Given that each opcode can
// request its evaluation result to be negated with kPolNegateEval you can
// achieve the negation of the total group evaluation. This means that if you
// need to express:
// 对一组opcode的裁决，默认每个comparsion都是AND操作，可以通过kPolUseOREval来置成
// OR操作
//             if (!(c1 && c2 && c3))
// You can do it by:
//             if ((!c1) || (!c2) || (!c3))
//

// Possible outcomes of policy evaluation.
```

一些常量、枚举：

```cpp
// Possible outcomes of policy evaluation.
enum PolicyResult { NO_POLICY_MATCH, POLICY_MATCH, POLICY_ERROR };

// Policy evaluation flags
// TODO(cpu): implement the options kStopOnErrors & kRankedEval.
// 
// Stop evaluating as soon as an error is encountered.
const uint32_t kStopOnErrors = 1;
// Ignore all non fatal opcode evaluation errors.
const uint32_t kIgnoreErrors = 2;
// Short-circuit evaluation: Only evaluate until opcode group that
// evaluated to true has been found.
const uint32_t kShortEval = 4;
// Discussed briefly at the policy design meeting. It will evaluate
// all rules and then return the 'best' rule that evaluated true.
const uint32_t kRankedEval = 8;
```

### `PolicyProcessor`

```cpp
// This class evaluates a policy-opcode stream given the memory where the
// opcodes are and an input 'parameter set'.
//
// This class is designed to be callable from interception points
// as low as the NtXXXX service level (it is not currently safe, but
// it is designed to be made safe).
//
// Its usage in an interception is:
//
//   POLPARAMS_BEGIN(eval_params)
//     POLPARAM(param1)
//     POLPARAM(param2)
//     POLPARAM(param3)
//     POLPARAM(param4)
//     POLPARAM(param5)
//   POLPARAMS_END;
//
//   PolicyProcessor pol_evaluator(policy_memory);
//   PolicyResult pr = pol_evaluator.Evaluate(ShortEval, eval_params,
//                                            _countof(eval_params));
//   if (NO_POLICY_MATCH == pr) {
//     EvalResult policy_action =  pol_evaluator.GetAction();
//     // apply policy here...
//   }
//
// Where the POLPARAM() arguments are derived from the intercepted function
// arguments, and represent all the 'interesting' policy inputs, and
// policy_memory is a memory buffer containing the opcode stream that is the
// relevant policy for this intercept.
class PolicyProcessor {
 public:
  // policy_buffer contains opcodes made with OpcodeFactory. They are usually
  // created in the broker process and evaluated in the target process.

  // This constructor is just a variant of the previous constructor.
  // 该构造器把PolicyBuffer传入，存储PolicyOpcode的容器与操纵者关联
  explicit PolicyProcessor(PolicyBuffer* policy) : policy_(policy) {
    SetInternalState(0, EVAL_FALSE);
  }

  // Evaluates a policy-opcode stream. See the comments at the top of this
  // class for more info. Returns POLICY_MATCH if a rule set was found that
  // matches an active policy.
  // 这个就是某个函数所有参数的Evaluate，传递了ParameterSet和count进去，其内部理应对每个参数进行
  // PolicyOpcode::Evaluate
  PolicyResult Evaluate(uint32_t options,
                        ParameterSet* parameters,
                        size_t parameter_count);

  // If the result of Evaluate() was POLICY_MATCH, calling this function returns
  // the recommended policy action.
  EvalResult GetAction() const;

 private:
  struct {
    size_t current_index_;
    EvalResult current_result_;
  } state_;

  // Sets the currently matching action result.
  void SetInternalState(size_t index, EvalResult result);

  PolicyBuffer* policy_;
  DISALLOW_COPY_AND_ASSIGN(PolicyProcessor);
};
```

关键的Evaluate，处理函数的parameters。

```cpp
PolicyResult PolicyProcessor::Evaluate(uint32_t options,
                                       ParameterSet* parameters,
                                       size_t param_count) {
  if (!policy_)
    return NO_POLICY_MATCH;
  if (0 == policy_->opcode_count)
    return NO_POLICY_MATCH;
  // 至少得置位kShortEval
  if (!(kShortEval & options))
    return POLICY_ERROR;

  MatchContext context;
  bool evaluation = false;
  bool skip_group = false;
  SetInternalState(0, EVAL_FALSE);
  size_t count = policy_->opcode_count;	
  // 共有count个opcode待处理，它们由comparsion和action组成
  // action作为组的定界
  
  // Loop over all the opcodes Evaluating in sequence. Since we only support
  // short circuit evaluation, we stop as soon as we find an 'action' opcode
  // and the current evaluation is true.
  //
  // Skipping opcodes can happen when we are in AND mode (!kPolUseOREval) and
  // have got EVAL_FALSE or when we are in OR mode (kPolUseOREval) and got
  // EVAL_TRUE. Skipping will stop at the next action opcode or at the opcode
  // after the action depending on kPolUseOREval.

  for (size_t ix = 0; ix != count; ++ix) {
    PolicyOpcode& opcode = policy_->opcodes[ix];
    // Skipping block.
    if (skip_group) {
      if (SkipOpcode(opcode, &context, &skip_group))
        continue;
    }
    // Evaluation block.
    // 这里使用具体的PolicyOpcode::Evaluate来对parameters中某个参数进行匹配
    EvalResult result = opcode.Evaluate(parameters, param_count, &context);
    switch (result) {
      case EVAL_FALSE:
        evaluation = false;
        // 如果某一个参数匹配失败了，那么看看是否设置了OR型匹配，如果没设置，就跳过本组
        // 因为本组已经失败了，说明传递进来的参数不匹配本组
        if (kPolUseOREval != context.options)
          skip_group = true;
        break;
      case EVAL_ERROR:
        if (kStopOnErrors & options)
          return POLICY_ERROR;
        break;
      case EVAL_TRUE:
        evaluation = true;
        // 如果某个匹配成功了，且设置了OR型匹配，那么本组就不必继续了，因为已经成功了
        // 这说明本组也不是想要找的参数匹配组
        if (kPolUseOREval == context.options)
          skip_group = true;
        break;
      default:
        // We have evaluated an action.
        // action opcode返回类型是action而非comparison，这说明本组匹配成功
    	// 这时要对第ix组进行结果的设置，然后返回
        SetInternalState(ix, result);
        return POLICY_MATCH;
    }
  }

  if (evaluation) {
    // Reaching the end of the policy with a positive evaluation is probably
    // an error: we did not find a final action opcode?
    return POLICY_ERROR;
  }
  return NO_POLICY_MATCH;
}
```

`SkipOpcode`非常简单，闭着眼睛都知道怎么处理的，当然是根据定界的action来移动：

```cpp
// Decides if an opcode can be skipped (not evaluated) or not. The function
// takes as inputs the opcode and the current evaluation context and returns
// true if the opcode should be skipped or not and also can set keep_skipping
// to false to signal that the current instruction should be skipped but not
// the next after the current one.
bool SkipOpcode(const PolicyOpcode& opcode,
                MatchContext* context,
                bool* keep_skipping) {
  if (opcode.IsAction()) {
    uint32_t options = context->options;
    context->Clear();
    *keep_skipping = false;
    return (kPolUseOREval != options);
  }
  *keep_skipping = true;
  return true;
}
```

### 管中窥豹

无论是opcode相关，还是操纵opcode group与`ParameterSet`的processor，他们都只是联合提供了一整套机制，封装成API供上层的某个组件使用。在头文件的描述中，我们已经知道了这个组件是low-level policy。low-level policy本身有一定篇幅，我们本节就不再继续深入，留到下一节分析。但抛开low-level policy，从policy_engine_unittest.cc和policy_opcodes_unittest.cc这两个单元测试代码中亦能可见一斑。

先看一下opcode的单元测试：

```cpp
// 这个TEST是测试ParameterSet的构造器和Get方法是否奏效
TEST(PolicyEngineTest, ParameterSetTest) {
  // 做出两个ParameterSet，设置对应的realType_和address_成员
  void* pv1 = reinterpret_cast<void*>(0x477EAA5);
  const void* pv2 = reinterpret_cast<void*>(0x987654);
  ParameterSet pset1 = ParamPickerMake(pv1);
  ParameterSet pset2 = ParamPickerMake(pv2);

  // Test that we can store and retrieve a void pointer:
  // 先确认ParameterSet本身没毛病
  const void* result1 = 0;
  uint32_t result2 = 0;
  EXPECT_TRUE(pset1.Get(&result1));
  EXPECT_TRUE(pv1 == result1);
  EXPECT_FALSE(pset1.Get(&result2));
  EXPECT_TRUE(pset2.Get(&result1));
  EXPECT_TRUE(pv2 == result1);
  EXPECT_FALSE(pset2.Get(&result2));

  // Test that we can store and retrieve a uint32_t:
  // 确保ParameterSet对uint32_t类型的处理是正确的（它是兼容uint32_t和指针两种处理的）
  uint32_t number = 12747;
  ParameterSet pset3 = ParamPickerMake(number);
  EXPECT_FALSE(pset3.Get(&result1));	// Get方法有多个重载，这里因为TYPE对不上理应返会false
  EXPECT_TRUE(pset3.Get(&result2));		// 这个则应该返回true
  EXPECT_EQ(number, result2);

  // Test that we can store and retrieve a string:
  // 测试字符串的ParameterSet有没有毛病
  const wchar_t* txt = L"S231L";
  ParameterSet pset4 = ParamPickerMake(txt);
  const wchar_t* result3 = nullptr;
  EXPECT_TRUE(pset4.Get(&result3));
  EXPECT_EQ(0, wcscmp(txt, result3));
}
```

`ParameterSet`的设计很有灵性，对于上面的测试代码，完全能够理解。

再看对true/false两种opcode的测试：

```cpp
TEST(PolicyEngineTest, TrueFalseOpcodes) {
  void* dummy = nullptr;
  ParameterSet ppb1 = ParamPickerMake(dummy);
  // 先做出一个工厂，memory作为PolicyOpcode+string的buffer
  char memory[kOpcodeMemory];
  OpcodeFactory opcode_maker(memory, sizeof(memory));

  // This opcode always evaluates to true.
  // op1是一个永远返回FALSE的opcode
  PolicyOpcode* op1 = opcode_maker.MakeOpAlwaysFalse(kPolNone);
  ASSERT_NE(nullptr, op1);
  // op1的evaluate一定是返回false
  EXPECT_EQ(EVAL_FALSE, op1->Evaluate(&ppb1, 1, nullptr));
  // op1是comparsion，不是action
  EXPECT_FALSE(op1->IsAction());

  // This opcode always evaluates to false.
  // op2永远返回true
  PolicyOpcode* op2 = opcode_maker.MakeOpAlwaysTrue(kPolNone);
  ASSERT_NE(nullptr, op2);
  // 这里应该返回true
  EXPECT_EQ(EVAL_TRUE, op2->Evaluate(&ppb1, 1, nullptr));

  // Nulls not allowed on the params.
  // 参数是null的情况，会返回ERROR（此时不会返回裁决值 true或false）
  EXPECT_EQ(EVAL_ERROR, op2->Evaluate(nullptr, 0, nullptr));
  EXPECT_EQ(EVAL_ERROR, op2->Evaluate(nullptr, 1, nullptr));

  // True and False opcodes do not 'require' a number of parameters
  EXPECT_EQ(EVAL_TRUE, op2->Evaluate(&ppb1, 0, nullptr));
  EXPECT_EQ(EVAL_TRUE, op2->Evaluate(&ppb1, 1, nullptr));

  // Test Inverting the logic. Note that inversion is done outside
  // any particular opcode evaluation so no need to repeat for all
  // opcodes.
  // 设置一个有着kPolNegateEval标记位得永远返回false的opcode
  PolicyOpcode* op3 = opcode_maker.MakeOpAlwaysFalse(kPolNegateEval);
  ASSERT_NE(nullptr, op3);
  // 由于kPolNegateEval，所以永远返回True
  EXPECT_EQ(EVAL_TRUE, op3->Evaluate(&ppb1, 1, nullptr));
  PolicyOpcode* op4 = opcode_maker.MakeOpAlwaysTrue(kPolNegateEval);
  ASSERT_NE(nullptr, op4);
  // 这个就永远返回false
  EXPECT_EQ(EVAL_FALSE, op4->Evaluate(&ppb1, 1, nullptr));

  // Test that we clear the match context
  PolicyOpcode* op5 = opcode_maker.MakeOpAlwaysTrue(kPolClearContext);
  ASSERT_NE(nullptr, op5);
  MatchContext context;
  context.position = 1;
  context.options = kPolUseOREval;
  // 对字符串连续匹配所用的辅助结构context的测试，实际上这里是用不到的
  // 只是单纯的测试options的值是否会正确的设置给context
  EXPECT_EQ(EVAL_TRUE, op5->Evaluate(&ppb1, 1, &context));
  EXPECT_EQ(0u, context.position);
  MatchContext context2;
  EXPECT_EQ(context2.options, context.options);
}
```

对工厂类的测试：

```cpp
TEST(PolicyEngineTest, OpcodeMakerCase1) {
  // Testing that the opcode maker does not overrun the
  // supplied buffer. It should only be able to make 'count' opcodes.
  void* dummy = nullptr;
  ParameterSet ppb1 = ParamPickerMake(dummy);

  char memory[kOpcodeMemory];
  OpcodeFactory opcode_maker(memory, sizeof(memory));
  size_t count = sizeof(memory) / sizeof(PolicyOpcode);

  // 放置count个false comparsion opcode到buffer中
  for (size_t ix = 0; ix != count; ++ix) {
    PolicyOpcode* op = opcode_maker.MakeOpAlwaysFalse(kPolNone);
    ASSERT_NE(nullptr, op);
    EXPECT_EQ(EVAL_FALSE, op->Evaluate(&ppb1, 1, nullptr));
  }
  // There should be no room more another opcode:
  PolicyOpcode* op1 = opcode_maker.MakeOpAlwaysFalse(kPolNone);
  ASSERT_EQ(nullptr, op1);
}

TEST(PolicyEngineTest, OpcodeMakerCase2) {
  SetupNtdllImports();
  // Testing that the opcode maker does not overrun the
  // supplied buffer. It should only be able to make 'count' opcodes.
  // The difference with the previous test is that this opcodes allocate
  // the string 'txt2' inside the same buffer.
  const wchar_t* txt1 = L"1234";
  const wchar_t txt2[] = L"123";

  ParameterSet ppb1 = ParamPickerMake(txt1);
  MatchContext mc1;

  char memory[kOpcodeMemory];
  OpcodeFactory opcode_maker(memory, sizeof(memory));
  size_t count = sizeof(memory) / (sizeof(PolicyOpcode) + sizeof(txt2));

  // Test that it does not overrun the buffer.
  // 放置count个字符串比较opcode到buffer
  for (size_t ix = 0; ix != count; ++ix) {
    PolicyOpcode* op = opcode_maker.MakeOpWStringMatch(
        0, txt2, 0, CASE_SENSITIVE, kPolClearContext);
    ASSERT_NE(nullptr, op);
    EXPECT_EQ(EVAL_TRUE, op->Evaluate(&ppb1, 1, &mc1));
  }

  // There should be no room more another opcode:
  PolicyOpcode* op1 =
      opcode_maker.MakeOpWStringMatch(0, txt2, 0, CASE_SENSITIVE, kPolNone);
  ASSERT_EQ(nullptr, op1);
}
```

继续测试各种comparison opcode的正确性：

```cpp
TEST(PolicyEngineTest, IntegerOpcodes) {
  const wchar_t* txt = L"abcdef";
  uint32_t num1 = 42;
  uint32_t num2 = 113377;

  // make三个ParameterSet，1个字符串两个uint32
  ParameterSet pp_wrong1 = ParamPickerMake(txt);
  ParameterSet pp_num1 = ParamPickerMake(num1);
  ParameterSet pp_num2 = ParamPickerMake(num2);

  char memory[kOpcodeMemory];
  // opcode工厂类
  OpcodeFactory opcode_maker(memory, sizeof(memory));

  // Test basic match for uint32s 42 == 42 and 42 != 113377.
  // 放入一个匹配uint32值为42、参数索引为0、标记为kPolNone的PolicyOpcode
  // 参数索引表示的是在ParameterSet数组中的索引，这里是单参数测试的，所以只有0
  // 这个索引是为了上层的匹配ParameterSet数组而设置的
  PolicyOpcode* op_m42 = opcode_maker.MakeOpNumberMatch(0, 42UL, kPolNone);
  ASSERT_NE(nullptr, op_m42);
  // 匹配pp_num1应该是返回true的，匹配pp_num2应该返回false，而匹配字符串因为类型不一致，会返回ERROR
  EXPECT_EQ(EVAL_TRUE, op_m42->Evaluate(&pp_num1, 1, nullptr));
  EXPECT_EQ(EVAL_FALSE, op_m42->Evaluate(&pp_num2, 1, nullptr));
  EXPECT_EQ(EVAL_ERROR, op_m42->Evaluate(&pp_wrong1, 1, nullptr));

  // Test basic match for void pointers.
  // void指针的匹配，同上
  const void* vp = nullptr;
  ParameterSet pp_num3 = ParamPickerMake(vp);
  PolicyOpcode* op_vp_null =
      opcode_maker.MakeOpVoidPtrMatch(0, nullptr, kPolNone);
  ASSERT_NE(nullptr, op_vp_null);
  EXPECT_EQ(EVAL_TRUE, op_vp_null->Evaluate(&pp_num3, 1, nullptr));
  EXPECT_EQ(EVAL_FALSE, op_vp_null->Evaluate(&pp_num1, 1, nullptr));
  EXPECT_EQ(EVAL_ERROR, op_vp_null->Evaluate(&pp_wrong1, 1, nullptr));

  // Basic range test [41 43] (inclusive).
  PolicyOpcode* op_range1 =
      opcode_maker.MakeOpNumberMatchRange(0, 41, 43, kPolNone);
  ASSERT_NE(nullptr, op_range1);
  EXPECT_EQ(EVAL_TRUE, op_range1->Evaluate(&pp_num1, 1, nullptr));
  EXPECT_EQ(EVAL_FALSE, op_range1->Evaluate(&pp_num2, 1, nullptr));
  EXPECT_EQ(EVAL_ERROR, op_range1->Evaluate(&pp_wrong1, 1, nullptr));
}
```

其他的测试用例都差不多，就不一一展开了，我们理解了`PolicyOpcode`， `OpcodeFactory`以及`ParameterSet`的使用即达到目的。

再看整个engine的测试代码：

```cpp
TEST(PolicyEngineTest, Rules1) {
  SetupNtdllImports();

  // Construct two policy rules that say:
  //
  // #1
  // If the path is c:\\documents and settings\\* AND
  // If the creation mode is 'open existing' AND
  // If the security descriptor is null THEN
  // Ask the broker.
  //
  // #2
  // If the security descriptor is null AND
  // If the path ends with *.txt AND
  // If the creation mode is not 'create new' THEN
  // return Access Denied.
  // 构造了两个rule：
  // 第一个规则的判断条件：
  // path得是c:\\documents and settings\\*且创建模式是open existing且安全描述符是null
  // 满足则action为ASK_BROKER
  // 第二个规则的判断条件：
  // 安全描述符是null且path以*.txt结尾且创建模式不是`create new`
  // 满足则action为Access Denied
  // 
  // 这几个参数就是要进行裁决的ParameterSet[]
  enum FileCreateArgs {
    FileNameArg,
    CreationDispositionArg,
    FlagsAndAttributesArg,
    SecurityAttributes
  };

  // 这里使用PolicyBuffer来new一个OpcodeFactory
  // 尺寸为1024，这里构造器的尺寸减去了0x40，我个人觉着减去0x4就可以了吧，不清楚是否是笔误
  // 当然无论是0x40还是0x4都不会影响正确性，仅仅是浪费了点空间
  const size_t policy_sz = 1024;
  PolicyBuffer* policy = reinterpret_cast<PolicyBuffer*>(new char[policy_sz]);
  OpcodeFactory opcode_maker(policy, policy_sz - 0x40);

  // Add rule set #1
  // 工厂类中添加第一套rule
  // 参数索引为1，值为L"c:\\documents and settings\\"，字符串起始位置0，大小写敏感，标记为kPolNone
  opcode_maker.MakeOpWStringMatch(FileNameArg, L"c:\\documents and settings\\",
                                  0, CASE_INSENSITIVE, kPolNone);
  // 参数索引为2，值为OPEN_EXISTING，标记为kPolNone
  opcode_maker.MakeOpNumberMatch(CreationDispositionArg, OPEN_EXISTING,
                                 kPolNone);
  // 参数索引为3，值为nullptr，标记为kPolNone
  opcode_maker.MakeOpVoidPtrMatch(SecurityAttributes, nullptr, kPolNone);
  // 这一group的comparsion opcode设置完毕了，接上满足匹配时的审判，这里是ASK_BROKER
  opcode_maker.MakeOpAction(ASK_BROKER, kPolNone);

  // Add rule set #2
  // 第二套规则仅仅判断两个参数
  opcode_maker.MakeOpWStringMatch(FileNameArg, L".TXT", kSeekToEnd,
                                  CASE_INSENSITIVE, kPolNone);
  opcode_maker.MakeOpNumberMatch(CreationDispositionArg, CREATE_NEW,
                                 kPolNegateEval);
  // 满足匹配时审判结果为FAKE_ACCESS_DENIED
  opcode_maker.MakeOpAction(FAKE_ACCESS_DENIED, kPolNone);
  // 更新count值
  policy->opcode_count = 7;

  // 设置一组参数
  const wchar_t* filename = L"c:\\Documents and Settings\\Microsoft\\BLAH.txt";
  uint32_t creation_mode = OPEN_EXISTING;
  uint32_t flags = FILE_ATTRIBUTE_NORMAL;
  void* security_descriptor = nullptr;

  // 这套宏非常关键，我们此前在注释中就看过了，应该是用于生成ParameterSet数组
  POLPARAMS_BEGIN(eval_params)
    POLPARAM(filename)
    POLPARAM(creation_mode)
    POLPARAM(flags)
    POLPARAM(security_descriptor)
  POLPARAMS_END;

  PolicyResult pr;
  PolicyProcessor pol_ev(policy);	// 架在PolicyBuffer上的processor，操纵者

  // Test should match the first rule set.
  // 看起来eval_params就是ParameterSet数组，这一组参数理应匹配到第一套规则
  // 设置kShortEval模式，匹配到了一个就会返回
  pr = pol_ev.Evaluate(kShortEval, eval_params, _countof(eval_params));
  // 结果应该是匹配到
  EXPECT_EQ(POLICY_MATCH, pr);、
  // 结果应该是ASK_BROKER
  EXPECT_EQ(ASK_BROKER, pol_ev.GetAction());

  // Test should still match the first rule set.
  // 再次审判，结果应该还是一样的
  pr = pol_ev.Evaluate(kShortEval, eval_params, _countof(eval_params));
  EXPECT_EQ(POLICY_MATCH, pr);
  EXPECT_EQ(ASK_BROKER, pol_ev.GetAction());

  // Changing creation_mode such that evaluation should not match any rule.
  // 修改一下参数的值，此时应该匹配不到规则
  creation_mode = CREATE_NEW;
  pr = pol_ev.Evaluate(kShortEval, eval_params, _countof(eval_params));
  EXPECT_EQ(NO_POLICY_MATCH, pr);

  // Changing creation_mode such that evaluation should match rule #2.
  // 再次修改一个参数值，此时应该匹配到第二套规则
  creation_mode = OPEN_ALWAYS;
  pr = pol_ev.Evaluate(kShortEval, eval_params, _countof(eval_params));
  EXPECT_EQ(POLICY_MATCH, pr);
  EXPECT_EQ(FAKE_ACCESS_DENIED, pol_ev.GetAction());

  delete[] reinterpret_cast<char*>(policy);
}
```

唯一的一点就是`POLPARAMS_BEGIN`等一整套宏机制了。在policy_params.h中找到了定义，算是本次分析之旅的漏网之鱼：

```cpp
// Warning: The following macros store the address to the actual variables, in
// other words, the values are not copied.
#define POLPARAMS_BEGIN(type) class type { public: enum Args {
#define POLPARAM(arg) arg,
#define POLPARAMS_END(type) PolParamLast }; }; \
  typedef sandbox::ParameterSet type##Array [type::PolParamLast];
```

而测试代码中使用的套路和此前看到的注释是有些差别的，因为测试代码做了简化，自己重写了这套宏：

```cpp
#define POLPARAMS_BEGIN(x) sandbox::ParameterSet x[] = {
#define POLPARAM(p) sandbox::ParamPickerMake(p),
#define POLPARAMS_END }
```

用它展开上面的代码：

```cpp
sandbox::ParameterSet eval_params[] = {
  sandbox::ParamPickerMake(filename),
  sandbox::ParamPickerMake(creation_mode),
  sandbox::ParamPickerMake(flags),
  sandbox::ParamPickerMake(security_descriptor),
}

// ParamPickerMake(&parameter) => ParameterSetEx<type>(address) => ParameterSet(ArgType, address)
```

相比较policy_params.h的定义，省去了类型的定义，直接使用了`ParameterSet`数组。

到此，通过对测试用例的解读，我们已经了解了使用这套policy engine的套路。下一节我们去看看它的上层操纵者：low-level policy。