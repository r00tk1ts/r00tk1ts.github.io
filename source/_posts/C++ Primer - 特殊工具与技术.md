---
title: C++ Primer - 特殊工具与技术
date: 2018-12-10 19:40:11
categories: programming-language
tags:
	- cpp
	- cpp-primer



---

《C++ Primer》第五版引入了11标准相关内容，我早年在初学C++时还只有第四版，近来想对C++做一个整体的复习+版本升级，借此机会过一遍第五版。本文是阅读第十九章“特殊工具与技术”时所做的笔记。

<!--more-->

# C++ Primer - 特殊工具与技术

## 控制内存分配

某些应用程序对内存分配有特殊的需求，需要通过重载operator new和operator delete来实现。

new表达式的背后机理：

- 调用名为operator new（或operator new[]）的标准库函数。该函数分配一块足够大的、原始的、未命名的内存空间存储特定类型的对象。
- 然后，编译器执行构造函数来构造对象。
- 此时对象构造完成，返回指向对象的指针。

delete表达式则相反：

- 对指针所指对象或数组中的元素执行析构函数。
- 编译器调用operator delete（或operator delete[]）来释放内存空间。

应用程序可以自己重载operator new和operator delete函数来自己控制内存分配。重载的两个运算符必须保证正确性。

编译器寻找operator new的顺序是，如果对象是类类型，则先在类及其基类的作用域中查找，如果有则调用，否则，就在全局作用域查找匹配的函数，如果找到了自定义的版本，就用该版本来完成new表达式的第一步，如果没有就用标准库的版本。

标准库定义了operator new和operator delete的8个版本，前四个版本的new可能会抛出bad_alloc异常，后四个版本则不会：

```cpp
void *operator new(size_t);	
void *operator new[](size_t);
void *operator delete(void*) noexcept;
void *operator delete[](void*) noexcept;

void *operator new(size_t, nothrow_t&) noexcept;
void *operator new[](size_t, nothrow_t&) noexcept;
void *operator delete(void*, nothrow_t&) noexcept;
void *operator delete[](void*, nothrow_t&) noexcept;
```

nothrow_t定义在new头文件，是一个struct，该类型不包含任何成员。new头文件还定义了一个名为nothrow的const对象，可以通过这个对象来请求new的非抛出版本。

operator new和operator new[]的返回类型必须是void*，第一个形参必须是size_t且该形参不能有默认实参。

可以自定义上面的版本中任何一个，但自定义的版本必须位于全局作用域或类作用域中。也可以提供额外的形参来自定义operator new，带有size_t以外形参的都要用placement new表达式来调用。

`void *operator new(size_t, void*);`不允许重新定义。

operator delete和operator delete[]的返回类型必须是void，第一个形参类型必须是void*。执行delete表达式将调用相应的operator函数。

operator delete或operator delete[]被定义成类的成员时，该函数可以有另一个类型为size_t的形参。此时，该形参的初始值是第一个形参所指对象的字节数。size_t形参可用于删除继承体系中的对象。如果基类有一个虚析构函数，则传递给operator delete的字节数将因待删除指针所指对象的动态类型不同而有所区别。而且，实际运行的operator delete函数版本也由对象的动态类型决定。

**我们所重载的是operator new和operator delete，而不是new表达式和delete表达式，它们不允许被重定义。**

可以用placement new形式来构造对象，通过为new传递一个地址：

```cpp
new (place_address) type
new (place_address) type (initializers)
new (place_address) type [size]
new (place_address) type [size] { braced initializer list }
```

place_address必须是一个指针，同时initializers中提供一个初始值列表，用于构造新分配的对象。

当仅通过一个地址值来调用时，placement new使用的就是operator new(size_t, void*)来分配内存，该函数不分配任何内存，而是简单地返回指针实参；然后由new表达式负责在指定的地址处初始化对象。实际上，placement new允许我们在一个特定的预先分配的内存地址上构造对象。

## RTTI

