---
title: Chromium-sandbox-LowLevelPolicy-analysis
date: 2018-06-03 11:22:11
categories: 源码剖析
tags:
	- chromium
	- chromium-sandbox
---

本篇是sandbox源码剖析的第十三篇，主要分析了架设在PolicyEngine之上的low-level Policy机制。Low-level policy为TargetPolicy（实际上是PolicyBase）所用，用以evaluate high-level操作的可行性（job+alternative desktop+stricted token(include IL/AC)几乎把target的路堵死了，所以需要借助一种渠道来执行一些操作，于是就有了子系统三大组件）。阅读本篇前，请先阅读前四篇及第十到十二篇。

想要流程的阅读本系列你需要以下几个条件：
1. 较高水平的C++编码能力（至少通读C++ Primer 5th，刷过课后题，有一定编码量）。
2. 熟悉Windows API编程，尤其是安全相关的内容。
3. 对二进制安全有一定了解，熟悉各类型安全漏洞成因、漏洞缓解措施及bypass手法。

<!--more-->

# chromium-sandbox-LowLevelPolicy-analysis

在了解了`PolicyOpcode`, `OpcodeFactory`以及`ParameterSet`的机理以后，终于可以看看架在其上的low-level policy。low-level policy与dispatcher和interception共同构成了子系统的三大组件。

low-level policy在policy_low_level.h中定义。chromium封装了两个类：`LowLevelPolicy`和`PolicyRule`。在头文件中已经给出了low-level policy的说明，以及简单的使用示例。

```cpp
// Low level policy classes.
// Built on top of the PolicyOpcode and OpcodeFatory, the low level policy
// provides a way to define rules on strings and numbers but it is unaware
// of Windows specific details or how the Interceptions must be set up.
// To use these classes you construct one or more rules and add them to the
// LowLevelPolicy object like this:
//
// 我们此前在OpcodeFactory的单元测试代码中看到过rule
// low-level policy只是提供了定义rule的接口，它对于windows的特定细节以及Interceptions如何被安装
// 并不关心。使用的套路就是定义多个PolicyRule对象，每个对象代表一组rule，添加match的opcode，然后
// add到管理者LowLevelPolicy对象，同样是add/fire的设计模式。
// 
//   PolicyRule rule1(ASK_BROKER);
//   rule1.AddStringMatch(IF, 0, L"\\\\/?/?\\c:\\*Microsoft*\\*.exe", true);
//   rule1.AddNumberMatch(IF_NOT, 1, CREATE_ALWAYS, EQUAL);
//   rule1.AddNumberMatch(IF, 2, FILE_ATTRIBUTE_NORMAL, EQUAL);
//
//   PolicyRule rule2(FAKE_SUCCESS);
//   rule2.AddStringMatch(IF, 0, L"\\\\/?/?\\Pipe\\Chrome.*", false));
//   rule2.AddNumberMatch(IF, 1, OPEN_EXISTING, EQUAL));
//
//   LowLevelPolicy policyGen(*policy_memory);
//   policyGen.AddRule(kNtCreateFileSvc, &rule1);
//   policyGen.AddRule(kNtCreateFileSvc, &rule2);
//   policyGen.Done();
//
// At this point (error checking omitted) the policy_memory can be copied
// to the target process where it can be evaluated.
// 根据注释的说明，LowLevelPolicy是由broker控制的，在fire之后，需要把policy_memory传给target进程。
```

## `LowLevelPolicy`

类头定义：

```cpp
// Provides the means to collect rules into a policy store (memory)
class LowLevelPolicy {
 public:
  // policy_store: must contain allocated memory and the internal
  // size fields set to correct values.
  // policy_store是个PolicyGlobal指针，这理应是个由外部分配的内存空间，具有某种设计的结构
  explicit LowLevelPolicy(PolicyGlobal* policy_store);

  // Destroys all the policy rules.
  ~LowLevelPolicy();

  // Adds a rule to be generated when Done() is called.
  // service: The id of the service that this rule is associated with,
  // for example the 'Open Thread' service or the "Create File" service.
  // returns false on error.
  // 增加rule的方法
  bool AddRule(int service, PolicyRule* rule);

  // Generates all the rules added with AddRule() into the memory area
  // passed on the constructor. Returns false on error.
  // 生成所有由AddRule添加的rule，导入到policy_store_
  bool Done();

 private:
  struct RuleNode {
    const PolicyRule* rule;
    int service;
  };
  std::list<RuleNode> rules_;	//规则RuleNode链表
  PolicyGlobal* policy_store_;//依赖于构造器传入的外部PolicyGlobal对象指针
  DISALLOW_IMPLICIT_CONSTRUCTORS(LowLevelPolicy);
};
```

