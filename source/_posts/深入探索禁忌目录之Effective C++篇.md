---
title: 深入探索禁忌目录之Effective C++篇
date: 2018-12-06 20:33:11
categories: programming-language
tags:
	- cpp


---

《Effective C++》已经很老了，我年少时便翻看过，但初时懵懂，能见者鲜，不解个中道理。时隔数年，功力与日俱增，然再看已无少年时。C++语法坑比比皆是，前赴后继的先驱者用命总结了这些使用法则，供后来人无涉险滩，后来人不解其中味，偏要以身犯险，一探究竟。

Effective系列都是好书，这些item我称之为禁忌目录。我将撸遍整个Effective系列，编写用例以深入探索。

<!--more-->

# 深入探索禁忌目录之Effective C++篇

## 条款1：尽量用const和inline而不用#define

### 改用const定义常量

对于常量的定义，应该抛弃C风格的#define预处理，而应使用const。这一点带来的好处在于编译器看不到那些被预处理代码替换掉的符号。比如：

```cpp
#define ASPECT_RATIO 1.653
```

编译器是看不到ASPECT_RATIO的，所以编译排错阶段没办法一步到位找到这个定义。另外调试起来也不明朗，因为ASPECT_RATIO不会出现在符号列表。

所以，但凡是定义常量，首选const。

```cpp
const double ASPECT_RATIO = 1.653;
```

### 类内const推荐用static

如果常量定义在类内部，最好定义成static，缘何如此？我来踩踩坑。

举个例子，在VS13上测试如下代码：

```cpp
class A
{
public:
	void printMember(){ std::cout << nonStatic << " " << NUM_TURNS << std::endl; }
private:
	const int NUM_TURNS;
	int nonStatic;
};

int main()
{
	A a;
	a.printMember();
	return 0;
}
```

编译会出现这样的错误：

```
Error	1	error C2512: 'A' : no appropriate default constructor available	f:\书\cpp\step-in-the-cpp-puddles\effective-cpp\item1\item1\source.cpp	14	1	item1
```

这是因为我们定义的类A中存在常量数据成员，编译器合成的默认构造器无法初始化NUM_TURNS，所以不予合成。

想要解决这一问题，有两种方式，第一种是自己定义默认构造器并用初始化列表初始化常量数据成员：

```cpp
class A
{
public:
	A() : NUM_TURNS(0), nonStatic(0) { }
	void printMember(){ std::cout << nonStatic << " " << NUM_TURNS << std::endl; }
private:
	const int NUM_TURNS;
	int nonStatic;
};
```

如此就可以利用初始化列表对NUM_TURNS进行初始化。

这里要注意的是定义的默认构造器必须对NUM_TURNS提供列表初始化，不能省略，也不能在构造器函数体内用赋值语句对NUM_TURNS赋值。

因为初始化和赋值是两回事，赋值的左侧对象必须是左值。

如果省略了常量数据成员的初始化，编译器会报错：

```
Error	1	error C2758: 'A::NUM_TURNS' : a member of reference type must be initialized	f:\书\cpp\step-in-the-cpp-puddles\effective-cpp\item1\item1\source.cpp	6	1	item1
	2	IntelliSense: "A::A()" provides no initializer for:
            const member "A::NUM_TURNS"	f:\书\cpp\step-in-the-cpp-puddles\effective-cpp\item1\item1\Source.cpp	6	6	item1
```

如果在函数体内对NUM_TURNS赋值，则编译器报错：

```
Error	2	error C2166: l-value specifies const object	f:\书\cpp\step-in-the-cpp-puddles\effective-cpp\item1\item1\source.cpp	6	1	item1
Error	1	error C2758: 'A::NUM_TURNS' : a member of reference type must be initialized	f:\书\cpp\step-in-the-cpp-puddles\effective-cpp\item1\item1\source.cpp	6	1	item1
	3	IntelliSense: "A::A()" provides no initializer for:
            const member "A::NUM_TURNS"	f:\书\cpp\step-in-the-cpp-puddles\effective-cpp\item1\item1\Source.cpp	6	6	item1
	4	IntelliSense: expression must be a modifiable lvalue	f:\书\cpp\step-in-the-cpp-puddles\effective-cpp\item1\item1\Source.cpp	6	8	item1
```

第二种方法就是如果编译器支持类内成员初始化，就直接使用。

```cpp
class A
{
public:
	void printMember(){ std::cout << nonStatic << " " << NUM_TURNS << std::endl; }
private:
	const int NUM_TURNS = 0;
	int nonStatic;
};
```

如此，编译器可以合成默认的构造函数，并对NUM_TURNS应用类内初始化。