运行时类型识别（run-time type identification）由两个运算符实现：

- typeid运算符，用于返回表达式的类型。
- dynamic_cast运算符，用于将基类的指针或引用安全地转换成派生类的指针或引用。

适用的场景：想用基类指针或引用来执行派生类的非虚函数成员。

dynamic_cast语句的转换目标如果是指针类型，失败的情况下返回0；而如果转换目标是引用，则抛出bad_cast异常。

```cpp
if(Derived *dp = dynamic_cast<Derived*>(bp))
{
    //试用dp指向的Derived对象
}else{
    //试用bp指向的Base对象
}

void f(const Base &b)
{
    try{
        const Derived &d = dynamic_cast<const Derived&>(b);
    }catch(bad_cast){
        //处理类型转换失败的情况
    }
}
```

typeid运算符用来返回一个常量对象的引用，该对象的类型时标准库类型type_info或者type_info的公有派生类型。定义在typeinfo头文件中。

typeid可以作用域任意类型表达式。

顶层const会被忽略，引用会被忽略。

特别的是，当typeid作用于数组或函数时，并不会执行向指针的标准类型转换。

当运算对象不属于类类型或者是不包含虚函数的类时，返回运算对象的静态类型。而当运算对象是定义了至少一个虚函数的类的左值时，typeid的结果要在运行时才会计算。

typeid作用于指针时（而非指针所指的对象），返回的结果是该指针的静态编译时类型。

运行时求值的情况下，如果作用于*p但p为空指针，typeid就会抛出bad_typeid的异常。

| type_info的操作 |                                                              |
| --------------- | ------------------------------------------------------------ |
| t1 == t2        | 如果type_info对象t1和t2表示同种类型，返回true；否则返回false |
| t1 != t2        | 与上相反                                                     |
| t.name()        | 返回一个C风格字符串，表示类型名字的可打印形式。类型名字的生成方式因系统而异 |
| t1.before(t2)   | 返回一个bool值，表示t1是否位于t2之前。before所采用的顺序关系是依赖于编译器的 |

type_info没有默认构造，也没有拷贝移动控制成员。有公有的虚析构函数成员。因此，只能通过typeid运算符生成type_info对象。

## 枚举类型

枚举可以让我们把一组整型常量组织在一起。每个枚举类型定义了一种新的类型。枚举属于字面值常量类型。

C++包含两种枚举：限定作用域和不限定作用域的。C++11引入了限定作用域的枚举类型（scoped enumeration）。

```cpp
//限定枚举
enum class open_modes { input, output, append };

//非限定枚举
enum color {red, yellow, green};	
enum {floatPrec = 6, doublePrec = 10, double_doublePrec = 10};//匿名枚举类型
```

限定作用域的枚举类型中，枚举成员的名字遵循常规的作用域准则，在枚举类型的作用域外不可访问。而非限定枚举类型中，枚举成员的作用域与枚举类型本身的作用域相同：

```cpp
enum color {red, yellow, green};	//非限定
enum stoplight {red, yellow, green};	//错误：重复定义了枚举成员
enum class peppers {red, yellow, green};//正确：枚举成员被隐藏了
color eyes = green;	//正确，非限定枚举成员位于有效的作用域
peppers p = green;	//错误：peppers的枚举成员不在有效的作用域中
					//color::green在有效的作用域中，但是类型错误
color hair = color::red;	//正确：允许显式访问枚举成员
peppers p2 = peppers::red;	//正确，使用peppers的red
```

默认情况，枚举成员从0开始，依次加1。也可以为多个枚举成员指定专门的值，值可以重复。

枚举成员是const，因此初始化枚举成员时提供的初始值必须是常量表达式。

如此，可以定义枚举类型的constexpr变量，可以将enum作为switch的case标签，也可以作为非类型模板形参，或者在类的定义中初始化枚举类型的静态数据成员。

尽管enum都定义了惟一的类型，但实际上enum是由某种整数类型表示的。C++11允许我们在enum的名字后面用冒号接这一整数类型：

