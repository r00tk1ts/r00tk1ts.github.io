---
title: C++ Primer - 函数
date: 2018-11-20 21:10:11
categories: programming-language
tags:
	- cpp
	- cpp-primer


---

《C++ Primer》第五版引入了11标准相关内容，我早年在初学C++时还只有第四版，近来想对C++做一个整体的复习+版本升级，借此机会过一遍第五版。本文是阅读第六章“函数”时所做的笔记。

<!--more-->

# C++ Primer - 函数

## 函数基础

一个典型的函数定义包括返回类型（return type）、函数名字、由0个或多个形式参数（parameter，简称形参）组成的列表和函数体（function body）。

这些都是继承自C的概念。

但在C++中，函数调用本身被视为调用运算符，用以执行函数。

```cpp
int fact(int val)
{
    int ret = 1;    
    while (val > 1)
    ret *= val--;   
    return ret;     
}

int main()
{  
 	int j = fact(5);
  	cout << "5! is " << j << endl;
  	return 0;
}
```

函数调用完成两项工作：

- 用实参初始化对应的形参。
- 将控制权从主调函数转移给被调函数。此时，主调函数（calling function）的执行被暂时中断，被调函数（called function）开始执行。

`return`语句结束函数的执行过程，完成两项工作：

- 返回return语句中的值（可能没有值）。
- 将控制权从被调函数转移回主调函数，函数的返回值用于初始化调用表达式的结果。

> void类型返回值的函数可能没有显式的return。void类型不返回任何值。

### 实参和形参

初学者懵逼的一大议题，掌握了C，对此一定已是了如指掌。

实参是形参的初始值，实参的类型必须与对应的形参类型匹配，函数调用规定实参数量应该与形参一致，所以形参一定会被初始化。

### 局部对象

形参和函数体内定义的变量统称为局部变量。

局部变量本身没什么好说的，栈空间上的自生自灭。但静态局部对象大相径庭，可以把它理解成一种特殊的全局变量，它只在函数内部可用，且有状态（即上一次的值变更会驻留到下一次调用）。

> static这个关键字在C中是个很好的考点，不同的地方使用static有着不同的意义。

### 声明

和变量本质上是一样的，一次定义，到处声明。函数声明也叫做函数原型（function prototype）。

函数应该在头文件中声明，在源文件中定义。定义函数的源文件应该包含含有函数声明的头文件。

## 传参

形参初始化的机理与变量初始化一样。

形参的类型决定了形参和实参交互的方式：

- 当形参是引用类型时，它对应的实参被引用传递（passed by reference），函数被传引用调用（called by reference）。引用形参是它对应实参的别名。
- 当形参不是引用类型时，形参和实参是两个相互独立的对象，实参的值会被拷贝给形参（值传递，passed by value），函数被传值调用（called by value）。

> C++中的传引用解决了C“庸人自扰”的困境，C中常言传值还是传址，实际上传址即通过指针传递本身还是值传递，只不过拷贝给形参的值是一个地址值。而C++的传引用则是它本身语法引入的一种新机制。

### 传值

如果形参不是引用类型，则函数对形参做的所有操作都不会影响实参。

受旧式C风格影响，使用指针类型的形参可以访问或修改函数外部的对象。

C++推荐使用引用形参代替指针形参。

### 传引用

通过引用形参，函数可以改变实参的值。

引用形参可以避免拷贝操作，尤其是对类类型对象来说这意味着拷贝构造的消耗，且不谈某些类根本不支持拷贝构造。涉及到类类型的议题，在后续会相继详谈。

**除了内置类型、函数对象和标准库迭代器外，其他类型的参数建议以引用方式传递。**

如果函数无须改变引用形参的值，最好将其声明为常量引用。

### const形参和实参

当形参有顶层const时，传递给它常量对象或非常量对象都是可以的。

可以使用非常量对象初始化一个底层const形参，反之则不行（因为不可变更）。

把函数不会改变的形参定义成普通引用会极大地限制函数所能接受的实参类型，同时也会给别人一种误导，即函数可以修改实参的值。

### 数组形参

又是个游离于三界之外的小东西。继承了C风格。形参虽然可以写成数组的形式，但实际上传递的是指向数组首元素的指针。