根据头的设计，我们可以轻易的判断出，`rules_`是负责处理add操作，而fire时再把`rules_`的内容倒换到`policy_store_`中。`policy_store_`是broker需要copy到target的buffer。

`RuleNode`的设计已经很清楚了，只是关联了`PolicyRule`和`service`，那么`PolicyGloabl`是什么样的设计呢？

```cpp
// Defines the memory layout of the policy. This memory is filled by
// LowLevelPolicy object.
// For example:
//
//  [Service 0] --points to---\
//  [Service 1] --------------|-----\
//   ......                   |     |
//  [Service N]               |     |
//  [data_size]               |     |
//  [Policy Buffer 0] <-------/     |
//  [opcodes of]                    |
//  .......                         |
//  [Policy Buffer 1] <-------------/
//  [opcodes]
//  .......
//  .......
//  [Policy Buffer N]
//  [opcodes]
//  .......
//   <possibly unused space here>
//  .......
//  [opcode string ]
//  [opcode string ]
//  .......
//  [opcode string ]
struct PolicyGlobal {
  PolicyBuffer* entry[kMaxServiceCount];	// kMaxServiceCount是个const，值为64
  size_t data_size;
  PolicyBuffer data[1];
};
```

`PolicyBuffer`我们上一篇已经了解了，实际上是若干`PolicyOpcode`：

```cpp
struct PolicyBuffer {
  size_t opcode_count;
  PolicyOpcode opcodes[1];
};
```

那么这一设计就很明显了，因为我们知道`PolicyBuffer`是不定长的，所以不能简单的通过`PolicyGlobal.data[index]`方式来定位`PolicyBuffer`，所以`PolicyGlobal`中有一个entry指针数组，用以索引每个`PolicyBuffer`。

而上面的图示已经描摹的很清楚了。另一点要注意的就是`kMaxServiceCount`这个值，看起来它和crosscall机制中的service有关系，每个service对应一个`PolicyBuffer`，至于他们具体是如何联系的，等分析具体子系统时就会看到。

### 构造/析构器

其实都没什么好说的，无非就是成员的初始化和清理：

```cpp
LowLevelPolicy::LowLevelPolicy(PolicyGlobal* policy_store)
    : policy_store_(policy_store) {}

LowLevelPolicy::~LowLevelPolicy() {
  // Delete all the rules.
  typedef std::list<RuleNode> RuleNodes;
  for (RuleNodes::iterator it = rules_.begin(); it != rules_.end(); ++it) {
    delete it->rule;
  }
}
```

### `Add`

```cpp
// Adding a rule is nothing more than pushing it into an stl container. Done()
// is called for the rule in case the code that made the rule in the first
// place has not done it.
// add方法不过是把PolicyRule和service二元组置入容器中
bool LowLevelPolicy::AddRule(int service, PolicyRule* rule) {
  // 如果已经fire了，那就不能再add了
  if (!rule->Done()) {
    return false;
  }

  // 看起来PolicyRule支持复制构造，也就是说rules_容器中存储的不是传入的PolicyRule本体
  // 而是一个复制对象，等到我们分析PolicyRule时再看看是否有显式定义复制构造器
  PolicyRule* local_rule = new PolicyRule(*rule);
  RuleNode node = {local_rule, service};
  rules_.push_back(node);
  return true;
}
```

### `Done`

