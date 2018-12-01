---
title: C++ Primer - 面向对象程序设计
date: 2018-12-01 19:46:11
categories: programming-language
tags:
	- cpp
	- cpp-primer


---

《C++ Primer》第五版引入了11标准相关内容，我早年在初学C++时还只有第四版，近来想对C++做一个整体的复习+版本升级，借此机会过一遍第五版。本文是阅读第十五章“面向对象程序设计”时所做的笔记。

<!--more-->

# C++ Primer - 面向对象程序设计

## OOP概述

OOP三大核心思想：抽象、继承和多态（动态绑定）。

- 数据抽象将类的接口与实现分离
- 继承可以定义相似的类型并对其相似关系建模
- 多态则在一定程度上忽略相似类型的区别，实现统一方式来使用一组对象

对继承来说，层次关系的根部类叫基类，其他类可以直接或间接从基类继承而来，它们叫派生类。基类负责定义在层次关系中所有类共同拥有的成员，派生类定义各自特有的成员。

而一旦有了继承，也就可以应用多态。想要解释继承和多态，再多的语言也是苍白无力讳莫如深。不如从一个例子说起：

```cpp
class Quote{
public:
  	std::string isbn() const;
  	virtual double net_price(std::size_t n) const;//定义为虚函数，实现多态
};
class Bulk_quote : public Quote{	// Bulk_quote继承了Quote
public:
  	double net_price(std::size_t) const override;
};
```

Quote为基类，Bulk_quote为Quote的派生类（子类）。派生类通过类派生列表(class derivation list)明确指出从哪个(哪些)基类继承而来。其中关键字public表示公有继承，先不解释其作用。

基类的函数net_price前面加上virtual表示其是一个虚函数，虚函数的作用是为了实现多态。一旦基类定义了虚函数，那么派生的子类就可以通过重新定义来覆盖基类的虚函数实现。派生类可以省略virtual关键字，尽管子类中不声明virtual，覆盖函数定义仍然还是虚函数。

C++11标准允许派生类显式地注明它将使用哪个成员函数改写基类的虚函数，这就是上例中override关键字的作用。

virtual实现的动态绑定怎么用呢？

```cpp
double print_total(ostream &os, const Quote &item, size_t n)
{
  	//如果item是Quote对象，调用Quote::net_price
  	//如果item是Bulk_quote对象，调用Bulk_quote::net_price
    double ret = item.net_price(n);
  	os << "ISBN: " << item.isbn()	//调用Quote::isbn
      	<< " # sold: " << n << " total due: " << ret << endl;
  	return ret;
}

print_total(cout, basic, 20);	//basic是Quote对象
print_total(cout, bulk, 20);	//bulk是Bulk_quote对象
```

可以看到尽管形参是一个const Quote对象的引用，但可以传入一个派生类对象作为实参，而一旦如此，对形参调用类成员函数时，就会进行动态绑定，即派生类调用派生类重新定义的虚函数体，基类调用基类定义的虚函数体。

> 如果net_price不是虚函数，即使子类中重新定义了一模一样的net_price，对该例来说，无论传给item的对象是子类还是派生类，最终调用的都是父类的net_price。所以说虚函数才支持动态绑定。深入一点说，拥有虚函数的类对象都有虚表，忽略基类型别动态绑定到子类成员函数的过程实际上是依赖于类对象的虚表指针，因为无论对象被看成基类还是子类，它的虚表指针始终是指向正确的解绑函数的。

所以，多态的存在可以让我们在程序设计上使用父类指针（或引用）指向子类对象，也就是所谓的“一定程度上忽略相似类型的区别，实现统一方式来使用一组对象”。

## 定义基类和派生类

```cpp
class Quote{
public:
  	Quote() = default;	//合成默认构造函数
  	Quote(const std::string &book, double sales_price):bookNo(book), price(sales_price){}
  	std::string isbn() const {return bookNo;}
  	//返回给定数量的书籍的销售总额
  	//派生类负责改写并使用不同的折扣计算算法
  	virtual double net_price(std::size_t n) const
    {return n * price;}
  	virtual ~Quote() = default;	//对析构函数进行动态绑定
private:
  	std::string bookNo;	//书籍的ISBN编号
protected:
  	double price = 0.0;	//代表普通状态下不打折的价格
};
```

