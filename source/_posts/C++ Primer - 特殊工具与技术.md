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

成员函数可以用作可调用对象，此时需要借助成员函数指针来传递。首先利用`.*`或`->*`将指针绑定到特定的对象上，因此，成员函数指针不同于普通函数指针，它不是一个可调用对象，不支持函数调用运算符。

因为成员指针不是可调用对象，所以不能直接将指向成员函数的指针传递给算法：

```cpp
auto fp = &string::empty;	//fp指向string的empty函数
//错误，必须使用.*或->*调用成员指针
find_if(svec.begin(), svec.end(), fp);
//find_if内部会试图调用fp(*it)，但fp不支持调用运算符
```

因此，我们需要对fp进行适配。有三种方法：

- 使用function生成可调用对象

  ```cpp
  function<bool (const string&)> fcn = &string::empty;
  find_if(svec.begin(),svec.end(),fcn);
  ```

  告知模板function：empty是接受string参数并返回bool值的函数。functino内部会在`fcn(*it)`时展开为`((*it).*p)()`，p为fcn内部的指向成员函数的指针。

  当定义function对象时，必须指定该对象所能表示的函数类型，即可调用对象的形式。如果可调用对象是一个成员函数，则第一个形参必须表示该成员是在哪个对象上执行的。同时，传递给function的形式中还必须指明对象是否是以指针或引用的形式传入的。

  对于上例，我们在string对象的序列上调用find_if，因此我们要求function生成一个接受string对象的可调用对象。又因为我们的vector保存的是string的指针，所以必须指定function接受指针：

  ```cpp
  vector<string*> pvec;
  function<bool (const string*)> fp = &string::empty;
  //fp接受一个指向string的指针，然后使用->*调用empty
  find_if(pvec.begin(), pvec.end(), fp);
  ```

- 使用mem_fn生成可调用对象

  标准库功能mem_fn可以让编译器负责推断成员的类型。它可以从成员指针生成一个可调用对象，无需向function那样显式指定可调用对象的类型：

  ```cpp
  find_if(svec.begin, svec.end(), mem_fn(&string::empty));
  
  auto f = mem_fn(&string::empty);	//f接受一个string或者一个string*
  f(*svec.begin());	//正确：传入一个string对象，f使用.*调用empty
  f(&svec[0]);	//正确，传入一个string指针，f使用->*调用empty
  ```

- 使用bind生成一个可调用对象

  也可以用bind从成员函数生成一个可调用对象：

  ```cpp
  //选择范围中的每个string，并将其bind到empty的第一个隐式实参上
  auto it = find_if(svec.begin(), svec.end(), bind(&string::empty, _1));
  
  auto f = bind(&string::empty, _1);
  f(*svec.begin());	//正确：实参是string，f使用.*调用empty
  f(&svec[0]);	//正确：实参是string指针，f使用->*调用empty
  ```

  类似function，使用bind必须把执行对象的隐式形参转换成显式的。也类似mem_fn，bind生成的可调用对象的第一个实参既可以是string指针，也可以是string的引用。

## 嵌套类

类可以定义在另一个类的内部，称为嵌套类。嵌套类是独立的类，与外层类基本没什么关系。外层类的对象和嵌套类的对象是相互独立的。

嵌套类的名字在外层类作用域中是可见的，在外层类作用域之外不可见。

与普通的类定义没什么差别，只是在必要的时候增加嵌套类所在类的作用域符来修饰。

## union：一种节省空间的类

union在C中就有了，union意在节省空间，他可以有多个数据成员，但任意时刻却只有一个数据成员可以有值（或者说有意义）。给union的某个成员赋值后，其他的就是未定义的状态。分配给union的空间要足够容纳最大成员。

union不能有引用类型的成员（很正常，因为引用天生是“const”的，必须得在初始化时绑定）。C++11以后，含有构造函数和析构函数的类类型也可以作为union的成员类型。union也可以为成员指定public、protected或private权限。默认情况下是public，与struct相同。

union可以定义包括构造函数和析构函数在内的成员函数。但由于union不能继承自其它类，也不能作为基类使用，所以union不能有虚函数。

```cpp
union Token{
    char cval;
    int ival;
    double dval;
};

Token first_token = {'a'};	//初始化cval成员
Token last_token;		//未初始化Token对象
Token *pt = new Token;	//指向未初始化的Token对象指针

last_token.cval = 'z';	//初始化或赋值都会令其他数据成员变成未定义状态
pt->ival = 42；
```

> 标准的未定义就意味着访问的结果是不确定的。

union可以匿名：

```cpp
union {
    char cval;
    int ival;
    double dval;
};
//一旦定义了匿名union，编译器会自动为该union创建一个未命名的对象
cval = 'c';	//为刚刚定义的未命名的匿名union对象赋新值
ival = 42;	//该对象当前保存的值是42
```

