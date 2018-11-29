---
title: C++ Primer - 重载运算与类型转换
date: 2018-11-29 22:26:11
categories: programming-language
tags:
	- cpp
	- cpp-primer


---

《C++ Primer》第五版引入了11标准相关内容，我早年在初学C++时还只有第四版，近来想对C++做一个整体的复习+版本升级，借此机会过一遍第五版。本文是阅读第十四章“重载运算与类型转换”时所做的笔记。

<!--more-->

# C++ Primer - 重载运算与类型转换

## 基本概念

重载的运算符是具有特殊名字的函数（关键字operator接要定义的运算符号）。

既然是函数，所以也有返回类型、参数列表和函数体。其中参数数量与该运算符作用的运算对象一样多。一元运算符有一个参数，二元运算符有两个（唯一的一个三元运算符不能重载）。对二元运算符来说，左侧运算对象传递给第一个参数，而右侧运算对象传递给第二个参数。除了重载的函数调用运算符`operator()`之外，其他重载运算符不能含有默认实参。

> 有的运算符既可以当一元也可以当二元，这个时候要根据重载参数个数来判断语义。

如果一个运算符函数是成员函数，则它的第一个（左侧）运算对象绑定到隐式的this指针上，因此，成员运算符函数（显式）的参数数量比运算符的运算对象总是少一个。

运算符函数要么是类的成员，要么至少含有一个类类型参数，这就意味着无法对内置类型的运算对象进行运算符重载。

> 比如`int operator+(int, int);`就是错误的语法，不能改变内置类型的运算符行为。

只能重载已有的运算符，不能搞新发明。不是所有的运算符都能重载。

| 可重载运算符      |      |      |       |        |          |
| ----------- | ---- | ---- | ----- | ------ | -------- |
| +           | -    | *    | /     | %      | ^        |
| &           | \|   | ~    | !     | ,      | =        |
| <           | >    | <=   | >=    | ++     | --       |
| <<          | >>   | ==   | !=    | &&     | \|\|     |
| +=          | -=   | /=   | %=    | ^=     | &=       |
| \|=         | *=   | <<=  | >>=   | []     | ()       |
| ->          | ->*  | new  | new[] | delete | delete[] |
| **不能重载运算符** |      |      |       |        |          |
| ::          | .*   | .    | ? :   |        |          |

运算符函数一般通过间接调用，当然也可以直接调用。

```cpp
data1 + data2;
operator+(data1, data2);	

data1 += data2;
data1.operator+=(data2);
```

尽管有些运算符可以重载，但大多数情况下不建议重载，它们是`, & || &&`。

运算符重载虽然可以为所欲为，但最好让他们的含义与内置类型一致，不要违直觉定义。

- 如果类执行IO操作，那么<<和>>就应该与内置类型的IO一致。
- 如果类的某个操作是检查相等性，则定义operator==，通常也应该有operator!=。
- 如果类包含一个内在的单序比较操作，则定义operator<，如果有了<，一般也有其他关系操作。
- 重载运算符的返回类型通常应与内置版本的返回类型兼容：逻辑运算符和关系运算符应该返回bool，算术运算符应该返回类类型的值，赋值运算符和复合赋值运算符应该返回左侧运算对象的一个引用。

定义重载运算符时，必须要先决定是声明为类成员函数还是普通的非成员函数。对此，有一些准则：

- 赋值、下标、调用和成员访问箭头运算符必须是成员。
- 复合赋值运算符一般来说应该是成员，但并非必须。
- 改变对象状态的运算符或者与给定类型密切相关的运算符，如递增、递减和解引用运算符，应该是成员。
- 具有对称性的运算符可能转换任意一端的运算对象，例如算术、相等性、关系和位运算符等，它们应该是普通的非成员函数（成员函数则会引发`string u = "hi" + s;`错误的灾难）。
- 输入输出运算符必须是非成员函数。


## 输入和输出运算符

### 重载<<

通常输出运算符的第一个形参是一个非常量ostream对象引用。非常量是因为向流写入内容会改变其状态，而引用则是因为ostream不能拷贝。第二个形参一般是一个常量的引用，该常量是我们想要打印的类类型。这里的引用是因为我们希望避免复制实参，常量则是因为打印对象通常不会改变对象内容。

为了与其他<<一致，operator<<一般返回它的ostream形参。

```cpp
ostream &Sales_data::operator<<(ostream &os, const Sales_data &item)
{
    os << item.isbn() << " " << item.units_sold << " " << item.revenue << " " << item.avg_price();
  	return os;
}
```

IO运算符往往需要读写类的非公有数据成员，所以IO运算符一般被声明为友元。

### 重载>>

```cpp
istream &operator>>(istream &is, Sales_data &item)
{
    double price;
  	is >> item.bookNo >> item.units_sold >> price;
  	if(is)	//检查输入是否成功
      	item.revenue = item.units_sold * price;
  	else
      	item = Sales_data();	//输入失败，对象被赋予默认的状态
  	return is;
}
```