> nonStatic也可以类内初始化，否则值是未定义的，我测试了VS13的处理，发现无论是将A的对象定义在栈还是堆上，nonStatic都是垃圾值。

虽然默认构造器的问题解决了，但拷贝控制成员还是有问题，合成的拷贝构造器是可行的，但赋值操作符却不行：

```cpp
class A
{
public:
	void printMember(){ std::cout << nonStatic << " " << NUM_TURNS << std::endl; }
private:
	const int NUM_TURNS = 0;
	int nonStatic = 0;
};

int main()
{
	A a, b;
	A c = a;
	b = a;		//error C2582: 'operator =' function is unavailable in 'A'	
	return 0;
}
```

因为编译器默认合成的赋值运算符函数内部实现是逐个对非static成员进行赋值操作。常量不是左值，所以不能作为赋值运算符的左侧对象，因此，我们只好自己重新定义赋值运算：

```cpp
A& operator=(const A& rhs)
{
  nonStatic = rhs.nonStatic;
  return *this;
}
```

**所以，非static常量数据成员会影响合成的默认构造器、拷贝控制成员。虽然很多时候，我们不使用合成的版本而会自己定义，但绝不应该为了常量数据成员而去定义。**

另外，定义成static的类常量成员还有两个额外好处，一是可以省空间，因为static类成员本质上是全局的，不会拷贝来拷贝去，二比较隐晦，可以把整型常量用作类内部其他数组成员的维度。

```cpp
class A
{
public:
	void printMember(){ }
private:
	 const int NUM_TURNS = 5;
	 int nonStatic = NUM_TURNS;//这是OK的
  	 int scores[NUM_TURNS];		//不行，除非NUM_TURNS是static的
};
```

关于这一点我没有去确认标准，我想应该是因为定义和初始化是分阶段的，同属定义阶段时NUM_TURNS还不能被看做是常量。但如果定义成static却又不一样了，因为本质上是全局的，不隶属于某个特定对象。

> 另外，旧的编译器可能不支持类内初始化，这时候对static成员来说，要在类外定义：
>
> ```cpp
> const int A::NUM_TURNS = 5;
> ```
>
> 但实际上这又会牵扯很多坑，对于当前来说，这一点意义已经不打了，不展开了。只要记得用类内初始化就万事大吉了。

### 使用enum来代替static常量

对于早期编译器不支持类内初始化的情况，有一种流行的规避手法：

```cpp
class A
{
public:
	void printMember(){ }
private:
	enum{ NUM_TURNS = 5 };
	int nonStatic = NUM_TURNS;
	int scores[NUM_TURNS];
};
```

### 预处理与inline

C中经常会用形如`#define max(a,b) ((a) > (b) ? (a) : (b))`这样的宏而不是定义成函数，一方面为了代码编写简洁而另一方面，又不增加函数调用的开销。

为什么要加()呢，因为怕运算符优先级问题引起歧义，这在C中已是路人皆知的技巧。

但实际上这种手法捉襟见肘，缺陷很多，比如:

```cpp
int a = 5, b = 0;
max(++a, b);	//a的值会增加2次
max(++a, b+10);	//a的值只增加了1次
```

虽然有点故意刁难的意思。

C++的inline完全规避了这种缺陷，可以改为inline函数：

```cpp
template<typename T>
inline const T& max(const T& a, const T& b){ return a > b ? a : b; }
```

inline还有一个好处就是现代的编译器都比程序员聪明，你显式声明inline实际上最终不一定是inline，而有一些即使不声明inline也会被编译器优化成inline，这是C++的一大性能优化。

## 条款2：尽量用`<iostream>`而不用`<stdio.h>`

iostream的优势在于<<和>>运算符，而类可以重载这两个运算符。scanf和printf则必须自己处理格式。

但是iostream历来被诟病，我对这东西研究还不深入，直接摘录这段文本吧：

> 第一，有些iostream的操作实现起来比相应的C stream效率要低，所以不同的选择会给你的程序有可能带来很大的不同。但请牢记，这不是对所有的iostream而言，只是一些特殊的实现；第二，在标准化的过程中，iostream库在底层做了很多修改，所以对那些要求最大可移植性的应用程序来说，会发现不同的厂商遵循标准的程度也不同。第三，iostream库的类有构造函数而<stdio.h>里的函数没有，在某些涉及到静态对象初始化顺序的时候，如果可以确认不会带来隐患，用标准C库会更简单实用。

C++11又扩充了iostream的格式化，届时再扯吧。

## 条款3：尽量用new和delete而不用malloc和free

### new/delete和malloc/free的区别

主要是语义上new和delete内部实现与malloc和free并不一致。

摘录C++ Primer 5th对new的说法：