```cpp
// Here is where the heavy byte shuffling is done. We take all the rules and
// 'compile' them into a single memory region. Now, the rules are in random
// order so the first step is to reorganize them into a stl map that is keyed
// by the service id and as a value contains a list with all the rules that
// belong to that service. Then we enter the big for-loop where we carve a
// memory zone for the opcodes and the data and call RebindCopy on each rule
// so they all end up nicely packed in the policy_store_.
// 把rules_部署到PolicyGlobal中，经历了一个stl map的整理以及copy到最终的memory的过程
bool LowLevelPolicy::Done() {
  typedef std::list<RuleNode> RuleNodes;
  typedef std::list<const PolicyRule*> RuleList;
  typedef std::map<uint32_t, RuleList> Mmap;
  Mmap mmap;	// 整理所用map，service id做键，RuleList存储一堆PolicyRule

  // 简单的迭代，把rules_打入到mmap
  for (RuleNodes::iterator it = rules_.begin(); it != rules_.end(); ++it) {
    mmap[it->service].push_back(it->rule);
  }

  // 定位到PolicyGlobal中第一个PolicyBuffer处
  PolicyBuffer* current_buffer = &policy_store_->data[0];
  // buffer_end指向这块memory的末尾，用于后续尺寸检查防止溢出
  char* buffer_end =
      reinterpret_cast<char*>(current_buffer) + policy_store_->data_size;
  size_t avail_size = policy_store_->data_size;

  // 对mmap迭代
  for (Mmap::iterator it = mmap.begin(); it != mmap.end(); ++it) {
    uint32_t service = (*it).first;
    // 首先service的id不应该超过kMaxServiceCount，否则无处安放，一个id一个坑
    if (service >= kMaxServiceCount) {
      return false;
    }
    // 调整entry[service]指针指向第一个坑位，也就是data[0]
    policy_store_->entry[service] = current_buffer;

    RuleList::iterator rules_it = (*it).second.begin();
    RuleList::iterator rules_it_end = (*it).second.end();

    size_t svc_opcode_count = 0;

    // 迭代该service id的PolicyRule
    for (; rules_it != rules_it_end; ++rules_it) {
      const PolicyRule* rule = (*rules_it);
      // 获取本次PolicyRule的opcode个数
      size_t op_count = rule->GetOpcodeCount();

      // 算出opcode的尺寸
      size_t opcodes_size = op_count * sizeof(PolicyOpcode);
      // 剩余的可用空间是否足以容纳opcodes
      if (avail_size < opcodes_size) {
        return false;
      }
      // 扣除opcodes本身的尺寸
      size_t data_size = avail_size - opcodes_size;
      // 索引到本次处理的PolicyBuffer中第svc_opcode_count个PolicyOpcode
      // 存储opcode，opcodes在rule中维护，所以copy是它的事情。buffer_end和data_size应该是用于
      // 防止溢出和回置data使用的尺寸
      PolicyOpcode* opcodes_start = &current_buffer->opcodes[svc_opcode_count];
      if (!rule->RebindCopy(opcodes_start, opcodes_size, buffer_end,
                            &data_size)) {
        return false;
      }
      // 再度扣除data尺寸，调整buffer_end和avail_size
      size_t used = avail_size - data_size;
      buffer_end -= used;
      avail_size -= used;
      svc_opcode_count += op_count;	// 每处理一个PolicyRule，都会存一组PolicyOpcode
      							  // 所以不是简单的自增1，而是自增op_count
    }

    current_buffer->opcode_count += svc_opcode_count;
    // 这里计算的方式有点奇怪，current_buffer[0]的尺寸应该是一个PolicyOpcode+size_t的大小
    // 而svc_opcode_count * sizeof(PolicyOpcode)是svc_opcode_count个PolicyOpcode的大小
    // 假设svc_opcode_count是N个，那么就是
    // (N * sizeof(PolicyOpcode) / (sizeof(PolicyOpcode)+4))
    // 此时算出来的值会小于N，而current_buffer在调整位置时也很奇怪
    // 如果是按照我的理解，应该写成：
    // current_buffer = &(current_buffer->opcodes[svc_opcode_count]);
    // 但这里是把PolicyOpcode的尺寸换成了PolicyBuffer单位个
    // current_buffer[policy_byte_count+1]中多出来的1个应该是给current_buffer->opcode_count
    // 这样的计算就会导致新的current_buffer和上一次数据的结尾之间会有一段空白
    // 而另一方面，rule->RebindCopy设置的data_size就很关键了，它是否和此处计算的一致呢
    // 到目前为止，看起来这是个bug
    size_t policy_byte_count =
        (svc_opcode_count * sizeof(PolicyOpcode)) / sizeof(current_buffer[0]);
    current_buffer = &current_buffer[policy_byte_count + 1];
  }

  return true;
}
```

