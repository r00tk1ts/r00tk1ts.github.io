---
title: 深入探索禁忌目录之Effective C++篇
date: 2018-12-16 14:02:11
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

## 条款7：预先准备好内存不够的情况

头文件`<new>`里提供了new-handler函数机制，当operator new不能满足请求时，会在抛出异常之前调用程序员指定的出错处理函数。

```cpp
typedef void (*new_handler)();
new_handler set_new_handler(new_handler p) throw();
```

set_new_handler用来设置new_handler类型的函数。

operator new不能满足内存分配请求时，new-handler函数不只调用一次，而是不断重复，直至找到足够的内存。一个设计得好的new-handler函数必须实现下面功能中的一种。

- 产生更多的可用内存。这将使operator 
  new下一次分配内存的尝试有可能获得成功。实施这一策略的一个方法是：在程序启动时分配一个大的内存块，然后在第一次调用new-handler时释放。释放时伴随着一些对用户的警告信息，如内存数量太少，下次请求可能会失败，除非又有更多的可用空间。
- 安装另一个不同的new-handler函数。如果当前的new-handler函数不能产生更多的可用内存，可能它会知道另一个new-handler函数可以提供更多的资源。这样的话，当前的new-handler可以安装另一个new-handler来取代它(通过调用set_new_handler)。下一次operator 
  new调用new-handler时，会使用最近安装的那个。(这一策略的另一个变通办法是让new-handler可以改变它自己的运行行为，那么下次调用时，它将做不同的事。方法是使new-handler可以修改那些影响它自身行为的静态或全局数据。)
- 卸除new-handler。也就是传递空指针给set_new_handler。没有安装new-handler，operator 
  new分配内存不成功时就会抛出一个标准的std::bad_alloc类型的异常。
- 抛出std::bad_alloc或从std::bad_alloc继承的其他类型的异常。这样的异常不会被operator 
  new捕捉，所以它们会被送到最初进行内存请求的地方。(抛出别的不同类型的异常会违反operator 
  new异常规范。规范中的缺省行为是调用abort，所以new-handler要抛出一个异常时，一定要确信它是从std::bad_alloc继承来的)。
- 没有返回。典型做法是调用abort或exit。

用类模板实现每个继承的特定类都实现自己的operator new：

```cpp
template<class t>	// 提供类set_new_handler支持的
class newhandlersupport {	// 混合风格”的基类
public:
	static new_handler set_new_handler(new_handler p);
	static void * operator new(size_t size);

private:
	static new_handler currenthandler;
};

template<class t>
new_handler newhandlersupport<t>::set_new_handler(new_handler p)
{
	new_handler oldhandler = currenthandler;
	currenthandler = p;
	return oldhandler;
}

template<class t>
void * newhandlersupport<t>::operator new(size_t size)
{
	new_handler globalhandler =
		std::set_new_handler(currenthandler);
	void *memory;
	try {
		memory = ::operator new(size);
	}
	catch (std::bad_alloc&) {
		std::set_new_handler(globalhandler);
		throw;
	}
	
	std::set_new_handler(globalhandler);
	return memory;
}
// this sets each currenthandler to 0

template<class t>
new_handler newhandlersupport<t>::currenthandler;
```

定义自己的类，可以直接继承：

```cpp
// note inheritance from mixin base class template. (see
// my article on counting objects for information on why
// private inheritance might be preferable here.)
class x: public newhandlersupport<x> {

...		// as before, but no declarations for
};		// set_new_handler or operator new
```

远古C++要求内存分配失败时operator new返回0而不是抛出std::bad_alloc异常。为了向下兼容，C++的语法可以用placement new替代：

```cpp
class widget { ... };

widget *pw1 = new widget;// 分配失败抛出std::bad_alloc if

if (pw1 == 0) ...	// 这个检查一定失败

widget *pw2 = new (nothrow) widget; 	// 若分配失败返回0
	
if (pw2 == 0) ...	// 这个检查可能会成功

```

这种形式称为“nothrow”，与“Regular”形式的new作以区分。无论采用哪种形式，关键点都在于要为内存分配失败做好准备。

## 条款8：写operator new和operator delete时要遵循常规

重写operator new时要保证其行为和系统缺省的operator new一致。

非类成员形式的operator new伪代码类似如下：

```cpp
void * operator new(size_t size)        // operator new还可能有其它参数
{                                       

  if (size == 0) {                      // 处理0字节请求时，
    size = 1;                           // 把它当作1个字节请求来处理
  }                                     
  while (1) {
    分配size字节内存;

    if (分配成功)
      return (指向内存的指针);

    // 分配不成功，找出当前出错处理函数
    new_handler globalhandler = set_new_handler(0);
    set_new_handler(globalhandler);

    if (globalhandler) (*globalhandler)();
    else throw std::bad_alloc();
  }
}
```

因此，跳出循环的唯一办法是分配成功，或者globalhandler完成了条款7描述的事件中的一种：得到了更多的可用内存；安装了一个新的new-handler(出错处理函数)；卸除了new-handler；抛出了一个std::bad_alloc或其派生类型的异常；或者返回失败。

## 条款9：避免隐藏标准形式的new

如果类里增加了一个带多个参数的operator new函数，那么就会导致标准形式的new被隐藏。

```cpp
class x {
public:
  void f();

  // operator new的参数指定一个
  // new-hander(new的出错处理)函数
  static void * operator new(size_t size, new_handler p);
};

void specialerrorhandler();          // 定义在别的地方

x *px1 =
  new (specialerrorhandler) x;       // 调用x::operator new
x *px2 = new x;                      // 错误!
```

解决的办法有两种，第一种是在类里增加对单参数operator new的定义：

```cpp
class x {
public:
  void f();

  static void * operator new(size_t size, new_handler p);

  static void * operator new(size_t size)
  { return ::operator new(size); }
};

x *px1 =
  new (specialerrorhandler) x;      // 调用 x::operator
                                    // new(size_t, new_handler)
x* px2 = new x;                     // 调用 x::operator
                                    // new(size_t)
```

第二种办法是为每一个增加到operator new的参数提供缺省值：

