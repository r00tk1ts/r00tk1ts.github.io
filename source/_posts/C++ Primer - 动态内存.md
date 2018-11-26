---
title: C++ Primer - 泛型算法
date: 2018-11-26 19:50:11
categories: programming-language
tags:
	- cpp
	- cpp-primer





---

《C++ Primer》第五版引入了11标准相关内容，我早年在初学C++时还只有第四版，近来想对C++做一个整体的复习+版本升级，借此机会过一遍第五版。本文是阅读第十二章“动态内存”时所做的笔记。

<!--more-->

# C++ Primer - 动态内存

除了此前所用的静态内存和栈内存，程序还可以使用堆内存。程序用堆（heap）来存储动态分配（dynamically allocate）的对象。动态对象的生存期由程序控制，即我们需要显式的申请和销毁它们。

## 动态内存与智能指针

C++中的动态内存管理通过一对运算符完成：`new`在动态内存中为对象分配空间并返回指向该对象的指针，可以选择对对象进行初始化；`delete`接受一个动态对象的指针，销毁该对象并释放与之关联的内存。

如果忘记释放内存，就会产生内存泄露。而如果在尚有指针引用内存的情况下释放内存，可能产生野指针的使用。

为了安全，标准库提供了两种智能指针(smart pointer)类型来管理动态对象。智能指针的行为类似常规指针，区别在于它负责自动释放所指向的对象。新标准库提供的这两种智能指针的区别在于管理底层指针的方式：shared_ptr允许多个指针指向同一个对象；unique_ptr则独占所指向的对象。标准库还定义了名为weak_ptr的伴随类，它是一种弱引用，指向shared_ptr所管理的对象。三者均在memory头文件中定义。

### shared_ptr

也是类模板。

```cpp
shared_ptr<string> p1;	//shared_ptr，可以指向string
shared_ptr<list<int>> p2;	//shared_ptr, 可以指向int的list
```

默认初始化的智能指针保存空指针。

使用方式类似普通指针，通过解引用返回对象。

| shared_ptr与unique_ptr都支持的操作 |                                          |
| --------------------------- | ---------------------------------------- |
| `shared_ptr<T> sp`          | 空智能指针，可以指向类型为T的对象                        |
| `unique_ptr<T> up`          |                                          |
| p                           | 将p用作一个条件判断，若p指向一个对象，则为true               |
| *p                          | 解引用p，获得它指向的对象                            |
| p->mem                      | 等价于(*p).mem                              |
| p.get()                     | 返回p中保存的指针。要小心使用，若智能指针释放了其对象，返回的指针所指向的对象也就消失了 |
| swap(p, q)                  | 交换p和q中的指针                                |
| p.swap(q)                   |                                          |

| shared_ptr独有的操作        |                                          |
| ---------------------- | ---------------------------------------- |
| `make_shared<T>(args)` | 返回一个shared_ptr，指向一个动态分配的类型为T的对象。使用args初始化此对象 |
| `shared_ptr<T> p(q)`   | p是shared_ptr q的拷贝；此操作会递增q中的计数器。q中的指针必须能转换为T* |
| p = q                  | p和q都是shared_ptr，所保存的指针必须能相互转换。此操作会递减p的引用计数，递增q的引用计数；若p的引用计数变为0，则将其管理的原内存释放 |
| p.unique()             | 若p.use_count()为1，返回true；否则返回false        |
| p.use_count()          | 返回与p共享对象的智能指针数量；可能很慢，主要用于调试              |

make_shared标准库函数是分配使用动态内存最安全的方式。定义在memory头文件中。

```cpp
shared_ptr<int> p3 = make_shared<int>(42);//42
shared_ptr<string> p4 = make_shared<string>(10, '9');//"9999999999"
shared_ptr<int> p5 = make_shared<int>();//值初始化，即为0
auto p6 = make_shared<vector<string>>();//更通用的定义
```

### shared_ptr的拷贝和赋值

拷贝或赋值时，每个shared_ptr都会记录有多少个其他shared_ptr指向相同的对象：