## `PolicyRule`

```cpp
// Provides the means to collect a set of comparisons into a single
// rule and its associated action.
class PolicyRule {
  friend class LowLevelPolicy;

 public:
  explicit PolicyRule(EvalResult action);
  PolicyRule(const PolicyRule& other);
  ~PolicyRule();

  // Adds a string comparison to the rule.
  // rule_type: possible values are IF and IF_NOT.
  // parameter: the expected index of the argument for this rule. For example
  // in a 'create file' service the file name argument can be at index 0.
  // string: is the desired matching pattern.
  // match_opts: if the pattern matching is case sensitive or not.
  bool AddStringMatch(RuleType rule_type,
                      int16_t parameter,
                      const wchar_t* string,
                      StringMatchOptions match_opts);

  // Adds a number match comparison to the rule.
  // rule_type: possible values are IF and IF_NOT.
  // parameter: the expected index of the argument for this rule.
  // number: the value to compare the input to.
  // comparison_op: the comparison kind (equal, logical and, etc).
  bool AddNumberMatch(RuleType rule_type,
                      int16_t parameter,
                      uint32_t number,
                      RuleOp comparison_op);

  // Returns the number of opcodes generated so far.
  size_t GetOpcodeCount() const { return buffer_->opcode_count; }

  // Called when there is no more comparisons to add. Internally it generates
  // the last opcode (the action opcode). Returns false if this operation fails.
  bool Done();

 private:
  void operator=(const PolicyRule&);
  // Called in a loop from AddStringMatch to generate the required string
  // match opcodes. rule_type, match_opts and parameter are the same as
  // in AddStringMatch.
  bool GenStringOpcode(RuleType rule_type,
                       StringMatchOptions match_opts,
                       uint16_t parameter,
                       int state,
                       bool last_call,
                       int* skip_count,
                       base::string16* fragment);

  // Loop over all generated opcodes and copy them to increasing memory
  // addresses from opcode_start and copy the extra data (strings usually) into
  // decreasing addresses from data_start. Extra data is only present in the
  // string evaluation opcodes.
  bool RebindCopy(PolicyOpcode* opcode_start,
                  size_t opcode_size,
                  char* data_start,
                  size_t* data_size) const;
  PolicyBuffer* buffer_;	// 内部当然得有一个PolicyBuffer来承载opcode groups
  OpcodeFactory* opcode_factory_;	// 指向管理这个group的工厂类
  EvalResult action_;
  bool done_;
};
```

`Add`和`Done`接口，单从函数参数来看，其用意已十分明显。值得注意的是该类重载了赋值操作符，但注意它是private权限，即只能在内部使用。

### 构造/析构器

```cpp
PolicyRule::PolicyRule(EvalResult action) : action_(action), done_(false) {
  // memory是size_t + PolicyOpcode + 4096的大小
  char* memory = new char[sizeof(PolicyBuffer) + kRuleBufferSize];
  buffer_ = reinterpret_cast<PolicyBuffer*>(memory);
  buffer_->opcode_count = 0;
  // new出一个工厂类，top是buffer_->opcodes[0]，bottom是memory[sizeof(PolicyBuffer) + kRuleBufferSize]
  // 使用的是第一个参数为PolicyBuffer *的构造器
  opcode_factory_ =
      new OpcodeFactory(buffer_, kRuleBufferSize + sizeof(PolicyOpcode));
}

PolicyRule::~PolicyRule() {
  delete[] reinterpret_cast<char*>(buffer_);
  delete opcode_factory_;
}
```

复制构造有点意思：