```cpp
// 尽管形式不同，但这3个print等价
// 每个函数都有一个const int *类型的形参
void print(const int*);
void print(const int[]);    
void print(const int[10]);  
```

以数组作为形参的函数必须确保使用数组时不会越界。

如果函数不需要对数组元素执行写操作，应该把数组形参定义成指向常量的指针。

### 数组引用形参

数组也可以传引用：

```cpp
void print(int (&arr)[10])
{
    for(auto elem : arr)
      cout << elem << endl;
}
```

> (&arr)的括号必须要有，因为优先级的关系，必然变成了XX引用类型的数组，和指针数组、数组指针一个道理。

另外，形式上传递多维数组时，要指定低维度的具体尺寸。

###含有可变形参的函数

C++11新标准提供了两种主要方法处理实参数量不定的函数。

- 如果实参类型相同，可以使用`initializer_list`标准库类型。

  ```cpp
  oid error_msg(initializer_list<string> il)
  {
      for (auto beg = il.begin(); beg != il.end(); ++beg)
      cout << *beg << " " ;
      cout << endl;
  }
  ```

- 如果实参类型不同，可以定义可变参数模板（这个以后再说）。

> C++还可以使用省略符形参传递可变数量的实参，但这种功能一般只用在与C函数交换的接口程序中。这些代码使用了名为varargs的C标准库功能。

> 省略符形参仅仅用于C和C++通用的类型，大多数类类型的对象传递给省略符形参时无法正确拷贝。

## 返回类型和return语句

分为无返回值和有返回值，与C没有什么区别。

函数返回一个值的方式和初始化一个变量或形参的方式完全一样：返回的值用于初始化调用点的一个临时量，该临时量就是函数调用的结果。如果函数返回引用类型，则该引用仅仅是它所引用对象的一个别名。

**调用一个返回引用的函数会得到左值，其他返回类型得到右值。**

C++11规定，函数可以返回用花括号包围的值的列表。同其他返回类型一样，列表也用于初始化表示函数调用结果的临时量。如果列表为空，临时量执行值初始化；否则返回的值由函数的返回类型决定。

- 如果函数返回内置类型，则列表内最多包含一个值，且该值所占空间不应该大于目标类型的空间。

- 如果函数返回类类型，由类本身定义初始值如何使用。

  ```cpp
  vector<string> process()
  {
      // . . .
      // expected and actual are strings
      if (expected.empty())
          return {};  // return an empty vector
      else if (expected == actual)
          return {"functionX", "okay"};  // return list-initialized vector
      else
          return {"functionX", expected, actual};
  }
  ```

### 返回数组指针

因为数组不能被拷贝，所以函数不能返回数组，但可以返回数组的指针或引用。

#### 尾置返回类型

C++11允许使用尾置返回类型（trailing return type）简化复杂函数声明。尾置返回类型跟在形参列表后面，并以一个`->`符号开头。为了表示函数真正的返回类型在形参列表之后，需要在本应出现返回类型的地方添加auto关键字。

```cpp
// func接受一个int类型的实参，返回一个指针，该指针指向含有10个整数的数组
auto func(int i) -> int(*)[10];
```

> 尾置返回类型在模板元编程中非常常见。

#### 使用decltype

如果我们知道函数返回的指针将指向哪个数组，就可以使用decltype关键字声明返回类型。但decltype并不会把数组类型转换成指针类型，所以还要在函数声明中添加一个`*`符号。

```cpp
int odd[] = {1,3,5,7,9};
int even[] = {0,2,4,6,8};
// 返回一个指针，该指针指向含有5个整数的数组
decltype(odd) *arrPtr(int i)
{
    return (i % 2) ? &odd : &even;  // 返回一个指向数组的指针
}
```

## 函数重载

C++允许函数名字相同但形参列表不同，这就是函数重载。

```cpp
void print(const char *cp);
void print(const int *beg, const int *end);
void print(const int ia[], size_t size);
```

一般来说定义重载函数都是基于参数不同但每个函数执行的操作大抵相似。编译器会根据实参类型来推断出使用的重载函数。

> main不能重载。

返回类型不同但形参列表完全相同会导致语法错误，这不是有效的重载，毕竟编译器推断是基于参数列表。

### 重载和const形参