```cpp
class x {
public:
  void f();

  static
    void * operator new(size_t size,                // p缺省值为0
                        new_handler p = 0);         // 
};

x *px1 = new (specialerrorhandler) x;               // 正确
x* px2 = new x;                                     // 也正确
```

无论哪种，只要保证不隐藏掉标准形式的new即可。

## 条款10：如果写了operator new就要同时写operator delete

自定义的内存管理程序可以很好地改善程序的性能，operator new和operator delete需要同时工作，那么你写了operator new，就也一定要写operator delete。否则可能会引起多种情景下的内存泄露。

## 条款11：为需要动态分配内存的类声明一个拷贝构造函数和一个赋值操作符

这个其实很好理解，因为合成的拷贝控制成员是值拷贝，因此对于动态分配内存的类来说，不应该拷贝指针的值（浅拷贝），而应该自己完成深拷贝。

用VS13写个例子：

```cpp
#include <iostream>

using namespace std;

namespace r00tk1t
{
	class string
	{
	public:
		string(const char *value);
		~string();
		void print();
	private:
		char *data;	//指针成员
	};

	string::string(const char *value)
	{ 
		if (value)
		{
			data = new char[strlen(value) + 1];
			strcpy_s(data, strlen(value)+1, value);
		}
		else
		{
			data = new char[1];
			*data = '\0';
		}
	}

	inline string::~string(){ delete[] data; }
	
	void string::print()
	{
		printf("data:%s, address:%x\n", data, data);
	}
}

int main()
{
	r00tk1t::string a("hello"), b("world");
	a.print();	//data:hello, address:9b65b0
 	b.print();	//data:world, address:9b8dc8

	b = a;		//这会调用合成的拷贝赋值运算符函数
	b.print();	//data:hello, address:9b65b0

	return 0;
}
```

可以看到浅拷贝了指针的值，这还不是最致命的。这段程序会崩溃，因为当a和b析构的时候会调用到类string的析构函数，而二者delete[]的是同一块内存，第二次的delete[]会导致程序崩溃（标准对二次释放的情形是未定义的，取决于编译器实现）。另外，b=a的时候也会导致b原本申请的动态内存丢失，导致经典的内存泄露。

对于拷贝构造函数来说也是一样的，在调用函数传值、函数返回值类型、拷贝初始化的场合都会出现相同的情景，不一一编码尝试了。

所以，拷贝控制成员必须要自定义，修改一下上面的类：

```cpp
namespace r00tk1t
{
	class string
	{
	public:
		string(const char *value);
		string(const string &rhs);
		string& operator=(const string &rhs);
		~string();
		void print();
	private:
		char *data;	//指针成员
	};

	string::string(const char *value)
	{ 
		if (value)
		{
			data = new char[strlen(value) + 1];
			strcpy_s(data, strlen(value)+1, value);
		}
		else
		{
			data = new char[1];
			*data = '\0';
		}
	}

	string::string(const string &rhs)
	{ 
		*this = rhs;
	}

	string& string::operator=(const string &rhs)
	{
		delete[] data;
		if (rhs.data)
		{
			data = new char[strlen(rhs.data) + 1];
			strcpy_s(data, strlen(rhs.data) + 1, rhs.data);
		}
		else
		{
			data = new char[1];
			*data = '\0';
		}

		return *this;
	}

	inline string::~string(){ delete[] data; }
	
	void string::print()
	{
		printf("data:%s, address:%x\n", data, data);
	}
}
```

## 条款12：尽量使用初始化而不要在构造函数里赋值

初始化列表的好处在于，有些情况必须得用初始化，无法在构造函数体内进行赋值，比如const成员和引用成员。

对象的构造分两步：数据成员初始化 + 执行被调用的构造函数体内的动作。

初始化列表可以为第一步提供成员初始化信息。

当然初始化也不是在任何情况下都比构造函数体内赋值要优越，对于冗余的固定类型数据成员的初始化往往非常啰嗦（比如类有若干个基础类型int成员），此时使用赋值未尝不可。

静态类成员有自己的初始化方式，与本条款无缘。

## 条款13：初始化列表中成员列出的顺序和它们在类中声明的顺序相同

这是C++语法的一个坑，程序员编写的初始化列表的顺序并不是真正的初始化顺序，真正的初始化顺序只受类中数据成员声明的顺序所影响。

用一个例子来踩坑：

```cpp
#include <iostream>
#include <vector>

using namespace std;

namespace r00tk1t
{
	template<typename T>
	class array
	{
	public:
		array(int lowbound, int highbound);
		vector<T>& getData(){ return data; }
	private:
		vector<T> data;
		size_t size;
		int lbound, hbound;
	};

	template<typename T>
	array<T>::array(int lowbound, int highbound) : 
		size(highbound - lowbound + 1), lbound(lowbound), hbound(highbound), data(size)
	{ }
}

int main()
{
	r00tk1t::array<int> arr(0, 10);
	vector<int> v = arr.getData();
	cout << v.size() << endl;	//输出0

	return 0;
}
```

因为声明顺序的原因，尽管初始化列表把size放在了前面，但实际上先初始化的还是data。至于输出0的问题，我不确定是标准的要求还是编译器自己的处理，反汇编了VS生成的code：

```assembly
r00tk1t::array<int> arr(0, 10);
012397E0 6A 1C                push        1Ch  
012397E2 8D 4D D0             lea         ecx,[arr]  
012397E5 E8 9D 7D FF FF       call        r00tk1t::array<int>::__autoclassinit2 (01231587h)  
012397EA 6A 0A                push        0Ah  
012397EC 6A 00                push        0  
012397EE 8D 4D D0             lea         ecx,[arr]  
	r00tk1t::array<int> arr(0, 10);
012397F1 E8 31 7E FF FF       call        r00tk1t::array<int>::array<int> (01231627h)  
012397F6 C7 45 FC 00 00 00 00 mov         dword ptr [ebp-4],0  
```

经过了`r00tk1t::array<int>::__autoclassinit2`之后，栈空间上原本的0xCC都被擦除了。在构造函数体内：

