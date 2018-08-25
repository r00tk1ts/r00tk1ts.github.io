---
title: C++ Templates Note- 函数模板
date: 2018-08-25 12:29:11
categories: programming-language
tags:
	- cpp
	- cpp-templates
---
《C++ Templates》更新了第二版，内容上更新了C++ 11/15/17标准中模板元编程的大量内容。本文是阅读第一章“函数模板”时做的笔记。

<!--more-->
#C++ Templates Note- 函数模板

函数模板是模板元编程部分最简单的内容了。函数模板是一个函数族，它可以被不同类型的参数调用，长相酷似普通函数，但其某些参数是未决的，需要被函数实例参数化。

## 初探函数模板

### 定义函数模板

```cpp
template<typename T>
T max(T a, T b)
{
  	// if b < a then yield a else yield b
    return b < a ? a : b;
}
```

定义函数模板需要在函数头前加一行`template< comma-separated-list-of-parameters>`。`typename`是关键字，`T`是类型参数，名称随意。只要某种类型支持`<`操作符就可以去适配模板的`T`类型。

> 历史原因，`class`关键字可以替代`typename`，`typename`是C++98标准引入的，此前是`class`。

### 使用函数模板

```cpp
#include "max1.hpp"
#include <iostream>
#include <string>
int main()
{
  int i = 42;
  std::cout << "max(7,i): " << ::max(7,i) << ’\n’;
  double f1 = 3.4; double f2 = -6.7;
  std::cout << "max(f1,f2): " << ::max(f1,f2) << ’\n’;
  std::string s1 = "mathematics"; std::string s2 = "math";
  std::cout << "max(s1,s2): " << ::max(s1,s2) << ’\n’;
}
```

此`::max`非彼`std::max`，是我们上面自己定义的。

函数模板不会为每一种类型都编译出函数实体，而是用到了哪个就编译出哪个。对上面的例子，会编译出三个函数：

```cpp
int max(int, int);
double max(double, double);
std::string max(std::string, std::string);
```

### 两阶段转译

如果某种类型不支持模板内定义的某个操作，但程序中却使用了这种类型作为参数调用模板函数，那么编译时会报错。

```cpp
std::complex<float> c1, c2;	//doesn't provide operator <
...
::max(c1,c2);	// ERROR at compile time
```

模板在编译时有两个阶段：

1. 定义时不会实例化，模板代码本身会检查忽略模板参数以外的正确性，这包括：
   1. 语法错误，比如缺少符号。
   2. 使用不识别的名称且不依赖于模板参数。
   3. 不依赖模板参数的静态断言会被检查。
2. 在实例化阶段，模板代码会检查是否所有的代码都是有效的。也就是说，所有依赖模板参数的部分都会被第二次检查。

```cpp
template<typename T>
void foo(T t)
{
	undeclared(); // first-phase compile-time error ifundeclared() unknown
	undeclared(t); // second-phase compile-time error if undeclared(T) unknown
	static_assert(sizeof(int) > 10, "int too small"); // always fails if sizeof(int)<=10 
	static_assert(sizeof(T) > 10, "T too small"); //fails if instantiated for T with size <=10
}
```

> 某些编译器不会在第一阶段进行完整的检查。所以在模板代码至少实例化一次之前可能不会发现错误。

两阶段转译引入了一个严重的问题：函数模板实例化时编译器需要找到模板的定义，而普通函数编译时仅仅需要声明即可，其定义是链接阶段的事儿。所以如果模板实现不是简单的在一个头文件内部而是分散到了其他源文件中，那么编译时就会有问题。这一问题有着数种方法来解决，以后再了解。

## 模板参数推导

根据使用时传入参数的类型，编译器会自动推导模板参数类型。对于简单的情形当然一目了然。然而，当类型推导存在类型转换时问题就复杂多了。

对模板来说，自动类型转换在类型推导时是有以下限制的：

