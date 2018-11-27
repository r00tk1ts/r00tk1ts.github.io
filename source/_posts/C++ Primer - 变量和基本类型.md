---
title: C++ Primer - 变量和基本类型
date: 2018-10-28 19:56:11
categories: programming-language
tags:
	- cpp
	- cpp-primer

---

《C++ Primer》第五版引入了11标准相关内容，我早年在初学C++时还只有第四版，近来想对C++做一个整体的复习+版本升级，借此机会过一遍第五版。本文是阅读第二章“变量和基本类型”时所做的笔记。

<!--more-->

# C++ Primer - 变量和基本类型

## 基本内置类型

基本类型包括算术类型和void类型。

### 算术类型

C++标准定义的算术类型如下：

| 类型          | 含义        | 最小尺寸    |
| ----------- | --------- | ------- |
| bool        | 布尔型       | 未定义     |
| char        | 字符        | 8bit    |
| wchar_t     | 宽字符       | 16bit   |
| char16_t    | Unicode字符 | 16bit   |
| char32_t    | Unicode字符 | 32bit   |
| short       | 短整型       | 16bit   |
| int         | 整型        | 16bit   |
| long        | 长整型       | 32bit   |
| long long   | 长整型       | 64bit   |
| float       | 单精度浮点数    | 6位有效数字  |
| double      | 双精度浮点数    | 10位有效数字 |
| long double | 扩展精度浮点数   | 10位有效数字 |

> 注意这只是标准的定义，真实尺寸与精度取决于具体架构、语言实现，可以通过`sizeof`操作符获取具体类型大小。浮点数精度的大多数编译器实现都比标准的底线要高，一般`float`和`double`分别是7和17个有效位。此外可以看到这一标准制定考虑了各种兼容性，相对完善，C++11引入了`long long`。

布尔类型只有两个值，true或false。除了布尔和扩展的字符型（`wchar_t,char16_t,char32_t`）以外，其他整型都有带符号和不带符号两种。缺省是带符号数的，可以显示的用`signed`前缀修饰，如果是无符号，则用`unsigned`修饰。

一个`char`的大小和一个机器字节一样，确保可以存放机器基本字符集中任意字符对应的数字值。`wchar_t`确保可以存放机器最大扩展字符集中的任意一个字符。

浮点型可表示单精度（single-precision）、双精度（double-precision）和扩展精度（extended-precision）值，分别对应`float`、`double`和`long double`类型。

### 类型转换

C++的隐式类型转换继承了C的特性。这种自动转换虽然很多时候很nice，但也容易万劫不复。实际上C++也继承了C的显示类型转换，只是C++自己又定义了一套较为优雅的转换机制。

进行类型转换时，类型所能表示的值的范围决定了转换的过程。

- 把非布尔类型的算术值赋给布尔类型时，初始值为0则结果为false，否则结果为true。
- 把布尔值赋给非布尔类型时，初始值为false则结果为0，初始值为true则结果为1。
- 把浮点数赋给整数类型时，进行近似处理，结果值仅保留浮点数中的整数部分。
- 把整数值赋给浮点类型时，小数部分记为0。如果该整数所占的空间超过了浮点类型的容量，精度可能有损失。
- 赋给无符号类型一个超出它表示范围的值时，结果是初始值对无符号类型表示数值总数（8比特大小的unsigned char能表示的数值总数是256）取模后的余数。
- 赋给带符号类型一个超出它表示范围的值时，结果是未定义的（undefined）。

### 字面值常量

不是一个对新手很友好的概念，写程序的时候虽然承载数据的主要是变量，但很多时候也可以通过类似的手法定义常量(`const`)，甚至直接写字面值常量(例如`42,3.14159L,0.1F,42ULL,'a',L'a',"Hello, world!"`以及各种转义序列(`\n \v \t \r`)还有特殊的`true,false,nullptr`)，一些常量值会有后缀来表明它最小匹配的类型。

浮点型字面值默认是一个double。

字符串字面值的类型是由常量字符构成的数组（array）。编译器在每个字符串的结尾处添加一个空字符`'\0'`，因此字符串字面值的实际长度要比它的内容多一位。 