```assembly
01233060 55                   push        ebp  
01233061 8B EC                mov         ebp,esp  
01233063 81 EC CC 00 00 00    sub         esp,0CCh  
01233069 53                   push        ebx  
0123306A 56                   push        esi  
0123306B 57                   push        edi  
	private:
		vector<T> data;
		size_t size;
		int lbound, hbound;
	};

	template<typename T>
	array<T>::array(int lowbound, int highbound) : 
		size(highbound - lowbound + 1), lbound(lowbound), hbound(highbound), data(size)
	{ }
0123306C 51                   push        ecx  
0123306D 8D BD 34 FF FF FF    lea         edi,[ebp-0CCh]  
01233073 B9 33 00 00 00       mov         ecx,33h  
01233078 B8 CC CC CC CC       mov         eax,0CCCCCCCCh  
0123307D F3 AB                rep stos    dword ptr es:[edi]  
0123307F 59                   pop         ecx  
01233080 89 4D F8             mov         dword ptr [this],ecx  
01233083 8B 45 F8             mov         eax,dword ptr [this]  
01233086 8B 48 10             mov         ecx,dword ptr [eax+10h]  
01233089 51                   push        ecx  
0123308A 8B 4D F8             mov         ecx,dword ptr [this]  
0123308D E8 18 E5 FF FF       call        std::vector<int,std::allocator<int> >::vector<int,std::allocator<int> > (012315AAh)  
01233092 8B 45 0C             mov         eax,dword ptr [highbound]  
01233095 2B 45 08             sub         eax,dword ptr [lowbound]  
01233098 83 C0 01             add         eax,1  
0123309B 8B 4D F8             mov         ecx,dword ptr [this]  
0123309E 89 41 10             mov         dword ptr [ecx+10h],eax  
012330A1 8B 45 F8             mov         eax,dword ptr [this]  
012330A4 8B 4D 08             mov         ecx,dword ptr [lowbound]  
012330A7 89 48 14             mov         dword ptr [eax+14h],ecx  
012330AA 8B 45 F8             mov         eax,dword ptr [this]  
012330AD 8B 4D 0C             mov         ecx,dword ptr [highbound]  
012330B0 89 48 18             mov         dword ptr [eax+18h],ecx  
012330B3 8B 45 F8             mov         eax,dword ptr [this]  
012330B6 5F                   pop         edi  
012330B7 5E                   pop         esi  
012330B8 5B                   pop         ebx  
012330B9 81 C4 CC 00 00 00    add         esp,0CCh  
012330BF 3B EC                cmp         ebp,esp  
012330C1 E8 8C E2 FF FF       call        __RTC_CheckEsp (01231352h)  
012330C6 8B E5                mov         esp,ebp  
012330C8 5D                   pop         ebp  
012330C9 C2 08 00             ret         8  
```

由于初始化顺序的问题，在`call        std::vector<int,std::allocator<int> >::vector<int,std::allocator<int> > (012315AAh)`时传入的参数是0，最终传到：

```assembly
explicit vector(size_type _Count)
		: _Mybase()
		{	// construct from _Count * value_type()
01239230 55                   push        ebp  
01239231 8B EC                mov         ebp,esp  
01239233 6A FF                push        0FFFFFFFFh  
01239235 68 F8 A3 23 01       push        123A3F8h  
0123923A 64 A1 00 00 00 00    mov         eax,dword ptr fs:[00000000h]  
01239240 50                   push        eax  
```

传入的_Count是size所在的位置，因为之前init的关系它已经是0了。

关于这个初始化内存空间的举措我不清楚是标准所要求还是编译器自己所处理的。

因为初始化有着依赖关系，所以我们调换一下次序：

```cpp
template<typename T>
	class array
	{
	public:
		array(int lowbound, int highbound);
		vector<T>& getData(){ return data; }
	private:
		size_t size;
		int lbound, hbound;
		vector<T> data;
	};

	template<typename T>
	array<T>::array(int lowbound, int highbound) : 
		size(highbound - lowbound + 1), lbound(lowbound), hbound(highbound), data(size)
	{ }
```

再次输出的尺寸就是正确的11了。

**既然如此，就有一个疑惑了，为什么语法非要要求按声明次序初始化，而不是初始化列表次序呢？**

这是因为对于对象的成员来说，析构函数被调用的顺序是和构造函数里被创建的顺序相反的，如果支持某一个构造函数按初始化列表次序初始化，那么可能各种构造函数的初始化次序各不相同，这时每个对象析构时其对象的析构顺序也就各不相同，编译器就得为每个对象而不是每类对象去维护析构顺序。

但如果忽略初始化顺序，无论何种构造，都要按成员声明顺序构建，那这一类对象析构的顺序也就是固定的。

与条款12一样，静态数据成员有自己的机制，不受该条款影响。

## 条款14：确定基类有虚析构函数

这个就人尽皆知了，因为多态的使用本身会出现对指向派生类的基类指针进行delete的操作。如果基类析构是非虚的，那么delete的结果就是不确定的（C++标准的阐述）。

另一方面，如果类不作为基类使用，那么析构函数不应该被设置为虚函数。虚函数的存在会为对象引入vptr，对每个对象来说体积都会增加4字节(32位)。

很多人总结：当且仅当类里包含至少一个虚函数的时候才去声明虚析构函数。其实这种说法并不完全正确。对于类模板来说，可能没有虚函数，但类模板会被继承下去，这种情况虚析构也是需要的。因此，本质上来讲，只要类作为基类使用且有析构函数，就要定义成虚析构。

## 条款15：让operator=返回*this的引用

既不要返回void，也不要返回const对象的引用，前者虽然合理但妨碍了链式赋值，后者则会导致链式赋值的失败（无法为返回的常量赋值）。

本质上来说，C++语言提供运算符重载的初衷就是希望程序员可以像使用基础类型那样使用自定义的类类型。因此，对operator=保持基础类型的语义就很自然而然了。

类中重载的赋值运算符函数，总是要返回`*this`，因为this是隐式传递进来的=号左侧的对象指针。

## 条款16：在operator=中对所有数据成员赋值

