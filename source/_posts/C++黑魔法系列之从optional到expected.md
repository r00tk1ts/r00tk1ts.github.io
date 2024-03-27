---
title: C++黑魔法系列之从optional到expected
date: 2024-02-01 10:45:35
tags: [cpp, cpp-templates]
category: 源码剖析
---

众所周知，想要在C++中写出通用的框架、组件代码并不简单，一来C++本身庞然大物包罗万象，大部分开发者并不了解像是”茴有四种写法“这种语言律师津津乐道的课题，对于晦涩难懂的模板元更是谈之色变，二来是历史悠久，在发展过程中为了向前兼容，遗留了各种特例特办的技术债，导致系统臃肿不堪。因此，尽管C++生态相当茂盛，但权威的第三方库却屈指可数（甚至其标准库都是风风雨雨缝缝补补，~~偷得人家boost就剩个底裤~~）。阅读优秀的开源代码是提升代码水平的捷径，本篇文章我们解读modern C++中常用的脚手架`std::optional`，并进阶到folly库所实现的更加强大的`folly::Expected`。

<!-- more -->

# C++黑魔法系列之从optional到expected
在日常编码过程中，我们经常会遇到这样一个问题：某些场景我们需要区分空值和默认值（一般是零值）。比如，对于一个返回值类型为`int`的函数，当值为0时，它的物理意义是什么，是真正的零值、还是代表了一种表达为空的属性（是否存在，是否有效等物理意义）？这是函数设计者需要结合实际场景来认真考虑的事。当然，二者的区分也不仅仅限定于某个语种，更不限定于函数的返回值。

## 前菜：protobuf的缺省与默认值
大名鼎鼎的protobuf协议便在这个问题上纠结了很久，从v2的required/optional tag的强硬派重拳出击，比如协议：

```protobuf
// proto2
message SearchRequest {
  required string query = 1;                    # 必需
  optional int32 page_number = 2 [default=0];   # 可选，默认值为0，有hasPageNumber() method
}
```

在proto2中，被标记为required的字段是必需品，而标记了optional的字段则是可缺省的，我们可以为其赋予默认值（比如page_number被显式地设置了一个默认值0）。每一个optional字段都会生成相应的`hasXXX` method桩代码，用以检查是否缺省。

> 实际上proto2的序列化有坑，不论optional字段是真的缺省，还是被显式赋予了一个默认值，序列化以后这个信息都会被擦除，反序列化后，haxXXX总是返回false。归根结底是把默认值和缺省混为一谈了。

> 对proto2来说，若无显式设置默认值，则使用对应类型的零值做默认值，上例的page_number实际上不需要显式设置，这里只是为了演示才画蛇添足。

在经历了大刀阔斧的升级后，v3干脆移除了required/optional tag，所有的字段通通都是optional，但同时对于原始的基础类型，也不再生成`haxXXX`这样的桩代码。那么，对于上述协议：

```protobuf
// proto3
message SearchRequest {
  string query = 1;
  int32 page_number = 2;
}
```

page_number是一个基础类型，我怎么区分page_number确实是0，还是压根没填呢？在损失了`hasXXX`方法后，我们只得另辟蹊径，一般来说，有两种迂回的办法：使用oneof嵌套，或是定义wrapper类型。前者是借助了oneof的特性，后者则是绕开了只对基础型移除`hasXXX`的限制。设计师面对上述两种黑魔法的大行其道，最终还是向用户妥协，自3.15版本起，protobuf又重新支持了optional tag，此时就又可以像v2那样对page_number生成`hasXXX`的桩代码了。

摆上这样一道前菜只是为了热身，更多关于protobuf的内容就不在这里展开，以免喧宾夺主。

> v3的改革历来争议很大，核心争论点主要集中于required，详见stackoverflow高赞回答：https://stackoverflow.com/questions/31801257/why-required-and-optional-is-removed-in-protocol-buffers-3


## 函数的多返回值设计
通过前面的热身我们不难发现，默认值与空值，根本就是两个维度上的事儿，protobuf的变革历程深刻地印证了这一点。那么，对于函数返回值的设计来说，也就意味着需要返回两个值，第一个是值对应的类型，第二个则是一个布尔，用来表示是否存在或者是否成功等等。