顶层const不影响传入函数的对象。拥有顶层const的形参无法和另一个没有顶层const的形参区分开来：

```cpp
Record lookup(Phone);
Record lookup(const Phone);	//重复声明了Record lookup(Phone)

Record lookup(Phone*);
Record lookup(Phone* const);//重复声明了Record lookup(Phone*)
```

而如果形参是某种类型的指针或引用，则通过区分其指向的是常量对象还是非常量对象可以实现函数重载，此时const是底层的：

```cpp
//对于接受引用或指针的函数来说，对象是常量还是非常量对应的形参不同
//定义了4个独立的重载函数
Record lookup(Account &);		//作用于Account的引用
Record lookup(const Account &);	//作用于常量引用

Record lookup(Account *);		//作用于指向Account的指针
Record lookup(const Account*);	//作用于指向常量的指针
```

编译器可以通过实参是否是常量来推断应该调用哪个函数。因为const不能转换成其他类型，所以我们只能把const对象(或指向const的指针)传递给const形参。相反的，因为非常量可以转换成const，所以上面的4个函数都能作用于非常量对象或者指向非常量对象的指针。

当传递一个非常量对象或者非常量对象的指针时，编译器会优选非常量版本的函数。

### const_cast

const_cast常常应用于重载函数。

假设有：

```cpp
const string &shorterString(const string &s1, const string &s2)
{
    return s1.size() <= s2.size() ? s1 : s2;
}
```

参数和返回类型都是const string引用，但这个函数可以传入两个非常量的string实参，但是尽管如此，结果依然是const string引用，如果我们想要一版返回string引用的函数，那么就可以通过const_cast来简单的完成重载：

```cpp
string &shorterString(string &s1, string &s2)
{
    auto &r = shorterString(const_cast<const string&>(s1),
                           const_cast<const string&>(s2));
    return const_cast<string&>(r);
}
```

如你所见，尽管const_cast很危险，但这种使用情境下我们是知晓这种转换不会带来副作用的。

## 特殊用途语言特性

默认实参、内联函数和constexpr函数。

### 默认实参

C中很不方便的就是没有默认实参，懒人科技。

默认实参作为形参的初始值出现在形参列表中。可以为一到多个形参定义默认值，但一旦某个形参被赋予了默认值，它后面的所有形参都得有默认值。所以一般把那些具有默认值的参数放到最后（当然要如此，否则传参时鬼知道哪个参数被省略了）。

函数传参时如果不传递实参，则使用默认实参初始化。

### 内联函数和constexpr函数

内联函数可以避免函数调用的开销，所谓内联，就是在调用点“内联地”展开。类似C中宏的概念。

在函数的返回类型前加上inline关键字，就可以声明为内联函数。

但实际上内联inline标志只是向编译器发起一个请求，编译器会根据实际情况选择忽略还是采用。

> 实际上，现代编译器都会自己处理哪些要内联，早就不是我比编译器聪明系列了。

constexpr函数是指能用于常量表达式的函数。定义constexpr函数的方法和其他函数类似，但要遵守几项约定：函数的返回类型及所有形参的类型都得是字面值类型，且函数体中必须有且只有一条return语句:

```cpp
constexpr int new_sz(){return 42;}
constexpr int foo = new_sz();	//正确，foo一个常量表达式
```

复杂一点：

```cpp
constexpr size_t scale(size_t cnt){return new_sz() * cnt;}

int arr[scale(2)];	//正确，scale(2)是常量表达式
int i = 2;			//i不是常量表达式
int a2[scale(i)];	//错误，scale(i)不是常量表达式
```

所以说，constexpr函数不一定返回常量表达式。

内联函数和constexpr函数通常放在头文件内，不然就得保证每次的定义都一致，这无异于自找麻烦。

### 调试

C++引入了assert预处理宏。它实际上是一个预处理变量，类似于内联函数。`assert(expr);`会首先对expr求值，如果表达式为false，则assert输出信息，程序终止执行流，如果是true则什么都不做。

assert宏位于cassert头文件。预处理名字由预处理器而非编译器管理，所以可以直接使用预处理名字而无需提供using声明。

assert的行为依赖于一个叫NDEBUG的预处理变量的状态。如果定义了NDEBUG，那么assert什么也不做，默认状态是未定义的，所以直接用assert即可。