```cpp
auto p = make_shared<int>(42);	//p指向的对象只有p一个引用者
auto q(p);	//p和q指向相同对象，此对象有两个引用者
```

每个shared_ptr都有一个引用计数。拷贝shared_ptr会递增计数器。用一个shared_ptr初始化另一个shared_ptr，或作为参数传递给一个函数或作为函数的返回值时，引用计数都会递增。而shared_ptr赋予其他值时或是shared_ptr被销毁时（比如局部的shared_ptr离开其作用域），计数器会递减。

一旦shared_ptr计数器变为0，它会自动释放自己管理的对象。

```cpp
auto r = make_shared<int>(42);	//r指向的int只有一个引用者
r = q;	//给r赋值，令它指向另一个地址
		//递增q指向的对象的引用计数
		//递减r原本指向对象的引用计数
		//r原本指向对象的引用计数变为0，自动释放
```

shared_ptr类的自动销毁对象是通过其析构函数完成的。析构函数会递减它所指向对象的引用计数，如果引用计数变为0，shared_ptr的析构函数会销毁对象并释放空间。

如果将shared_ptr存放于容器中，一段时间过后不需要全部元素，而只使用其中一部分，应该用erase删除不再需要的元素，如此才能得以释放内存（如果没有其他的引用的话）。

程序使用动态内存通常出于以下三种原因之一：

- 不确定需要使用多少对象。
- 不确定所需对象的准确类型。
- 需要在多个对象间共享数据。

### 直接管理内存

直接使用new和delete是C++的一把双刃剑。

默认情况下，动态分配的对象是默认初始化的。所以内置类型或组合类型的对象的值将是未定义的，而类类型对象将用默认构造函数进行初始化。

```cpp
string *ps = new string;	//初始化为空字符串
int *pi = new int;	//pi指向未初始化int
```

可以使用值初始化方式、直接初始化方式、传统构造方式（圆括号`()`）或新标准下的列表初始化方式（花括号`{}`）初始化动态分配的对象。

```cpp
int *pi = new int(1024);	
string *ps = new string(10, '9');
vector<int> *pv = new vector<int>{0,1,2,3,4,5,6,7,8,9};
string *ps1 = new string;	//默认初始化，空串
string *ps = new string();	//值初始化，空串
int *pi1 = new int;		//默认初始化，值未定义
int *pi2 = new int();	//值初始化为0
```

对定义了构造函数的类类型来说，无论是值初始化还是默认初始化，都会调用默认构造函数。而内置类型则不同，值初始化有着良好定义的值，默认初始化则未定义。

可以用new分配const对象，返回指向const类型的指针。动态分配的const对象必须初始化。

如果动态内存被耗尽，new表达式就会失败，默认情况下new失败时会抛出类型为bad_alloc的异常。我们可以改变使用new的方式来阻止它抛异常：

```cpp
//如果分配失败，new返回一个空指针
int *p1 = new int;	//分配失败，则new抛出std::bad_alloc
int *p2 = new (nothrow) int;	//如果分配失败，则new返回一个空指针
```

后者的new形式称为定位new。定位new表达式可以传递额外的参数，这里传递了一个标准库定义的名为nothrow的对象，意为不要抛出异常。

> nothrow和bad_alloc定义在new头文件中。

释放内存通过delete表达式。

```cpp
delete p;	//p必须指向一个动态分配的对象或是一个空指针
```

释放并非new分配的内存，或者将相同的指针值释放多次，行为是未定义的：

```cpp
int i, *pi1 = &i, *pi2 = nullptr;
double *pd = new double(33), *pd2 = pd;
delete i;	//错误：i不是指针
delete pi1;	//未定义：pi1指向一个局部变量
delete pd;	//正确
delete pd2;	//未定义：pd2指向的内存已经被释放了
delete pi2;	//正确：释放空指针总是没有错误的
```

new出来的const对象也可以delete，尽管对象本身不能改变。

### shared_ptr和new的联用

new返回的指针可以初始化智能指针，该构造函数是explicit的，因此必须使用直接初始化形式：

```cpp
shared_ptr<int> p1 = new int(1024);	//错误：必须使用直接初始化
shared_ptr<int> p2(new int(1024));	//正确：使用了直接初始化
```

