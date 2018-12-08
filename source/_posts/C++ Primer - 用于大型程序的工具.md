---
title: C++ Primer - 用于大型程序的工具
date: 2018-12-08 20:52:11
categories: programming-language
tags:
	- cpp
	- cpp-primer



---

《C++ Primer》第五版引入了11标准相关内容，我早年在初学C++时还只有第四版，近来想对C++做一个整体的复习+版本升级，借此机会过一遍第五版。本文是阅读第十八章“用于大型程序的工具”时所做的笔记。

<!--more-->

# C++ Primer - 用于大型程序的工具

## 异常处理

耳熟能详的异常机制。好在因为自身缺陷，C++的异常没被滥用。

### 抛出异常

通过throw语句来抛出异常，throw之后的语句不会再执行。此时程序控制权转移到throw匹配的catch块中，该catch可能是同一个函数中的局部catch（要看是否对该类型异常进行了捕获），也可能是直接或间接调用了发生异常的函数的另一个函数中（简单来说，就是调用嵌套）。

当throw出现在一个try块内部时，检查与该try块关联的catch子句。如果匹配该类型异常，就在该catch中处理异常。否则，就向上沿调用链匹配catch块，以此类推。这一过程被称为stack unwinding。

如果找不到匹配的catch，则程序将调用标准库函数terminate而终止。

stack unwinding过程中，位于调用链上的语句块可能会提前退出，此时块中的局部对象也会随之销毁，如果局部对象有类类型对象，则对象会被析构。

如果块中有堆内存对象，就要自己在catch中处理，否则就是经典的内存泄露。

如果异常发生在构造函数中，则当前对象可能只构造了一部分。有的成员已经初始化，但另外一些还没有。尽管如此，也要确保已构造的成员能被正确地销毁，这是程序员的工作。

异常也可能发生于数组或标准库容器的元素初始化过程中。此时，如果已经构造了一部分元素，程序员也要保证它们被正确地销毁。

所有标准库类型都能确保它们的析构函数不会引发异常。

异常对象是一种特殊的对象，编译器使用异常抛出表达式来对异常对象进行拷贝初始化。因此，throw语句中的表达式必须拥有完全类型。如果该表达式是类类型的话，则相应的类必须有一个可访问的析构函数和一个可访问的拷贝或移动构造函数。如果该表达式是数组或函数类型，则表达式将被转换成与之对应的指针类型（数组和函数指针特有的退化机制）。

异常对象位于由编译器管理的空间中，编译器确保无论最终调用的是哪一个catch子句，都能访问该空间。当异常处理完毕后，异常对象被销毁。

当异常被抛出时，沿着调用链的块会依次退出直到匹配catch。退出某个块时，局部对象会被释放，所以，不能抛出一个指向局部对象的指针。换句话说，抛出指针的话就要保证指针所指的对象必须存在。

抛出表达式时，表达式的**静态编译时类型**决定了异常对象的类型。这就意味着对继承体系来说，throw表达式解引用一个指向派生类对象的基类指针，会导致抛出的对象被切掉一部分，只抛出基类部分。

### 捕获异常

catch子句的异常声明决定了所能捕获的异常类型。这个类型必须是完全类型，可以是左值引用，但不能是右值引用。

异常声明中的参数如果是非引用类型，则参数就是值拷贝，如果是引用类型，就是对象本身的别名。

此外，如果参数是基类类型，则可以用派生类类型的异常对象对其初始化。此时，如果参数是非引用类型，则异常对象被阉割，如果是基类的引用，则参数会绑定到异常对象上。

异常声明的静态类型将决定catch语句所能执行的操作。如果catch的参数是基类类型，则无法使用派生类的特有成员。

通常，如果catch接受的异常与某个继承体系有关，则最好将该catch参数定义为引用类型。

异常和catch异常声明的匹配规则有诸多限制。大部分类型转换都不被允许，除了下面的一些特例外，C++要求异常类型和catch声明的类型是精确匹配的：