为什么析构函数要定义成虚函数呢？因为我们往往使用多态时，会使用父类指针指向子类对象，而后续可能会delete父类指针，如果析构函数不是虚函数，那么delete一个子类对象不会调用子类对象的析构，而是直接调用父类的析构了，这与预期不符。

所以，拥有虚函数的父类的析构函数往往也是虚函数。

这里的一个疑点：类成员protected权限是什么？

### 成员函数与继承

派生类可以继承基类的成员，基类的成员函数有两种：希望派生类进行覆盖而被声明为virtual的虚函数、希望派生类直接使用的函数。

任何构造函数之外的非静态函数都可以是虚函数。

> 构造函数为什么不能virtual呢？很简单，因为构造子类理应递归的调用父类的构造器，如果父类构造器被virtual化了，那么子类就无法调用到父类的构造器了。

普通的成员函数的解析过程发生在编译阶段，虚函数的解析过程发生在运行时(从虚表取函数地址)。

### 访问控制与继承

派生类可以继承基类的成员，但这并不意味着派生类内部可以随意使用基类的成员。在public继承条件下，如果父类的成员是private权限，那么派生类内部无法访问，如果父类的成员是public权限，那么派生类内部可以访问。

那么，有的时候我们希望父类的一些成员也可以被子类访问，但不希望被其他外部非亲非戚的访问，private和public就都不好用了，于是，就有了protected权限。protected修饰的成员意味着派生类可以访问，外部不行。

继续定义派生类：

```cpp
class Bulk_quote : public Quote{
public:
  	Bulk_quote() = default;
  	Bulk_quote(const std::string &, double, std::size_t, double);
  	//覆盖基类的虚函数，隐式virtual
  	double net_price(std::size_t) const override;
private:
  	std::size_t min_qty = 0;	//自己的成员，折扣政策下最低购买量
  	double discount = 0.0;		//折扣额
}
```

派生类对象包含多个组成部分：含有派生类自己定义的(非静态)成员的子对象，以及一个与该类继承的基类对应的子对象，如果有多个基类，那么也就有对应多个子对象。

Bulk_quote对象

| bookNo |	从Quote继承

| price |

| min_qty |	Bulk_quote自定义的成员

| discount |

之所以能完成继承，本质上是因为派生类对象中拥有基类对象。

C++标准没有规定派生类对象的内存如何分布，有兴趣可以看看《深度探索C++对象模型》，当然由于这本书比较老了，所以现在主流编译器的设计和书中内容有较大差异，但仍然极有价值，毕竟授人以渔。

因为派生类对象中含有与其基类对应的组成部分，所以可以把派生类对象当成基类对象使用，也能将基类指针或引用绑定到派生类对象的基类部分上。

```cpp
Quote item;			//基类对象
Bulk_quote bulk;	//派生类对象
Quote *p = &item;	//p指向Quote对象
p = &bulk;			//p指向bulk的Quote部分
Quote &r = bulk;	//r绑定到bulk的Quote部分
```

这种称为派生类到基类的类型转换，编译器会隐式执行派生类到基类的转换。

### 派生类构造函数

```cpp
Bulk_quote(const std::string &book, double p, std::size_t qty, double disc) : Quote(book, p), min_qty(qty), discount(disc){}
```

初始化列表中调用了Quote的构造函数，用来负责初始化基类部分。

除非特别指出，否则派生类对象的基类部分会像数据成员一样执行默认初始化。如果想使用其他的基类构造函数，我们需要以类名加圆括号内的实参列表的形式为构造函数提供初始值。

派生类的构造器总是先初始化基类部分，再按声明顺序依次初始化派生类成员。

### 派生类使用基类的成员

```cpp
double Bulk_quote::net_price(size_t cnt) const
{
    if(cnt >= min_qty)
      	return cnt * (1 - discount) * price;
  	else
      	return cnt * price;
}
```

### 继承与静态成员

如果基类定义了静态成员，则整个继承体系中只存在该成员的唯一定义。不论基类中派生出多少个派生类，对每个静态成员来说都只存在唯一的一个实例。

> 因为静态成员实际上是全局的，当然是单例。只是语法上为了关系结构，把它放在类中定义。

另外，静态成员也遵循访问控制权限。