因为如果不定义operator=，编译器会自动合成一个，但一旦自己定义了，那么就要接管所有的数据成员赋值，不要妄想我只赋值个别想要特别处理的成员，其他的成员还可以让编译器去默认处理。

添加类数据成员要记得更新operator=。

对继承体系来说，派生类的赋值运算符也必须处理它的基类成员的赋值。

踩踩坑：

```cpp
class base {
public:
	base(int initialvalue = 0) : x(initialvalue) {}

private:
	int x;
};

class derived : public base {
public:
	derived(int initialvalue)
		: base(initialvalue), y(initialvalue) {}

	derived& operator=(const derived& rhs);

private:
	int y;
};

// erroneous assignment operator
derived& derived::operator=(const derived& rhs)
{
	if (this == &rhs) return *this;    

	y = rhs.y;             

	return *this;                      
}


int main()
{
	derived d1(0), d2(1);

	d1 = d2;	//d1.x=0, d1.y=1!
	return 0;
}
```

因为派生类自定义的operator=只接管了自己的数据成员，基类的成员没有管。

如果溢出derived的operator=的自定义，会发现编译器默认合成的版本会类似构造器一样，递归的先调用基类base的operator=，这里base的operator=也是默认合成的（此时d1里的x和y都变成了1）。

那么当我们接管了operator=时，也应该去在派生类中显式的调用基类的operator=（我们没法直接为x赋值，因为是基类的private，就算是public或protected，也不要这样处理，基类的成员交给基类自己去处理）。

```cpp
// erroneous assignment operator
derived& derived::operator=(const derived& rhs)
{
	if (this == &rhs) return *this;    

	base::operator=(rhs);
  	// 这种写法比较怪异，用于适配那些拒绝对基类赋值运算符调用的编译器
  	// static_cast<base&>(*this) = rhs;
	y = rhs.y;             

	return *this;                      
}
```

operator=的这一点对于拷贝构造来说也是一样的，派生类不要忘记在拷贝构造时，在初始化列表中调用基类的拷贝构造。

```cpp
derived(const derived& rhs): base(rhs), y(rhs.y) {}
```

## 条款17：在operator=中检查给自己赋值的情况

这一点非常容易忽略，因为自己赋值给自己在语法上是正确的，所以对这种情况来说，operator=如果不检查自赋值的情况，那么就会浪费感情。

另外，检查自赋值还可以保证赋值的正确性，对指针成员来说，赋值运算符可能会先释放一个对象的资源，再分配新的资源（这一点在C++ Primer中被重点敲黑板了，要先分配再释放，保证自赋值的语义正确性）。如果自赋值那就导致在拷贝的时候资源已经被第一步释放掉了。

这个坑就不再踩了，学C++ Primer的时候已经踩过一次了。

## 条款18: 争取使类的接口完整并且最小

设计类接口的目标：完整且最小。

所谓完整就是指提供所有合理的接口，不能因为接口残疾而让用户完不成应该完成的任务。最小则是指提供的接口要尽量少，不用互相牵扯混淆。

这是个设计上的问题，不是语法上的坑。

## 条款19: 分清成员函数，非成员函数和友元函数

如果需要动态绑定，那只能选用成员函数的设计，因为成员函数才可以是虚函数。

如果不需要动态绑定，就需要根据实际的语义进行区分。这其实也是一个经验性的问题，甚至可以说是潜规则。某些函数的语义设计成成员函数更佳，而另一部分设计成非成员函数更好。对于运算符重载函数亦是如此。C++语法上对运算符做了限制，使得某些运算符只能重载成成员函数(自增自减)，某些则是非成员函数(operator<<)，还有一些是二者均可，但往往也会因为语义而沿用其他类似的设计。

非成员函数如果需要访问类的非public成员，就要记得在类的设计中增加友元函数的设定。

## 条款20: 避免public接口出现数据成员

public和private最开始设计出来就是为了功能分离（抽离接口和隐藏数据），如果数据成员以public出现，那么外部就可以肆无忌惮的乱搞。

在public接口里放上数据成员无异于自找麻烦，所以要把数据成员安全地隐藏在与功能分离的高墙后。

> 仅仅打包一团数据，用struct关键字更有辨识度，虽然struct和class在C++语法上只是默认访问权限不同。这里所谓的功能分离不适配于这种情况。

## 条款21: 尽可能使用const

const在C++中的使用非常宽泛，在类的外面，可以用于全局或名字空间常量，以及静态对象。类内则可以用于静态和非静态成员。对指针来说，可以指定指针本身为const，也可以指定所指数据为const，或二者兼具（顶层const和底层const）。

const的坑太多了，不是三言两语就可能穷尽的。但无论是何种坑，只要对const的具体使用情景有深刻的理解，就能跳出来。

C++11以后，还要考虑const与auto、decltype等相互作用。

## 条款22: 尽量用“传引用”而不用“传值”

C只能传值，尽管传递指针看起来和引用功能一样，但却决然不同。指针本身也是一个变量，它的值是存储指向对象的地址。传递指针只是把实参的地址值拷贝给了形参，所以本质上还是值传递。

而C++的引用本身不是一个对象，它只是一个其他对象的别名。因此传引用是C++引入的概念，C是不支持的。

> 这里的传引用是值左值引用。C++11以后为了移动语义，还有右值引用。

传引用不会引起对象的拷贝（传对象的指针也可以避免对象的拷贝），传递引用也避免了语法的繁琐（C玩家饱受嵌套指针做参数的折磨）。

某些情境下不能传递引用。对基本类型来说，传值比传引用效果更好。

## 条款23: 必须返回一个对象时不要试图返回一个引用

某些语义是应该返回对象的，不能返回一个引用。返回局部对象的引用是错误的（后面有条款专述），返回参数引用在语义上也不符。

比如：

```cpp
class rational {
public:
  rational(int numerator = 0, int denominator = 1);

  ...

private:
  int n, d;              // 分子和分母

friend
  const rational                      // 参见条款21：为什么
    operator*(const rational& lhs,    // 返回值是const
              const rational& rhs)     
};

inline const rational operator*(const rational& lhs,
                                const rational& rhs)
{
  return rational(lhs.n * rhs.n, lhs.d * rhs.d);
}

```