- 允许从非常量到常量的类型转换，即非常量对象的throw可以匹配接受常量引用的catch
- 允许从派生类向基类的类型转换
- 数组和函数可以退化成对应类型的指针

有时，单独的catch捕获到了异常后也不会完整的处理该异常。执行某些必要操作后，可以通过`throw;`语句重新抛出，继续沿调用链传递。

catch的异常声明如果写成`...`，就表示捕获所有异常，而非指定类型的异常。

### try块与构造函数

构造函数的初始值列表初始化可能也出现异常，可以通过try接初始值列表来处理：

```cpp
template<typename T>
Blob<T>::Blob(std::initializer_list<T> il) try:data(std::make_shared<std::vector<T>>(il))
{ }catch(const std::bad_alloc &e) {handle_out_of_memory(e);}
```

该catch既能处理初始化列表的异常，也能处理函数体内的异常。

### noexcept

C++11可以通过提供noexcept说明来指定某个函数不会抛出异常。 

```cpp
void recoup(int) noexcept;	//不抛出异常
void alloc(int);	//可能抛异常
```

noexcept要么出现在该函数所有声明和定义语句中，要么一次也不出现。位于尾置返回类型之前。

函数指针的声明和定义中也可指定noexcept。typedef或类型别名中则不能出现noexcept。成员函数中，noexcept需要跟在const及引用限定符之后，在final、override或虚函数的=0之前。

noexcept并不是说函数一定不会抛异常，它只是用来通知编译器的，开发者应该保证它不抛出异常，含有throw语句的noexcept函数也会编译通过，不会因为这种违反异常说明的情况而报错。

早期的C++也有一套机制，在函数后面使用throw说明符。它的位置和noexcept相同。

```cpp
void recoup(int) throw();	//不抛出异常
```

noexcept运算符常与noexcep说明符混用，noexcept是一元运算符，返回值是bool类型的右值常量表达式，指示给定的表达式是否会抛出异常。noexcept不会对运算对象求值。

```cpp
noexcept(recoup(i));	//如果recoup不抛出异常则结果为true，否则为false
```

当`noexcept(e)`的e调用的所有函数都做了不抛出说明且本身不含有throw语句时，返回true，否则返回false。

```cpp
void f() noexcept(noexcept(g()))	//f和g异常说明一致
```

尽管noexcept说明符不属于函数类型的一部分，但是仍然会影响函数的使用。

- 函数指针及该指针所指的函数的异常说明必须一致。
- 派生的虚函数必须和基类虚函数的异常说明一致。
- 如果对所有成员和基类的所有操作都承诺了noexcept，那么合成的拷贝控制成员也是noexcept的。

### 异常类层次

标准库异常类的继承体系

- exception
  - bad_cast
  - runtime_error
    - overflow_error
    - underflow_error
    - range_error
  - logic_error
    - domain_error
    - invalid_argument
    - out_of_range
    - length_error
  - bad_alloc

exception定义了拷贝构造、拷贝赋值运算符、虚析构和名为what的虚函数。what返回const char*，指向一个null结尾的字符数组，确保不会抛异常。

exception、bad_cast和bad_alloc定义了默认构造函数。类runtime_error和logic_error没有默认构造函数，但有一个可以接受C风格字符串或标准库string类型实参的构造函数，它们负责提供关于错误的更多信息。这些类中，what负责返回用于初始化异常对象的信息。

## 命名空间

应用程序使用多个库时，不可避免的会发生命名冲突。C++引入命名空间来避免名称的污染。每个命名空间都是一个作用域。

采用namespace关键字定义，后面跟名称：

```cpp
namespace cplusplus_primer {
    class Sales_data { /* ... */ };
    Sales_data operator+(const Sales_data&, const Sales_data&);
    class Query { /* ... */ };
    class Query_base { /* ... */ };
}//命名空间结束后无须分号
```