派生类的声明不能包含派生列表，直接`class Bulk_quote;`就行了。

C++11可以定义一种不允许其他类继承的类。

```cpp
class NoDerived final{/* */};	//NoDerived不能做基类
```

嗯，C++居然反向抄袭了java。

### 类型转换与继承

通常当把引用或指针绑定到一个对象时，引用或指针的类型得和对象的类型一致，或者对象类型含有一个可接受的const类型转换规则。但对于继承类来说还有一个特例，那就是可以把基类的指针或引用绑定到派生类对象上，为了实现多态。

> 智能指针也支持这一类型转换，所以可以将派生类对象指针存在基类智能指针之内。

- 从派生类向基类的类型转换只对指针或引用类型有效。
- 基类向派生类不存在隐式类型转换。
- 派生类向基类的类型转换也可能会由于访问受限而变得不可行。

## 虚函数

对虚函数的调用在运行时被解析。派生类覆盖虚函数定义需要保证型别完全一致。

> 有一个例外就是虚函数在基类中如果返回基类指针或引用时，派生类中是可以返回派生类的指针或引用的，这是唯一的一个可行的型别不一致的地方，但这种不一致也有前提条件，即派生类到基类的类型转换是可访问的（不能访问受限）。

```cpp
struct B{
    virtual void f1(int) const;
  	virtual void f();
  	void f3();
};

//这个例子可以看出override的用处，可以直观的找出错误，没有override编译器会曲解原本的意图
struct D1 : B{
    void f1(int) const override;	//正确，f1与基类型别一致
  	void f2(int) override;	//错误，B没有该函数
  	void f3() override;		//错误，f3不是虚函数
  	void f4() override;		//错误，B没有f4
};

struct D2 : B{
  	//继承B的f2(),f3(),覆盖f1(int)
    void f1(int) const final;	//不允许后续的其他类覆盖f1(int)
};

struct D3 : D2{
  	void f2();				//正确：覆盖从间接基类B继承而来的f2()
  	void f1(int) const;		//错误，D2已经声明f1为final了
};
```

虚函数也可以声明final来阻止派生类覆盖。

### 回避虚函数的机制

有时候不希望动态绑定，而是调用某个虚函数的特定版本，可以通过作用域运算符来实现：

```cpp
double undiscounted = baseP->Quote::net_price(42);
```

无论baseP实际上是啥类型，最后调用的都是Quote的net_price，这是编译时期确定的。

通常只有成员函数(或友元)的代码才需要这种hack技巧。比如派生类虚函数想要调用父类的虚函数版本。

##  抽象基类

有时候父类指向声明一个函数接口，不想实际定义，希望由派生类来定义。C++允许这种设计，可以在virtual的基础上定义纯虚函数：

```cpp
class Disc_quote : public Quote{
public:
  	Disc_quote() = default;
  	Disc_quote(const std::string &book, double price, std:size_t qty, double disc):Quote(book, price), quantity(qty), discount(disc) {}
  	double net_price(std::size_t) const = 0;//=0表示纯虚函数
protected:
  	std::size_t quantity = 0;
  	double discount = 0.0;
}
```

纯虚函数无需定义，=0只能出现在类内部的虚函数声明语句处。

纯虚函数也可以定义，但必须在类外部定义，大部分情况不会定义，因为这与我们的使用意图相悖。

类只要含有纯虚函数，就是一个抽象基类，抽象基类负责定义接口，后续的其他类来覆盖接口。不能创建抽象基类的对象，抽象基类需要派生类去继承。

```cpp
class Bulk_quote : public Disc_quote{
public:
  	Bulk_quote() = default;
  	Bulk_quote(const std::string &book, double price, std::size_t qty, double disc):Disc_quote(book, price, qty, disc){}
  	double net_price(std::size_t) const override;
};
```

直接基类是Disc_quote，间接基类是Quote。各个类控制自己的构造器，构造器会递归下去，先执行根基类构造器，最后执行自身的构造。继承链的构造器形成了层的概念。

## 访问控制与继承

protected的一个坑：