> new表达式实际执行了三步操作：第一步，调用operator new(或operator new[])的标准库函数，分配一块足够大的、原始的、未命名的内存空间以便存储特定类型的对象（或对象的数组）。第二步，编译器运行相应的构造函数以构造这些对象，并传入初始值。第三步，对象被分配了空间并构造完成，返回一个指向该对象的指针。

而delete则相反：

> 第一步，对sp所指的对象或arr所指的数组中所有元素执行相应的析构函数。第二步，编译器调用名为operator delete(或operator delete[])的标准库函数释放内存空间。

所以，关键问题在于malloc和free对类对象来说，没有构造和析构的概念。

举这样一个例子：

```cpp
#include <iostream>

using namespace std;

class A
{
public:
	A() : member(0) { }
	A(int m) : member(m) { }
	int getMember() const { return member; }
private:
	int member;
};

int main()
{
	A *p1 = static_cast<A*>(malloc(sizeof(A)));
	A *p2 = new A(10);

	cout << sizeof(A) << endl;	// 4
	cout << p1->getMember() << endl;//-842150451
	cout << p2->getMember() << endl;//10

	return 0;
}
```

程序编译运行都没什么问题，由于p1只是指向了一个可以容纳类A对象的堆空间，但实际上并没有构造对象，所以member是个垃圾值。而p2则很正常。

> 可以在构造器中增加打印来验证。

尽管这种用法奇怪，但是好歹没出问题，但是，如果我修改一下：

```cpp
class A
{
public:
	A() : member(0) { }
	A(int m) : member(m) { }
	virtual int getMember() const { return member; }
	void setMember(int m) { member = m; }
private:
	int member;
};

int main()
{
	A *p1 = static_cast<A*>(malloc(sizeof(A)));
	A *p2 = new A(10);

	cout << sizeof(A) << endl;	// 8
	cout << p1->getMember() << endl;//error
	p1->setMember(5);	//ok
	cout << p1->getMember() << endl;//error
	cout << p2->getMember() << endl;
	p2->setMember(5);
	cout << p2->getMember() << endl;

	return 0;
}
```

将getMember改为虚函数（虽然程序设计上这个virtual没什么意义，纯粹是为了演示），那么运行期就会崩溃。

这是因为A类有虚函数，类对象就会有虚表，sizeof(A)也可以看出输出的不再是4而是8，就是因为多出了虚表的大小。而虚表是在构造器中由编译器隐式创建的，那么一个没有调用构造器的“类对象”调用由虚表索引的虚函数时，无疑会导致程序崩溃(比如access violation)。

> 但p1->setMember(5)却没有问题，因为普通函数并不由虚表来查找，语法上p1是指向A对象的指针，那么p1->setMember确实会匹配到A类的对应成员函数。

## 条款4：尽量使用C++风格的注释

//的风格C标准其实也引入了，这还是最近的一次C从C++标准中抄来的特性。。。

这种好处在于行尾注释//可以避免/**/的嵌套。

这点没什么好说的，现在的编译器已经踩不到坑了。

## 条款5：对应的new和delete要采用相同的形式

### new[]/delete[]和new/delete

C++为了兼容C，所以数组这个东西在C++的各种语义上都要特别处理（我称它游离于三界之外）。

new[]用来创建数组，delete[]用来删除。因为数组创建的写法所致往往不会错用成new，但释放时却极容易写成delete而非delete[]，这可能会导致内存泄露。

之所以说是可能，是因为C++标准对这种情况是未定义的，所以具体的处理就交由编译器。

所以，准则就是不要混用。这也意味着new也不能用delete[]来删除，结果也是未定义的。

引申的一个问题，new[]和delete[]如何知道尺寸呢？

//TODO: 先mark，回头调试一下MSVC。

### 结合typedef的一个坑

如果使用了类似:`typedef string addresslines[4];`的语法定义对象数组，那么后面一半就常常写成：

```cpp
string *pal = new addresslines;	//虽然写成new，但实际上调用的是new[]
delete pal;		//Error
delete[] pal;	//OK
```

这个坑非常容易被踩到。

所以，最好的办法就是杜绝对数组类型使用typedef。

> C++11以后还有using。

## 条款6：析构函数里对指针成员调用delete

一般在构造函数或是拷贝构造、赋值运算符函数中忘记处理类对象的指针成员，弊端会很容易显现，但一个很容易被忽略的点是析构函数要记得delete指针成员。

这是一个经典的内存泄露姿势。

由于C++标准要求可以delete空指针，所以写析构函数就不用畏首畏尾，干净利落的对指针成员进行delete即可（当然，悬垂指针处理不了）。

## 待续。。。