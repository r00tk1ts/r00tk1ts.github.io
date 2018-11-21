---
title: C++ Primer - 类
date: 2018-11-21 19:12:11
categories: programming-language
tags:
	- cpp
	- cpp-primer


---

《C++ Primer》第五版引入了11标准相关内容，我早年在初学C++时还只有第四版，近来想对C++做一个整体的复习+版本升级，借此机会过一遍第五版。本文是阅读第七章“类”时所做的笔记。

<!--more-->

# 

# C++ Primer - 类

类的基本思想是数据抽象（data abstraction）和封装（encapsulation）。数据抽象是一种依赖于接口（interface）和实现（implementation）分离的编程及设计技术。类的接口包括用户所能执行的操作；类的实现包括类的数据成员、负责接口实现的函数体以及其他私有函数。

## 定义抽象数据类型

使用class或是struct关键字可以定义类类型，struct是为了兼容旧式C风格结构体，对于定义类类型来说，struct和class的唯一区别在于默认访问权限不同。

成员函数（member function）的声明必须在类的内部，定义则既可以在类的内部也可以在类的外部。定义在类内部的函数是隐式的inline函数。

```cpp
struct Sales_data{
    // 新成员：关于Sales_data对象的操作
  	std::string isbn() const {return bookNo;}
  	Sales_data& combine(const Sales_data&);
  	double avg_price() const;
  	// 数据成员
  	std::string bookNo;
  	unsigned units_Sold = 0;
  	double revenue = 0.0;
};
// Sales_data的非成员接口函数
Sales_data add(const Sales_data&, const Sales_data&);
std::ostream &print(std::ostream&, const Sales_data&);
std::istream &read(std::istream&, Sales_data&);
```

成员函数通过一个名为this的隐式参数来访问调用它的对象。this是一个常量指针，无法改变this中保存的地址。

### const成员函数

isbn函数的参数列表后跟随了一个const关键字，这里的const是用来修改隐式this指针的类型。

默认情况下，this类型是指向类类型非常量版本的常量指针。比如Sales_data类成员函数中，this的类型是`Sales_data *const`。尽管this是隐式的，但它仍然需要遵守初始化规则，这就意味着我们不能把this绑定到一个常量对象上。因此，受这一限制，我们无法在一个常量对象上调用普通的成员函数。

因为this是隐式的，我们没有办法像修饰其他参数一样，去声明其为指向常量的指针，所以C++的语法只好提供实现途径——把const关键字放在成员函数的参数列表之后，此时，this就是一个指向常量的指针，这种const成员函数被叫做常量成员函数。

```cpp
//伪代码，说明隐式this指针是如何使用的
//下面的代码非法：我们不能显式地定义自己的this指针
//此处的this是一个指向常量的指针，因为isbn是一个常量成员
std::string Sales_data::isbn(const Sales_data *const this)
{return this->isbn;}
```

因为this指向常量，所以常量成员函数不能改变调用它的对象的内容。只读不可写。

成员函数可以在外部定义。

```cpp
double Sales_data::avg_price() const{
    if(units_sole)
      	return revenue/units_sold;
  	else
      	return 0;
}
```

注意需要加作用域运算符，否则谁知道你定义的是谁的成员。

为了实现连续调用链，可以令成员函数返回this对象本身：

```cpp
Sales_data& Sales_data::combine(const Sales_data &rhs)
{
    units_sold += rhs.units_sold;
  	revenue += rhs.revenue;
  	return *this;	// 返回调用该函数的对象
}
```

类的作者往往还需要定义一些辅助函数，这些函数不作为类的成员函数，但也作为类的接口的一部分。

```cpp
istream &read(istream &is, Sales_data &item)
{
    double price = 0;
  	is >> item.bookNo >> item.units_sold >> price;
  	item.revenue = price * item.units_sold;
  	return is;
}
ostream &print(ostream &os, const Sales_data &item)
{
    os << item.isbn() << " " << item.units_sold << " "
      	<< item.revenue << " " < item.avg_price();
  	return os;
}
```

### 构造函数

每个类都定义了对象被初始化的方式，类通过一个或几个特殊的成员函数来控制其对象的初始化过程，这些函数叫构造函数。构造函数的任务是初始化类对象的数据成员，无论何时只要类的对象被创建，就会执行构造函数。

构造函数的名字和类名一致，没有返回类型，构造函数可以重载，不能被声明成const（因为创建一个const对象时，会先调用构造函数，再得到const属性）。

#### 合成的默认构造函数

如果类不定义任何构造函数，编译器会提供一个没有任何实参的默认构造函数。编译器创建的构造函数也叫合成的默认构造函数。

合成的默认构造函数完成以下任务：