```cpp
class Base{
protected:
  	int prot_mem;
};
class Sneaky : public Base{
    friend void clobber(Sneaky&);	//可以访问Sneaky::prot_mem
  	friend void clobber(Base&);	//不能访问Base::prot_mem
  	int j;	//j默认是private
};

void clobber(Sneaky &s){s.j = s.prot_mem = 0;}//clobber可以访问Sneaky的private和protected成员
void clobber(Base &b){b.prot_mem = 0;}//clobber不能访问Base的protected成员
```

之所以有这种限制，是因为如果第二个用法合法的话，那么就可以通过定义一个形如Sneaky的新类来规避掉protected提供的访问保护了。

所以，派生类的成员和友元只能访问派生类对象中的基类部分的受保护成员，而不能访问普通的基类对象中的成员。

### public、private和protected

```cpp
class Base{
public:
  	void pub_mem()
protected:
  	int prot_mem;
private:
  	char priv_mem;
};
struct Pub_Derv : public Base{
    int f(){return prot_mem;}//正确：派生类能访问protected成员
  	char g(){return priv_mem;}//错误：private成员对于派生类来说是不可访问的
};
struct Priv_Derv : private Base{
    //private不影响派生类的访问权限
  	int f1() const{return prot_mem;}
};
struct Prot_Derv : protected Base{
    int f1() const{return prot_mem;}	//依然是protected
}
Pub_Derv d1;	//继承自Base的成员遵循原有的访问说明符
Priv_Derv d2;	//继承自Base的成员无论此前是什么权限，都变成private
d1.pub_mem();	//正确：pub_mem在派生类中是public的
d2.pub_mem();	//错误：pub_mem在派生类中是private的

Prot_Derv d3;	//继承自Base的成员如果是public，会变成protected，其他不变
d3.pub_mem();	//错误，pub_mem是protected，只能成员和友元访问，外部不行
```

### 派生类向基类转换的可访问性

- 只有当D公有继承B时，用户代码才能使用派生类向基类的转换，如果D继承B的方式是受保护或私有继承，则用户代码不能使用该转换。
- 无论D以什么方式继承B，D的成员函数和友元都能使用派生类向基类的转换，派生类向直接基类的类型转换对于派生类的成员和友元来说永远是可访问的。
- 如果D继承B的方式是公有的或者受保护的，则D的派生类的成员和友元可以使用D向B的类型转换；反之，如果D继承B的方式是私有的，则不能使用。

说白了就一个规则：

**对代码中某个给定节点，如果基类的公有成员是可访问的，则派生类向基类的转换就是可访问的，反之则不行。**

**友元关系不能继承。**

class默认继承权限是private，struct是public。这一点和类成员权限很相似。

## 继承中的类作用域

派生类的作用域位于基类作用域之内，因此才可以实现派生类访问基类成员。

如果派生类重用了基类的成员名字，那么基类的对应成员就会被隐藏。此时想要访问隐藏的成员，就要通过域运算符，这一手法类似调用特定虚函数版本。

```cpp
class Base{
public:
  	virtual int fcn();
};
class D1 : public Base{
public:
  	//隐藏基类的fcn，这个fcn不是虚函数
  	//D1继承了Base::fcn()的定义
  	int fcn(int);	//形参列表与Base中的fcn不一致
  	virtual void f2();	//新的虚函数，在Base中不存在
};
class D2 : public D1{
public:
  	int fcn(int);	//非虚函数，隐藏了D1::fcn(int)
  	int fcn();		//覆盖了Base的虚函数fcn
  	void f2();		//覆盖了D1的虚函数f2
};

Base bobj;
D1 d1obj;
D2 d2obj;

Base *bp1 = &bobj, *bp2 = &d1obj, *bp3 = &d2obj;
bp1->fcn();	//虚调用，运行时调用Base::fcn
bp2->fcn();	//虚调用，运行时调用Base::fcn
bp3->fcn();	//虚调用，运行时调用D2::fcn

D1 *d1p = &d1obj; D2 *d2p = &d2obj;
bp2->f2();	//错误，Base没有f2成员
d1p->f2();	//虚调用，运行时调用D1::f2()
d2p->f2();	//虚调用，运行时调用D2::f2()

Base *p1 = &d2obj; D1 *p2 = &d2obj; D2 *p3 = &d2obj;
p1->fcn(42);	//错误：Base中没有接受一个int的fcn
p2->fcn(42);	//静态绑定，调用D1::fcn(int)
p3->fcn(42);	//静态绑定，调用D2::fcn(int)
```