`operator*`返回的是对象，而不是对象引用，但这并没有未被尽量传引用而不传值的准则（返回值本身也有值传递和引用传递，与参数传递如出一辙）。

这是因为`operator*`语义上应该返回一个新的对象，而不是原本已有的对象。

假设这样改写：

```cpp
inline const rational& operator*(const rational& lhs,
                                 const rational& rhs)
{
  rational result(lhs.n * rhs.n, lhs.d * rhs.d);
  return result;
}
```

返回了局部对象是严重的错误，因为局部对象在栈空间，随着函数的返回而被自动析构，返回的引用的对象已无“生命体征”，显然有问题。这点以后还会讨论。

那如果改用堆空间呢？

```cpp
inline const rational& operator*(const rational& lhs,
                                 const rational& rhs)
{
  rational *result =
    new rational(lhs.n * rhs.n, lhs.d * rhs.d);
  return *result;
}
```

看起来好像没什么问题，但带来的隐患却是巨大的。在`operator*`中new，那在哪里delete呢？该由谁负责delete呢？而且，每次`operator*`以后都要记得delete对开发者来说不太现实，一次处理不当，就导致了内存泄露的问题。毕竟，从语义上来说，开发者不大能接受`operator*`内部new的对象还要自己去控制的规则。

```cpp
rational w, x, y, z;
w = x * y * z;
```

按上面的写法，内存泄露就已经发生了，oops!

还有第三种坑：

```cpp
inline const rational& operator*(const rational& lhs,
                                 const rational& rhs)
{
  static rational result;      // 将要作为引用返回的
                               // 静态对象

  lhs和rhs 相乘，结果放进result；

  return result;
}
```

纯粹是自作聪明，这种设计的思想是：既然局部对象不行，那就弄个伪局部？

这种设计就是没有搞清楚局部static变量的意义，实际上局部static只是限制了语义为局部，本质上可以看做一个全局对象，每次`operator*`的调用都会修改局部static的result对象，这就意味着`operator*`的后一次调用会影响前一次调用的结果值（因为后一次修改了result对象，而前一次返回的却是这个被修改的对象的引用，当然受了影响）。

所以，别折腾了，该返回值的时候就要返回值，不能省。

## 条款24: 在函数重载和设定参数缺省值间慎重选择

允许一个函数以多种方式被调用有两种手段：重载和默认形参。

选择哪一个要根据具体的情景，一般来说缺省值会对函数体产生一定程度的限制，而重载则会代码膨胀。

关于这一点其实没什么好说的，按经验判断即可。

## 条款25: 避免对指针和数字类型重载

因为C++11以前是没有独立出nullptr的，旧式C风格的NULL本质上是个宏，可能定义成`#define NULL 0`或者是`#define NULL ((void*)0)`等。

那么如果重载接受一个int或某类型指针参数的函数时，就会导致在传0值时0的二义性，是应该匹配int版本还是指针版本呢？当然可以用`static_cast<t*>`来避免二义性。

尽管在大部分情况下推导的函数是确定的，不会出现编译错误，但这种设计无疑是自掘坟墓。所以，作为重载函数的设计者，归根结底最基本的一条是，只要有可能，就要避免对一个数字和一个指针类型重载。

## 条款26: 当心潜在的二义性

坑1：

```cpp
class 
B;		// 对类B提前声明
		// 
class A {
public:
 A(const B&);	// 可以从B构造而来的类A
};

class B {
public:
 operator A() const;	// 可以从A转换而来的类B
};

void f(const A&);
B b;
f(b);			// 错误!——二义
```

上例中，f的调用需要传入一个A的对象，但上面的定义却有两种可行性，第一种是通过A的构造函数来生成，另一种是通过B的类型转换运算符来生成，于是编译器懵逼了。

这种定义虽然合法，而且在未使用到二义性的语句时也不会出问题，但总归是挖了坑。

坑2：

```cpp
void f(int);
void f(char);

double d = 6.02;
f(d);			// 错误!——二义
```

因为两种隐式转换都可以，所以编译器还是懵逼。尽管可以通过`static_cast<>`来转型，但依然不舒服。

坑3：

```cpp
class Base1 {
public:
 int doIt();
};

class Base2 
{
public:
 void doIt();
};

class Derived: public Base1	// Derived没有声明
		public Base2 {	// 一个叫做doIt的函数
...
};

Derived d;
d.doIt();		// 错误!——二义
```

这个就是非常常见的坑了，多重继承引起的基类成员同名的二义性。

消除的手法可以用`d.Base1::doIt()`来调用，但语法上就啰嗦多了。

## 条款27: 如果不想使用隐式生成的函数就要显式地禁止它

C++11以前不支持对合成的构造、复制构造和赋值运算符函数进行=default/=delete的操作，因此对于需要显式禁止的场合，是通过一种编程技巧来实现的。

通过将不希望默认合成的函数显式地声明为private成员而不去定义，就可以实现=delete的效果。

## 条款28: 划分全局名字空间

全局空间最大的问题在于它本身仅有一个。在大的软件项目中，经常会有不少人把他们定义的名字都放在这个单一的空间中，从而不可避免地导致名字冲突。

所以我们自己写代码尤其是希望作为第三方库使用时，将代码分离在一个甚至多个独立的命名空间中很有必要。

使用时，只需要通过using引入或者直接通过`namespace::`访问即可。

## 条款29: 避免返回内部数据的句柄

返回内部数据的指针或引用往往会引起其失效问题，容易导致悬垂句柄。

对于const成员函数来说，返回句柄是不明智的，因为它会破坏数据抽象。对于非const成员函数来说，返回句柄会带来麻烦，特别是涉及到临时对象时。

当然，只能说要尽量避免，并不绝对化。

## 条款30: 避免这样的成员函数：其返回值是指向成员的非const指针或引用，但成员的访问级比这个函数要低

踩踩坑：

```cpp
#include <iostream>
#include <vector>
#include <string>

using namespace std;

class person{
public:
	person(const string& addr) :address(addr) { }
	string& getAddress(){ return address; }//注意返回的是引用
private:
	string address;
};

int main()
{
	person p("China");
	string &address = p.getAddress();
	cout << address << endl;		//China
	address = "America";
	cout << p.getAddress() << endl;	//America
	return 0;
}
```

