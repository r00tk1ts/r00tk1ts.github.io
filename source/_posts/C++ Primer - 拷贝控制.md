---
title: C++ Primer - 拷贝控制
date: 2018-11-29 22:25:11
categories: programming-language
tags:
	- cpp
	- cpp-primer


---

《C++ Primer》第五版引入了11标准相关内容，我早年在初学C++时还只有第四版，近来想对C++做一个整体的复习+版本升级，借此机会过一遍第五版。本文是阅读第十三章“拷贝控制”时所做的笔记。

<!--more-->

# C++ Primer - 拷贝控制

定义一个类，会显式或隐式指定此类型的对象拷贝、移动、赋值和销毁时做什么。类通过定义五种特殊的成员函数来控制这些操作，包括：拷贝构造函数（copy constructor）、拷贝赋值运算符（copy-assignment operator）、移动构造函数（move constructor）、移动赋值运算符（move-assignment operator）和析构函数（destructor）。

其中拷贝构造和移动构造器定义了当用同类型的另一个对象初始化本对象时的行为。而拷贝赋值和移动赋值定义了将一个对象赋予同类型的另一个对象时的行为。析构函数定义了此类型对象销毁时做什么。这些统称为拷贝控制操作。

对于没有显式定义这些成员的，编译器会自动定义默认的版本。一些类必须要自己定义拷贝控制成员，另外一些则不需要。所以，何时需要自己去定义就考察程序员的功底了。

## 拷贝、赋值与销毁

移动语义是C++11新引入的，过后再谈。

### 拷贝构造函数

仅有一个参数为自身类类型引用的构造函数就是拷贝构造函数，形如：

```cpp
class Foo{
public:
    Foo();				//默认构造函数
    Foo(const Foo&);	//拷贝构造函数
}
```

该参数必须是引用类型，一般是const引用。由于拷贝构造函数会在几种情况下隐式地调用，所以一般不是explicit。

如果自己不定义，编译器就会合成一个默认的。合成的拷贝构造函数会把参数成员逐个拷贝到正在创建的对象中（非static成员）。

成员的类型决定了拷贝的方式：类类型的成员会用它自己的拷贝构造函数来拷贝；内置类型则直接值拷贝。数组会逐个复制，如果数组成员是类类型，会逐个调用成员本身的拷贝构造函数。

```cpp
class Sales_data{
public:
    Sales_data(const Sales_data&);
private:
    std::string bookNo;
    int units_sold = 0;
    double revenue = 0.0;
};

//与Sales_data的合成拷贝构造函数等价
Sales_data::Sales_data(const Sales_data &orig) : 
	bookNo(orig.bookNo), //使用string的拷贝构造函数
	units_sold(orig.units_sold),	//拷贝orig.units_sold
	revenue(orig.revenue)	//拷贝orig.revenue
    {}	//空函数体
```

#### 拷贝初始化

拷贝初始化和直接初始化的差异：

```cpp
string dots(10,',');	//直接初始化
string s(dots);			//直接初始化
string s2 = dots;		//拷贝初始化
string null_book = "9-999-99999-9";	//拷贝初始化
string nines = string(100, '9');	//拷贝初始化
```

拷贝初始化一般由拷贝构造函数完成，之所以说一般是因为移动语义的引入，导致如果类由移动构造函数时，拷贝初始化有时会使用移动构造函数而非拷贝构造函数。

拷贝初始化不仅在用=定义变量时发生，在下列情形也会发生：

- 将一个对象作为实参传递给一个非引用类型的形参
- 从一个返回类型为非引用类型的函数返回一个对象
- 用花括号列表初始化一个数组中的元素或一个聚合类中的成员

某些类类型还会对它们所分配的对象使用拷贝初始化。如初始化标准库容器或调用其insert或push成员时，容器会对其元素进行拷贝初始化。而emplace创建的元素都是直接初始化。

#### 参数和返回值

拷贝构造函数被用来初始化非引用类类型参数，所以拷贝构造函数自身的参数必须是引用类型。不然的话，就二者矛盾而无限循环了。

> 为了调用拷贝构造函数，我们必须拷贝它的实参，但为了拷贝实参，我们又需要调用拷贝构造函数。

### 拷贝赋值运算符

```cpp
Sales_data trans, accum;
trans = accum;	//使用Sales_data的拷贝赋值运算符
```

如果类未定义，编译器会合成一个。

这个函数的定义涉及了重载运算符的概念，这里重载的是赋值运算符。

重载运算符本质上是函数，名字由operator关键字接要定义的运算符符号组成。所以，赋值运算符就对应operator=的函数。

重载运算符的参数表示运算符的运算对象，某些运算符包括赋值必须定义为成员函数。如果一个运算符是一个成员函数，其左侧运算对象就绑定到隐式的this参数。对一个二元运算符，例如赋值运算符，右侧运算对象作为显式参数传递。

拷贝赋值运算符接受一个与其所在类相同类型的参数：

```cpp
class Foo{
public:
    Foo &operator=(const Foo&);	//赋值运算符
    // ...
}
```

为了与内置类型的赋值保持一直，赋值运算符通常返回一个指向其左侧运算对象的引用。另外，标准库通常要求保存在容器中的类型具有赋值运算符，且返回值是左侧运算符对象的引用。

编译器合成的拷贝赋值运算符类似拷贝构造，也是逐一进行成员拷贝（非static），类类型通过它自身的拷贝赋值运算符来完成，数组成员为类类型的，也会逐一调用自身的拷贝赋值运算符。最后，返回一个指向左侧运算对象的引用。

```cpp
//等价于合成拷贝赋值运算符
Sales_data& Sales_data::operator=(const Sales_data &rhs)
{
    bookNo = rhs.bookNo;	//调用string::operator=
    units_sold = rhs.units_sold;	//使用内置的int赋值
    revenue = rhs.revenue;	//使用内置的double赋值
    return *this;	//返回左侧对象的引用
}
```

### 析构函数

与构造执行的操作相反。

析构函数名字比构造函数多了一个~。没有返回值，也没有参数。

```cpp
class Foo{
public:
    ~Foo();	//析构函数
    ...
};
```

析构函数不能被重载，是惟一的。

调用析构的时机：

- 变量在离开作用域时被销毁
- 当一个对象被销毁时，其成员被销毁
- 容器被销毁时（标准库容器或数组），其元素被销毁
- 动态分配的对象，当对指向它的指针应用delete时被销毁
- 临时对象，当创建它的完整表达式结束时被销毁

```cpp
{//新作用域
    //p和p2指向动态分配对象
    Sales_data *p = new Sales_data;//p是一个内置指针
    auto p2 = make_shared<Sales_data>();	//p2是一个shared_ptr
    Sales_data item(*p);	//拷贝构造函数将*p拷贝到item中
    vector<Sales_data> vec;	//局部对象
    vec.push_back(*p2);		//拷贝p2指向的对象
    delete p;				//对p指向的对象执行析构函数
}//退出局部作用域；对item、p2和vec调用析构函数
//销毁p2会递减其引用计数；如果引用计数变为0，则对象释放
//销毁vec会销毁它的元素
```

如果类未定义析构，则编译器会自动合成。

```cpp
class Sales_data{
public:
    //成员会被自动销毁，除此之外不需要做其他事情
    ~Sales_data(){}
    //其他成员的定义
    ...
};
```

析构函数体（空）执行完毕后，成员会被自动销毁。本例中string的析构函数会被调用，释放bookNo的内存。析构函数体本身不直接销毁成员，它们是在函数体之后隐含的析构阶段中被销毁的。析构函数体只是析构过程的一部分。

### 三五法则

#### 需要析构函数的类也需要拷贝和赋值操作

因为析构函数需要去手工delete成员指针。这种情况下，编译器合成的拷贝构造和赋值运算符就会有问题，因为仅仅只是完成了浅拷贝，拷贝了成员指针的地址值，这可能引起问题。所以这种情况我们要自己写深拷贝代码。

#### 需要拷贝操作的类也需要赋值操作，反之亦然

因为语义上拷贝构造和赋值操作是一致的，只是调用时机不同。提供了一个就说明需要特化某些操作，那么对应的另一个也要一致。但需要二者却不一定需要一个析构。

### =default

=default可以显式地要求编译器生成合成的版本。

```cpp
class Sales_data{
public:
    Sales_data() = default;
    Sales_data(const Sales_data&) = default;
    Sales_data &operator=(const Sales_data &);
    ~Sales_data() = default;
    //其他成员
    ...
};
Sales_data &Sales_data::operator=(const Sales_data&) = default;
```

类内使用=default声明，合成的函数会隐式地声明为inline。

### =delete

有些情况我们希望阻止类的拷贝或赋值。比如iostream就阻止了拷贝，避免多个对象写入或读取相同的IO缓冲。

```cpp
struct NoCopy{
    NoCopy() = default;	//合成的默认构造函数
    NoCopy(const NoCopy&) = delete;	//阻止拷贝
    NoCopy& operator=(const NoCopy&) = delete;	//阻止赋值
    ~NoCopy() = default;
};
```

=delete通知编译器，不希望定义这些成员。

注意，析构函数不能删除，其他任何函数都可以指定=delete。虽然语法上允许析构函数指定=delete，但这样一来涉及到该类的对象都不能用，因为它无法销毁。

所以，记着析构函数不能加=delete这条软规则即可。

如果一个类有数据成员不能默认构造、拷贝、复制或销毁，那么对应的成员函数将被定义为删除的。这就意味着，composite模式的数据成员自身残疾将影响整个团队残疾。

具有引用成员或无法默认构造的const成员的类，编译器不会合成默认构造函数。如果类有const成员，则它不能使用合成的拷贝赋值运算符（新值是不能给const对象的）。

在没有=delete之前，C++是通过private权限限制拷贝构造函数和拷贝赋值运算符来阻止拷贝的。这种方法有一个疏漏，就是友元函数和成员函数是可以进行拷贝的。

## 拷贝控制和资源管理

类一旦管理了类外资源，往往就需要自定义析构，根据三五法则也就意味着要自定义拷贝构造和拷贝赋值运算符。

而定义拷贝控制成员时，首先要确定类的拷贝语义，我们是让类的行为看起来像值还是像指针。

- 如果是像值，比如string、标准库容器类等，它们的拷贝会使得副本对象和原对象完全独立，改变副本不会影响原对象。
- 如果是像指针，比如shared_ptr，那么拷贝的就是指针，指向的是同一个对象。
- 当然，也可以设置为不允许拷贝或赋值，此时既不像值也不像指针。

### 行为像值的类

```cpp
class HasPtr{
public:
  	HasPtr(const std::string &s = std::string()):ps(new std::string(s)), i(0){}
  	//对ps指向的string，每个HasPtr对象都有自己的拷贝
  	HasPtr(const HasPtr &p):ps(new std::string(*p.ps)), i(p.i){}
  	HasPtr& operator=(const HasPtr &);
  	~HasPtr(){delete ps;}
private:
  	std::string *ps;
  	int i;
};

HasPtr& HasPtr::operator=(const HasPtr &rhs)
{
  	//这里一定要先new再delete，因为赋值操作赋值给自己是合法的
  	//如果赋值给自己，先delete意味着rhs.ps就丢了
    auto newp = new string(*rhs.ps);	//拷贝底层string
  	delete ps;	//释放旧内存
  	ps = newp;	//从右侧运算对象拷贝数据到本对象
  	i = rhs.i;
  	return *this;	//返回本对象
}
```

> 赋值运算符要谨记一个好习惯，在销毁左侧运算对象资源之前先拷贝右侧运算对象资源。

### 行为像指针的类

```cpp
class HasPtr{
public:
  	//构造函数分配新的string和新的计数器，将计数器置为1
  	HasPtr(const std::string &s = std::string()):ps(new std::string(s)), i(0), use(new std::size_t(1)){}
  	//拷贝构造函数拷贝所有3个数据成员，并递增计数器
  	HasPtr(const HasPtr &p):ps(p.ps), i(p.i), use(p.use){++*use;}
  	HasPtr& operator=(const HasPtr&);
  	~HasPtr();
private:
  	std::string *ps;
  	int i;
  	std::size_t *use;	//用来记录有多少个对象共享*ps的成员
};

HasPtr::~HasPtr()
{
    if(--*use == 0){	//如果引用计数变为0
        delete ps;		//释放string内存
      	delete use;		//释放计数器内存
    }
}

HasPtr& HasPtr::operator=(const HasPtr &rhs)
{
    ++*rhs.use;	//递增右侧运算对象的引用计数
  	if(--*use == 0){	//然后递减本对象的引用计数
        delete ps;		//如果没有其他用户
      	delete use;		//释放本对象分配的成员
    }
  	ps = rhs.ps;	//将数据从rhs拷贝到本对象
  	i = rhs.i;
  	use = rhs.use;
  	return *this;	//返回本对象
}
```

赋值运算符要考虑自赋值的情况，所以在左侧递减引用计数之前先递增右侧引用计数。

## 交换操作

除了拷贝控制成员外，管理资源的类一般还定义一个swap函数。对与重排元素顺序的算法一起使用的类来说，swap非常重要，因为这些算法交换两个元素时会调用swap。

如果类自己定义了swap，算法就使用自定义版本，否则使用标准库定义的swap。

```cpp
class HasPtr{
    friend void swap(HasPtr&, HasPtr&));
  	//其他成员定义
  	...
};

inline void swap(HasPtr &lhs, HasPtr &rhs)
{
    using std::swap;
  	swap(lhs.ps, rhs.ps);	// 交换指针，而不是string数据
  	swap(lhs.i, rhs.i);		// 交换int成员
}
```

swap不是必要的，但对分配了资源的类来说，定义swap是一种很重要的优化手段。

swap定义的一个坑：

```cpp
//Foo有类型为HasPtr的成员h
void swap(Foo &lhs, Foo &rhs)
{
    //错误：这个函数使用了标准库版本的swap，而不是HasPtr版本
  	std::swap(lhs.h, rhs.h);
  	// 交换类型Foo的其他成员
}

//正确的写法：
void swap(Foo &lhs, Foo &rhs)
{
    using std::swap;
  	swap(lhs.h, rhs.h);	//使用HasPtr版本的swap
  	//交换类型Foo的其他成员
}
```

这种未加限定的写法之所以可行，本质上是因为类型特定的swap版本匹配程度优于声明的std::swap版本。而对std::swap的声明可以使得在找不到类型特定版本时可以正确的找到std中的版本。

swap常用于赋值运算符，它可以一步到位完成拷贝并交换的技术。

```cpp
//注意rhs是按值传递的，意味着HasPtr的拷贝构造函数将右侧运算对象中的string拷贝到rhs
HasPtr& HasPtr::operator=(HasPtr rhs)
{
    //交换左侧运算对象和局部变量rhs的内容
  	swap(*this, rhs);	//rhs现在指向本对象曾经使用的内存
  	return *this;	//rhs被销毁，从而delete了rhs中的指针
}
```

这里的参数不是引用，右侧运算对象是值传递，所以rhs是右侧运算对象的副本。因此直接swap就一步到位了，自动销毁rhs时就自动销毁了原对象（执行析构）。

**使用拷贝和交换的赋值运算符天生异常安全，且能正确处理自赋值。**

## 对象移动

C++11引入了一个特性：可以移动而非拷贝对象。移动而非拷贝对象会大幅度提升性能。

旧版本即使在不必拷贝对象的情况下，也不得不拷贝，对象如果巨大，那么拷贝的代价是昂贵的。在旧版本的标准库中，容器所能保存的类型必须是可拷贝的。但在新标准中，可以用容器保存不可拷贝，但可移动的类型。

标准库容器、string和shared_ptr类既支持移动也支持拷贝。IO类和unique_ptr类可以移动但不能拷贝。

### 右值引用

为了支持移动操作，C++11引入了一个新的引用类型——右值引用(rvalue reference)。所谓右值引用就是必须绑定到右值的引用。通过&&来获得右值引用（左值引用是通过&）。**右值引用只能绑定到一个将要销毁的对象。**因此，才得以自由地将一个右值引用的资源转移给另一个对象。

```cpp
int i = 42;	
int &r = i;		//正确：r引用i，r是左值引用
int &&rr = i;	//错误：右值引用不能绑定到左值上
int &r2 = i*42;	//错误：i*42是右值
const int &r3 = i*42;	//正确：可以将一个const引用绑定到一个右值上
int &&rr2 = i*42;	//正确：将rr2绑定到乘法结果上
```

> 最特别的就是const左值引用是可以绑定到右值的。

变量表达式都是左值，所以不能将一个右值引用直接绑定到一个变量上，即使这个变量的类型是右值引用也不行。

```cpp
int &&rr1 = 42;		//正确：字面常量是右值
int &&rr2 = rr1;	//错误：表达式rr1是左值
```

左值是持久的，右值是短暂的。

### 标准库move函数

虽然右值引用不能绑定到左值，但可以显式地将左值转换为对应的右值引用类型。调用`move`函数可以获得绑定在左值上的右值引用，此函数定义在头文件utility中。

```cpp
int &&rr3 = std::move(rr1);	//正确
```

move告诉编译器：我们有一个左值，但我们希望像一个右值一样处理它。但使用move就意味着承诺：除了对rr1赋值或销毁它外，我们将不再使用它。

可以销毁一个移后源对象，也可以赋予它新值，但不能使用移后源对象的值。

调用move函数的代码应该使用std::move而非move，这样做可以避免潜在的名字冲突。

### 移动构造函数和移动赋值运算符

移动构造函数类似拷贝构造，第一个参数是该类类型的引用。不同于拷贝构造函数，这个引用参数在移动构造函数中是一个右值引用。其他任何额外参数都必须有默认值（与拷贝构造一致）。

除了完成资源移动，移动构造函数还要保证移后源对象处于一个状态：销毁它是无害的。移动之后，源对象必须不再指向被移动的资源，这些资源归新对象所有。

```cpp
StrVec::StrVec(StrVec &&s) noexcept	//移动构造不应该抛任何异常
  //成员初始化器接管s中资源
  : elements(s.elements), first_free(s.first_free), cap(s.cap)
{
	//令s进入这样一个状态————对其运行析构函数是安全的
    s.elements = s.first_free = s.cap = nullptr;
}
```

移动构造函数不会分配任何新内存，它接管给定的StrVec的内存。接管之后，源对象的指针置nullptr。

移动操作通常不分配任何资源，因此移动操作通常不抛出任何异常。而通过noexcept可以通知标准库构造函数不会抛出异常，如果不通知，那么标准库会认为移动构造函数可能会抛出异常，为此会做一些额外的工作。

为什么要指出移动操作不抛出异常呢？因为标准库能对异常发生时其自身的行为提供保证，比如vector保证push_back时发生异常不会改变vector本身。

之所不异常时不改变vector，是因为拷贝构造函数中发生异常时，旧元素的内存空间是没有变化的，至于新内存空间尽管发生了异常，vector可以直接释放新分配的内存（尚未成功构造）并返回，这不会影响vector原有的元素。但移动语义就不同，如果移动了部分元素时发生了异常，那么这时源元素就已经被改变了，这就无法满足自身保持不变的要求了。

所以除非vector知道元素类型的移动构造函数不会抛异常，否则在重新分配内存时，它必须使用拷贝构造而不是移动构造。基于此，如果希望vector重新分配内存时可以使用自定义类型对象的移动操作而不是拷贝操作，那就要显式的声明我们的移动构造函数是noexcept的。

### 移动赋值运算符

类似移动构造，如果不抛出任何异常，也要标记为noexcept。

```cpp
StrVec &StrVec::operator=(StrVec &&rhs) noexcept
{
    //直接检测自赋值
  	if(this != &rhs){
        free();	//释放已有元素
      	elements = rhs.elements;	/从rhs接管资源
        first_free = rhs.first_free;
      	cap = rhs.cap;
      	//将rhs置于可析构状态
      	rhs.elements = rhs.first_free = rhs.cap = nullptr;
    }
  	return *this;
}
```

### 合成的移动操作

如果自己不定义，编译器也会自动合成移动操作，但这和拷贝操作不同，它需要一些条件。

- 如果一个类定义了自己的拷贝构造函数、拷贝赋值运算符或者析构函数，编译器就不会合成移动操作。

- 只有当一个类没有定义任何自己版本的拷贝控制成员，且类的每个非static数据成员都可以移动时，编译器才会为它合成移动构造和移动赋值运算符。

  ```cpp
  //编译器为X和hasX合成移动操作
  struct X{
      int i;	//内置类型可以移动
    	std::string s;	//string定义了自己的移动操作
  };
  struct hasX{
      X mem;	//X有合成的移动操作
  };
  X x, x2 = std::move(x);	//使用合成的移动构造函数
  hasX hx, hx2 = std::move(hx);	//使用合成的移动构造函数
  ```

  与拷贝操作不同，移动操作永远不会被隐式定义为删除的函数。但如果显式地要求编译器生成=default的移动操作，且编译器不能移动全部成员，则移动操作会被定义为删除的函数。

  定义了移动构造或移动赋值的类也必须定义自己的拷贝操作，否则拷贝操作默认被定义为删除的。

  如果类既有移动构造，也有拷贝构造，那么编译器使用普通的函数匹配规则来确定使用哪个构造函数。赋值也类似。

  ```cpp
  StrVec v1, v2;
  v1 = v2;	//v2是左值，使用拷贝赋值
  StrVec getVec(istream &s);	//getVec返回一个右值
  v2 = getVec(cin);	//getVec(cin)是一个右值，使用移动赋值
  ```

  如果类有拷贝构造，但没有移动构造，函数匹配规则会保证该类型的对象会被拷贝：

  ```cpp
  class Foo{
  public:
    	Foo() = default;
    	Foo(const Foo&);
    	...
  };
  Foo x;
  Foo y(x);	//拷贝构造函数，x是左值
  Foo z(std::move(x));	//拷贝构造函数，因为未定义移动构造函数
  ```

  在未定义移动构造的情境下，`Foo z(std::move(x)`之所以可行，是因为我们可以把`Foo&&`转换为一个`const Foo&`。

五个拷贝控制成员应该当成一个整体来对待。如果一个类需要任何一个拷贝操作，它就应该定义所有五个操作。

C++11标准库定义了移动迭代器（move iterator）适配器。一个移动迭代器通过改变给定迭代器的解引用运算符的行为来适配此迭代器。移动迭代器的解引用运算符返回一个右值引用。调用`make_move_iterator`函数能将一个普通迭代器转换成移动迭代器。原迭代器的所有其他操作在移动迭代器中都照常工作。

最好不要在移动构造函数和移动赋值运算符这些类实现代码之外的地方随意使用move操作。std::move是危险的。

### 右值引用和成员函数

在非static成员函数的形参列表后面添加引用限定符（reference qualifier）可以指定this的左值/右值属性。引用限定符可以是`&`或者`&&`，分别表示this可以指向一个左值或右值对象。引用限定符必须同时出现在函数的声明和定义中。

```cpp
class Foo
{
public:
    Foo &operator=(const Foo&) &; // 只能向可修改的左值赋值
    // 其他成员
};

Foo &Foo::operator=(const Foo &rhs) &
{
    // 执行将rhs赋予本对象所需的工作
    return *this;
}
```

一个非static成员函数可以同时使用const和引用限定符，此时引用限定符跟在const限定符之后。

```cpp
class Foo
{
public:
    Foo someMem() & const;      // error
    Foo anotherMem() const &;   // ok
};
```

引用限定符也可以区分成员函数的重载版本。

```cpp
class Foo
{
public:
    Foo sorted() &&;        // 可用于可改变的右值
    Foo sorted() const &;   // 可用于任何类型的Foo
  	//Foo其他成员
private:
  	vector<int> data;
};

//本对象为右值，因此可以原址排序
Foo Foo::sorted() &&
{
    sort(data.begin(), data.end());
  	return *this;
} 
//本对象是const或是一个左值，哪种情况我们都不能对其进行原址排序
Foo Foo::sorted() const &{
    Foo ret(*this);
  	sort(ret.data.begin(), ret.data.end());
  	return ret;
}

retVal().sorted();	//retVal()是右值，调用Foo::sorted() &&
retFoo().sorted();	//retFoo()是左值，调用Foo::sorted() const &
```

如果定了两个或两个以上具有相同名字和相同参数列表的成员函数，要么都加引用限定符，要么都不加，这一点不受const this的影响。

```cpp
class Foo
{
public:
    Foo sorted() &&;
    Foo sorted() const;    // 错误：必须加上引用限定符
    // Comp是函数类型的类型别名
    // 此函数类型可以用来比较int值
    using Comp = bool(const int&, const int&);
    Foo sorted(Comp*);  		// 正确：不同的参数列表
  	Foo sorted(Comp*) const;	//正确：两个版本都没有引用限定符
};
```



