---
title: C++ Templates Note - 类模板
date: 2018-08-25 12:31:11
categories: programming-language
tags:
	- cpp
	- cpp-templates
---
《C++ Templates》更新了第二版，内容上更新了C++ 11/15/17标准中模板元编程的大量内容。本文是阅读第二章“类模板”时所做的笔记。

<!--more-->
# C++ Templates Note - 类模板

类也可以用模板类型参数化，称作类模板。这些模板类型使用在类内部。

## 类模板Stack实现

```cpp
#include <vector>
#include <cassert>

template<typename T>
class Stack{
private:
  	std::vector<T> elems;	// elements
public:
  	void push(T const &elem);	// push elem
  	void pop();				// pop element
  	T const &top() const;	// return top element
  	bool empty() const{		// return whether the stack is empty
        return elems.empty();
    }
};

template<typename T>
void Stack<T>::push(T const & elem)
{
    elems.push_back(elem);	// append copy of passed elem
}

template<typename T>
void Stack<T>::pop()
{
    assert(!elems.empty());
  	elems.pop_back();	// remove last element
}

template<typename T>
T const &Stack<T>::top() const
{
    assert(!elems.empty());
  	return elems.back();	// return copy of last element
}
```

> 类模板Stack<>实现基于`std::vector<>`。内存管理，拷贝构造，赋值操作符就不必自己操心了。

### 类模板声明

与函数模板差不多：

```cpp
template<typename T>
class Stack{
    ...
};
```

定义拷贝构造和赋值操作符：

```cpp
template<typename T>
class Stack{
    ...
    Stack(Stack const &);	// copy constructor
  	Stack& operator= (Stack const &);	// assignment operator
  	...
};
```

等价于

```cpp
template<typename T>
class Stack{
    ...
    Stack(Stack<T> const &);	// copy constructor
  	Stack<T>& operator= (Stack<T> const &);	// assignment operator
  	...
};
```

类内部推荐使用第一种方式，但如果是在类外部，就不能省略`<T>`。

```cpp
template<typename T>
bool operator== (Stack<T> const &lhs, Stack<T> const &rhs);
```

此外，类模板只能被定义在全局/命名空间或是在其他类的定义中，不能够声明或定义在函数或代码块中。

### 成员函数的实现

```cpp
template<typename T>
void Stack<T>::push(T const &elem)
{
    elems.push_back(elem);	// append copy of passed elem
}

template<typename T>
T Stack<T>::pop()
{
    assert(!elems.empty());
  	T elem = elems.back();	// save copy of last element
  	elems.pop_back();		// remove last element
  	return elem;			// return copy of saved element
}

template<typename T>
T const &Stack<T>::top() const
{
    assert(!elems.empty());
  	return elems.back();	// return copy of last element
}
```

除了外部定义以外，也可以在类内部定义成inline函数：

```cpp
template<typename T>
class Stack{
    ...
    void push(T const &elem){
        elems.push_back(elem);	// append copy of passed elem
    }
  ...
};
```

## 类模板Stack的使用

C++17以前，使用类模板必须显式地指定模板参数。

```cpp
#include "stack1.hpp"
#include <iostream>
#include <string>

int main()
{
    Stack<int>	intStack;			// stack of ints
  	Stack<std::string> stringStack;	// stack of strings
  
  	// manipulate int stack
  	intStack.push(7);
  	std::cout << intStack.top() << '\n';
  
  	// manipulate string stack
  	stringStack.push("hello");
  	std::cout << stringStack.top() << '\n';
  	stringStack.pop();
}
```

只有被调用到的模板函数才会实例化，这可以省空间和时间。

上例中，默认构造器，`push()`和`top()`会为`int`和`std::string`两种类型都实例化。`pop()`只为后者实例化。如果类模板拥有静态成员，它们也会为各类型都实例化一个函数。

## 类模板的局部用法

模板参数类型不必支持所有类模板内部用到的操作，只需要支持自己所用到的操作即可。

```cpp
template<typename T>
class Stack{
    ...
    void printOn(std::ostream& strm) const{
        for(T const & elem : elems){
            strm << elem << ' ';	// call << for each element
        }
    }
};
```

尽管如此，不支持<<操作符的T也是可以使用的。

```cpp
Stack<std::pair<int, int>> ps;	// note: std::pair<> has no operator<< defined
ps.push({4, 5});	// OK
ps.push({6, 7});	// OK
std::cout << ps.top().first << '\n';	// OK
std::cout << ps.top().second << '\n';	// OK
```

然而一旦使用了:

```cpp
ps.printOn(std::cout);	// ERROR: operator<< not supported for element type
```

代码就会报错。

### 思想