如果getAddress返回的是引用，就破坏了设定为private的初衷。外部可以绕过private访问修改对象的address成员。

同样的：

```cpp
class person{
public:
	person(const string& addr) :address(addr) { }
	string* getAddress(){ return &address; }//返回指针
private:
	string address;
};

int main()
{
	person p("China");
	string *address = p.getAddress();
	cout << *address << endl;		//China
	*address = "America";
	cout << *p.getAddress() << endl;	//America
	return 0;
}
```

返回指针也具有相同的问题。

所以，如果想对private成员提供只读接口，那么应该使用值返回。

## 条款31: 千万不要返回局部对象的引用，也不要返回函数内部用new初始化的指针的引用

局部对象是在被定义时创建，在离开生命空间时被销毁的。返回一个局部对象的引用当然是错误的。

函数内部的指针本质上也是局部对象，所以也不能返回它的引用，虽然函数内部用指针去new一个对象，然后返回该指针的引用看起来好像中规中矩：new出的对象没有被释放啊！但指针本身这个对象会随着离开生命空间而被销毁，好嘛，还引起了内存泄露。

这个议题在C语言中就是新手阶段常见的大坑，对引入了引用概念的C++来说更是不遑多让。

## 条款32: 尽可能地推迟变量的定义

这个没啥好说的，因为更容易控制和确定变量的生命周期。其实C99已经有了，要求所有变量都只能定义在头部的说法是C89的legacy。

## 条款33: 明智地使用内联

都8102年了，这个放现在来看已经意义不大了，毕竟不再有“我比编译器聪明系列”。

## 条款34: 将文件间的编译依赖性降至最低

为了提高编译效率，防止未修改文件的重新编译。这也算是C时代的legacy了。

## 条款35: 使公有继承体现 "是一个" 的含义

类D公有继承类B，意味着每个D的对象也是一个B的对象，但每个B的对象不一定是D的对象。B比D更为宽泛，D比B更为特化。

公有继承应该体现出D“是一个”B这样的语义，所以设计上就要慎重考虑，当符合D是一个B时，应该采用公有继承。

## 条款36: 区分接口继承和实现继承

纯粹的接口继承就是基类中定义的纯虚函数，而纯粹的实现继承则是非虚函数，在二者之间还有一个虚函数，可以由派生类决定沿用父类的实现还是自己重新diy。

设计一个类时，要根据需求，确定基类成员应该提供哪一种继承方式。

## 条款37: 决不要重新定义继承而来的非虚函数

这是C++老生长谈的话题，对子类重新定义父类的非虚函数，会导致父类的实现继承被覆盖。

踩坑：

```cpp
class Base{
public:
	void print(){ cout << "Base" << endl; }
};

class Derived : public Base{
public:
	void print(){ cout << "Derived" << endl; }
};

int main()
{
	Derived d;
	d.print();	//Derived

	Base *b = new Derived();
	b->print();	//Base
	return 0;
}
```

基类的print被覆盖，而更好的设计应该是定义成虚函数。

为什么说这种定义不好呢？因为虚函数才能实现多态，上面的指针b就是个例子。

而且这种隐藏也不彻底，依然可以通过d来调用Base的print实现：

```cpp
d.Base::print();	//Base
(static_cast<Derived*>(b))->print();//Derived
```

都是奇奇怪怪的写法，坑自己。

改成virtual就无需这种坑自己的写法了。

当然，改为virtual后，想要调用Base的print也是可以的：

```cpp
Derived d;
d.Base::print();

Base *b = new Derived();
b->Base::print();
```

视具体情况而用。

## 条款38: 决不要重新定义继承而来的缺省参数值

根据条款37的规则，这里值得考虑的就是“继承一个有缺省参数值的虚函数”。

基于的理由非常明显：虚函数是动态绑定而缺省参数值是静态绑定的。

踩踩坑：

```cpp
enum ShapeColor { RED, GREEN, BLUE };
// 一个表示几何形状的类
class Shape {
public:
	// 所有的形状都要提供一个函数绘制它们本身
	virtual void draw(ShapeColor color = RED) const = 0;
};

class Rectangle : public Shape {
public:
	// 注意：定义了不同的缺省参数值 ---- 不好!
	virtual void draw(ShapeColor color = GREEN) const{ cout << color << endl; };
};

class Circle : public Shape {
public:
	virtual void draw(ShapeColor color) const{ cout << color << endl; };
};



int main()
{
	Shape *ps;
	Shape *pc = new Circle;
	Shape *pr = new Rectangle;

	pc->draw();	//0
	pr->draw();	//0

	return 0;
}
```

两次输出都是0！尽管Rectangle覆盖了虚函数，将默认传参改为GREEN，但实际上通过动态绑定调用时，依然是基类的参数。

但如果不利用多态，直接定义派生类型，则默认传参确实是GREEN：

```cpp
Rectangle *pr2 = new Rectangle;
pr2->draw(); //1
```

因此，这就是标准的自掘坟墓，变着法子坑自己。

## 条款39: 避免 "向下转换" 继承层次

如果不借用动态绑定，那么如果用父类指针或引用指向子类对象，编译器是一无所知的，此时，我们就需要自己去通过static_cast进行“向下转换”。

在某些特定场景下这可能很有用，但对于大部分成熟的设计来说，应该转而使用虚函数来支持动态绑定。毕竟，派生类只有1个的时候，static_cast是很好用的，但一旦有多个派生类，那么当程序员把它们的基类指针混杂在一起之后，就彻底分不清谁究竟是哪个派生类对象了。

这个坑没什么好踩的，归根结底是设计的思维，总之，在使用到static_cast的时候多想想是不是自己设计得有问题，是不是有其他的方式来代替？

## 条款40: 通过分层来体现 "有一个" 或 "用...来实现"

所谓分层，就是指某个类的对象称为另一个类的数据成员。

公有继承提供的语义是Derived对象“是一个”Base对象，而分层提供的语义是“A有一个B”，或者是“A用B来实现”。