cplusplus_primer是一个命名空间，包含4个成员，三个类和一个重载的+运算符。

命名空间外想要使用，就需要`cplusplus_primer::Query`。

命名空间不一定非要连续定义，可以在多处重复定义相同的namespace，所有的片段拼在一起组成完整的namespace命名空间。

命名空间可以嵌套：

```cpp
namespace cplusplus_primer {
    namespace QueryLib {
        class Query { /* ... */ };
        Query operator&(const Query&, const Query&);
        //...
    }
    namespace Bookstore {
        class Quote { /* ... */ };
        class Disc_quote : public Quote { /* ... */ };
        //...
    }
}
```

使用时相应的嵌套`::`运算符：

```cpp
cplusplus_primer::QueryLib::Query
```

C++11以后，命名空间可以内联，内联命名空间中的名字可以被外层命名空间直接使用。

```cpp
inline namespace FifthEd {
    
}
namespace FifthEd {	//隐式内联
    class Query_base { /* ... */ };
    //其他与Query有关的声明
}
```

inline虽出现了一次，但所有的片段定义都是内联的。

假设内联命名空间FifthEd在FifthEd.h中定义：

```cpp
namespace cplusplus_primer {
#include "FifthEd.h"
}
```

如此，`cplusplus_primer::`可以直接获得FifthEd的成员。

namespace之后如果不跟随名称，就是一个匿名的命名空间。匿名命名空间中定义的变量拥有静态生命周期：它们在第一次使用前参加，并直到程序结束时才销毁。

匿名空间可以在给定的文件内不连续，但不能跨文件。

在C++引入命名空间之前，程序一般用C风格的static来声明对单个文件有效。static的这一语义在C++中已经被namespace取缔了。

可以使用命名空间别名：

```cpp
namespace primer = cplusplus_primer;
```

如此，就可以用`primer::`来替代`cplusplus_primer::`。

using声明可以引入命名空间中的一个成员，这一使用非常广泛。而using指示则一次性把命名空间注入到全局作用域中，它一般出现在最外层作用域中。

```cpp
namespace A{
    int i, j;
}
void f()
{
    using namespace A;//using指示把A全部灌入全局作用域
    cout << i*j << endl;	//使用A中的i和j
    //...
}
```

如果全局注入后出现了标识符冲突，编译器会因为二义性而报错。

尽量避免using指示，使用using声明来一次只导入一个命名空间成员，对于存在二义性的则放弃using声明，直接用原始的namespace::语法来引用。

## 多重继承与虚继承

### 多重继承

派生类可以继承多个基类：

```cpp
class Bear : public ZooAnimal {};
class Panda : public Bear, public Endangered {};
```

C++没有规定可以继承的基类个数，但每个基类只能出现一次。

多重继承的派生类对象包含每个基类的子对象。Panda中有Bear的子对象，还有Endangered的子对象。

构造派生类对象会同时构造并初始化它的所有基类子对象。多重继承的派生类的构造函数初始值也只能初始化它的直接基类：

```cpp
//显式地初始化所有基类
Panda::Panda(std::string name, bool onExhibit) : Bear(name, onExhibit, "Panda"), Endangered(Endangered::critical){ }

//隐式地使用Bear的默认构造函数初始化Bear子对象
Panda::Panda():Endangered(Endangered::critical){ }
```

基类的构造顺序与派生列表中基类的出现顺序保持一致，与派生类构造函数初始值列表中基类的顺序无关。

C++11以后允许派生类从它的一个或几个基类中继承构造函数。如果从多个基类中继承了相同的构造函数，会导致语法错误。

```cpp
struct D2 : public Base1, public Base2 {
    using Base1::Base1;	//从Base1继承构造函数
    using Base2::Base2;	//从Base2继承构造函数
    //D2必须自定义一个接受string的构造函数
    D2(const string &s) : Base1(s), Base2(s) { }
    D2() = default;	//一旦D2定义了它自己的构造函数，则必须出现
}
```