## 变量

### 变量定义

和C一致，类型说明符+变量名，多个变量名用逗号分隔。定义时可以初始化变量值。C++不太一样的地方在于有了namespace。

**初始化和赋值是两回事。**

> 这是新手特别容易忽略的重点，对于内置类型还好说，一旦涉及了class，那么这一概念就尤为重要。

C++11还引入了多种初始化方法（列表初始化）：

```cpp
int units_sold = 0;
int units_sold = {0};
int units_sold{0};
int units_sold(0);
```

花括号对类型有严格要求，其他情况则正确执行（可能会丢精度）。

> 例如 long double i = 3.14; int j = {i}则会报错，而int j = i; 和 int j(i)则只会丢失精度。

C++的内置类型定义时未指定初值，表现和C是一致的。即全局会被初始化为0，局部的不会初始化，而是栈上的垃圾值。而类类型则有自己的体系。

### 变量声明和定义

和C一样，用于拆分来支持分离式编译机制。一处定义，到处声明。

```cpp
extern int i;
```

如果写成了：

```cpp
extern int i = 3;
```

就变成定义而不再是声明了。

声明这种东西，对只学过动态类型语言如js, py的孩子来说不太友好。

### 标识符

和C一致，字母、数字、下划线组成，不能以数字开头。注意避开关键字。变量命名要有规范，比如驼峰标志命名法。

标识符的长度没有限制，但是对大小写字母敏感。C++为标准库保留了一些名字。用户自定义的标识符不能连续出现两个下划线，也不能以下划线紧连大写字母开头。此外，定义在函数体外的标识符不能以下划线开头。

C++的保留关键字也比C多很多：

| alignas  | alignof   | asm        | auto         | bool             | break       |
| -------- | :-------- | ---------- | ------------ | ---------------- | ----------- |
| case     | catch     | char       | char16_t     | char32_t         | class       |
| const    | constexpr | const_cast | continue     | decltype         | default     |
| delete   | do        | double     | dynamic_cast | else             | enum        |
| explicit | export    | extern     | false        | float            | for         |
| friend   | goto      | if         | inline       | int              | long        |
| mutable  | namespace | new        | noexcept     | nullptr          | operator    |
| private  | protected | public     | register     | reinterpret_cast | return      |
| short    | signed    | sizeof     | static       | static_assert    | static_cast |
| struct   | switch    | template   | this         | thread_local     | throw       |
| true     | try       | typedef    | typeid       | typename         | union       |
| unsigned | using     | virtual    | void         | volatile         | wchar_t     |
| while    |           |            |              |                  |             |

### 作用域

也是老生长谈的东西。C++名字的作用域跟着大括号走，大括号内定义的，出了大括号谁也不认识，但大括号里面都认识，包括嵌套的大括号。

同时，允许在内层作用域中重新定义外层作用域已有的名字，此时内层作用域中新定义的名字将屏蔽外层作用域的名字。

> 但这种做法是非常不推荐的，因为往往是自掘坟墓。

C++多了`::`操作符来显示的指定作用域。

## 复合类型

基于其他类型定义的类型。复合类型千变万化，但有两个非常关键：**引用和指针**。

### 引用

严格上讲应该叫左值引用。引用就是给对象起了个别名。

> 这意味着引用只能绑定在对象上，不能与字面值或某个表达式的计算结果（右值）绑定在一起。

```cpp
int ival = 1024;
int &refVal = ival;	//refVal是ival的别名
int &refVal2;		//报错：引用必须被初始化
```

引用一旦指定，不能再重新绑定到其他对象，因此引用必须初始化。

> 定义引用的初衷是为了摆脱传参时繁琐易错的指针语法，尽管C++的语法是我见过最臃肿的。C++的各种扩展定义可以看出，想要极力摆脱直接使用指针的局面。

### 指针

和C是一样的。指针可以实现其他对象的间接访问。

- 指针本身也是对象，允许对指针赋值和拷贝，而且在生命周期内它可以先后指向不同的对象。
- 指针可以不进行初始化。如果局部变量指针未初始化，则是一个野指针，全局未初始化的则为空指针。

**注意野指针、空指针以及极有灵性的`void *`**。