和<<类似，但>>要额外考虑失败的情况。

> 流含有错误类型的数据读取操作，或是到达文件末尾或遇到输入流的其他错误时会失败。

## 算术和关系运算符

### 重载==、!=

```cpp
bool operator==(const Sales_data &lhs, const Sales_data &rhs)
{
    return lhs.isbn() == rhs.isbn() && lhs.units_sold == rhs.units_sold && lhs.revenue == rhs.revenue;
}

bool operator!=(const Sales_data &lhs, const Sales_data &rhs)
{
    return !(lhs == rhs);
}
```

### 重载关系运算符

定义了相等运算符的类一般也包含关系运算符。特别的，关联容器需要用到小于运算符，所以定义operator<很有用。

通常情况下，关系运算符应该：

1. 定义顺序关系，令其与关联容器中对关键字的要求一致，并且
2. 如果类同时含有==运算符的话，则定义一种关系令其与==保持一致。特别是，如果两个对象是!=的，那么一个对象应该<另一个。

对Sales_data类来说，关系运算符没有什么必要，因为语义上违直觉。

## 赋值运算符

除了拷贝赋值和移动赋值运算符以外，类还可以定义其他赋值运算符，把别的类型作为右侧运算对象。

比如vector支持操作：

```cpp
vector<string> v;
v = {"a", "an", "the"};
```

之所以可以这样赋值，是因为vector类似这样重载了=运算符：

```cpp
class StrVec
{
public:
  	StrVec &operator=(std::initializer_list<std::string>);
}
```

复合赋值运算符虽然不一定非要是类成员，但语义上作为类成员函数更符合直觉。

## 下标运算符

如果一个类包含下标运算符，则通常会定义两个版本：一个返回普通引用，另一个是类的常量成员并且返回常量引用。

```cpp
class StrVec{
public:
  	std::string& operator[](std::size_t n)
    {
        return elements[n];
    }
  	const std::string& operator[](std::size_t n) const
    {
        return elements[n];
    }
private:
  	std::string *elements;	//指向数组首元素的指针
}；
```

常量对象取下标会匹配调用const版本。

## 递增和递减

这个比较特别，有前置版本和后置版本，所以也要定义两个版本。语义上建议作为成员函数。

### 前置

```cpp
class StrBlobPtr{
public:
  	StrBlobPtr& operator++();	//前置
  	strBlobPtr& operator--();
};

StrBlobPtr& StrBlobPtr::operator++()
{
    check(curr, "increment past end of StrBlobPtr");
  	++curr;
  	return *this;
}
StrBlobPtr& StrBlobPtr::operator--()
{
  	--curr;
    check(curr, "decrement past bgin of StrBlobPtr");
  	return *this;
}
```

前置运算符返回的是递增或递减后的对象引用。

### 后置

为了区分前置和后置，后置接受一个额外的不被使用的int型形参。

```cpp
class StrBlobPtr{
public:
  	StrBlobPtr operator++(int);	//后置
  	StrBlobPtr operator--(int);	
};
StrBlobPtr StrBlobPtr::operator++(int)
{
    StrBlobPtr ret = *this;
  	++*this;
  	return ret;
}
StrBlobPtr StrBlobPtr::operator--(int)
{
  	StrBlobPtr ret = *this;
    --*this;
  	return ret;
}
```

## 重载*/->

```cpp
class StrBlobPtr{
public:
  	std::string& operator*() const
    {
        auto p = check(curr, "dereference past end");
      	return (*p)[curr];	//(*p)是对象所指的vector
    }
  	std::string* operator->() const
    {
        return & this->operator*();
    }
}
```

## 重载()

最特别的一个。

```cpp
struct absInt{
    int operator()(int val) const{
        return val < 0 ? -val : val;
    }
};

int i = -42;
absInt absObj;
int ui = absObj(i);	//i传递给absObj.operator()
```

只能作为类成员定义，可以重载多个函数，以参数区分。

**类如果定义了调用运算符，那么该类的对象就被称为函数对象。**

lambda会被编译器翻译成一个未命名类的未命名对象。lambda表达式产生的类中含有一个重载的函数调用运算符，所以lambda表达式实际上是函数对象。

标准库也定义了一组函数对象，plus类定义了一个函数调用运算符用于对一对运算对象执行+操作，modules类定义了调用运算符执行二元%操作，equal_to类执行==等。

这些类都是类模板，需要使用时指定具体应用类型。

```cpp
plus<int> intAdd;
negate<int> intNegate;
int sum = intAdd(10, 20);
sum = intNegate(intAdd(10,20));
sum = intAdd(10, intNegate(10));
```

它们定义在functional头文件中。