几个预处理器定义的辅助调试的名字：

- `__FILE__`: 存放文件名的字符串字面值
- `__LINE__`: 存放当前行号的整型字面值
- `__TIME__`: 存放文件编译时间的字符串字面值
- `__DATE__`: 存放文件编译日期的字符串字面值

## 函数匹配

大部分情况直接根据实参来完全匹配形参就可以确定该调用哪个重载函数。但很多时候，几个重载函数形参数量相等且某些形参的类型可以由其他类型转换得来时，就比较麻烦了。

```cpp
void f();
void f(int);
void f(int, int);
void f(double, double = 3.14);
f(5.6);	//调用void f(double, double)
```

### 确定候选函数和可行函数

匹配的第一步是挑出候选函数集。特征：与调用函数同名、其声明在调用点可见。

上面的例子中有4个候选函数。

接下来分析实参，挑选能被这组实参调用的函数，这一子集叫可行函数。

可行函数特征：形参数量与实参相等，每个实参类型与对应的形参类型也相同或是可以转换成形参的类型。

上例中，可行函数就只有`void f(int);`和`void f(double, double = 3.14);`。

最后，寻找可行函数的最佳匹配，基本思想就是实参类型与形参类型越接近就越好。

### 实参类型转换

为了确定最佳匹配，编译器把实参到形参的转换分成几个等级，排序如下：

1. 精准匹配，包括：
   1. 实参类型和形参类型相同。
   2. 实参从数组类型或函数类型转换成对应的指针类型。
   3. 向实参添加顶层const或者从实参中删除顶层const。
2. 通过const转换实现的匹配。
3. 通过类型提升实现的匹配。
4. 通过算术类型转换或指针转换实现的匹配。
5. 通过类类型转换实现的匹配。

注意算数类型转换如果引起二义性，那么也会报错。当然，这都是程序员给自己挖的坑，正常的程序不会有这种情况。

### 函数匹配和const实参

```cpp
Record lookup(Account &);
Record lookup(const Account &);
const Account a;
Account b;

lookup(a);	//匹配Record lookup(const Account &);
lookup(b);	//匹配Record lookup(Account &);
```

## 函数指针

函数指针指向函数而非对象。函数指针指向某种特定类型。函数的类型由返回类型和形参类型共同决定，与函数名无关。

```cpp
bool (*pf)(const string &, const string &);
```

定义了一个指向`bool(const string&, const string&)`类型的函数指针。

> 同样的,(*pf)两侧的小括号非常重要。

当把函数名作为值使用时，函数名自动转换成指针。

```cpp
pf = lengthCompare;		//pf指向名为lengthCompare的函数
pf = &lengthCompare;	//等价的赋值语句，取地址符是可选的

bool b1 = pf("hello", "goodbye");	//调用lengthCompare函数
bool b2 = (*pf)("hello", "goodbye");	//等价调用
bool b3 = lengthCompare("hello", "goodbye");	//等价调用
```

不同函数类型的指针不可以互相转换，但是可以让函数指针指向nullptr或0值，表示空指针。

也可以返回函数类型的指针。

```cpp
// 类型别名的方式
using F = int(int *, int);	//F是函数类型，不是指针
using PF = int(*)(int *, int);	//PF是指针类型

PF f1(int);	//正确：PF是指向函数的指针，f1返回指向函数的指针
F f1(int);	//错误：F是函数类型，f1不能返回一个函数
F*f1(int);	//正确：显式地指定返回类型是指向函数的指针

// 原生定义的方式
// 这种也是可以的，由内向外解读，f1是个函数，返回的是个指针
// 而这个指针比较特别，形式上也有参数列表和返回值，所以f1返回的是个函数指针
int (*f1(int))(int *, int);

// 尾置返回类型的方式
auto f1(int) -> int(*)(int *, int);
```

如果明确知道返回的函数是哪一个，就可以用decltype简化书写函数指针返回类型的过程。

```cpp
string::size_type sumLength(const string&, const string&);
string::size_type largerLength(const string&, const string&);
//根据形参的取值，getFcn函数返回指向sumLength或largerLength的指针
decltype(sumLength) * getFcn(const string &);
//这里一定要加一个*号，因为decltype本身返回的是函数类型，而非指针类型
```