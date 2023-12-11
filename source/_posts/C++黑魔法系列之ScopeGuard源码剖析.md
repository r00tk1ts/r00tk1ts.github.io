---
title: C++黑魔法系列之ScopeGuard源码剖析
date: 2023-12-01 17:32:06
tags: [cpp, cpp-templates]
category: 源码剖析
---

众所周知，想要在C++中写出通用的框架、组件代码并不简单，一来C++本身庞然大物包罗万象，大部分开发者并不了解像是”茴字有四种写法“这种语言律师津津乐道的课题，对于晦涩难懂的模板元更是谈之色变，二来是历史悠久，在发展过程中为了向前兼容，遗留了各种特例特办的技术债，导致系统臃肿不堪。因此，尽管C++生态相当茂盛，但权威的第三方库却屈指可数（甚至其标准库都是风风雨雨缝缝补补，~~偷得人家boost就剩个底裤(bushi)~~）。

本篇文章我们来深度剖析大名鼎鼎的Facebook folly库所实现的ScopeGuard。管中窥豹，可见一斑。

<!-- more -->

# C++黑魔法系列之ScopeGuard源码剖析
ScopeGuard是一种通用的RAII思想的实现。它可以保证某个函数在离开当前作用域时一定会被执行，基于这一语义，我们可以方便的编写出异常安全的代码，也可以实现像go语言中defer那样的语法糖。Facebook大名鼎鼎的folly库实现了一种通用的ScopeGuard实现，它被广泛应用于各种基础组件之上，通过模拟语法糖的形式让编写异常安全的代码变得简单。

本文则是对其源码实现的解剖，但与常规解析有所不同的是，我不会一上来就逐行逆向去讨论每一处代码的细节，而是提取出全局设计思想，一步一步按部就班实现所需的能力，最终reach源代码。

## 异常触发的清洁工：makeGuard
在ScopeGuard的源码中，开局就给了一个应用场景案例：

```cpp
 void User::addFriend(User& newFriend) {
    // add the friend to memory
    friends_.push_back(&newFriend);
 
    // If the db insertion that follows fails, we should
    // remove it from memory.
    auto guard = makeGuard([&]() { friends_.pop_back(); });
 
    // this will throw an exception upon error, which
    // makes the ScopeGuard execute UserCont::pop_back()
    // once the Guard's destructor is called.
    db_->addFriend(GetName(), newFriend.GetName());
 
    // an exception was not thrown, so don't execute
    // the Guard.
    guard.dismiss();
}
```