| 算术                 | 关系                    | 逻辑                  |
| ------------------ | --------------------- | ------------------- |
| `plus<Type>`       | `equal_to<Type>`      | `logical_and<Type>` |
| `minus<Type>`      | `not_equal_to<Type>`  | `logical_or<Type>`  |
| `multiplies<Type>` | `greater<Type>`       | `logical_not<Type>` |
| `divides<Type>`    | `greater_equal<Type>` |                     |
| `modules<Type>`    | `less<Type>`          |                     |
| `negate<Type>`     | `less_equal<Type>`    |                     |

函数对象的一个坑：

```cpp
vector<string *> nameTable;
sort(nameTable.begin(), nameTable.end(), [](string *a, string *b){return a < b;});//错误，nameTable中指针彼此之间没有关系，所以<将产生未定义行为
sort(nameTable.begin(), nameTable.end(), less<string*>());//正确，标准库规定指针的less是定义良好的
```

后者可以用指针地址值来排序，标准库规定其函数对象对于指针同样适用，而手写的lambda就不行了。

C++语言中几种可调用的对象：函数、函数指针、lambda表达式、bind创建的对象以及重载了函数调用运算符的类。调用形式指明了调用返回的类型以及传递给调用的实参类型。不同的可调用对象可能具有相同的调用形式。

标准库`function`类型是一个模板，定义在头文件*functional*中，用来表示对象的调用形式。

| function的操作               |                                          |
| ------------------------- | ---------------------------------------- |
| `function<T> f;`          | f是一个用来存储可调用对象的空function，这些可调用对象的调用形式应该与函数类型T相同（即T是`retType(args)`） |
| `function<T> f(nullptr);` | 显式地构造一个空function                         |
| `function<T> f(obj);`     | 在f中存储可调用对象obj的副本                         |
| `f`                       | 将f作为条件：当f含有一个可调用对象时为真，否则为假               |
| `f(args)`                 | 调用f中的对象，参数是args                          |
| 定义为`function<T>`的成员的类型    |                                          |
| result_type               | 该function类型的可调用对象返回的类型                   |
| argument_type             | T有一个或两个实参时定义的类型。如果T只有一个实参，               |
| first_argument_type       | 则argument_type是该类型的同义词；如果T有两个实参，         |
| second_argument_type      | 则first_argument_type和second_argument_type分别代表两个实参的类型 |

```cpp
function<int(int,int)> f1 = add;		//函数指针
function<int(int,int)> f2 = devide();	//函数对象类的对象
function<int(int,int)> f3 = [](int i, int j){return i * j;};	//lambda

cout << f1(4,2) << endl;	//6
cout << f2(4,2) << endl;	//2
cout << f3(4,2) << endl;	//8
```

## 重载、类型转换与运算符

转换构造函数和类型转换运算符共同定义了类类型转换（class-type conversion）。

### 类型转换运算符

这是类的一种特殊成员函数，负责将一个类类型的值转为其他类型。它不能声明返回类型，形参列表也必须为空，形式如下：

```cpp
operator type() const;
```

类型转换运算符可以面向除了void以外的任意类型进行定义。

```cpp
class SmallInt
{
public:
    SmallInt(int i = 0): val(i)
    {
        if (i < 0 || i > 255)
            throw std::out_of_range("Bad SmallInt value");
    }   
    operator int() const { return val; }
    
private:
    std::size_t val;
};

// 内置类型转换将double实参转换成int
SmallInt si = 3.14;     // 调用SmallInt(int)构造函数
// SmallInt类型转换运算符将si转换成int
si + 3.14;     // 内置类型转换将所得的int继续转换成double
```

应该避免过度使用类型转换函数。如果在类类型和转换类型之间不存在明显的映射关系，则这样的类型转换可能具有误导性。

C++11引入了显示的类型转换运算符（explicit conversion operator）。和显式构造函数一样，编译器通常不会将显式类型转换运算符用于隐式类型转换。

一旦给了类型转换运算符explicit标志，那么：

```cpp
SmallInt si = 3;	//正确，SmallInt的构造函数不是显式的
si + 3;				//错误：此处需要隐式的类型转换，但类型转换运算符是显式的
static_cast<int>(si)+3;	//正确，显式地请求类型转换
```

如果表达式被用作条件，则编译器会隐式地执行显式类型转换。

- if、while、do语句的条件部分。
- for语句头的条件表达式。
- 条件运算符`? :`的条件表达式。
- 逻辑非运算符`!`、逻辑或运算符`||`、逻辑与运算符`&&`的运算对象。

在两种情况下可能产生多重转换路径：

- A类定义了一个接受B类对象的转换构造函数，同时B类定义了一个转换目标是A类的类型转换运算符。
- 类定义了多个类型转换规则，而这些转换涉及的类型本身可以通过其他类型转换联系在一起。

可以通过显式调用类型转换运算符或转换构造函数解决二义性问题，但不能使用强制类型转换，因为强制类型转换本身也存在二义性。

**所以，请避免有二义性的类型转换。**