实际设计中，要根据需求考虑是采用分层来提供“A用B来实现”的语义，还是提供“A是一个B”的语义。对于后者来说，谨遵条款35，应该符合对B成立的条件对A应该也都成立这一事实，如果不符合，就要考虑“A用B来实现”的语义。

## 条款41: 区分继承和模板

何时使用类模板，何时使用继承？

最本质的要点在于要判断：类型T是否影响类的行为。如果不影响就可以使用模板（因为代码对各种可能的类型是common的），如果影响那就只能用虚函数（这个说得比较绝对，其实模板也可以通过特化来解决个别的不兼容问题），从而要使用继承。

## 条款42: 明智地使用私有继承

公有继承意味着派生类“是一个”基类，分层则意味着A“有一个”B或“A用B来实现...”，那么私有继承意味着什么呢？

先明确私有继承的两个特征：

- 如果两个类之间的继承关系为私有，编译器一般不会将派生类对象转换成基类对象。
- 私有继承二来的成员都成为了派生类的私有成员，无论在基类中是保护或公有成员。

私有继承意味着 "用...来实现"。如果使类D私有继承于类B，这样做是因为你想利用类B中已经存在的某些代码，而不是因为类型B的对象和类型D的对象之间有什么概念上的关系。因而，私有继承纯粹是一种实现技术。

私有继承意味着只是继承实现，接口会被忽略。如果D私有继承于B，就是说D对象在实现中用到了B对象，仅此而已。私有继承在软件 "设计" 过程中毫无意义，只是在软件 "实现" 时才有用。

“用...来实现”与分层的用途有所混淆（分层也有这一层意思），那么二者应该如何抉择？答案很简单：尽可能地使用分层，必须时才使用私有继承。什么时候必须呢？这往往是指有保护成员和/或虚函数介入的时候。

换句话说，当需要权限控制和虚函数参于到“用...来实现”这一设计的时候，分层是达不到效果的，这时只能用也必须用私有继承。

书中给出的一个例子：

```cpp
class GenericStack {
protected:
  GenericStack();
  ~GenericStack();

  void push(void *object);
  void * pop();

  bool empty() const;

private:
  struct StackNode {
    void *data;                    // 节点数据
    StackNode *next;               // 下一节点

    StackNode(void *newData, StackNode *nextNode)
    : data(newData), next(nextNode) {}
  };

  StackNode *top;                          // 栈顶

  GenericStack(const GenericStack& rhs);   // 防止拷贝和
  GenericStack&                            // 赋值(参见
    operator=(const GenericStack& rhs);    // 条款27)
};

GenericStack s;                   // 错误! 构造函数被保护
```

GenericStack没有public成员，这意味着该类不是暴露给用户使用的直接类，这种设计是因为考虑到GenericStack是不安全的通用类，不希望被直接使用，而是被具体类继承使用：

```cpp
class IntStack: private GenericStack {
public:
  void push(int *intPtr) { GenericStack::push(intPtr); }
  int * pop() { return static_cast<int*>(GenericStack::pop()); }
  bool empty() const { return GenericStack::empty(); }
};

class CatStack: private GenericStack {
public:
  void push(Cat *catPtr) { GenericStack::push(catPtr); }
  Cat * pop() { return static_cast<Cat*>(GenericStack::pop()); }
  bool empty() const { return GenericStack::empty(); }
};

IntStack is;                     // 正确
CatStack cs;                     // 也正确
```

当私有继承之后，GenericStack的protected成员被继承过来，且权限变为private。这一权限的控制是分层所做不到的（分层的话即使IntStack包含GenericStack对象，GenericStack对象的protected成员是无法被访问的，除非设置友元，这会导致关系混乱，或者protected成员定义成public，但定义成public意味着直接暴露了GenericStack给用户）。

私有继承可以访问到基类的protected成员，同时又阻止了基类的暴露。

进一步可以用模板来优化：

```cpp
template<class T>
class Stack: private GenericStack {
public:
  void push(T *objectPtr) { GenericStack::push(objectPtr); }
  T * pop() { return static_cast<T*>(GenericStack::pop()); }
  bool empty() const { return GenericStack::empty(); }
};
```

编译器将根据需要自动生成所有的接口类。

## 条款43: 明智地使用多继承

MI带来了单继承所没有的复杂性。比如，经典的基类成员重名的二义性。

踩坑：

```cpp
#include <iostream>

using namespace std;

class Lottery {
public:
	virtual void draw(){ cout << "Lottery draw!" << endl; }
};

class GraphicalObject {
public:
	virtual void draw() { cout << "GraphicalObject draw!" << endl; }
};

class LotterySimulation : public Lottery, public GraphicalObject {

	// 没有声明draw
};


int main()
{
	LotterySimulation *pls = new LotterySimulation;

	pls->draw();                   // 错误! ---- 二义
	pls->Lottery::draw();          // 正确
	pls->GraphicalObject::draw();  // 正确


	return 0;
}
```

这里的二义性是显而易见的，然而隐藏比较深的坑在这里：

```cpp
class GraphicalObject {
private:
	virtual void draw() { cout << "GraphicalObject draw!" << endl; }
};
```

即使修改了GraphicalObject的draw为private权限，二义性一样存在，即便客观上看起来只能访问得到Lottery的draw，但语法上依然存在错误。

此时，如果为LotterySimulation也定义一个draw：

```cpp
class LotterySimulation : public Lottery, public GraphicalObject {
public:
	virtual void draw() { cout << "LotterySimulation draw!" << endl; }
};
```

那么对于该调用：

```cpp
LotterySimulation *pls = new LotterySimulation;

pls->draw();                   // 正确
pls->Lottery::draw();          // 正确
pls->GraphicalObject::draw();  // 正确
```

就是正确的。

但对多继承来说，动态绑定不是那么好用：

```cpp
Lottery *pls = new LotterySimulation;

	pls->draw();                   // 正确，调用LotterySimulation
	pls->Lottery::draw();          // 正确
	pls->GraphicalObject::draw();  // 错误
```

此时pls是找不到GraphicalObject对象的，它不在同一条继承链上，无法动态绑定过去。