- 如果存在类内的初始值，用它来初始化成员。(units_sold和revenue)
- 否则，默认初始化该成员。(bookNo初始化成空串)

一旦定义了一个构造函数，那么编译器不再合成默认构造函数，即使我们定义的构造函数并不是没有参数的默认构造函数。

为Sales_data定义合适的构造函数：

```cpp
struct Sales_data{
    Sales_data() = default;
  	Sales_data(const std::string &s):bookNo(s){}
  	Sales_data(const std::string &s, unsigned n, double p):bookNo(s), units_sold(n), revenue(p*n){}
  	Sales_data(std::istream &);
  
  	std::string isbn()const{return bookNo;}
  	Sales_data& combine(const Sales_data&);
  	double avg_price() const;
  	std::string bookNo;
  	unsigned units_sold = 0;
  	double revenue = 0.0;
};
```

=default是C++引入的， 用于显式要求编译器合成默认构造函数（因为定义了其他构造函数，编译器不会自动合成默认构造函数，但我们又想要编译器提供的默认构造函数，所以这是一种偷懒的语法糖）。

=default可以出现在类的内部，也可以出现在外部，内部意味着inline。

紧跟在构造函数参数列表之后在花括号之前的部分是初始值列表。它负责为新创建的对象的一个或几个数据成员赋初值。被忽略的成员则将以合成默认构造函数相同的方式隐式初始化。

> C++的编译器并不都支持类内初始值。为了可移植性，最好用初始值列表。

### 拷贝、赋值和析构

除了构造函数以外，类还有3个特殊的成员函数：拷贝构造、赋值操作和析构。

拷贝构造会在拷贝初始化变量和值传递方式传递或返回一个对象时被调用。

赋值操作则在对类对象使用赋值运算符时会被调用。

而析构函数则在对象被销毁时被调用。

与构造函数类似，如果不去定义这3个成员，编译器也会默认合成。关于这一议题，后面第13章会单独讲解。届时可以掌握什么时候可以用合成的版本，什么时候不行，不行的话又该如何正确的定义。

## 访问控制与封装

C++用访问说明符加强了类的封装性：

- 定义在public说明符之后的成员在整个程序内都可以被访问，public成员定义类的接口。
- 定义在private说明符之后的成员仅可以被类的内部成员函数访问，外部代码无法访问，private封装了类的实现细节。

```cpp
class Sales_data{
public:
  	Sales_data() = default;
  	Sales_data(const std::string &s, unsigned n, double p):bookNo(s), units_sold(n), revenue(p*n){}
  	Sales_data(const std::string &s):bookNo(s){}
  	Sales_data(std::istream&);
  	std::string isbn() const {return bookNo;}
  	Sales_data &combine(const Sales_data&);
private:
  	doubkle avg_price() const
    {return units_sold ? revenue/units_sold : 0;}
  	std::string bookNo;
  	unsigned units_sold = 0;
  	double revenue = 0.0;
};
```

struct的默认访问权限是public，class的默认访问权限是private。

### 友元

加上了权限之后，一些外部接口函数就无法访问类的private成员，这种情况要么提供public接口，要么就使用友元。

```cpp
class Sales_data{
	friend Sales_data add(const Sales_data&, const Sales_data&);
	friend std::istream &read(std::istream&, Sales_data&);
	friend std::ostream &print(std::ostream&, const Sales_data&);
  	...
}
```

friend关键字用于表示这三个函数是类Sales_data的友元函数，如此这三个函数可以访问类的private成员。friend相当于白名单，除了友元函数以外，还可以定义友元类。

```cpp
class Screen{
  	// Window_mgr的成员可以访问Screen类的private成员
	friend class Window_mgr;
  	...
}
```

每个类负责控制自己的友元类和友元函数。

有时候整个类作为友元比较冒险，可以只对类的几个成员函数声明为友元：

```cpp
class Screen{
  	// Window_mgr::clear必须在Screen类之前被声明
    friend void Window_mgr::clear(ScreenIndex);
  	...
}
```

## 类的其他特性

在外部定义的成员函数也可以通过inline关键字来显式内联。

成员函数也是可以重载的。

使用关键字`mutable`可以声明可变数据成员（mutable data member）。可变数据成员永远不会是const的，即使它在const对象内。因此const成员函数可以修改可变成员的值。

```cpp
class Screen 
{
public:
    void some_member() const;
private:
    mutable size_t access_ctr;  // may change even in a const object
    // other members as before
};

void Screen::some_member() const
{
    ++access_ctr;   // keep a count of the calls to any member function
    // whatever other work this member needs to do
}
```

> 这个mutable我是没见过。。。

类内初始值除了=初始化形式以外，还可以用花括号形式（C++11）：