- 如果调用参数被声明为引用类型，那么类型推导时不会进行类型转换（即使是无关紧要的转换也不行），两个使用同一个模板参数T的参数必须严格一致。
- 如果按值声明参数，那么推导时仅支持无关紧要的类型转换（decay）：const或volatile会被忽略，引用会转换为引用类型，原生数组或函数会转换成对应的指针类型。两个使用同一模板参数T的参数，其decayed类型必须一致。

```cpp
template<typename T>
T max (T a, T b);
… 
int const c = 42;
max(i, c); // OK: T is deduced as int
max(c, c); // OK: T is deduced as int
int& ir = i;
max(i, ir); // OK: T is deduced as int
int arr[4];
foo(&i, arr); // OK: T is deduced as int*
max(4, 7.2);	//ERROR: T can be deduced as int or double
std::string s;
foo("hello", s); //ERROR: T can be deduced as char const[6] or std::string
```

有三种方法来处理这些错误：

1. 做参数类型转换，使二者一致：`max(static_cast<double>(4), 7.2); //OK`
2. 显式指定T的类型来阻止编译器做自动类型推导：`max<double>(4, 7.2); //OK`
3. 指定参数可以使用不同类型。

==============================================

类型推导对默认调用参数并不起作用。例如：

```cpp
template<typename T>
void f(T = "");
...
f(1);	//OK: deduced T to be int, so that it calls f<int>(1)
f();	// ERROR: cannot deduce T
```

如果想要模板去支持默认参数，也是有办法的：

```cpp
template<typename T = std::string>
void f(T = "");
...
f();	//OK
```

## 多重模板参数

模板参数可以有多个，类型也可以不一致。

```cpp
template<typename T1, typename T2>
T1 max (T1 a, T2 b)
{  
  return b < a ? a : b;
}
...
auto m = ::max(4, 7.2);	// OK, but type of first argument defines return type
```

虽然看起来不错但实际上引入了问题，因为a和b类型并不一致，如果返回类型被强制设定成和a一致，那么返回类型永远是a的类型，即使返回了b，也会被自动转换成a的类型。

C++的处理方式：

- 为返回类型引入第三个模板参数类型
- 让编译器决定返回类型
- 声明返回类型为两个参数类型的“通用类型”

### 返回类型的模板参数 

```cpp
template<typename T1， typename T2, typename RT>
RT max(T1 a, T2 b);
```

看起来很美好，但实际上这种不负责任的写法编译器根本无法推导出RT是什么，因为没有任何线索可循。

所以一旦如此定义，那么调用时就必须显式的指定RT的类型。

```cpp
::max<int, double, double>(4, 7.2);	// OK, but tedious
```

这种写法比较啰嗦，可以在定义时利用一点小技巧遮丑：

```cpp
template<typename RT, typename T1, typename T2>
RT max(T1 a, T2 b);
...
::max<double>(4, 7.2)	// OK: return type is doublke, T1 and T2 are deduced
```

此时RT被显式指定，T1和T2由编译器推导。

### 推导返回类型

也可以全权交给编译器来仲裁：

```cpp
template<typename T1, typename T2>
auto max(T1 a, T2 b)
{
    return b < a ? a : b;
}
```

利用了这个自作聪明的关键字`auto`。它会根据函数体内的返回语句来推导其类型。

可以利用C++11的trailing return type语法来使用调用参数。可以声明返回类型继承自operator?:。

```cpp
template<typename T1, typename T2>
auto max(T1 a, T2 b) -> decltype(b<a?a:b)
{  
  return b < a ? a : b;
}
```

如此，返回类型会由operator ?:来决定，以返回一个直观上期望的结果。

注意，

```cpp
template<typename T1, typename T2>
auto max(T1 a, T2 b) -> decltype(b<a?a:b);
```

仅仅只是声明，编译器会使用operator ?:来决定返回类型，但函数实现体内不一定非要使用operator ?:来返回。写成`decltype(true?a:b)`其实是一样的。

然而，在一些情况下这种定义有一个显著的缺点：返回类型可能是个引用类型，因为一些条件下T可能是个引用类型。因此你应该返回T的decayed类型。