```cpp
int *ip1, *ip2;     // ip1和ip2都是int型指针
double dp, *dp2;    // dp2是double型指针，dp是double类型变量
```

不能定义指向引用的指针，因为引用本身不是对象，没有实际地址。

指针通过解引用`*`间接访问对象：

```cpp
int ival = 42;
int *p = &ival; // p存放了ival的地址，或者说p是指向ival的指针
cout << *p;     // 由符号*得到p指向的对象，输出42
```

对定义空指针来说，`nullptr`关键字取代了C式的NULL(cstdlib)。

## const限定符

用于定义常量，顾名思义，常量就是不能变化的量，而变量则可能会改变值。

```cpp
const int bufSize = 512;	//int型常量
bufSize = 512;				//抱歉，常量不能被赋值，哪怕是等值也不行
```

常量定义时也必须初始化。

```cpp
const int i = get_size();	//正确：运行时初始化（临时量）
const int j = 42;			//正确：编译时初始化
const int k;				//错误：未初始化
```

`const`对象默认仅在文件内有效。如果想定义跨文件的常量，对于编译时已确定值的常量来说，直接定义在头文件即可，而对于RT时确定的常量，有一种蹩脚的办法：

```cpp
//file_1.cc定义并初始化了一个常量，该常量可以被其他文件访问
extern const int bufSize = fcn();
//file_1.h头文件
extern const int bufSize;	//与file_1.cc中定义的bufSize是一个
```

在头文件声明，源文件中定义，都要加extern关键字。

### const的引用

把引用绑定在const对象上即为对常量的引用（reference to const）。常量的引用不能修改它所绑定的对象。

```cpp
const int ci = 1024;
const int &r1 = ci;
r1 = 42;	//错误，r1是对常量的引用
int &r2 = ci;	//错误，试图让一个非常量引用指向一个常量对象
```

引用类型要和绑定对象类型严格匹配，但有两个例外：

- 初始化常量引用时可以用任意表达式作为初始值，但该表达式的结果必须能转换成引用的类型。
- 允许常量引用绑定到非常量的对象、字面值或一般表达式。（这一规则在传参时非常常见）

### 指针和const

极容易混淆的几个表达式：

```cpp
const double pi = 3.14;
double *ptr = &pi;	//错误，ptr是个普通指针
const double *cptr = &pi;	//正确，cptr可以指向一个双精度常量
*cptr = 42;	//错误，不能给cptr赋值
```

也可以定义常量指针，而不是指向常量的指针。常量指针就是一旦初始化指定了哪个变量就不能再改变，但可以通过间接引用修改变量值。

```cpp
int errNumb = 0;
int *const curErr = &errNumb;	//curErr一直指向errNumb
const double pi = 3.14159;
const double *const pip = &pi;	//pip是指向常量对象的常量指针
```

> 紧紧盯住当`const`和指针名挨在一起时，就是常量指针，一旦`const`和指针名间有*，不管前面的`const`和变量类型怎么折腾，都是指向常量的指针变量。

类似引用，常量对象的地址只能用指向常量的指针存放，但指向常量的指针可以指向一个非常量对象。

类似的，常量指针必须被初始化，因为它不能被二次赋值。

###顶层const

**顶层`const`表示指针常量，底层`const`表示指向常量的指针变量。**

```cpp
int i = 0;
int *const p1 = &i;     // 无法修改p1的值，这是一个顶层const
const int ci = 42;      // 无法修改ci的值，这是一个顶层const
const int *p2 = &ci;    // 可以修改p2的值，这是一个底层const
const int *const p3 = p2; // 靠右的const是顶层const，靠左的是底层const
const int &r = ci;      // 用于声明引用的const都是底层const
```

拷贝操作会受顶层const和底层const影响。顶层无影响而底层则约束拷入对象。简单来说，非常量可以转为常量，而常量不能转为非常量。

### constexpr和常量表达式

常量表达式指值不会变且在编译时就已经可以计算出结果的表达式，比如字面值。常量表达式初始化的`const`对象也是常量表达式。这是个很实用的东西，只是对新手的理解不太友好。