这个案例源于Andrie大佬的这篇文章：[Generic: Change the Way You Write Exception-Safe Code — Forever](https://drdobbs.com/cpp/generic-change-the-way-you-write-excepti/184403758)，里面着重讲述了在C++中要如何简化编写异常安全的代码。

在函数`addFriend`执行过程中，可能会抛出某个异常，而从业务逻辑视角来看，我们希望一旦有异常抛出，那么就能够触发某些清理工作的执行。在上述代码中，传递给`makeGuard`的lambda函数就是清理任务，代码中的`db_->addFriend(...)`可能会抛异常，此时就需要执行清理任务lambda，而如果在执行过程中并未抛出任何异常，则清理任务不会执行。

对于这个例子，可能读者会比较奇怪，为什么要用这么”别扭“的写法呢？如果我写成：

```cpp
void User::AddFriend(User& newFriend)
{
    // Add the new friend to the database
    pDB_->AddFriend(GetName(), newFriend.GetName());
    // Add the new friend to the vector of friends
    friends_.push_back(&newFriend);
}
```

不是直接就大功告成？一旦`pDB_->AddFriend(GetName(), newFriend.GetName());`抛出了异常，那么根据C++的异常控制机理，下面的`friends_.push_back(&newFriend);`不就恰恰不会执行了吗？

你说得对，但是我是说如果，`pDB_->AddFriend(GetName(), newFriend.GetName())`在成功修改了外部数据库的信息后，`friends_.push_back(&newFriend)`抛出了异常，阁下又该如何应对？可能大部分业务都没这么敏感，往往也有一些纠错逻辑去补偿，但既然是例子，我们暂且钻一下牛角尖。

### try-catch 暴力解决
当然，对于这段简单的代码，看起来并不需要ScopedGuard这么高级的机制，我们只需要暴力编写成：

```cpp
void User::AddFriend(User& newFriend) {
    friends_.push_back(&newFriend);
    try {
        pDB_->AddFriend(GetName(), newFriend.GetName());
    } catch (...) {
        friends_.pop_back();
        throw;
    }
}
```

一样可以达成同样的效果。但是这种异常安全的代码，也带来了额外的开销：我们的程序变得臃肿，随着可能抛出异常代码块的增多，我们需要编写的try-catch块儿也更复杂，甚至还有需要嵌套的代码块儿，导致代码可维护性差，甚至晦涩难懂。

### RAII
那么，有没有什么更优雅的手法呢？本质上`friends_.push_back(&newFriend);`是一种资源泄露，我们可以借助RAII的思想来避免泄露：

```cpp
class VectorInserter
{
public:
    VectorInserter(std::vector<User*>& v, User& u)
    : container_(v), commit_(false)
    {
        container_.push_back(&u);
    }
    void Commit() throw()
    {
        commit_ = true;
    }
    ~VectorInserter()
    {
        if (!commit_) container_.pop_back();
    }
private:
    std::vector<User*>& container_;
    bool commit_;
};

void User::AddFriend(User& newFriend)
{
    VectorInserter ins(friends_, &newFriend);
    pDB_->AddFriend(GetName(), newFriend.GetName());
    // Everything went fine, commit the vector insertion
    ins.Commit();
}
```

这下我们通过引入`VectorInserter ins`这个局部变量来实现RAII，在代码末尾处我们通过执行`ins.Commit()`来设置其成员标记，使其在离开函数作用域之后析构之时，不去执行清理代码`container_.pop_back();`。而如果前面的代码抛出了异常，那么`ins.Commit();`将没有机会得到执行，此时`commit`就是初始值`false`，在函数作用域退出时依然会触发`ins`的析构，此时就执行了清理代码。

通过对RAII思想的利用，这下子`AddFriend`的异常安全逻辑编写变得简洁多了。但是，由于我们需要编写伴生的RAII class，从整体代码设计来看依然不够优雅。另一方面，在C++中想要设计一个功能完备又正确的类可没这么简单，像是上面的`VectorInserter`，如何优雅的处理好Big Five，也是相当麻烦。

### ScopeGuard Tool
RAII的思想没错，但是需要开发者自己设计伴生类代价太大了，我们自己做一下封装，提供最精致的接口供开发者使用。考虑到资源清理的手段多种多样，比如最通用的手法是调用某个函数对象，为了分离flag和具体手段，在设计上可以将类拆分成层级结构：

```cpp
class ScopeGuardImplBase {
  public:
    void dismiss() noexcept { dismissed_ = true; }
  protected:
     ScopeGuardImplBase(bool dismissed = false) noexcept : dismissed_(dismissed) {}
  protected:
    bool dismissed_;
};

template<typename FunctionType>
class ScopeGuardImpl : public ScopeGuardImplBase {
  public:
    explicit ScopeGuardImpl(FunctionType& fn) : function_(std::as_const(fn)) { }
    explicit ScopeGuardImpl(const FunctionType& fn) : function_(fn) { }
    explicit ScopeGuardImpl(FunctionType&& fn) : function_(std::move(fn)) { }

    ScopeGuardImpl(ScopeGuardImpl&& other) : function_(std::move(other.function_)) {
      dismissed_ = std::exchange(other.dismissed_, true);
    }

    ~ScopeGuardImpl() {
      if (!dismissed_) {
        function_();
      }
    }
  
  private:
    FunctionType function_;
};
```

而对外提供的makeGuard可以设计成这样：

```cpp
template <typename F>
ScopeGuardImpl<std::decay_t<F>> makeGuard(F&& f) {
  return ScopeGuardImpl<std::decay_t<F>>(std::forward<F>(f));
}
```

嗯，看起来这样就行了，我们写个例子执行一下试试。

#### 风调雨顺，岁月静好
当一切都朝着你期望的方向推进时，这代码简直泰裤辣：

```cpp
void unexpected() {throw std::runtime_error("oops...");}

void expected() { std::cout << "everything goes well" << std::endl; }

void func(std::vector<int>& friends) {
  friends.push_back(1);

  auto guard = makeGuard([&](){friends.pop_back();});
  expected();
  unexpected();
  guard.dismiss();
}

int main() {
  std::vector<int> friends;
  try {
    func(friends);
  }catch(std::exception &ex) {
    std::cerr << "exception: " << ex.what() << std::endl;
  }

  std::cout << "size=" << friends.size() << std::endl;
  return 0;
}
```

由于`unexpected`在执行期间抛出了某个异常，最终`friends`中的成员`1`被清理。而当我们注释掉`unexpected`的调用后，清理工作不会执行，因为`guard`被成功`dismiss`了。

#### 异常，还是异常
这样就够了吗？不，ScopeGuard的真正实现远比上面的要复杂得多。

我们传递给`makeGuard`的是一个lambda对象，事实上，我们可以传递任意一个可调用对象，在C++中，它可以是一个函数、一个lambda或者是一个重载了`operator()`的`class/struct`。另一方面，`makeGuard`可以传入一个左值引用或是右值引用，于此同时，它还可以有CV限定(`const`, `volatile`)，以满足日常编程所有场景的需求。

上述实现代码中，面对lvalue reference、const lvalue reference和rvalue reference，各自实现了构造器(以下简称为ctor)。其中前两者会触发`FunctionType`的copy ctor，而末者则会触发`FunctionType`的move ctor(如果有的话)。到此，就出现了第一个问题：如果`FunctionType`的copy/move ctor抛了异常，我们又该如何（一般来说，move ctor在设计上是不会抛异常的，但C++是自由的，它允许你发癫）？

`makeGuard`能够保证垃圾被清理的前提，在于要成功构造出`ScopeGuardImpl`对象，而`FunctionType`的ctor一旦会抛异常，那么我们的构造就会失败，此时，垃圾清理函数当然得不到执行。

比如：

```cpp
class Functor {
  public:
    explicit Functor(std::vector<int>& friends) : friends_(friends) {}

    Functor(const Functor &rhs) : friends_(rhs.friends_) { 
      // 这里模拟一下，抛个异常
      throw std::runtime_error("throw exception in Functor copy ctor...");
    }

    void operator()() { friends_.pop_back(); }

  private:
    std::vector<int>& friends_;
};

void unexcepted() { throw std::runtime_error("oops..."); }

void func(std::vector<int> &friends) {
    friends.push_back(1);
    Functor f(friends);
    // 实参是左值引用，此时会调用到Functor的copy ctor，触发异常
    auto guard = makeGuard(f);
    unexcepted();
    guard.dismiss();
}

int main() {
  std::vector<int> friends;
  try {
    func(friends);
  }catch(std::exception &ex) {
    std::cerr << "exception: " << ex.what() << std::endl;
  }

  std::cout << "size=" << friends.size() << std::endl;
  return 0;
}
```

执行结果：
```shell
exception: throw exception in Functor copy ctor...
size=1
```

可以看到代码在执行过程中抛出了异常，但是垃圾并没有被回收。一个小小的copy ctor异常就破了我们的招。

### 异常安全的ScopeGuard
那怎么办呢？有没有什么办法可以兜住ctor的异常呢？有，而且甚秒。

Folly库在实现ScopeGuardImpl时，引入了一个叫`makeFailsafe`的成员函数，它通过在构造`ScopeGuardImpl`对象期间，嵌套构造另一个`ScopeGuardImpl`对象，并借助模板元编程中一种叫做tag-dispatch的技术对可能会抛出异常的`FunctionType`用`std::reference_wrapper`做了二次包裹，利用`std::reference_wrapper` nothrow的ctor，避免了`ScopeGuardImpl`对象构造期间可能抛出的异常。

听起来相当套娃，事实上它的实现也确实复杂，我们来看一下代码：

待续。。。