```cpp
class Window_mgr{
private:
  	//默认情况下，一个Window_mgr包含一个标准尺寸的空白Screen
  	std::vector<Screen> screens{Screen(24, 80, ' ')};
}
```

const成员函数以引用形式返回*this，则它的返回类型将是常量引用。

成员函数可以基于const重载：

```cpp
class Screen{
public:
  	Screen &display(std::ostream &os){do_display(os);return *this;}
  	const Screen &display(std::ostream &os) const {do_display(os);return *this;}
private:
  	void do_display(std::ostream &os) const {os << contents;}
};
```

因为非常量版本的函数对于常量对象是不可用的，所以只能在常量对象上调用const成员函数。尽管明面上参数列表相同，但实际上隐式的this指针类型是不同的，区别在于是否有底层const。

## 构造函数再探

初始值列表提供了成员初始化的机会，如果在构造函数体内对成员进行赋值，那执行的就是赋值操作了，对于类类型来说，初始化和赋值操作可能行为不一致。

如果成员是const、引用，或者是某种未定义默认构造函数的类类型，必须在初始值列表中将其初始化。

```cpp
class ConstRef
{
public:
    ConstRef(int ii);
private:
    int i;
    const int ci;
    int &ri;
};

ConstRef::ConstRef(int ii): i(ii), ci(ii), ri(i) { }
```

最好令构造函数初始值的顺序与成员声明的顺序一致，并且尽量避免使用某些成员初始化其他成员。

#### 委托构造函数

C++11引入了委托构造函数：

```cpp
class Sales_data
{
public:
   	//非委托构造函数使用对应实参初始化成员
  	Sales_data(std:string s, unsigned cnt, double price):bookNo(s), units_sold(cnt), revenue(cnt*price){}
  	//其余构造函数全都委托给另一个构造函数
  	Sales_data():Sales_data("", 0, 0){}
  	Sales_data(std::string s):Sales_data(s, 0, 0){}
  	Sales_data(std::istream &is):Sales_data(){read(is, *this);}
  	...
};
```

默认初始化的发生情况：

- 在块作用域内不使用初始值定义非静态变量或数组。
- 类本身含有类类型的成员且使用合成默认构造函数。
- 类类型的成员没有在构造函数初始值列表中显式初始化。

值初始化的发生情况：

- 数组初始化时提供的初始值数量少于数组大小。
- 不使用初始值定义局部静态变量。
- 通过`T()`形式（*T*为类型）的表达式显式地请求值初始化。

类必须包含一个默认构造函数以便在上述情况下使用。

### 隐式的类类型转换

如果构造函数只接受一个实参，则它实际上定义了转换为此类类型的隐式转换机制。这种构造函数被称为转换构造函数（converting constructor）。

```cpp
string null_book = "9-999-99999-9";
// 构造一个临时的Sales_data对象
// 该对象的units_sold和revenue等于0，bookNo等于null_book
item.combine(null_book);
```

类类型转换只允许一步，这意味着：

```cpp
//错误，需要两次转换
item.combine("9-999-99999-9");
//正确，显式转换为string，隐式转为Sales_data
item.combine(string("9-999-99999-9"));
```

如果构造函数声明为explicit，就可以抑制隐式的类类型转换。

## 类的静态成员

使用关键字`static`可以声明类的静态成员。静态成员存在于任何对象之外，对象中不包含与静态成员相关的数据。

```cpp
class Account
{
public:
    void calculate() { amount += amount * interestRate; }
    static double rate() { return interestRate; }
    static void rate(double);
    
private:
    std::string owner;
    double amount;
    static double interestRate;
    static double initRate();
};
```

> 静态成员实际上是全局变量，只不过通过语法的封装，让他和所属类建立了耦合的关系。

由于静态成员不与任何对象绑定，因此静态成员函数不能声明为const，也不能在静态成员函数内使用this指针。

用户代码可以使用作用域运算符访问静态成员，也可以通过类对象、引用或指针访问。类的成员函数可以直接访问静态成员。

```cpp
class Account{
public:
  	void calculate(){amount += amount * interestRate;}
  	static double rate(){return interestRate;}
  	static void rate(double);
private:
  	std::string owner;
  	double amount;
  	static double interestRate;
  	static double initRate();
};

double r;
r = Account::rate();

Account ac1;
Account *ac2 = &ac1;
r = ac1.rate();
r = ac2->rate();
```

成员函数无需作用域运算符即可使用静态成员。

在类外部定义静态成员时，不能重复static关键字，其只能用于类内部的声明语句。

由于静态数据成员不属于类的任何一个对象，因此它们并不是在创建类对象时被定义的。通常情况下，不应该在类内部初始化静态成员。而必须在类外部定义并初始化每个静态成员。一个静态成员只能被定义一次。一旦它被定义，就会一直存在于程序的整个生命周期中。