```cpp
PolicyRule::PolicyRule(const PolicyRule& other) {
  if (this == &other)
    return;
  action_ = other.action_;
  done_ = other.done_;
  // buffer_size是size_t + PolicyOpcode + 4096的大小，这和构造器中的计算一致
  size_t buffer_size = sizeof(PolicyBuffer) + kRuleBufferSize;
  char* memory = new char[buffer_size];
  // 需要重新开辟memory，然后深拷贝，所以要显式定义复制构造
  buffer_ = reinterpret_cast<PolicyBuffer*>(memory);
  memcpy(buffer_, other.buffer_, buffer_size);

  // opcode_buffer指向的是PolicyOpcode
  // next_opcode指向的是所有的PolicyOpcode的末尾
  char* opcode_buffer = reinterpret_cast<char*>(&buffer_->opcodes[0]);
  char* next_opcode = &opcode_buffer[GetOpcodeCount() * sizeof(PolicyOpcode)];
  // 新的工厂类使用的是上一次剩余的那部分free空间
  // 所以next_opcode指向上一次的top，而other.opcode_factory_->memory_size()是上一次剩余的free大小
  // 新的工厂类的top和bottom实际上是上一次的free空间
  opcode_factory_ =
      new OpcodeFactory(next_opcode, other.opcode_factory_->memory_size());
}
```

教科书般的复制构造，两个点都注意到了：一是buffer的深拷贝，二是工厂类对象。

### 两个`Add`

```cpp
// There are 'if' rules and 'if not' comparisons
enum RuleType {
  IF = 0,
  IF_NOT = 1,
};

// Possible comparisons for numbers
enum RuleOp {
  EQUAL,
  AND,
  RANGE  // TODO(cpu): Implement this option.
};

bool PolicyRule::AddNumberMatch(RuleType rule_type,
                                int16_t parameter,
                                uint32_t number,
                                RuleOp comparison_op) {
  if (done_) {
    // Do not allow to add more rules after generating the action opcode.
    return false;
  }
  // rule_type映射成kPolNegateEval标记
  uint32_t opts = (rule_type == IF_NOT) ? kPolNegateEval : kPolNone;

  // comparison_op映射成整型match的类别，是相等还是逻辑与还是范围(这个还没实现)
  // 然后调用工厂类的具体make接口做出PolicyOpcode
  if (EQUAL == comparison_op) {
    if (!opcode_factory_->MakeOpNumberMatch(parameter, number, opts))
      return false;
  } else if (AND == comparison_op) {
    if (!opcode_factory_->MakeOpNumberAndMatch(parameter, number, opts))
      return false;
  }
  ++buffer_->opcode_count;
  return true;
}

enum StringMatchOptions {
  CASE_SENSITIVE = 0,    // Pay or Not attention to the case as defined by
  CASE_INSENSITIVE = 1,  // RtlCompareUnicodeString windows API.
  EXACT_LENGTH = 2       // Don't do substring match. Do full string match.
};

bool PolicyRule::AddStringMatch(RuleType rule_type,
                                int16_t parameter,
                                const wchar_t* string,
                                StringMatchOptions match_opts) {
  if (done_) {
    // Do not allow to add more rules after generating the action opcode.
    return false;
  }

  const wchar_t* current_char = string;
  uint32_t last_char = kLastCharIsNone;
  int state = PENDING_NONE;
  int skip_count = 0;       // counts how many '?' we have seen in a row.
  base::string16 fragment;  // accumulates the non-wildcard part.

  // 这里是处理windows路径转义的操作，算法就不细说了，工厂类的Make封装在GenStringOpcode
  while (L'\0' != *current_char) {
    switch (*current_char) {
      case L'*':
        if (kLastCharIsWild & last_char) {
          // '**' and '&*' is an error.
          return false;
        }
        if (!GenStringOpcode(rule_type, match_opts, parameter, state, false,
                             &skip_count, &fragment)) {
          return false;
        }
        last_char = kLastCharIsAsterisk;
        state = PENDING_ASTERISK;
        break;
      case L'?':
        if (kLastCharIsAsterisk == last_char) {
          // '*?' is an error.
          return false;
        }
        if (!GenStringOpcode(rule_type, match_opts, parameter, state, false,
                             &skip_count, &fragment)) {
          return false;
        }
        ++skip_count;
        last_char = kLastCharIsQuestionM;
        state = PENDING_QMARK;
        break;
      case L'/':
        // Note: "/?" is an escaped '?'. Eat the slash and fall through.
        if (L'?' == current_char[1]) {
          ++current_char;
        }
        FALLTHROUGH;
      default:
        fragment += *current_char;
        last_char = kLastCharIsAlpha;
    }
    ++current_char;
  }

  if (!GenStringOpcode(rule_type, match_opts, parameter, state, true,
                       &skip_count, &fragment)) {
    return false;
  }
  return true;
}
```