```cpp
#include <type_traits>
template<typename T1, typename T2>
auto max(T1 a, T2 b) -> typename std::decay<decltype(true?a:b)>::type
{
  return b < a ? a : b;
}
```

`std::decay<>::type`用来返回decayed类型，它在标准库的`<type_trait>`中定义。

另外auto的初始化总是会decay。

```cpp
int i = 42;
int const & ir = i;	// ir refers to i
auto a = ir;		// a is declared as new object of type int
```

> 初始化总是会decay，但模板函数定义却未必使用auto初始化，所以才要`std::decay<decltype(true?a:b)>::type` ？

###  返回通用类型

C++11以后，C++标准库提供了一种方法来指定更为通用的类型。`std::common_type<>::type`会返回两到多个不同类型的通用类型并以此作为模板参数。

```cpp
#include <type_traits>
template<typename T1, typename T2>
std::common_type_t<T1, T2> max(T1 a, T2 b)
{
    return b < a ? a : b;
}
```

`std::common_type`是一个`type trait`，在`<type_traits>`中定义。

> C++ 11中需要写成`typename std::common_type<T1,T2>::type`，但是C++14中可以简化为`std::common_type_t<T1,T2>`。

关于`std::common_type_t<>`的实现机制，着实好奇。

## 默认模板参数

模板参数也可以有默认值。

```cpp
#include <type_traits>
template<typename T1, typename T2,
		typename RT = std::decay_t<decltype(true ? T1():T2())>>
RT max(T1 a, T2 b)
{
	return b < a ? a : b;          
}
```

除了`std::decay_t<>`用来屏蔽返回引用类型以外，对这个例子来说还需要有能力调用T1和T2的默认构造器（另一个解决方案是使用`std::declval`，但这更为复杂）。

也可以用`std::common_type<>`来指定返回类型的默认值。

```cpp
#include <type_traits>
template<typename T1, typename T2,
		typename RT = std::common_type_t<T1,T2>>
RT max(T1 a, T2 b)
{
     return b < a ? a : b;             
}
```

**综合来说，1.3.2的方法“推导返回类型”是最佳的。**

## 重载函数模板

函数模板也可以重载，即所谓的同名异构。

```cpp
// maximum of two int values:
int max(int a, int b)
{
    return b < a ? a : b;
}

// maximum of two values of any type:
template<typename T>
T max(T a, Tb)
{
    return b < a : a : b;
}

int main()
{
    ::max(7, 42);	// calls the nontemplate for two ints
  	::max(7.0, 42.0);	// calls max<double> (by argument deduction)
  	::max<>(7, 42);	// calls max<int> (by argument deduction)
  	::max<double>(7, 42);	// calls max<double> (no argument deduction)
  	::max('a', 42.7);	// calls the nontemplate for two ints
}
```

非模板函数可以和模板函数混用。如果参数类型完全匹配，非模板比模板生成函数更优先，如果不完全匹配则优先考虑是否可以推导模板，如果模板推导不成再考虑类型转换适配非模板函数。

> 归根结底，非模板函数可以做自动类型转换，但模板参数推导却不行。

重载模板函数有一个特例，可以仅仅改变返回类型：

```cpp
template<typename T1, typename T2>
auto max(T1 a, T2 b)
{
    return b < a ? a : b;
}
template<typename RT, typename T1, typename T2>
RT max(T1 a, T2 b)
{
    return b < a ? a : b;
}

auto a = ::max(4, 7.2);	// uses first template
auto b = ::max<long double>(7.2, 4);	// uses second template
auto c = ::max<int>(4, 7.2);	// ERROR: both function templates match
```

依我看，这纯粹是比较二逼的用法，写这种重载就是坑自己。

为指针和C字符串重载模板函数的例子：