派生类的析构函数只负责清除派生类本身分配的资源，派生类的成员及基类都是自动销毁的。合成的析构函数体为空。

析构的调用顺序与构造相反。

在只有一个基类的情况下，派生类指针或引用可以自动转换为可访问基类的指针或引用。多个基类的情况也类似。可以令某个可访问基类的指针或引用直接指向一个派生类对象。

```cpp
//接受Panda的基类引用的一系列操作
void print(const Bear&);
//void print(const Endangered&);	//这会导致编译出错，因为基类转换没有优劣
void highlight(const Endangered&);
ostream& operator<<(ostream&, const ZooAnimal&);
Panda ying_yang("ying_yang");
print(ying_yang);	
highlight(yin_yang);
cout << yin_yang << endl;
```

只有一个基类时，派生类的作用域嵌套在直接基类和间接基类的作用域中。查找过程沿着继承体系自底向上进行。派生类的名字会隐藏基类的同名成员。而在多重继承的情况下，相同的查找过程在所有直接基类中同时进行。所以，如果基类之间存在重名时，直接使用名字就会出现二义性。此时，要加前缀限定符来使用该名字。

### 虚继承

尽管派生列表中同一个类只能出现一次，但实际上派生类可以多次继承同一个类，即两个直接父类继承了同一个间接祖父类，那么该子类就继承了两次祖父类。

默认情况下，派生类中含有继承链上每个类对应的字部分，所以如果出现了这种情况，就会在子类对象中存在多个祖父类子对象。

有时我们不希望重复继承祖父类，希望两个直接父类能够共享一个祖父类对象，C++语法上可以通过虚继承来实现。虚继承可以令某个类做出声明，承诺愿意共享它的基类。共享的基类子对象被称为虚基类。

指定虚基类的方式是派生列表中添加关键字virtual：

```cpp
//public和virtual的顺序随意
class Raccoon : public virtual ZooAnimal { ... };
class Bear : virtual public ZooAnimal { ... };

class Panda : public Bear, public Raccoon, public Endangered{ ...  };
```

此时，Panda对象中只有一个ZooAnimal子对象，为Raccoon和Bear两个子对象所共享。

不论基类是不是虚基类，派生类对象都可以被可访问基类的指针或引用操作。

由于虚基类中只有唯一一个共享的子对象，所以该基类的成员可以被直接访问。

虚派生中，虚基类是由最低层的派生类初始化的。之所以如此是因为如果采用普通的初始化，就会导致虚基类在多条继承路径上重复初始化。

```cpp
Bear::Bear(std::string name, bool onExhibit) : ZooAnimal(name, onExhibit, "bear"){ }
Raccoon::Raccoon(std::string name, bool onExhibit) : ZooAnimal(name, onExhibit, "Raccoon"){ }

Panda::Panda(std::string name, bool onExhibit) : ZooAnimal(name, onExhibit, "Panda"), Bear(name, onExhibit), Raccoon(name, onExhibit), Endangered(Endangered::critical), sleeping_flag(flase) { }
```

因此，含有虚基类的对象的构造函数顺序是先给最低层派生类构造函数的初始值初始化该对象的虚基类子部分，接下来按派生列表的顺序对直接基类初始化。

对Panda来说就是：

- 先用Panda的构造函数初始值列表中提供的初始值构造虚基类ZooAnimal
- 接下来构造Bear
- 接下来构造Raccoon
- 然后构造第三个直接基类Endangered
- 最后构造Panda

如果Panda没有显式初始化ZooAnimal基类，则ZooAnimal的默认构造函数将被调用。如果ZooAnimal没有默认构造函数，代码就会出错。

> 虚基类永远先于非虚基类构造，与继承体系中的顺序和位置无关。

一个类可以有多个虚基类，此时，这些虚的子对象按照派生列表中出现的顺序从左到右依次构造。

无论是哪种继承体系，对象的销毁顺序都与构造顺序相反。