默认情况下，用来初始化智能指针的内置指针必须指向动态内存，因为智能指针默认使用delete释放它所管理的对象。如果要将智能指针绑定到一个指向其他类型资源的指针上，就必须提供自定义操作来代替delete。

| 定义和改变shared_ptr的其他方法     |                                          |
| ------------------------ | ---------------------------------------- |
| `shared_ptr<T> p(q)`     | p管理内置指针q所指向的对象；q必须指向new分配的内存，且能够转换为T*类型  |
| `shared_ptr<T> p(u)`     | p从unique_ptr u那里接管了对象的所有权；将u置为空          |
| `shared_ptr<T> p(q, d)`  | p接管了内置指针q所指向的对象的所有权。q必须能转为T*类型。p将使用可调用对象d来代替delete |
| `shared_ptr<T> p(p2, d)` | p是shared_ptr p2的拷贝，唯一的区别是p将用可调用对象d来代替delete |
| p.reset()                | 若p是唯一指向其对象的shared_ptr，reset会释放此对象。       |
| p.reset(q)               | 若传递了可选的参数内置指针q，会令p指向q，否则会将p置空。           |
| p.reset(q, d)            | 若还传递了参数d，将会调用d而不是delete来释放q              |

不要混合使用内置指针和智能指针。当将shared_ptr绑定到内置指针后，资源管理就应该交由shared_ptr负责。不应该再使用内置指针访问shared_ptr指向的内存。

```cpp
// 函数被调用时ptr被创建并初始化
void process(shared_ptr<int> ptr)
{
    // 使用ptr
}   // ptr离开作用域，被销毁

int *x(new int(1024));   // 危险：x是一个普通指针，不是智能指针
process(x);     // 错误：无法转换 int* 到 shared_ptr<int>
process(shared_ptr<int>(x));    // 合法，但是内存会被释放
int j = *x;     // 未定义的：x是悬垂指针

shared_ptr<int> p(new int(42));   // 引用计数为1
process(p);     // 拷贝p会增加它的引用计数，process中引用计数为2
int i = *p;     // 正确：引用计数为1
```

智能指针的`get`函数返回一个内置指针，指向智能指针管理的对象。。主要用于向不能使用智能指针的代码传递内置指针。使用get返回指针的代码不能delete此指针。

```cpp
shared_ptr<int> p(new int(42));	//引用计数为1
int *q = p.get();	//正确：但使用q要注意，不要让它管理的指针被释放
{	//新程序块
    //未定义：两个独立的shared_ptr指向相同的内存
  	shared_ptr<int>(q);
}	//程序块结束，q被销毁，它指向的内存被释放
int foo = *p;	//未定义：p指向的内存已经被释放了
```

花式作死的另一种用法。

> 永远不要用get初始化另一个智能指针或为另一个智能指针赋值。

reset可以将新指针赋予shared_ptr：

```cpp
p = new int(1024);	//错误：不能将一个指针赋予shared_ptr
p.reset(new int(1024));	//正确：p指向一个新对象
```

与赋值类似，reset会更新引用计数，如果需要的话，会释放p指向的对象。reset成员经常与unique一起用，来控制多个shared_ptr共享的对象。

```cpp
if(!p.unique())
  	p.reset(new string(*p));	//不是唯一用户；分配新的拷贝
*p += newVal;	//现在我们知道自己是唯一的用户，可以改变对象的值
```

### 智能指针和异常

如果使用智能指针，即使程序块过早结束，智能指针类也能确保在内存不再需要时将其释放。

```cpp
void f()
{
    int *ip = new int(42);
  	//这段代码抛出异常，且在f中未被捕获
  	delete ip;	//退出之前释放内存
}
```

标准的内存泄露。

而如果:

```cpp
void f()
{
    shared_ptr<int> sp(new int(42));
  	//这段代码抛出异常，且在f中未捕获
}//函数结束时shared_ptr自动释放内存
```

默认情况下shared_ptr假定其指向动态内存，使用delete释放对象。创建shared_ptr时可以传递一个（可选）指向删除函数的指针参数，用来代替delete。这个删除器(deleter)函数必须能够完成对shared_ptr中保存的指针进行释放的操作。