```cpp

#include <cstring>
#include <string>

// maximum of two values of any type:
template<typename T>
T max(T a, T b)
{
    return b < a ? a : b;
}

// maximum of two pointers:
template<typename T>
T *max(T *a, T *b)
{
    return *b < *a ? a : b;
}

// maximum of two C-strings:
char const *max(char const *a, char const *b)
{
    return std::strcmp(b,a) < 0 : a : b;
}

int main()
{
    int a = 7;
  	int b = 42;
  	auto m1 = ::max(a,b);	// max() for two values of type int
  	std::string s1 = "hey";
  	std::string s2 = "you";
 	auto m2 = ::max(s1, s2);	// max() for two values of type std::string
  
  	int *p1 = &b;
  	int *p2 = &a;
  	auto m3 = ::max(p1, p2);	// max() for two pointers
  
  	char const *x = "hello";
  	char const *y = "world";
  	auto m4 = ::max(x,y);	// max() for two C-Strings
}
```

所有的重载都是按值传递。通常来说，重载函数模板时尽可能避免修改并限制参数数量的改动以及显式化指定模板参数。

一个经典的错误范例：

```cpp
#include <cstring>
// maximum of two values of any type (call-by-reference)
template<typename T>
T const &max(T const &a, T const &b)
{
    return b < a ? a : b;
}
// maximum of two C-Strings (call-by-value)
char const *max(char const *a, char const *b)
{
    return std::strcmp(b, a) < 0 ? a : b;
}

// maximum of three values of any type (call-by-reference)
template<typename T>
T const &max(T const &a, T const &b, T const &c)
{
    return max(max(a,b),c);	//error if max(a,b) uses call-by-value
}

int main()
{
    auto m1 = ::max(7, 42, 68);		//OK
  	char const *s1 = "frederic";
  	char const *s2 = "anica";
  	char const *s3 = "lucas";
  	auto m2 = ::max(s1, s2, s3);	//run-time ERROR
}
```

因为对于C字符串来说，`max(a, b)`创建一个新的临时的局部变量并按引用返回，但是该局部变量在return语句执行后返回到main中就生命期失效了（它是调用者一层max的局部变量）。这种错误非常难以鉴别。而` ::max(7, 42, 68)`之所以正确是因为这三个临时局部变量生命周期在main中。

## 用法忌讳

### 传值还是传引用？

为什么一定要传值而不是传引用。通常来说，对基本类型以外的类型传引用都是被推崇的，因为这可以省去无谓的copy。

然而，对于模板来说，传值更好，有以下几个原因：

- 语法简单。
- 编译器优化的更好。
- move语法使得拷贝变得cheap。
- 有些时候根本就没有拷贝和移动。

### 为什么不能用inline？

模板函数无需inline。我们可以在头文件中定义非inline模板函数，在多个源文件中include该头文件也不会有问题。

> 对指定类型完全特化的模板函数则比较特别，因为它的返回值不再是可生成的（所有模板参数都被定义过了）。

inline仅仅意味着函数可以多次在同一个应用程序中出现。然而，对编译器来说这意味着调用该函数需要展开函数体。对特定情境inline具有更高的效率，但是很多时候也会让代码变得低效。对当代编译器来说，编译器决定是否应用inline关键字比程序员本身更为可靠，所以请相信编译器！

### 为什么不能用constexpr?

C++11中可以用`constexpr`来提供一种能力——在编译期计算某些值。

```cpp
template<typename T1, typename T2>
constexpr auto max(T1 a, T2 b)
{
    return b < a ? a : b;
}
```

如此，你可以声明这样一个尺寸的原生数组或`std::array<>`：

```cpp
int a[::max(sizeof(char),1000u)];
std::array<std::string, ::max(sizeof(char), 1000u)> arr;
```

## 总结

- 函数模板为不同类型模板参数定义了一个函数族。
- 函数模板按照传递的参数类型，推导出对应的实例化函数，其参数类型与传入类型保持一致。
- 可以显式化限定模板参数。
- 模板参数可以有默认参数。
- 函数模板可以被重载。
- 使用其他函数模板重载函数模板时，需要保证使用时只有一个会匹配。
- 重载函数模板时，改动限制在：显式地指定模板参数。
- 函数模板的所有重载版本都要在使用位置前声明，确保编译器可以看到。