对于早期的编程语言，比如C，由于严格贯彻数学上函数的定义，返回值只能有一个，彼时，对于有多个返回值要求的场合，一般有两种解法：

1. 聚合多个返回值构建成新的struct
2. 通过参数传入，也就是OUT型参数

```c
/// function 1
struct result {
  int value;
  bool exist;
};

// 此时还要注意返回指针的生命期问题
struct result* func1() { // ... }

/// function 2
int func2(bool *exist) { // ... }
```

这其中第二种解法比较有年代感，早期C标准库、linux内核代码常见这种设计，而Windows SDK的C API则都是这种风格，甚至变本加厉，通过IN/OUT宏来标记参数去辅助接口的说明。比如，下面是Windows创建进程的C API（这个其实还算是相对清爽的）：

```c
BOOL CreateProcessA(
  [in, optional]      LPCSTR                lpApplicationName,
  [in, out, optional] LPSTR                 lpCommandLine,
  [in, optional]      LPSECURITY_ATTRIBUTES lpProcessAttributes,
  [in, optional]      LPSECURITY_ATTRIBUTES lpThreadAttributes,
  [in]                BOOL                  bInheritHandles,
  [in]                DWORD                 dwCreationFlags,
  [in, optional]      LPVOID                lpEnvironment,
  [in, optional]      LPCSTR                lpCurrentDirectory,
  [in]                LPSTARTUPINFOA        lpStartupInfo,
  [out]               LPPROCESS_INFORMATION lpProcessInformation
);
```

而第一种方法则略显笨重，对于天生需要封装的结构体，这种设计是自然而然顺水推舟，但如果像是我们的需求，每次都把某种值类型和一个bool型封装成一个struct，那就显得太臃肿了。

现代的编程语言大多打破了函数只能有一个返回值的常规，最经典的比如go，它可以有任意多个返回值。因此，我们可以编写：

```go
func func3() (int, bool) { // ... }

func main() {
    value, exist := func3()
    fmt.Printf("value=%d,exist=%v", value, exist)
}
```

另一方面，尽管在语法上提供了任意多个返回值的直接支持，但在编写的代码中很少会遇到超过3个的情况。一般来说，返回值超过了3个，要么是函数的职责拆分不合理，要么是缺少结构的必要抽象(往往存在基本类型偏执)。此时，设计者会更倾向于上述的第一种解法。因此，尽管语言本身支持，但在实际编程中却另有取舍。

C++是C的超集，虽然发展到如今的C++23乃至26，早就和C标准分道扬镳了，但从C继承过来的历史包袱却是浑身难受（C风格数组、函数指针、`const char[N]`的字符串字面量等等都太过于原始，与现代语言的设计格格不入，在2024年，你很难想象一个现代编程语言不支持原生的字符串基本型）。

C++的STL弥补了很多语言天然的缺陷，再加上伟大的boost发光发热，硬是让这一难用的语言撑过了最艰难的蜕变期。STL提供了很多好用的容器，对于我们的需求，可以使用`std::pair<int, bool>`来满足：

```cpp
std::pair<int, bool> func4() { return std::make_pair(0, false); }

// before C++11
int main() {
    std::pair<int, bool> ret = func4();
    std::cout << ret.first << "|" << ret.second << std::endl;
    return 0;
}
```

在C++11之前，由于不支持返回类型推导，我们需要显式地定义返回值`ret`的类型，另一方面，对`pair`成员的访问，需要用`.first`，`.second`的形式。虽无go那般优雅，但也差强人意。

`pair`对于那些需要成对存在的场合也相当好用，比如KV型结构`std::map<Key,Value>`的`value_type`就是`std::pair<const Key, Value>`类型。

而在C++11又引入了另一个非常强大的异质容器：`std::tuple`。相比于`std::pair`，`tuple`可以容纳任意多个成员，自然，`pair`能干的活它都能干：

```cpp
std::tuple<int, bool> func5() { return std::make_tuple(0, false); }

// in C++11
int main() {
    auto ret = func5();
    std::cout << std::get<0>(ret) << "|" << std::get<1>(ret) << std::endl;
    return 0;
}
```

相比于`pair`，`tuple`的成员获取显得更加丑陋，这是为了泛用性而不得不做的牺牲。C++14和C++17相继引入了`std::tie`和语法层面的结构化绑定来遮羞：