```cpp
struct destination;  
struct connection;     
connection connect(destination*);   
void disconnect(connection);   
void end_connection(connection *p)
{
    disconnect(*p);
}

void f(destination &d /* 其他参数 */)
{
    connection c = connect(&d);
    shared_ptr<connection> p(&c, end_connection);
    // 使用连接
    // f退出时（即使是异常退出），connection会被正确关闭
}
```

**智能指针规范**：

- 不使用相同的内置指针值初始化或reset多个智能指针。
- 不释放get返回的指针。
- 不使用get初始化或reset另一个智能指针。
- 使用get返回的指针时，如果最后一个对应的智能指针被销毁，指针就无效了。
- 使用shared_ptr管理并非new分配的资源时，应该传递删除函数。

### unique_ptr

与shared_ptr不同，同一时刻只能有一个unique_ptr指向给定的对象。当unique_ptr被销毁时，它指向的对象也会被销毁。

`make_unique`函数（C++14新增，定义在头文件*memory*中）在动态内存中分配一个对象并初始化它，返回指向此对象的unique_ptr。

```cpp
unique_ptr<int> p1(new int(42));
// C++14
unique_ptr<int> p2 = make_unique<int>(42);
```

由于unique_ptr独占其指向的对象，因此unique_ptr不支持普通的拷贝或赋值操作。

| unique_ptr操作            |                                          |
| ----------------------- | ---------------------------------------- |
| `unique_ptr<T> u1`      | 空unique_ptr，可以指向类型为T的对象。u1会使用delete来释放它的指针；u2会使用一个类型为D的可调用对象来释放它的指针 |
| `unique_ptr<T, D> u2`   |                                          |
| `unique_ptr<T, D> u(d)` | 空unique_ptr，指向类型为T的对象，用类型为D的对象d代替delete  |
| u = nullptr             | 释放u指向的对象，将u置空                            |
| u.release()             | u放弃对指针的控制权，返回指针，并将u置空                    |
| u.reset()               | 释放u指向的对象                                 |
| u.reset(q)              | 如果提供了内置指针q，令u指向这个对象，否则u置空                |
| u.reset(nullptr)        |                                          |

unique_ptr不能拷贝或赋值，但可以转移：

```cpp
//将所有权从p1转移给p2
unique_ptr<string> p2(p1.release());	//release将p1置空
unique_ptr<string> p3(new string("Trex"));	
//将所有权从p3转移给p2
p2.reset(p3.release());	//reset释放了p2原来指向的内存

p2.release();	//错误：p2不会释放内存，而且我们弄丢了指针
auto p = p2.release();	//正确，但我们必须记得delete(p)
```

不能拷贝unique_ptr的规则有一个例外：我们可以拷贝或赋值一个将要被销毁的unique_ptr。

```cpp
unique_ptr<int> clone(int p){
    //正确：从int*创建一个unique_ptr<int>
  	return unique_ptr<int>(new int(p));
}
```

还可以返回一个局部对象的拷贝：

```cpp
unique_ptr<int> clone(int p){
    unique_ptr<int> ret(new int(p));
  	//...
  	return ret;
}
```

类似shared_ptr，默认情况下unique_ptr用delete释放其指向的对象。unique_ptr的删除器同样可以重载，但unique_ptr管理删除器的方式与shared_ptr不同。定义unique_ptr时必须在尖括号中提供删除器类型。创建或reset这种unique_ptr类型的对象时，必须提供一个指定类型的可调用对象（删除器）。

```cpp
//p指向一个类型为objT的对象，并使用一个类型为delT的对象释放objT对象
//它会调用一个名为fcn的delT类型对象
unique_ptr<objT, delT> p(new objT, fcn);

void f(destination &d/* 其他参数 */)
{
    connection c = connect(&d);
  	unique_ptr<connection, decltype(end_connection)*> p(&c, end_connection);
}
```

### weak_ptr