内部的`GenStringOpcode` private函数才是调用工厂类Make接口的关键：

```cpp
// This function get called from a simple state machine implemented in
// AddStringMatch() which passes the current state (in state) and it passes
// true in last_call if AddStringMatch() has finished processing the input
// pattern string and this would be the last call to generate any pending
// opcode. The skip_count is the currently accumulated number of '?' seen so
// far and once the associated opcode is generated this function sets it back
// to zero.
bool PolicyRule::GenStringOpcode(RuleType rule_type,
                                 StringMatchOptions match_opts,
                                 uint16_t parameter,
                                 int state,
                                 bool last_call,
                                 int* skip_count,
                                 base::string16* fragment) {
  // The last opcode must:
  //   1) Always clear the context.
  //   2) Preserve the negation.
  //   3) Remove the 'OR' mode flag.
  // 根据传入的参数，设置相应的标记
  uint32_t options = kPolNone;
  if (last_call) {
    if (IF_NOT == rule_type) {
      options = kPolClearContext | kPolNegateEval;
    } else {
      options = kPolClearContext;
    }
  } else if (IF_NOT == rule_type) {
    options = kPolUseOREval | kPolNegateEval;
  }

  PolicyOpcode* op = nullptr;

  // The fragment string contains the accumulated characters to match with, it
  // never contains wildcards (unless they have been escaped) and while there
  // is no fragment there is no new string match opcode to generate.
  if (fragment->empty()) {
    // There is no new opcode to generate but in the last call we have to fix
    // the previous opcode because it was really the last but we did not know
    // it at that time.
    if (last_call && (buffer_->opcode_count > 0)) {
      op = &buffer_->opcodes[buffer_->opcode_count - 1];
      op->SetOptions(options);
    }
    return true;
  }

  // 这里各找各妈，根据state状态传参调用Make
  if (PENDING_ASTERISK == state) {
    if (last_call) {
      op = opcode_factory_->MakeOpWStringMatch(parameter, fragment->c_str(),
                                               kSeekToEnd, match_opts, options);
    } else {
      op = opcode_factory_->MakeOpWStringMatch(
          parameter, fragment->c_str(), kSeekForward, match_opts, options);
    }

  } else if (PENDING_QMARK == state) {
    op = opcode_factory_->MakeOpWStringMatch(parameter, fragment->c_str(),
                                             *skip_count, match_opts, options);
    *skip_count = 0;
  } else {
    if (last_call) {
      match_opts = static_cast<StringMatchOptions>(EXACT_LENGTH | match_opts);
    }
    op = opcode_factory_->MakeOpWStringMatch(parameter, fragment->c_str(), 0,
                                             match_opts, options);
  }
  if (!op)
    return false;
  ++buffer_->opcode_count;
  fragment->clear();
  return true;
}
```

### `Done`

这个函数就很简单了：

```cpp
bool PolicyRule::Done() {
  if (done_) {
    return true;
  }
  if (!opcode_factory_->MakeOpAction(action_, kPolNone))	// action是构造器中传入的
    return false;
  ++buffer_->opcode_count;
  done_ = true;
  return true;
}
```

一组`AddXXX`操作后跟一个`Done`表示一个opcode group设置完毕。

### `RebindCopy`

此前在分析`LowLevelPolicy::Done`时，我们了解到将`PolicyRule`的buffer拷贝到`LowLevelPolicy`的`PolicyGlobal`的操作就是由它来完成的，其中还留下了一个疑问，即`data_size`的尺寸计算是否和`LowLevelPolicy::Done`最终的`current_buffer`挪移单位相吻合。