```cpp
enum intValues : unsigned long long{
    charTyp = 255, shortTyp = 65535, intTyp = 65535,
    longTyp = 4294967295UL,
    long_longTyp = 18446744073709551615ULL
};
```

限定枚举的默认情况是int类型。非限定枚举没有默认类型，只知道其潜在类型足以容纳枚举值。因此，在声明枚举时，对非限定枚举要指定成员类型，否则不知道它的大小：

```cpp
enum intValues : unsigned long long;	//非限定枚举必须指定成员类型
enum class open_modes;	//限定枚举类型可以使用默认的成员类型int
```

初始化enum对象必须用enum的另一个对象或它的枚举成员，而不能用某个整型值，哪怕它的值和枚举成员值相等，整型值不能作为函数的enum实参使用。

反之，我们可以将非限定枚举类型的对象或枚举成员传给整型实参。此时，enum被提升成int或更大的整型。

## 类成员指针

成员指针是指向类的非静态成员的指针。成员指针的类型囊括了类的类型以及成员的类型。初始化一个这样的指针时，我们令其指向类的某个成员，但是不指定该成员所属的对象；直到使用成员指针时，才提供成员所属的对象。

定义类Screen:

```cpp
class Screen{
public:
    typedef std::string::size_type pos;
    char get_cursor() const { return contents[cursor];}
    char get() const;
    char get(pos ht, pos wd) const;
private:
    std::string contents;
    pos cursor;
    pos height, width;
};
```

### 数据成员指针

```cpp
//pdata可以指向一个常量（非常量）Screen对象的string成员
const string Screen::*pdata;
```

pdata被声明为一个指向Screen类的const string成员的指针。

初始化成员指针或赋值时，要指定它所指的成员：

```cpp
pdata = &Screen::contents;
```

pdata指向某个非特定Screen对象的contents成员。

C++11中可以用auto或decltype简化：

```cpp
auto pdata = &Screen::contents;
```

初始化指针或赋值后，该指针仍没有指向有效的数据。只有解引用成员指针时菜提供对象的信息：

```cpp
Screen myScreen, *pScreen = &myScreen;
//.*解引用pdata以获得myScreen对象的contents成员
auto s = myScreen.*pdata;
//->*解引用pdata获得pScreen所指对象的contents成员
s = pScreen->*pdata;
```

`.*`用来解引用对象，`->*`用来解引用指针，获得对象成员。

### 成员函数指针

类似的定义方法：

```cpp
//pmf是一个指针，它可以指向Screen的某个常量成员函数
//前提是该函数不接受任何实参，并返回一个char
auto pmf = &Screen::get_cursor;

char (Screen::*pmf2)(Screen::pos, Screen::pos) const;
pmf2 = &Screen::get;
```

成员函数和指向该成员的指针之间不存在自动转换规则：

```cpp
pmf2 = &Screen::get;	//必须显式地使用取地址运算符
pmf2 = Screen::get;	//错误：成员函数和指针之间不存在自动转换规则
```

使用成员函数指针就和数据成员是一样的：

```cpp
Screen myScreen, *pScreen = &myScreen;
//通过pScreen所指对象调用pmf所指函数
char c1 = (pScreen->*pmf)();
//通过myScreen对象将实参0，0传给含有两个形参的get函数
char c2 = (myScreen.*pmf2)(0,0);
```

因为优先级的问题，注意括号的使用。

可以用类型别名来让成员指针更容易理解：

```cpp
using Action = char (Screen::*)(Screen::pos, Screen::pos) const;

Action get = &Screen::get;//get指向Screen的get成员
//action接受一个Screen的引用，和一个指向Screen成员函数的指针
Screen& action(Screen&, Action = &Screen::get);

Screen myScreen;
action(myScreen);				//使用默认实参
action(myScreen, get);			//使用我们之前定义的变量get
action(myScreen, &Screen::get);	//显式地传入地址
```

## 嵌套类