weak_ptr是一种不控制所指向对象生存期的智能指针，它指向一个由shared_ptr管理的对象。将weak_ptr绑定到shared_ptr不会改变shared_ptr的引用计数。如果shared_ptr被销毁，即使有weak_ptr指向对象，对象仍然有可能被释放。

| weak_ptr            |                                          |
| ------------------- | ---------------------------------------- |
| `weak_ptr<T> w`     | 空weak_ptr可以指向类型为T的对象                     |
| `weak_ptr<T> w(sp)` | 与shared_ptr sp指向相同对象的weak_ptr。T必须能转换为sp指向的类型 |
| w = p               | p可以是一个shared_ptr或一个weak_ptr。赋值后w与p共享对象   |
| w.reset()           | w置空                                      |
| w.use_count()       | 与w共享对象的shared_ptr的数量                     |
| w.expired()         | 若w.use_count()为0，返回true，否则返回false        |
| w.lock()            | 如果expired为true，返回空shared_ptr；否则返回一个指向w的对象的shared_ptr |

创建一个weak_ptr时，需要使用shared_ptr来初始化它。

```cpp
auto p = make_shared<int>(42);
weak_ptr<int> wp(p);    // wp弱共享p，p引用计数不变
```

由于对象可能不存在，所以weak_ptr访问对象前，要先lock：

```cpp
if(shared_ptr<int> np = wp.lock()){//np不为空则条件成立
    ...
}
```

## 动态数组

### new和数组

数组是个很特别的存在，对于数组的动态分配和释放C++定义了相应的手法。

#### new []

```cpp
int *pia = new int[get_size()];	// pia指向第一个int，调用get_size确定分配多少个int
```

方括号中的大小必须是整型，但不一定非要常量。

```cpp
typedef int arrT[42];
int *p = new arrT;	//实际上还是用的new[]而非new
```

new返回的是一个元素类型的指针，指向第一个分配的成员。

C++中，**动态数组不是数组类型**，所以begin或end是不可以使用的，也不能用范围for语句来处理动态数组。

默认情况下，new分配的对象是默认初始化的。可以对数组中的元素进行值初始化，方法是在大小后面跟一对空括号`()`。在新标准中，还可以提供一个元素初始化器的花括号列表。如果初始化器数量大于元素数量，则new表达式失败，不会分配任何内存，并抛出bad_array_new_length异常。

```cpp
int *pia = new int[10];     // 10个未初始化的int
int *pia2 = new int[10]();    // 10个值初始化为的int
string *psa = new string[10];    // 10个空string
string *psa2 = new string[10]();    // 10个空string
//10个int分别用列表中对应的初始化器初始化
int *pia3 = new int[10] { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9 };
//10个string，前4个用给定的初始化器初始化，剩余的进行值初始化
string *psa3 = new string[10] { "a", "an", "the", string(3,'x') };
```

虽然可以使用空括号对new分配的数组元素进行值初始化，但不能在括号中指定初始化器。这意味着不能用auto分配数组。

动态分配一个空数组是合法的，此时new会返回一个合法的非空指针。对于零长度的数组来说，该指针类似尾后指针，不能解引用。

#### delete []

对应`new[]`，使用`delete[]`释放动态数组。

```cpp
delete p;	//p必须指向一个动态分配的对象或空
delete [] pa;	//pa必须指向动态分配的数组或空
```

数组中元素按逆序销毁。如果忽略了方括号，那么行为未定义。

```cpp
typedef int arrT[42];
int *p = new arrT;
delete [] p;	//这个方括号是必须的。
```

#### 智能指针和动态数组

unique_ptr可以直接管理动态数组，定义时需要在对象类型后添加一对空方括号`[]`。

```cpp
unique_ptr<int[]> up(new int[10]);
up.release();	//自动使用delete []来销毁指针
```

指向数组的unique_ptr的操作有些不同：

| 指向数组的unique_ptr        |                                 |
| ---------------------- | ------------------------------- |
| `unique_ptr<T[]> u`    | u可以指向一个动态分配的数组，数组元素类型为T         |
| `unique_ptr<T[]> u(p)` | u指向内置指针p所指向的动态分配的数组。p必须能转换为类型T* |
| u[i]                   | 返回u拥有的数组中i处的对象，u必须指向一个数组        |