```cpp
bool PolicyRule::RebindCopy(PolicyOpcode* opcode_start,	// 用于存储PolicyOpcode的buffer起始
                            size_t opcode_size,			// 要存储多少个PolicyOpcode
                            char* data_start,			// 存储buffer的末尾
                            size_t* data_size) const {
  // 这一组有多少个opcode
  size_t count = buffer_->opcode_count;
  for (size_t ix = 0; ix != count; ++ix) {
    if (opcode_size < sizeof(PolicyOpcode)) {
      return false;
    }
    PolicyOpcode& opcode = buffer_->opcodes[ix];
    // 这里又调用了PolicyOpcode的复制构造
    *opcode_start = opcode;
    if (OP_WSTRING_MATCH == opcode.GetID()) {
      // For this opcode argument 0 is a delta to the string and argument 1
      // is the length (in chars) of the string.
      const wchar_t* str = opcode.GetRelativeString(0);
      size_t str_len;
      opcode.GetArgument(1, &str_len);
      str_len = str_len * sizeof(wchar_t);
      if ((*data_size) < str_len) {
        return false;
      }
      // data_size扣除了字符串的尺寸
      *data_size -= str_len;
      data_start -= str_len;
      // 如果是字符串匹配的话，还要把字符串拷贝到GlobalPolicy的末尾
      memcpy(data_start, str, str_len);
      // Recompute the string displacement
      ptrdiff_t delta = data_start - reinterpret_cast<char*>(opcode_start);
      // arguments_[0]保存的始终是offset，这里要重新调整opcode_start的arguments_[0]
      opcode_start->SetArgument(0, delta);
    }
    ++opcode_start;
    opcode_size -= sizeof(PolicyOpcode);
  }

  return true;
}
```

再回到`LowLevelPolicy::Done`中，看看上下文：

```cpp
size_t data_size = avail_size - opcodes_size;
PolicyOpcode* opcodes_start = &current_buffer->opcodes[svc_opcode_count];
if (!rule->RebindCopy(opcodes_start, opcodes_size, buffer_end,
                      &data_size)) {
  return false;
}
// used表示的是多个PolicyOpcode和字符串共用的尺寸
size_t used = avail_size - data_size;
// 这里buffer_end向前调整了used，我觉着也有问题，应该只减去字符串占用的长度吧
// 如果减掉used尺寸这意味着浪费了大量的空间，而且avail_size的扣除就有问题了，这种处理实际上应该减去
// 2倍的opcodes_size
buffer_end -= used;
avail_size -= used;
svc_opcode_count += op_count;
```

data_size只是在内部扣除了string的尺寸，所以和下面的计算也并不一致：

```cpp
current_buffer->opcode_count += svc_opcode_count;
size_t policy_byte_count =
  (svc_opcode_count * sizeof(PolicyOpcode)) / sizeof(current_buffer[0]);
current_buffer = &current_buffer[policy_byte_count + 1];
```

另外，在`Done`中，内层循环执行一轮之后，`avail_size`理应再扣除`PolicyBuffer`的`opcode_count`大小，但在源码中却没有看到。

此后，就是之前阅读时，发现的最为奇怪的`PolicyBuffer`向后移动操作：

```cpp
// 这一句是没有问题的
current_buffer->opcode_count += svc_opcode_count;
// 莫名其妙的计算，假设PolicyOpcode尺寸为S,count为N，那就是
// N*S/(S+4)，完全不知道他在算什么。结果估计是N-1，假设它是N-1，我们再看
// 此后调整current_buffer向后移动，指向了current_buffer[N]，向后移动了N个
// 如此调整，current_buffer和前面的数据之间势必有存在空隙，avail_size就更不对了
// 这写的什么东西啊。。。
size_t policy_byte_count =
  (svc_opcode_count * sizeof(PolicyOpcode)) / sizeof(current_buffer[0]);
current_buffer = &current_buffer[policy_byte_count + 1];
```

感觉这个`Done`的处理问题很大，不知道是怎么过的审。后续需要调试来确定不是我分析有误，而是本来写的就有问题。