C++11标准规定，可以将变量声明为`constexpr`类型以让编译器来验证变量的值是否是一个常量表达式。

```cpp
constexpr int mf = 20;
constexpr int limit = mf + 1;
constexpr int sz = size();	//只有当size()是一个constexpr函数时，才正确
```

> 如果你认定变量是一个常量表达式，就把它声明为`constexpr`。

指针和引用都能定义成constexpr，但是初始值受到严格限制。constexpr指针的初始值必须是0、nullptr或者是存储在某个固定地址中的对象。

函数体内定义的普通变量一般并非存放在固定地址中，因此constexpr指针不能指向这样的变量。相反，函数体外定义的变量地址固定不变，可以用来初始化constexpr指针。

在constexpr声明中如果定义了一个指针，限定符constexpr仅对指针本身有效，与指针所指的对象无关。constexpr把它所定义的对象置为了顶层const。

```cpp
const int *p = nullptr;	//p是一个指向整型常量的指针变量
constexpr int *q = nullptr;	//q是一个指向整数的常量指针
```

const和constexpr限定的值都是常量。但constexpr对象的值必须在编译期间确定，而const对象的值可以延迟到运行期间确定。

## 处理类型

### 类型别名

某种类型的同义词。

```cpp
typedef double wages;	//wages是double同义词
typedef wages base, *p;	//base是double同义词，p是double*同义词
```

> typedef 的一个坑：如果我们定义typedef char *pstring; 这个时候const pstring指的就是一个指向char的常量指针（指向的地址不变）。而如果把他替换成原来的形式 const char *就变成了指向常量字符的指针，意思就变了。

C++11标准规定了一种新方法，使用**别名声明**来定义类型的别名：

```cpp
using SI = Sales_item;	//SI是Sales_item的同义词
```

> 看起来C的语法又被嫌弃了。

### auto类型说明符

C++11标准引入，模糊类型定义，由编译器来通过初始值推算类型，因此，`auto`定义的变量也必须要初始化：

```cpp
auto item = val1 + val2;
```

使用`auto`定义多个变量时，如果用逗号隔开，那么所有变量类型都是一个。

编译器推断的auto类型，在一些情况下和初始值类型不同：

- 如果引用被用作初始值，则auto类型为引用对象类型
- auto一般忽略顶层const，所以如果想要生成一个顶层const，那就显示指定const auto。

### decltype类型指示符

从表达式的类型推断出要定义的变量的类型，但不用表达式的值初始化变量。C++11引入了`decltype`:

```cpp
decltype(f()) sum = x;	//sum的类型就是函数f的返回类型
```

编译器不会实际调用函数f，而是使用当调用发生时f的返回值类型作为`sum`的类型。

decltype处理顶层const和引用的方式与auto有些不同，如果decltype使用的表达式是一个变量，则decltype返回该变量的类型（包括顶层const和引用）。

```cpp
const int ci = 0, &cj = ci;
decltype(ci) x = 0;     // x类型为const int
decltype(cj) y = x;     // y类型为const int&，y绑定到变量x
decltype(cj) z;     // 错误：z是一个引用，必须初始化
```

如果decltype使用的表达式不是一个变量，则decltype返回表达式结果对应的类型。如果表达式的内容是解引用操作，则decltype将得到引用类型。如果decltype使用的是一个不加括号的变量，则得到的结果就是该变量的类型；如果给变量加上了一层或多层括号，则decltype会得到引用类型，因为变量是一种可以作为赋值语句左值的特殊表达式。

> 因为加了括号，就变成了表达式，即使原本是左值，也变成了右值。

`decltype((var))`的结果永远是引用，而`decltype(var)`的结果只有当var本身是一个引用时才会是引用。

## 自定义数据结构

`Sales_item`就是个例子。库类型也有很多预置的自定义数据结构，如`string, istream, ostream`等。C++中引入class的概念，实际上是对C的`struct`一种OO特性扩充。C++本身也有`struct`，`struct`和`class`除了默认权限不同以及class可作为模板类型关键字（已不推荐）以外，没有什么区别。

关于这些，到具体章节时再展开。

此外还有些关于预编译相关的话题，和C基本一致，不再赘述。