------

以上都是些小问题，如果用户想要在派生类中同时重写两个父类的draw要如何呢？

有一种技巧可以办到：

```cpp
class AuxLottery: public Lottery {
public:
  virtual int lotteryDraw() = 0;

  virtual int draw() { return lotteryDraw(); }
};

class AuxGraphicalObject: public GraphicalObject {
public:
  virtual int graphicalObjectDraw() = 0;

  virtual int draw() { return graphicalObjectDraw(); }
};


class LotterySimulation: public AuxLottery,
                         public AuxGraphicalObject {
public:
  virtual int lotteryDraw();
  virtual int graphicalObjectDraw();
};
```

通过两个中间类的周转，就可以在最后的派生类中同时重写两个draw，而使用时就可以根据各自链上的动态绑定利用指针型别的变换来访问：

```cpp
LotterySimulation *pls = new LotterySimulation;

Lottery *pl = pls;
GraphicalObject *pgo = pls;

// 调用LotterySimulation::lotteryDraw
pl->draw();

// 调用LotterySimulation::graphicalObjectDraw
pgo->draw();
```

**这是一个集纯虚函数，简单虚函数和内联函数综合应用之大成的方法，值得牢记在心。**

以上就是二义性引来的坑以及解决问题的手段。

------

现在开始谈谈多重继承中的菱形继承（虚继承）。

B和C派生自类A，D多重继承B和C，那么此时就要判断是否是虚继承。

当A是一个虚基类时，D就是虚继承，形成菱形结构；而A不是虚基类时，D中有两个A祖父类对象，这会导致D复杂得多。

大部分情况，设计上都会选择虚继承：

```cpp
class A { ... };
class B : virtual public A { ... };
class C : virtual public A { ... };
class D: public B, public C { ... };
```

A是非虚基类时，D的对象内存布局：

```
A部分+ B部分+ A部分 + C部分 + D部分
```

A是虚基类时，D的对象内存布局可能是（取决于编译器实现）：

```
        -----------------------------
        |                           |
        |                          \|/  
B部分 + 指针 + C部分 + 指针 + D部分 + A部分
                      |            /|\
                      |             |
                      ---------------
```

虚继承因为指针的存在，会带来额外的空间消耗。

多重继承的议题太多了，也太过于复杂，我现阶段的理解过于浅薄了，以后再述。

## 条款44: 说你想说的；理解你所说的

设计思想的对应关系：

- 共同的基类意味着共同的特性。如果类D1和类D2都把类B声明为基类，D1和D2将从B继承共同的数据成员和/或共同的成员函数。见条款43。
- 公有继承意味着 "是一个"。如果类D公有继承于类B，类型D的每一个对象也是一个类型B的对象，但反过来不成立。见条款35。
- 私有继承意味着 "用...来实现"。如果类D私有继承于类B，类型D的对象只不过是用类型B的对象来实现而已；类型B和类型D的对象之间不存在概念上的关系。见条款42。
- 分层意味着 "有一个" 或 "用...来实现"。如果类A包含一个类型B的数据成员，类型A的对象要么具有一个类型为B的部件，要么在实现中使用了类型B的对象。见条款40。

只适用于公有继承的情况：

- 纯虚函数意味着仅仅继承函数的接口。如果类C声明了一个纯虚函数mf，C的子类必须继承mf的接口，C的具体子类必须为之提供它们自己的实现。见条款36。
- 简单虚函数意味着继承函数的接口加上一个缺省实现。如果类C声明了一个简单（非纯）虚函数mf，C的子类必须继承mf的接口；如果需要的话，还可以继承一个缺省实现。见条款36。
- 非虚函数意味着继承函数的接口加上一个强制实现。如果类C声明了一个非虚函数mf，C的子类必须同时继承mf的接口和实现。实际上，mf定义了C的 "特殊性上的不变性"。见条款36。

## 条款45: 弄清C++在幕后为你所写、所调用的函数

永远要搞清楚编译器何时会自动合成构造器、拷贝控制成员等成员函数，何时不会生成，C++11还引入了移动控制成员。

## 条款46: 宁可编译和链接时出错，也不要运行时出错

只要有可能，就要让出错检查（除了异常）从运行时退回到链接时，或者，最理想的是，编译时。

这种方法带来的好处不仅仅在于程序的大小和速度，还有可靠性。如果程序通过了编译和链接而没有产生错误信息，你就可以确信程序中没有编译器和链接器能检查得到的任何错误，仅此而已。

运行时错误非常难调，所以写C++要足够谨慎。

## 条款47: 确保非局部静态对象在使用前被初始化

非局部静态对象指的是这样的对象：

- 定义在全局或名字空间范围内
- 在一个类中被声明为static，或
- 在一个文件范围被定义为static

第三种还好处理，因为只在一个文件内可用，那么提前初始化比较容易保证。

前两种就比较困难了，如果不是在定义时初始化，那么初始化的时机就很可能搞错，因为工程做大了以后，初始化的顺序很难保证（C++只是说它们在函数调用过程中初次碰到对象的定义时被初始化），而这引起的未初始化使用问题也很难调试。

**设计模式有种单例模式，可以在C++中如下应用：把每个非局部静态对象转移到函数中，声明它为static。其次，让函数返回这个对象的引用。这样，用户将通过函数调用来指明对象。换句话说，用函数内部的static对象取代了非局部静态对象。**

这种应用就保证了使用前一定会被初始化。另外，还带来另一个好处：如果这个模拟非局部静态对象的函数从没有被调用，也就永远不会带来对象构造和销毁的开销。

## 条款48: 重视编译器警告

早期C不重视warning，结果运行时错误一大坨，调试何等费时，渐渐发现很多问题其实warning都有暗示，但却因自信忽略而自讨没趣。

C++的warning就更重要了，毕竟C++的坑要多得多。

## 条款49: 熟悉标准库

C++11以后，标准库越来越变态了，源码之前，了无秘密（只要你看得懂）。

## 条款50: 提高对C++的认识

形而上学。致力于把各种语言特性和设计目标穿针引线、融会贯通。吾将上下而求索。