## 构造函数与拷贝控制

### 虚析构函数

这一点已经说过了，如果基类的析构函数不是虚函数，则delete一个指向派生类对象的基类指针将产生未定义行为。

```cpp
Quote *itemP = new Quote;
delete itemP;	//调用Quote的析构
itemP = new Bulk_quote;	//静态类型与动态类型不一致
delete itemP;	//如果Quote析构是虚函数，则调用Bulk_quote，否则调用Quote析构
```

三五准则中曾言，如果类需要析构函数，往往也需要拷贝构造和赋值操作，但对虚析构函数来说不遵守该准侧。

**虚析构函数将阻止合成移动操作。**

### 合成拷贝控制与继承

构造器的调用链：

- 合成的Bulk_quote默认构造函数运行Disc_quote的默认构造函数，后者又运行Quote的默认构造函数。
- Quote的默认构造函数将bookNo成员默认初始化为空串，同时使用类内初始化值将price初始化为0。
- Quote的构造函数完成后，继续执行Disc_quote的构造函数，它使用类内初始化qty和discount。
- Disc_quote的构造函数完成后，继续执行Bulk_quote的构造函数，但什么具体工作也没做。

类似的，合成的Bulk_quote的拷贝构造也一样，调用Disc_quote的拷贝构造，后者又调用了Quote的拷贝构造。

- 如果基类的默认构造、拷贝构造、拷贝赋值运算符或析构函数是被删除的函数或不可访问，则派生类中对应的成员也将是删除的。
- 如果基类中有一个不可访问或删除掉的析构函数，则派生类中合成的默认和拷贝构造函数是删除的，因为编译器无法销毁派生类对象的基类部分。
- 编译器不会合成一个删除掉的移动操作。使用=default请求一个移动操作时，如果基类中对应的操作是删除的或不可访问的，那么派生类中该函数将是被删除的，因为派生类对象的基类部分不可移动。同理，如果基类的析构函数是删除的或不可访问的，则派生类的移动构造函数也将是被删除的。

如果确实需要移动操作，那就应该在基类中自己去定义，否则会因为虚析构的存在而默认被delete。

```cpp
class Base{};
class D : public Base{
public:
  	D(const D& d) : Base(d)
    D(D&& d) : Base(std::move(d))
  	D &D::operator=(const D &rhs){
        Base::operator=(rhs);
      	return *this;
    }
};
```

### 继承的构造函数

C++11可以让派生类重用基类定义的构造函数。

```cpp
class Bulk_quote : public Disc_quote{
public:
  	using Disc_quote::Disc_quote;	//继承了Disc_quote的构造函数
  	double net_price(std::size_t) const;
};
```

编译器会生成形如:`derived(params) : base(args){}`的构造函数。这里的using作用给编译器，而不是当前作用域。

using声明语句不能指定constexpr或explicit，所以它继承基类的修饰。

基类构造函数含有默认实参时，默认实参不会被继承。相反，派生类或获得多个继承的构造函数，每个构造函数分别省略掉一个含有默认实参的形参。

如果基类有好几个构造函数，则大多数情况下派生类继承所有构造函数。除了两个例外，其一是派生类可以继承一部分构造函数，而为其他构造函数定义自己的版本。如果派生类定义的构造函数与基类构造函数具有相同的参数列表，则这些构造函数不会被继承。其二是默认、拷贝和移动构造不会被继承。它们按照正常规则来合成，游离于三界之外。

## 容器与继承

```cpp
vector<Quote> basket;
basket.push_back(Quote("0-201-82470-1", 50));
//正确，但是只能把对象的Quote部分拷贝给basket
basket.push_back(Bulk_quote("0-201-54858-8", 50, 10, .25));
//调用Quote定义的版本
cout << basket.back().net_price(15) << endl;
```

因为存放的是对象，所以类型转换上会阉割。

想要多态必须要间接访问，对容器来说也一样，我们要存放指针而不是对象。

```cpp
vector<shared_ptr<Quote>> basket;
basket.push_back(make_shared<Quote>("0-201-82470-1", 50));
basket.push_back(make_shared<Bulk_quote>("0-201-54848-8", 50, 10, .25));
cout << basket.back()->net_price(15) << endl;
```