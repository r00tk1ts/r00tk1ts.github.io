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

## 待续。。