如何知晓模板实例化时需要支持哪些操作呢？对C++标准库来说，它依赖于这样的思想：随机访问迭代器和默认可构造。

直到C++17标准，这些思想也没有标准化，仅仅有些文案描述。

C++11以后，可以用`static_assert`关键字和一些预定义类型traits来检查基本的限制。

```cpp
template<typename T>
class C
{
    static_assert(std::is_default_constructible<T>::value, "Class C requires default-constructible elements");
  	...
};
```

即使没有`static_assert`，编译也会失败，只是错误信息就太臃肿了，难以甄别。

## 友元

与其用`printOn()`打印堆栈内容，不如为stack也实现一个`operator<<`。

```cpp
template<typename T>
class Stack{
    ...
    void printOn(std::ostream& strm) const{
        ...
    }
  	friend std::ostream& operator<< (std::ostream& strm, Stack<T> const &s){
        s.printOn(strm);
      	return strm;
    }
};
```

如果先声明一个友元函数，随后在外部进行定义，就会把事情搞复杂。我们有两种选择：

1. 隐式地声明一个新的函数模板，它必须使用一个不同的模板参数，比如：

   ```cpp
   template<typename T>
   class Stack{
       ...
       template<typename U>
       friend std::ostream& operator<< (std::ostream&, Stack<U> const &);
   };
   ```

2. 前导声明

   ```cpp
   template<typename T>
   class Stack;
   template<typename T>
   std::ostream& operator<< (std::ostream&, Stack<T> const&);
   ```

   此时可以将函数声明为友元：

   ```cpp
   template<typename T>
   class Stack{
       ...
       friend std::ostream& operator<< <T>(std::ostream&, Stack<T> const&);
   };
   ```

这部分我看的不是很懂，以后遇到实例再分析吧。

## 类模板特化

特化就是指定具体模板参数类型。类模板被特化，成员函数都要连坐。

```cpp
template<>
class Stack<std::string>{
  ...
};
```

成员函数跟随着定义成普通成员函数：

```cpp
void Stack<std::string>::push(std::string const& elem)
{
    elems.push_back(elem);	// append copy of passed elem
}
```

为`std::string`定义的完整特化类模板实例：

```cpp
#include "stack1.hpp"
#include <deque>
#include <string>
#include <cassert>

template<>
class Stack<std::string>{
private:
	std::deque<std::string> elems;	// elements
public:
	void push(std::string const&);	// push elements
	void pop();					   // pop element
	std::string const &top() const;	//return top element
	bool empty() const{	// return whether the stack is empty
      	return elems.empty();
	}
};

void Stack<std::string>::push(std::string const& elem)
{
    elems.push_back(elem);	// append copy of passed elem
}

void Stack<std::string>::pop()
{
    assert(!elems.empty());
  	elems.pop_back();	// remove last element
}

std::string const& Stack<std::string>::top() const
{
    assert(!elems.empty());
  	return elems.back();	// return copy of last element
}
```

## 偏(局部)特化

类模板也可以局部特化。对特定场景提供特殊实现。

```cpp
#include "stack1.hpp"

// partial specialization of class Stack<> for pointers:
template<typename T>
class Stack<T*>{
private:
  	std::vector<T*> elems;	// elements
public:
  	void push(T*);	// push elements
  	T* pop();		// pop element
  	T* top() const;	// return top element
  	bool empty() const{	// return whether the stack is empty
        return elems.empty();
    }
};
template<typename T>
T* Stack<T*>::pop()
{
    assert(!elems.empty());
  	T* p = elems.back();
  	elems.pop_back();	// remove last element
  	return p;	// and return it (unlike in the general case)
}
template<typename T>
T* Stack<T*>::top() const
{
    assert(!elems.empty());
  	return elems.back();	// return copy of last element
}
```

接口有细微的不同。

```cpp
Stack<int*> ptrStack;	// stack of pointers (special implementation)
ptrStack.push(new int{42});
std::cout << *ptrStack.top() << '\n';
delete ptrStack.pop();
```

**多重参数的偏特化**

类模板也可以特化多重模板参数间的关系。

```cpp
template<typename T1, typename T2>
class MyClass{
  ...
};

// the following partial specializations are possible:
// partial specialization: both template parameters have same type
template<typename T>
class MyClass<T, T>{
    ...
};

// partial specialization: second type is int
template<typename T>
class MyClass<T, int>{
    ...
};

// partial specialization: both template parameters are pointer types
template<typename T1, typename T2>
class MyClass<T1*, T2*>{
  ...
};
```

使用匹配策略：

```cpp
MyClass<int, float> mif;	// uses MyClass<T1, T2>
MyClass<float, float> mff;	// uses MyClass<T, T>
MyClass<float, int> mfi;	// uses MyClass<T, int>
MyClass<int *, float *> mp;	// uses MyClass<T1*, T2*>
```

如果有两个以上的偏特化均匹配，那么声明将报错。

## 默认类模板参数

类模板也可以定义默认模板参数。

```cpp
#include <vector>
#include <cassert>
template<typename T, typename Cont = std::vector<T>>
class Stack{
private:
  	Cont elems;	// elements
public:
	void push(T const& elem);	// push element
	void pop();				// pop element
	T const &top() const;		// return top element
	bool empty() const{		// return whether the stack is empty
      	return elems.empty();
	}
}

template<typename T, typename Cont>
void Stack<T, Cont>::push(T const& elem)
{
    elems.push_back(elem);	// append copy of passed elem
}
template<typename T, typename Cont>
void Stack<T, Cont>::pop()
{
    assert(!elems.empty());
    elems.pop_back();		// remove last element
}
template<typename T, typename Cont>
T const& Stack<T,Cont>::top() const
{
    assert(!elems.empty());
  	return elems.back();	// return copy of last element
}
```

如此定义了默认模板参数`Cont`类型为`std::vector<T>`，使用时可以省略：

```cpp
#include "stack3.hpp"
#include <iostream>
#include <deque>

int main()
{
    // stack of ints
  	Stack<int> intStack;
  
  	// stack of doubles using a std::deque<> to manage the elements 
  	Stack<double,std::deque<double>> dblStack;
  	
  	// manipulate int stack
  	intStack.push(7);
  	std::cout << intStack.top() << '\n';
  	intStack.pop();
  
  	// manipulate double stack
  	dblStack.push(42.42);
  	std::cout << dblStack.top() << '\n';
  	dblStack.pop();
}
```

## 类型别名

C++的语法过于啰嗦，所以通过alias定义可以简化语法。

1. 使用`typedef`。

   ```cpp
   typedef Stack
   tpedef Stack<int> IntStack;	// typedef
   void foo(IntStack const& s);	// s is stack ofints
   IntStack istack[10];	// istack is array of 10 stacks of ints
   ```

   俗称的typedef-name。

2. 使用C++11引入的`using`关键字。

   ```cpp
   using IntStack = Stack<int>;	// alias declaration
   void foo(IntStack const& s);	// s is stack of ints
   IntStack istack[10];			// istack is array of 10 stacks of ints
   ```

   称作alias declaration。与`typedef`一样都只是定义了别名，而不是定义了新的类型。`using`语法简洁更为好用。

**模板别名**

C++之后引入

```cpp
template<typename T>
using DequeStack = Stack<T, std::deque<T>>;
```

如此，`DequeStack<int>`和`Stack<int, std::deque<int>>`就表示同一个类型。

**成员类型模板别名**

```cpp
struct MyType{
    using iterator = ...;
  	...
};

template<typename T>
using MyTypeIterator = typename MyType<T>::iterator;
```

这一技巧被C++14为标准库所用到，我们此前看过两个相似的例子。

`std::add_const_t<T>	// since C++14`用以替代`typename std::add_const<T>::type	// since C++11`。

```cpp
namespace std{
    template<typename T>
  	using add_const_t = typename add_const<T>::type;
}
```

## 类模板参数推导

C++17以前，必须为类模板传递所有的模板参数类型（哪怕有默认类型）。此后，该限制条件变得宽松了许多，如果构造器有能力去推导所有的模板参数时，就可以不必显式地指定。

```cpp
Stack<int> intStack1;	// stack of strings
Stack<int> intStack2 = intStack1;	// OK in all versions
Stack intStack3 = intStack1;	// OK since C++17
```

类模板参数推导，我是不会使用的。。。也没见什么人用。。。

## 模板化集合

集合类(不含用户提供的、显式的或继承的构造器、私有或保护权限非静态数据成员、虚函数、虚私有或保护继承基类的class/struct)也可以有模板。

```cpp
template<typename T>
struct ValueWithComment{
    T value;
  	std::string comment;
};

ValueWithComment<int> vc;
vc.value = 42;
vc.comment = "initial value";
```

## 总结

- 类模板是一族类，按模板参数留白。
- 把参数传给类模板，类模板会实例化该具体类。
- 对类模板来说，实例化类中只有被调用到的成员函数会被实例化。
- 可以为特定类型特化类模板。
- 可以为特定类型偏特化类模板。
- C++17以后类模板参数可以从构造器自动推导。
- 可以定义集合类模板。
- 模板类型的调用参数如果按值传递会decay。
- 模板仅可以声明和定义在全局/命名空间范围或者在类声明内部。