匿名union的定义所在的作用域内可以对该union对象的成员进行直接访问。

**匿名union不能包含受保护的成员或私有成员，也不能定义成员函数。**

## 局部类

定义在某个函数内部的类称为局部类。局部类定义的类型只在作用域内可见。

局部类的所有成员必须完整定义在类的内部。比起嵌套类它所受限制非常大。

```cpp
int a, val;
void foo(int val)
{
 	static int si;
    enum Loc {a = 1024, b};
    //Bar是foo的局部类
    struct Bar{
        Loc locVal;	//OK: 使用局部类型名
        int barVal;
        
        void fooBar(Loc l = a)	//正确：默认实参是Lock::a
        {
            barVal = val;	//错误：val是foo的局部变量
            barVal = ::val;	//正确：使用一个全局对象
            barVal = si;	//正确：使用一个静态局部对象
            locVal = b;		//正确：使用一个枚举成员
        }
    };
   // ...
}
```

## 固有的不可移植的特性

为了支持低层编程，C++定义了一些固有的不可移植的特性。所谓不可移植的特性是值因机器而异的特性（比如算术类型的大小在不同机器上不同）。

C++从C继承了两种不可移植特性：位域和volatile限定符。C++本身还有个链接指示。

### 位域

类可以将非静态数据成员定义成位域，在一个位域中含有一定数量的二进制位。当出现需要向其他程序或硬件设备传递二进制数据时，通常会用到位域。

```cpp
typedef unsigned int Bit;
class File{
    Bit mode: 2;	//mode占2位
    Bit modified: 1;	//modified占1位
    Bit prot_owner: 3;	//prot_owner占3位
    Bit prot_group: 3;	//prot_group占3位
    Bit prot_world: 3;	//prot_world占3位
    //File的操作和数据成员
public:
    //文件类型以八进制的形式表示
    enum modes { READ=01, WRITE=02, EXECUTE=03};
    File &open(modes);
    void close();
    void write();
    bool isRead() const;
    void setWrite();
};
```

五个位域可能会存储在同一个unsigned int中，能否压缩到一个unsigned int以及如何压缩与机器相关。

不能对位域应用取地址运算符&，因此任何指针都无法指向类的位域。

位域的使用和正常的类成员使用差不多。

### volatile限定符

volatile的确切含义与机器有关，只能通过阅读编译器文档来理解。

volatile和const用法相似，二者互相没有影响。

合成的拷贝对volatile对象无效。我们不能使用合成的拷贝、移动构造函数和赋值运算符初始化volatile对象或从volatile对象赋值。合成的成员接受的形参类型是（非volatile）常量引用，显然不能把一个非volatile引用绑定到一个volatile对象上。

如果希望拷贝、移动或赋值类的volatile对象，就必须自定义。

```cpp
class Foo{
public:
    Foo(const volatile Foo&);	//从volatile对象进行拷贝
    //将一个volatile对象赋值给一个非volatile对象
    Foo& operator=(volatile const Foo&);
    //将一个volatile对象赋值给一个volatile对象
    Foo& operator=(volatile const Foo&) volatile;
    // Foo类的剩余部分
};
```

### 链接指示: extern "C"

C++有时要调用其他语言编写的函数，比如C的函数。其他语言的函数名字也必须在C++中进行声明，该声明必须指定返回类型和形参列表。C++使用链接指示来指出任意非C++函数所用的语言。

```cpp
//可能出现在C++头文件<cstring>中的链接指示
//单语句链接指示
extern "C" size_t strlen(const char *);
//复合语句链接指示
extern "C"{
    int strcmp(const char*, const char*);
    char *strcat(char*, const char*);
}
```

编译器可能也支持其他语言的链接指示，比如extern "FORTRAN"等。

```cpp
//复合语句链接指示
extern "C"{
    #include <string.h>	//操作C风格字符串的C函数
}
```

这种用法可以把头文件声明的普通函数都声明为链接指示的语言所编写。链接指示可以嵌套，所以如果此头文件中包含自带链接指示的函数，则该函数不受本次影响。

可以应用于函数指针：

```cpp
//pf指向一个c函数，该函数接受一个int返回void
extern "C" void (*pf)(int);

void (*pf1)(int);	//指向一个C++函数
extern "C" void (*pf2)(int);	//指向C函数
pf1 = pf2;	//错误：pf1和pf2型别不同
```

由于C不支持重载函数，所以：

```cpp
extern "C" void print(const char*);
extern "C" void print(int);	//错误，重复定义
```

但如果一组重载函数有一个是C函数，其他都是C++函数：

```cpp
class SmallInt {/*...*/};
class BigNum {/*...*/};
//C函数可以在C或C++程序中调用
//C++函数重载了该函数，可以在C++中调用
extern "C" double calc(double);
extern SmallInt calc(const SmallInt&);
extern BigNum calc(const BigNum&);
```