```cpp
// in C++14
int main() {
    int value;
    bool exist;
    std::tie(value, exist) = func5();
    std::cout << value << "|" << exist << std::endl;
    return 0;
}
```

```cpp
// in C++17
int main() {
    auto [value, exist] = func5();
    std::cout << value << "|" << exist << std::endl;
    return 0;
}
```

嗯，这很modern。

## `std::optional`: 有还是没有啊？
上一节的拆分，我们既没有定义复杂的struct，也没有使用OUT型参数，而是选择了一个折中的方案：`pair`或是`tuple`来做wrapper。实际上，wrapper还可以有另一种封装方式，他不必携带多个返回值，他只需要携带额外的信息即可。对于上例来说，面向exist这个bool，使用C++的`std::optional`更为合适。

`std::optional`是在C++17才正式引入到标准库的，实际上这东西早在boost时代就已经发光发热相当长一段时间了。我们来看一下官方定义：

```cpp
/*
The class template std::optional manages an optional contained value,
i.e. a value that may or may not be present.

A common use case for optional is the return value of a function that may fail. 
As opposed to other approaches, such as std::pair<T, bool>, optional handles 
expensive-to-construct objects well and is more readable, as the intent is expressed 
explicitly.

Any instance of optional<T> at any given point in time either contains a value 
or does not contain a value.

If an optional<T> contains a value, the value is guaranteed to be allocated as part 
of the optional object footprint, i.e. no dynamic memory allocation ever takes place. 
Thus, an optional object models an object, not a pointer, even though operator*() 
and operator->() are defined.

When an object of type optional<T> is contextually converted to bool, 
the conversion returns true if the object contains a value and false 
if it does not contain a value.

The optional object contains a value in the following conditions:
- The object is initialized with/assigned from a value of type T 
  or another optional that contains a value.

The object does not contain a value in the following conditions:
- The object is default-initialized.
- The object is initialized with/assigned from a value of type std::nullopt_t 
  or an optional object that does not contain a value.
- The member function reset() is called.

There are no optional references; a program is ill-formed if it instantiates an optional 
with a reference type. In addition, a program is ill-formed if it instantiates an optional 
with the (possibly cv-qualified) tag types std::nullopt_t or std::in_place_t.
*/
template< class T >
class optional;
```

通过定义可以知晓，`optional<T>`的实例在某一时刻，要么包含一个类型为`T`的值，要么没有值。显然，对于我们的需求来说，它更加契合，毕竟，当`exist`为`false`的时候，其实返回的另一个`value`是没有实际意义的。关于这一点，相信有使用过go的开发者都深以为然，在go中，我们经常写：

```go
if data, err := fun(); err != nil {
    // do something for error case, the data generately is nil or zero-valued
    // ...
} else {
    // use data forward
    // ...
}
```

而对于`fun`的设计者，不得不这样编写他的函数：

```go
func fun() ([]string, error) {
    if !sanityCheck() {
      return nil, errors.New("incorrect msg")  
    }
    return []string{"1","2","3"}, nil
}
```

尽管返回值有两个，但其实同一时刻只有一个是有效的，另一个则按照语言的惯用法设置合理的值。

有了`std::optional`，就可以这样来编写：

```cpp
bool sanityCheck();
int calculate();

std::optional<int> func6() { 
    if (!sanityCheck()) {
        return std::nullopt;
    }
    return {calculate()};
}

// before C++11
int main() {
    if (auto ret = func6(); ret.has_value()) {
        std::cout << *ret << std::endl;
    } else {
        std::cout << "empty result" << std::endl;
    }
    return 0;
}
```

`std::optional<T>`的语义其实就是调用者告诉函数：“这个可以有”，而当函数返回的是一个`std::nullopt`时，就是反馈给调用者：“这个真没有”。

### std::optional的实现
正如官方手册所述，`std::optional`在实现上是一个类模板，本质上来讲它是一个`exist or not`的wrapper。

> If an optional<T> contains a value, the value is guaranteed to be allocated as part 
of the optional object footprint, i.e. no dynamic memory allocation ever takes place. 
Thus, an optional object models an object, not a pointer, even though operator*() 
and operator->() are defined.



## 参考链接

- [How to define an optional field in protobuf 3](https://stackoverflow.com/questions/42622015/how-to-define-an-optional-field-in-protobuf-3)
- [CreateProcessA function](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessa)