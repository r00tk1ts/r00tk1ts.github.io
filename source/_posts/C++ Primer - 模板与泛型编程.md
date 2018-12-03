---
title: C++ Primer - 模板与泛型编程
date: 2018-12-03 20:48:11
categories: programming-language
tags:
	- cpp
	- cpp-primer

---

《C++ Primer》第五版引入了11标准相关内容，我早年在初学C++时还只有第四版，近来想对C++做一个整体的复习+版本升级，借此机会过一遍第五版。本文是阅读第十六章“模板与泛型编程”时所做的笔记。

<!--more-->

# C++ Primer - 模板与泛型编程

模板是一种pattern，它用于创建一个类或函数。模板是C++泛型的基础，此前所学的标准库的vector、list等都是模板。

## 定义模板

### 函数模板

设计程序时往往希望同一个函数可以处理多种不同类型的参数，由于类型转换可能会引起各种各样的问题，我们需要对每一种所需的类型都重载一个几乎相同的函数。

```cpp
int compare(const string &v1, const string &v2)
{
    if(v1 < v2) return -1;
    if(v2 < v1) return 1;
    return 0;
}
int compare(const double &v1, const double &v2)
{
    if(v1 < v2) return -1;
    if(v2 < v1) return 1;
    return 0;
}
```

而如果利用函数模板就可以一步到位：

```cpp
template <typename T>
int compare(const T &v1, const T &v2)
{
    if(v1 < v2) return -1;
    if(v2 < v1) return 1;
    return 0;
}
```

以template关键字开始，后跟模板参数列表（template parameter list），由逗号分隔（如果有多个的话）。typename是类型关键字，旧的版本使用的是class，class现在已经不推荐使用了。

模板参数很像函数参数，在使用模板时，我们可以显式或隐式的指定模板实参，将其绑定到模板参数上。

对上例来说，T表示一种泛化的类型，它的实际类型在实例化模板时被确定。

#### 实例化模板

由于模板compare只是一个pattern，所以在使用参数调用它时要先确定T的类型，这一过程叫做模板实例化。

```cpp
//实例化出int compare(const int &, const int &)
cout << compare(1,0) << endl;	//T为int
//实例化出int compare(const vector<int>&, const vector<int>&)
vector<int> vec1{1, 2, 3}, vec2{4, 5, 6};
cout << compare(vec1, vec2) << endl;	//T为vector<int>
```

编译器会实例化出两个不同版本的compare。

#### 模板类型参数

compare函数有一个模板类型参数，该类型参数可以看成一个类型说明符，可以像内置类型说明符一样使用。类型参数可以用来指定返回类型或函数的参数类型，以及在函数体内用于变量声明或类型转换：

```cpp
//正确：返回类型和参数类型相同
template <typename T>
T foo(T* p)
{
    T tmp = *p;	// tmp的类型将是指针p指向的类型
    //...
    return tmp;
}
```

如果类型参数不止一个，那么他们不能同名，且每一个前面都需要typename关键字，用逗号隔开。

```cpp
template <typename T, typename U>
T calc(const T&, const U&);
```

#### 非类型模板参数

除了定义类型参数，还可以定义非类型参数(nontype parameter)。非类型参数表示一个值而非一个类型。

模板被实例化时，非类型参数被一个用户提供的或编译器推断出的值所代替。这些值必须是常量表达式，从而允许编译器在编译时实例化模板。

```cpp
template<unsigned N, unsigned M>
int compare(const char (&p1)[N], const char (&p2)[M])
{
    return strcmp(p1, p2);
}
```

处理两个字符数组，两个非类型模板参数是unsigned int值，该值在调用时确定，可以由程序员显式指定，也可以隐式推断。

```cpp
compare("hi", "mom");
//编译器会用字面常量的大小来代替N和M，从而实例化模板。
// int compare(const char (&p1)[3], const char (&p2)[4])
```

非类型参数可以是一个整型，或是一个指向对象或函数类型的指针或左值引用。绑定到非类型整型参数的实参必须是一个常量表达式。绑定到指针或引用非类型参数的实参必须具有静态的生存期。不能用一个普通局部变量或动态对象作为指针或引用非类型模板参数的实参。指针参数可以用nullptr或一个值为0的常量表达式来实例化。

模板定义内，模板非类型参数是一个常量值。在需要常量表达式的地方，可以使用非类型参数，例如，指定数组大小。

#### inline和constexpr函数模板

函数模板可以声明为inline或constexpr的，如同非模板函数一样。

```cpp
template <typename T>
inline T min(const T&, const T&);
```

#### 编写类型无关的代码

泛型代码要适配各种用到的类型。compare虽然简单，但说明了编写泛型代码的两个重要原则：

- 函数参数是const引用
- 条件判断仅仅使用<运算符

通过将函数参数设定为const引用，就保证了函数可以用于不能拷贝的类型。不拷贝也提高了效率。

仅仅使用<运算符降低了compare对处理类型的要求，只要适配的类型支持<运算符，就可以应用compare。

归根结底，核心的思想就是让模板程序尽量降低对实参类型的要求。

#### 模板编译

编译器遇到一个模板定义时，并不生成代码。只有当实例化模板时编译器才生成代码。

当调用一个函数时，编译器只需要掌握函数的声明，函数定义不必已经出现，即使不定义，编译也会通过，最终会在链接时才发现undefined symbol这个熟悉的错误。

但对于模板来说却不同，生成一个实例化版本，编译器需要掌握函数模板或类模板成员函数的定义。因此，与非模板代码不同，模板的头文件通常既包含声明也包含定义。

函数模板和类模板成员函数定义通常在头文件中。

#### 大多数编译错误在实例化期间报告

模板直到实例化时才生成代码，所以获得模板代码编译错误的时机较晚。编译器在3个阶段报告错误：

- 第一阶段是编译模板本身时，此时一般错误很少，只是检查语法错误，不检查依赖于类型的代码。
- 第二个阶段是遇到使用模板时，对函数模板调用编译器会检查实参数目是否正确。还要检查参数类型是否匹配。对类模板来说，编译器检查用户是否提供了正确的模板实参，但也仅限于此。
- 第三个阶段是模板实例化时，只有这个阶段才能发现类型相关的错误。依赖于编译器如何管理实例化，这类错误可能在链接时才报告。

编写模板代码不能针对特定类型，但模板代码通常对其所用的类型有一些假设。比如compare就会假设实参支持<运算符。

如果实例化T类型不支持<，那么就会在第三个阶段报错。

### 类模板

类也有pattern，称为类模板。我们已经见过很多了，标准库的vector, list, deque, shared_ptr都是类模板。

类模板和函数模板有所不同，编译器不能为类模板推断模板类型，需要显式的指定参数类型。

#### 定义

```cpp
template <typename T>
class Blob{
public:
    typedef T value_type;
    typedef typename std::vector<T>::size_type size_type;
    //构造函数
    Blob();
    Blob(std::initializer_list<T> il);
    // Blob中的元素数目
    size_type size() const {return data->size();}
    bool empty() const {return data->empty();}
    // 添加和删除元素
    void push_back(const T &t){data->push_back(t);}
    // 移动版本
    void push_back(T &&t){data->push_back(std::move(t));}
    void pop_back();
    // 元素访问
    T& back();
    T& operator[](size_type i);
private:
   	std::shared_ptr<std::vector<T>> data;
    //若data[i]无效，则抛出msg
    void check(size_type i, const std::string &msg) const;
};
```

Blob模板有一个名为T的模板类型参数，用来表示Blob保存的元素的类型。

#### 实例化类模板

需要显式指定模板实参列表。

```cpp
Blob<int> ia;	//空Blob<int>
Blob<int> ia2 = {0, 1, 2, 3, 4};//有5个元素的Blob<int>
```

ia和ia2使用相同的特定类型版本的Blob。编译器会实例化出一个这样的类：

```cpp
template <>
class Blob<int>{
public:
    typedef typename std::vector<int>::size_type size_type;
    Blob();
    Blob(std::initializer_list<int> il);
    ...
    int& operator[](size_type i);
private:
    std::shared_ptr<std::vector<int>> data;
    void check(size_type i, const std::string &msg) const;
};
```

编译器从Blob模板实例化出一个类时，会重写Blob模板，将模板参数T的每个实例替换为给定的模板实参，本例中是int。

对不同类型，编译器都生成一个不同的类。

#### 模板作用域中引用模板类型

一个类模板中的代码如果使用了另外一个模板，通常不将一个实际类型（或值）的名字用作其模板实参。相反的，我们通常将模板自己的参数当做被使用模板的实参。

换句话说，就是支持模板内部的嵌套。

```cpp
std::shared_ptr<std::vector<T>> data;
```

如此传入的就是模板参数类型T，而非实际类型。当实例化模板类时，data也会跟着实例化。

#### 类模板的成员函数

既可以在类模板内部，也可以在外部定义成员函数，内部默认隐式inline。

定义方法：

```cpp
template <typename T>
ret-type Blob<T>::member-name(parm-list)
```

完成上面类成员函数的定义：

```cpp
template <typename T>
void Blob<T>::check(size_type i, const std::string &msg) const
{
    if(i >= data->size())
        throw std::out_of_range(msg);
}

template <typename T>
T& Blob<T>::back()
{
    check(0, "back on empty Blob");
    return data->back();
}
template <typename T>
T& Blob<T>::operator[](size_type i)
{
    //如果i太大，check会抛出异常，阻止访问一个不存在的元素
    check(i, "subscript out of range");
    return (*data)[i];
}
template <typename T>
void Blob<T>::pop_back()
{
    check(0, "pop_back on empty Blob");
    data->pop_back();
}
```

Blob的构造函数：

```cpp
template <typename T>
Blob<T>::Blob() : data(std::make_shared<std::vector<T>>){ }

template <typename T>
Blob<T>::Blob(std::initializer_list<T> il):data(std::make_shared<std::vector<T>>(il)){ }
```

#### 类模板成员函数的实例化

类模板成员函数只有当程序用到时才会进行实例化。

```cpp
//实例化Blob<int>和接受initializer_list<int>的构造函数
Blob<int> squares = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9};
//实例化Blob<int>::size() const
for(size_t i=0;i!=squares.size();++i)
    squares[i] = i*i;	//实例化Blob<int>::operator[](size_t)
```

如果成员函数没有被用到，那么就不会实例化。这一特性使得即使某些成员函数不能应用于模板参数类型，但只要不被调用，就可以实例化模板类。

#### 类代码内简化模板类名的使用

使用一个类模板类型时必须提供模板实参，唯独在类模板自己的作用域中，可以直接使用模板名而不提供实参：

```cpp
//若试图访问一个不存在的元素，BlobPtr抛出异常
template <typename T>
class BlobPtr{
public:
    BlobPtr():curr(0){ }
    BlobPtr(Blob<T> &a, size_t sz = 0) : wptr(a.data), curr(sz){ }
    T& operator*() const
    {
        auto p = check(curr, "dereference past end");
        return (*p)[curr];	//(*p)为本对象指向的vector
    }
    //递增和递减
    BlobPtr& operator++();	//前置运算符
    BlobPtr& operator--();	//内部不用写成BlobPtr<T>&
private:
    //若检查成功，check返回一个指向vector的shared_ptr
    std::shared_ptr<std::vector<T>>
        check(std::size_t, const std::string&) const;
    //保存一个weak_ptr，表示底层vector可能被销毁
    std::weak_ptr<std::vector<T>> wptr;
    std::size_t curr;	//数组中的当前位置
};
```

#### 类模板外使用类模板名

```cpp
template <typename T>
BlobPtr<T> BlobPtr<T>::operator++(int)
{
    BlobPtr ret = *this;
    ++*this;
    return ret;
}
```

因为在类作用域外，所以返回类型就是`BlobPtr<T>`，不能省略`<T>`。

#### 类模板和友元

类包含友元声明时，类与友元各自是否是模板是相互无关的。如果类模板包含一个非模板友元，则友元被授权可以访问所有模板实例。如果友元自身是模板，类可以授权给所有友元模板实例，也可以只授权给特定实例。

**一对一友好关系**

```cpp
//前置声明，在Blob中声明友元所需要的
template <typename> class BlobPtr;
template <typename> class Blob;
template <typename T>
bool operator==(const Blob<T> &, const Blob<T>&);

template <typename T>
class Blob{
    //每个Blob实例将访问权限授予用相同类型实例化的BlobPtr和相等运算符
    friend class BlobPtr<T>;
    friend bool operator==<T>(const Blob<T>&, const Blob<T>&)；
    ...
}

Blob<char> ca;//BlobPtr<char>和operator==<char>都是本对象的友元
Blob<int> ia;//BlobPtr<int>和operator==<int>都是本对象的友元
```

一对一，各访问各的，不能走错片场。

#### 通用和特定的模板友好关系

一个类也可以将另一个模板的每个实例都声明为自己的友元，或者限定特定的实例为友元：

```cpp
//前置声明，在将模板的一个特定实例声明为友元时要用到
template <typename T> class Pal;
class C{//C时一个普通的非模板类
    friend class Pal<C>;	//用类C实例化的Pal是C的友元
    //Pal2的所有实例都是C的友元，这种情况无需前置声明
    template <typename T> friend class Pal2;
};
template <typename T> 
class C2{//C2本身是类模板
    //C2的每个实例将相同实例化的Pal声明为友元
    friend class Pal<T>;	//Pal的模板声明必须在作用域之内
    //Pal2的所有实例都是C2每个实例的友元，不需要前置声明
    template <typename X> friend class Pal2;
    //Pal3是一个非模板类，它是C2所有实例的友元
    friend class Pal3;	//不需要Pal3的前置声明
};
```

#### 令模板自己的类型参数称为友元

C++11支持将模板类型参数声明为友元：

```cpp
template <typename Type>
class Bar{
	friend Type;	//将访问权限授予用来实例化Bar的类型
	//...
};
```

依然可以用内置类型来实例化Bar。默认是允许内置类型的友好关系的。

#### 模板类型别名

```cpp
typedef Blob<string> StrBlob;

template <typename T> using twin = pair<T, T>;
twin<string> authors;//authors是一个pair<string, string>
twin<int> win_loss;	//win_loss是pair<int, int>
twin<double> area;	//area是pair<double,double>

//也可以固定一个或多个模板参数
template <typename T> using partNo = pair<T, unsigned>;
pairNo<string> books;	//books是一个pair<string, unsigned>
pairNo<Vehicle> cars;	//cars是一个pair<Vehicle, unsigned>
pairNo<Student> kids;	//kids是衣蛾pair<Student, unsigned>
```

#### 类模板的static成员

类模板可以声明static成员。

```cpp
template <typename T>
class Foo{
public:
    static std::size_t count(){return ctr;}
private:
    static std::size_t ctr;
};
```

每个Foo的实例都有自己的static成员实例。同一`Foo<X>`类型对象共享同一个ctr对象和count函数。

一个static成员函数只有在使用时才会实例化。

### 模板参数

参数名字本身没什么含义，T是大家用惯的习俗，你可以起其他的名字。

#### 使用类的类型成员

编译器遇到类似T::mem的代码时，无法判断mem是类型成员还是一个static数据成员。

> 非模板类型就没有这个问题，因为编译器知道具体类的内部定义是什么样的。该是什么就是什么。但对类模板来说不行，实例化时才能确定。

为了处理模板，编译器必须知道名字是否表示一个类型。不然遇到`T::size_type * p;`时就会出现二义性：要么表示定义一个名为p的变量，要么表示static数据成员与p相乘，这是无法确定的。

所以，C++语法上规定，如果希望使用一个模板类型参数的类型成员，就必须显式地告诉编译器该名字是一个类型，方法就是通过typename来声明：

```cpp
template <typename T>
typename T::value_type top(const T& c)
{
    if(!c.empty())
        return c.back();
    else
        return typename T::value_type();
}
```

#### 默认模板实参

模板实参也可以提供默认值。C++11之后，函数模板和类模板都能提供。

> 早期的函数模板不能提供默认实参。

```cpp
//compare有一个默认模板实参less<T>和一个默认函数实参F()
template <typename T, typename F = less<T>>
int compare(const T &v1, const T &v2, F f = F())
{
    if(f(v1, v2)) return -1;
    if(f(v2, v1)) return 1;
    return 0;
}
```

F表示可调用对象的类型，定义了一个新的函数参数f，绑定到可调用对象上。

为此模板参数提供了默认实参，并为其对应的函数参数也提供了默认实参。默认实参指出compare将使用标准库的less函数对象类，它是使用与compare一样的类型参数实例化的。默认函数实参指出f将是类型F的一个默认初始化的对象。

```cpp
bool i = compare(0, 42);	//使用less；i为-1
//结果依赖于item1和item2中的isbn
Sales_data item1(cin), item2(cin);
bool j = compare(item1, item2, compareIsbn);
```

#### 模板默认实参与类模板

```cpp
template <typename T=int>
class Numbers{ //T默认为int
public:
	Numbers(T v = 0) : val(v){ }
private:
	T val;
};

Numbers<long double> lots_of_precision;
Numbers<> average_precision;	//空<>表示使用默认类型，不能省略
```

### 成员模板

类可以包含本身是模板的成员函数。这种成员称为成员模板。成员模板不能是虚函数。

#### 普通类的成员模板

```cpp
//函数对象类，对给定指针执行delete
class DebugDelete{
public:
    DebugDelete(std::ostream &s = std::cerr) : os(s){ }
    //与任何函数模板相同，T的类型由编译器推断
    template <typename T>
    void operator()(T *p) const
    {
        os << "deleting unique_ptr" << std::endl;
        delete p;
    }
private:
    std::ostream &os;
};

double *p = new double;
DebugDelete d;
d(p);//调用DebugDelete:operator()(double*)
int *ip = new int;
DebugDelete()(ip);//在临时DebugDelete对象上调用operator()(int*)
```

可以把DebugDelete作为unique_ptr的删除器。

```cpp
//销毁p指向的对象
//实例化DebugDelete::operator()<int>(int *)
unique_ptr<int, DebugDelete> p(new int, DebugDelete());
//销毁sp指向的对象
//实例化DebugDelete::operator()<string>(string*)
unique_ptr<string, DebugDelete> sp(new string, DebugDelete());
```

在p的构造函数中提供了DebugDelete类型的未命名对象。

unique_ptr析构函数会调用DebugDelete的调用运算符。

#### 类模板的成员模板

```cpp
template <typename T>
class Blob{
    template <typename It> Blob(It b, It e);
    //...
};
```

成员模板是函数模板。外部定义需要相继声明两个模板参数列表：

```cpp
template <typename T>
template <typename It>
Blob<T>::Blob(It b, It e) : data(std::make_shared<std::vector<T>>(b, e)){ }
```

#### 实例化与成员模板

```cpp
int ia[] = {0,1,2,3,4,5,6,7,8,9};
vector<long> vi = {0,1,2,3,4,5,6,7,8,9};
list<const char*> w = {"now", "is", "the", "time"};
//实例化Blob<int>类及其接受两个int*参数的构造函数
Blob<int> a1(begin(ia), end(ia));
//实例化Blob<int>类的接受两个vector<long>::iterator的构造函数
Blob<int> a2(vi.begin(), vi.end());
//实例化Blob<string>及其接受两个list<const char*>::iterator参数的构造函数
Blob<string> a3(w.begin(), w.end());
```

### 控制实例化

因为模板只有使用时才会实例化，所以相同的实例如果出现在不同的对象文件中，多个源文件使用了相同的模板，并提供了相同的模板参数时，就会导致每个文件中都有该模板的一个实例。

这会引起代码体积膨胀，C++11以后允许通过显式实例化来避免这种开销。

形式：

```cpp
extern template declaration;	//实例化声明
template declaration;	//实例化定义
```

declaration是一个类或函数声明，其中所有模板参数都已被替换为模板实参。例如：

```cpp
extern template class Blob<string>;	//声明
template int compare(const int&, const int&);	//定义
```

编译器遇到extern模板声明时，不会在本文件中生成实例化代码。extern承诺在程序其他位置存在该实例化的非extern声明（定义）。

extern可以有多个，但定义必须只有一个。extern声明要出现在任何使用代码之前。

## 模板实参推断

从函数实参来确定模板的过程称为模板实参推断（template argument deduction）。

### 类型转换与模板类型参数

传递给函数模板的实参被用来初始化形参。如果形参的类型使用了模板类型参数，那么它采用特殊的初始化规则。只有很有限的几种类型转换会自动地应用于这些实参。编译器通常不是对实参进行类型转换，而是生成一个新的模板实例。

顶层const无论在形参中还是实参中都会被忽略。其他类型转换中，能在调用中应用于函数模板的有两种情况：

- const转换：可以将一个非const对象的引用（或指针）传递给一个const的引用（或指针）形参。
- 数组或函数指针转换：如果函数形参不是引用类型，则可以对数组或函数类型的实参应用正常的指针转换。数组实参可以转换为一个指向其首元素的指针。函数实参可以转换为指向一个该函数类型的指针。

其他类型转换，如算术转换、派生类向基类的转换以及用于自定义的转换，都不能应用于函数模板。

```cpp
template <typename T> T fobj(T, T);	//实参被拷贝
template <typename T> T fref(const T&, const T&);//引用
string s1("a value");
const string s2("another value");
fobj(s1, s2);	//调用fobj(string, string);顶层const被忽略
fref(s1, s2);	//调用fref(const string&， const string&);s1应用第一种情况，非const对象引用转为const引用

int a[10],b[42];	
fobj(a, b);	//调用f(int*, int*)，应用上述第二种情况
fref(a, b);	//错误：数组类型不匹配，引用不能转换成指针，所以a和b推导出的形参型别不一致
```

对于函数模板中已明确定义类型的形参来说，转换规则和普通函数是一致的，模板形参的转换规则是独立的，只作用于模板形参。

### 函数模板显式实参

一些情况下编译器无法推导出模板实参的类型，这时我们可以显式地控制模板实例化。最常见的两种情况：

- 指定显式模板实参

  ```cpp
  template <typename T1, typename T2, typename T3>
  T1 sum(T2, T3);
  //无法推断出T1的类型，因为T1不在函数参数中出现
  ```

  调用时可以为T1提供一个显式模板实参：

  ```cpp
  //T1是显式指定的，T2和T3是从函数实参类型推断出来的
  auto val3 = sum<long long>(i, lng);	//long long sum(int ,long)
  ```

  显式指定模板实参要严格按照自左向右的顺序来匹配，可以指定全部，也可以指定部分。这里把T1放在第一个，是为了可以省略T2和T3的显式指定，当T2和T3未指定时，走默认的参数推导路线。

- 正常类型转换应用于显式指定的实参

  显式指定实参后，由于不需要借助类型推导，所以原本的模板类型就可以应用普通类型的那些类型转换规则了。

  ```cpp
  long lng;
  compare(lng, 1024);	//错误，不匹配模板参数
  compare<long>(lng, 1024);	//正确，实例化compare(long, long)
  compare<int>(lng,1024);	//正确，实例化compare(int, int)
  ```

  显式指定后，就可以应用算术类型转换了。

### 尾置返回类型与类型转换

用显式指定模板参数来控制返回类型很有效，但不是唯一的手法，C++11支持尾置返回类型来指定返回类型。

```cpp
template <typename It>
auto fcn(It beg, It end) -> decltype(*beg)
{
    //处理
    return *beg;	//返回序列第一个元素的引用
}

vector<int> fi={1,2,3,4,5};
Blob<string> ca={"hi","bye"};
auto &i = fcn(vi.begin(), vi.end());//fcn返回int&
auto &s = fcn(ca.begin(), ca.end());//fcn返回string&
```

> decltype作用于表达式时，如果表达式返回的是左值，则decltype返回左值引用。

如果fcn想返回值而不是引用，该如何编写函数体？

传递的参数是迭代器，所有迭代器操作只能生成元素的引用。标准库的类型转换模板可以做到这一点，它们定义在type_traits头文件中。

这里可以用remove_reference获得元素类型。它有一个模板类型参数和一个名为type的类型成员。如果用引用类型实例化remove_reference，则type就表示被引用的类型。

```cpp
template <typename It>
auto fcn2(It beg, It end) -> typename remove_reference<decltype(*beg)>::type
{
	//处理
	return *beg;	//返回序列中一个元素的拷贝
}
```

因为type是类的成员，对于类模板来说，需要在前面用关键字typename声明才能告知编译器type是一个类型，而非静态成员。

| 标准类型转换模板      |                             |                    |
| --------------------- | --------------------------- | ------------------ |
| 对`Mod<T>`，其中Mod为 | 若T为                       | 则`Mod<T>::type`为 |
| remove_reference      | X&或X&&<br />否则           | X<br />T           |
| add_const             | X&、const X或函数<br />否则 | T<br />const T     |
| add_lvalue_reference  | X&<br />X&&<br />否则       | T<br />X&<br />T&  |
| add_rvalue_reference  | X&或X&&<br />否则           | T<br />T&&         |
| remove_pointer        | X*<br />否则                | X<br />T           |
| add_pointer           | X&或X&&<br />否则           | X*<br />T*         |
| make_signed           | unsigned X<br />否则        | X<br />T           |
| make_unsigned         | 带符号类型<br />否则        | unsigned X<br />T  |
| remove_extent         | X[n]<br />否则              | unsigned X<br />T  |
| remove_all_extents    | `X[n1][n2]`...<br />否则    | X<br />T           |

### 函数指针和实参推断

用函数模板初始化函数指针或为一个函数指针赋值时，编译器使用指针的类型来推断模板实参。

```cpp
template <typename T>
int compare(const T&, const T&);
//pf1指向实例int compare(const int&, const int&)
int (*pf1)(const int&, const int&) = compare;
```

pf1的参数类型决定了T的推导。

但有一个坑：

```cpp
//func的重载版本；每个版本接受一个不同的函数指针类型
void func(int(*)(const string&, const string&));
void func(int(*)(const int&, const int&));
func(compare);	//错误：谁知道该用哪个实例？
func(compare<int>);	//这样是可以的，显式指定，消除二义性
```

### 模板实参推断和引用

```cpp
template <typename T>
void f(T &p);
```

参数p是一个模板类型参数T的引用。

#### 从左值引用函数参数推断类型

当函数参数是模板类型参数的一个左值引用时，只能传递给它一个左值。实参可以是const，也可以不是。如果实参是const的，则T被推断为const。

```cpp
template <typename T>
void f1(T&);	//实参必须是一个左值
//对f1的调用使用实参所引用的类型作为模板参数类型
f1(i);	//i是int，模板参数类型T为int
f1(ci);	//ci是const int，模板参数T为const int
f1(5);	//错误，5字面值是右值
```

如果函数参数的类型是const T&，那么绑定规则上来说，可以传递给形参任何类型的实参，对象（const或非const）、临时对象或是字面常量值都可以。函数参数本身是const时，T推断不会是const类型。因为const已经是参数的一部分了。

```cpp
template <typename T>
void f2(const T&);	//可以接受一个右值
//f2的参数是const &;实参中的const是无关的
//每个调用中，f2的函数参数都被推断为const int&
f2(i);	//i是一个int；模板参数T为int
f2(ci);	//ci是const int，但模板参数T为int
f2(5);	//一个const &参数可以绑定到一个右值，T为int
```

#### 从右值引用函数参数推断类型

函数参数是右值引用时，我们可以传递给它一个右值。

```cpp
template <typename T>
void f3(T&&);
f3(42);	//实参是int类型的右值，模板参数T是int
```

引用不是对象，所以没有引用的引用，因此间接定义时，一旦出现引用的引用，就要进行折叠：

- X& &, X& &&, X&& &都折叠成X&
- X&& &&折叠成X&&

### 理解std::move

move可以获得一个绑定到左值上的右值引用，这是如何做到的呢？

move的定义：

```cpp
//在返回类型和类型转换中也要用到typename
template <typename T>
typename remove_reference<T>::type&& move(T&& t)
{
    return static_cast<typename remove_reference<T>::type&&>(t);
}
```

代码很短，但却非常精妙。move的参数T&&是指向模板类型参数的右值引用。因为引用折叠的关系，此参数可以与任何类型的实参匹配。

```cpp
string s1("hi!"), s2;
s2 = std::move(string("bye!"));//正确：从一个右值移动数据
s2 = std::move(s1);	//正确：但在赋值之后，s1的值是未定义的
```

第一个赋值中，传递给move的实参是右值。所以，对std::move(string("bye"))来说：

- 推断出的T的类型为string
- 因此，remove_reference用string实例化
- `remove_reference<string>`的type成员是string
- move的返回类型是string&&
- move的函数参数t的类型为string&&

实例化就是string&& move(string &&t);

函数体返回`static_cast<string&&>(t)`。t的类型已经是string&&, 所以这里不做任何事。

第二个赋值中，传递的是左值：

- 推断出的T的类型为string&
- 因此，remove_reference用string&实例化
- `remove_reference<string&>`的type成员是string
- move的返回类型仍然是string&&
- move的函数参数t实例化为string& &&，折叠为string&

函数体返回`static_cast<string&&>(t)`，此时的t为string&, cast转为string&&。

所以实例化为string&& move(string &t);

#### 从左值static_cast到一个右值引用是允许的

缝缝补补，最终实际上语言层面是基于static_cast允许左值到右值引用的类型转换。所以我们也可以用原生的static_cast来完成移动语义，但肯定直接用std::move最方便。

### 转发

某些函数会把实参连同类型不变地转发给其他函数，此时我们要保持实参的所有性质（实参类型是否是const以及实参是左值还是右值）。

```cpp
//接受一个可调用对象和另外两个参数的模板
//对”flip“的参数调用给定的可调用对象
//flip1是不完整实现，顶层const和引用丢失了
template <typename F, typename T1, typename T2>
void flip1(F f, T1 t1, T2 t2)
{
    f(t2, t1);
}
```

一般情况下该函数工作正常，但当希望它调用一个接受引用参数的函数时就有问题：

```cpp
void f(int v1, int &v2)//v2是引用
{
    cout << v1 << " " << ++v2 << endl;
}
```

如果通过flip1调用f：

```cpp
f(42, i);	//i会递增
flip1(f, j, 42);	//通过flip1调用不会改变j
```

因为实例化的T1是int类型而非int&。实例化为`void flip1(void(*fcn)(int, int&), int t1, int t2);`。

一种看似可以解决问题的方法：

```cpp
template <typename F, typename T1, typename T2>
void flip2(F f, T1 &&t1, T2 &&t2)
{
    f(t2,t1);
}
```

如此，调用flip2(f,j,42)时，t1由于是左值j，所以会根据引用折叠推导出T1的类型为int&，而T2是int &&。当flip2调用f时，f中引用参数v2被绑定到j。而t1依然是int &&。

但如果定义的f接受了右值引用参数：

```cpp
void g(int &&i, int & j)
{
    cout << i << " " << j << endl;
}
```

此时，通过flip2调用g：

```cpp
flip2(g, i, 42);//错误，不能从一个左值实例化int&&
```

还是有问题。

那要如何兼容呢？可以使用std::forward来缝补。

forward定义在utility头文件，forward必须通过显式模板实参来调用。forward返回该显式实参类型的右值引用。

`forward<Type>`会返回Type&&，而如果Type是左值引用，则引用重叠会令forward返回的依然是左值引用，所以forward就可以兼容上述的两种情形。既不改变那些左值引用参数，又提供了对右值引用传导的支持。

利用forward重写flip：

```cpp
template <typename F, typename T1, typename T2>
void flip(F f, T1 &&t1, T2 &&t2)
{
    f(std::forward<T2>(t2), std::forward<T1>(t1));
}
```

## 重载与模板

函数模板可以被另一个函数模板或普通函数重载。

函数匹配规则：

- 候选函数包括所有模板实参推断成功的函数模板实例
- 候选的函数模板总是可行的，因为模板实参推断会排除不可行的模板
- 可行函数按类型转换来排序（仅限可应用于函数的类型转换）。
- 如果恰有一个函数提供比任何其他函数都更好的匹配，则选择此函数。如果同时又多个函数提供同样好的匹配，就按规则取舍：
  - 如果其中只有一个是非模板函数，则选择此函数
  - 如果没有非模板函数，那么选择其中最特例化的那个
  - 否则，调用有歧义

模板推断是很复杂的，Primer只是渗透了一点点内容。

## 可变参数模板

C++11支持可变参数模板（variadic template），可变数目的参数被称为参数包。

```cpp
//Args是模板参数包，rest是函数参数包
//Args表示零或多个模板类型参数
//rest表示零或多个函数参数
template <typename T, typename... Args>
void foo(const T &t, const Args& ... rest);

int i = 0; double d = 3.14; string s = "how now brown cow";
foo(i,s,42,d);	//包中有3个参数
foo(s,42,"hi");	//包中有2个参数
foo(d,s);		//包中有1个参数
foo("hi");		//空包
```

编译器会实例化4个对应的版本。

可以通过`sizeof…`运算符来获得包中元素个数，它返回常量表达式。

### 可变参数函数模板

可变参数函数模板可以玩递归

```cpp
//用来终止递归并打印最后一个元素的函数
//此函数必须在可变参数版本的print定义之前声明
template <typename T>
ostream &print(ostream &os, const T &t)
{
    return os << t;	//包中最后一个元素之后不打印分隔符
}

template <typename T, typename... Args>
ostream &print(ostream &os, const T &t, const Args&...reset)
{
    os << t << ", ";//打印第一个实参
    return print(os, rest...);	//递归调用，打印其他参数
}
```

### 包扩展

参数包除了可以获取大小外，还可以对他进行扩展(expand)。扩展一个包，需要提供扩展元素的模式（pattern）。扩展一个包就是将它分解为构成的元素，对每个元素应用pattern，获得扩展后的列表。在pattern右边放一个省略号来触发扩展操作。

上例中用到了两处包扩展：

```cpp
template <typename T, typename... Args>
ostream &print(ostream &os, const T &t, const Args&... rest)	//扩展Args
{
	os << t << ", ";
	return print(os, rest...);	//扩展reset
}
```

实际上还有高级的用法:

```cpp
template <typename... Args>
ostream &errorMsg(ostream &os, const Args&... rest)
{
    // print(os, debug_rep(a1), debug_rep(a2),...,debug_rep(an))
    return print(os, debug_rep(rest)...);//print调用中对每个实参调用debug_rep
}
```

print使用了模式debug_rep(rest)。

可变参数模板与forward机制可以编写函数，实现将实参不变地传给其他函数。

标准库的emplace_back就是可变参数成员模板，在容器中直接构造一个元素。

```cpp
template <class... Args>
inline void StrVec::emplace_back(Args&&... args)
{
    chk_n_alloc();//如果需要的话重新分配StrVec空间
    alloc.construct(first_free++,std::forward<Args>(args)...);
}
```

先确保有内存空间，然后在first_free指向的位置构造了一个元素，construct调用中扩展为`std::forward<Args>(args)...`。

对于`svec.emplace_back(10,'c');`来说，扩展成：`std::forward<int>(10),std::forward<char>(c)`

## 模板特例化

模板可以定义特例化版本，将其中一个或多个模板参数指定为特定的类型。

### 函数模板特例化

特例化函数模板需要为每个模板参数都提供实参。使用template接<>表示特例化。

```cpp
template<>
int compare(const char* const &p1, const char* const &p2)
{
    return strcmp(p1, p2);
}
```

它是原型`template <typename T>int compare(const T&, const T&)`的一个特例化版本，其中T为const char*。

定义特例化版本实际上是人为的接管了这一种类型的编译器生成工作。

### 类模板特例化

类似函数模板特例化：

```cpp
namespace std{
template<>
struct hash<Sales_data>
{
	//用来散列一个无序容器的类型必须要定义下列类型
	typedef size_t result_type;
	typedef Sales_data	argument_type;	//默认情况下，此类型需要==
	size_t operator()(const Sales_data& s)const;
	//我们的类使用合成的拷贝控制成员和默认构造函数
};
size_t hash<Sales_data>::operator()(const Sales_data& s) const
{
    return hash<string>()(s.bookNo)^
        hash<unsigned>()(s.units_sold)^
        hash<double>()(srevenue);
}
}
```

为Sales_data定义特化版本。要注意将`std::hash<Sales_data>`声明为Sales_data的友元，因为它访问了私有成员。

### 类模板部分特化(偏特化)

类模板也可以不必为所有模板参数提供实参，只指定一部分模板参数，这叫部分特化（partial specialization）。部分特例化后依旧是模板，需要进一步由用户指定其余未指定的模板参数。

函数模板不支持偏特化。

```cpp
//原始版本
template <class T>
struct remove_reference
{
    typedef T type;
};

//部分特化版本，用于左值引用和右值引用
template <class T>
struct remove_reference<T&>//左值引用
{
	typedef T type;
};
template <class T>
struct remove_reference<T&&>//右值引用
{
    typedef T type;
}
```

部分特例化依然是模板，所以使用时和模板没什么差别：

```cpp
int i;
//decltype(42)为int，使用原始模板
remove_reference<decltype(42)>::type a;
//decltype(i)为int&，使用第一个(T&)部分特例化版本
remove_reference<decltype(i)>::type b;
//decltype(std::move(i))为int&&，使用第二个(即T&&)部分特例化版本
remove_reference<decltype(std::move(i))>::type c;
```

a,b,c都是int类型。

也可以只特例化特定成员而不是整个模板。

```cpp
template <typename T>
struct Foo{
    Foo(const T &t = T()) : mem(t){ }
    void Bar(){ /* ... */ }
    T mem;
    //Foo其他成员
}；
template<>	//特例化一个模板
void Foo<int>::Bar()	//特例化Foo<int>的成员Bar
{
	//进行应用于int的特例化处理
}

Foo<string>  fs;	//实例化Foo<string>::Foo()
fsb.Bar();			//实例化Foo<string>::Bar()
Foo<int> fi;		//实例化Foo<int>::Foo()
fi.Bar();			//使用我们特例化版本的Foo<int>::Bar()
```

只有实例化出`Foo<int>`类的时候，才会用到我们自己特例化的Bar成员。

一言以蔽之，特例化本质上就是接管编译器的工作。