指向数组的unique_ptr不支持成员访问运算符(.和->)。

shared_ptr不直接支持管理动态数组，如果希望使用shared_ptr管理动态数组，必须提供自定义的删除器：

```cpp
shared_ptr<int> sp(new int[10], [](int *p){delete[] p;});
sp.reset();	//使用我们提供的lambda释放数组，它使用delete []
```

如果未提供删除器，则代码是未定义的。

shared_ptr不支持下标运算符，且不支持指针的算数运算，所以需要借助get。

```cpp
for(size_t i=0;i!=10;++i){
    *(sp.get() + i) = i;	//使用get获取一个内置指针
}
```

### allocator类

allocator类也是一个类模板，定义时必须指定其分配的对象类型。

```cpp
allocator<string> alloc;    // 可以分配string的allocator对象
auto const p = alloc.allocate(n);   // 分配n个未初始化的string
```

| 标准库allocator类及其算法      |                                          |
| ---------------------- | ---------------------------------------- |
| `allocator<T> a`       | 定义了一个名为a的allocator对象，它可以为类型为T的对象分配内存     |
| `a.allocate(n)`        | 分配一段原始的、未构造的内存，保存n个类型为T的对象               |
| `a.deallocate(p, n)`   | 释放从T*指针p中地址开始的内存，这块内存保存了n个类型为T的对象；p必须是一个先前由allocate返回的指针，且n必须是p创建时所要求的大小。调用deallocate前，用户必须对每个在这块内存中创建的对象调用destroy |
| `a.construct(p, args)` | p必须是一个类型为T*的指针，指向一块原始内存；arg被传递给类型为T的构造函数，用来在p指向的内存中构造一个对象 |
| `a.destroy(p)`         | p为T*类型的指针，此算法对p指向的对象执行析构函数               |

allocator分配的内存是未构造的，程序需要在此内存中构造对象。新标准库的`construct`函数接受一个指针和零或多个额外参数，在给定位置构造一个元素。额外参数用来初始化构造的对象，必须与对象类型相匹配。

```cpp
auto q = p;     // q指向最后构造的元素之后的位置
alloc.construct(q++);    // *q为空字符串
alloc.construct(q++, 10, 'c');  // *q为cccccccccc
alloc.construct(q++, "hi");     // *q为hi
```

在未构造对象前使用原始内存是错误的：

```cpp
cout << *p << endl;	//正确：使用string的输出运算符
cout << *q << endl;	//灾难：q指向未构造的内存！
```

用完对象后，需要对每个构造的元素调用destroy来销毁它们。函数destroy接受一个指针，对指向的对象执行析构函数：

```cpp
while(q != p)
  	alloc.destroy(--q);	//释放我们真正构造的string
```

注意只能对已构造的对象进行destroy。

destroy后的内存可以重用。

全部destroy后，可以调用deallocate来归还内存给系统。

```cpp
alloc.deallocate(p, n);
```

标准库还为allocator类定义了两个伴随算法，可以在未初始化内存中创建对象。

| allocator算法                  |                                          |
| ---------------------------- | ---------------------------------------- |
| uninitialized_copy(b,e,b2)   | 从迭代器b和e指定的输入范围中拷贝元素到迭代器b2指定的未构造的原始内存中。b2指向的内存必须足够大。 |
| uninitialized_copy_n(b,n,b2) | 从迭代器b指向的元素开始，拷贝n个元素到b2开始的内存中             |
| uninitialized_fill(b,e,t)    | 在迭代器b和e指定的原始内存范围中创建对象，对象的值均为t的拷贝         |
| uninitialized_fill_n(b,n,t)  | 从迭代器b指向的内存地址开始创建n个对象。b必须指向足够大的未构造的原始内存   |

这些函数在给定目的位置创建元素，而不是由系统分配内存给它们。它们在memory头文件中。

```cpp
auto p = alloc.allocate(vi.size() * 2);
auto q = uninitialized_copy(vi.begin(), vi.end(), p);//返回递增后的目的位置迭代器。
uninitialized_fill_n(q, vi.size(), 42);
```