---
title: C++黑魔法系列之ScopeGuard源码剖析
date: 2023-12-01 17:32:06
tags: [cpp, cpp-templates]
category: 源码剖析
---

众所周知，想要在C++中写出通用的框架、组件代码并不简单，一来C++本身庞然大物包罗万象，大部分开发者并不了解像是”茴有四种写法“这种语言律师津津乐道的课题，对于晦涩难懂的模板元更是谈之色变，二来是历史悠久，在发展过程中为了向前兼容，遗留了各种特例特办的技术债，导致系统臃肿不堪。因此，尽管C++生态相当茂盛，但权威的第三方库却屈指可数（甚至其标准库都是风风雨雨缝缝补补，~~偷得人家boost就剩个底裤~~）。阅读优秀的开源代码是提升代码水平的捷径，本篇文章我们来深度剖析大名鼎鼎的Facebook folly库所实现的ScopeGuard。管中窥豹，可见一斑。

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
那么，有没有什么更优雅的手法呢？本质上`friends_.push_back(&newFriend);`是一种资源泄露，我们可以借助RAII(Resource Acquisition Is Initialization)的思想来避免泄露：

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

> 注：Big Five并非C++官方的说法，它是指destructor, copy constructor, copy assignment, move constructor, move assignment这五个类中非常重要的成员函数。这一说法最早它来源于台湾知名大神侯捷的译作，熟悉C++的老玩家自当会心一笑。

### ScopeGuard Tool
RAII的思想没错，但是需要开发者自己设计伴生类代价太大了，我总不能为了每一段这样的逻辑都编写一个伴生类吧，那岂不是违背了初心又本末倒置。因此，我们需要对伴生类的实现尽量通用化，提供最精致的接口供开发者使用。考虑到资源清理的手段多种多样，比如最通用的手法是调用某个函数对象，为了可以面向具体手段做扩展，`ScopeGuard`在设计上可以将类拆分成层级结构：

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

而对外提供的`makeGuard`可以封装一下：

```cpp
template <typename F>
ScopeGuardImpl<std::decay_t<F>> makeGuard(F&& f) {
  return ScopeGuardImpl<std::decay_t<F>>(std::forward<F>(f));
}
```

嗯，看起来这样就成了，我们写个例子执行一下试试。

#### 风调雨顺，岁月静好
当一切都朝着你期望的方向推进时，这代码简直泰裤辣：

```cpp
void unexpected() {throw std::runtime_error("oops...");}

void expected() { std::cout << "everything goes well" << std::endl; }

void func(std::vector<int>& friends) {
  friends.push_back(1);

  auto guard = makeGuard([&](){friends.pop_back();});
  expected();
  unexpected(); // 此处会抛出异常，导致下一行代码得不到执行
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

我们传递给`makeGuard`的是一个lambda对象，事实上，我们可以传递任意一个可调用对象，在C++中，它可以是一个函数、一个lambda、一个`std::function<>`、甚至是一个重载了`operator()`的`class/struct`。另一方面，`makeGuard`可以传入一个左值引用或是右值引用，于此同时，它还可以有CV限定(`const`, `volatile`)，以满足日常编程所有场景的需求。

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

可以看到代码在执行过程中抛出了异常，但是垃圾并没有被回收。**防不胜防啊，一个小小的copy ctor异常就破了我们的招。**

### 异常安全的ScopeGuard
那么，有没有什么办法可以兜住ctor的异常呢？答案是有，而且甚秒。

Folly库在实现ScopeGuardImpl时，引入了一个叫`makeFailsafe`的成员函数，它通过在构造`ScopeGuardImpl` A对象期间，嵌套构造另一个`ScopeGuardImpl` B对象，并借助模板元编程中一种叫做tag-dispatch的技术对可能会抛出异常的`FunctionType`用`std::reference_wrapper`做了二次包裹，利用`std::reference_wrapper`的`noexcept` ctor，确保了B在构造期间不会抛出异常，从而能够在构造A期间抛出异常的时刻，接管垃圾清理器。

这听起来就相当套娃，事实上它的实现更加复杂，我们来升级一下代码：

```cpp
class ScopeGuardImplBase {
  public:
    void dismiss() noexcept { dismissed_ = true; }
  protected:
    ScopeGuardImplBase(bool dismissed = false) noexcept : dismissed_(dismissed) {}
      
    static ScopeGuardImplBase makeEmptyScopeGuard() noexcept {
      return ScopeGuardImplBase{};
    }
  protected:
    bool dismissed_;
};

template<typename FunctionType>
class ScopeGuardImpl : public ScopeGuardImplBase {
  public:
    // 利用trait做tag-dispatch
    explicit ScopeGuardImpl(FunctionType& fn) 
      : ScopeGuardImpl(std::as_const(fn), makeFailsafe(std::is_nothrow_copy_constructible<FunctionType>{}, &fn)) { }

    explicit ScopeGuardImpl(const FunctionType& fn)
      : ScopeGuardImpl(fn, makeFailsafe(std::is_nothrow_copy_constructible<FunctionType>{}, &fn)) { }

    explicit ScopeGuardImpl(FunctionType&& fn) 
      : ScopeGuardImpl(std::move(fn), makeFailsafe(std::is_nothrow_move_constructbile<FunctionType>{}, &fn)) { }

    ScopeGuardImpl(ScopeGuardImpl&& other) : function_(std::move(other.function_)) {
      dismissed_ = std::exchange(other.dismissed_, true);
    }

    ~ScopeGuardImpl() {
      if (!dismissed_) {
        function_();
      }
    }
  private:
    static ScopeGuardImplBase makeFailsafe(std::true_type, const void*) noexcept {
      return makeEmptyScopeGuard();
    }

    template <typename Fn>
    static auto makeFailsafe(std::false_type, Fn* fn) noexcept
        -> ScopeGuardImpl<decltype(std::ref(*fn))> {
      return ScopeGuardImpl<decltype(std::ref(*fn))>{std::ref(*fn)};
    }

    template <typename Fn>
    explicit ScopeGuardImpl(Fn&& fn, ScopeGuardImplBase&& failsafe)
      : ScopeGuardImplBase{}, function_(std::forward<Fn>(fn)) {
      failsafe.dismiss();
    }
  private:
    FunctionType function_;
};

template <typename F>
ScopeGuardImpl<std::decay_t<F>> makeGuard(F&& f) {
  return ScopeGuardImpl<std::decay_t<F>>(std::forward<F>(f));
}
```

我们对构造器进行了改造，若传入的是lvalue reference，则根据其copy ctor是否会抛异常而调用不同的makeFailsafe，若传入的是rvalue reference，则根据其move ctor是否会抛异常而调用不同的makeFailsafe。
makeFailsafe的设计是一种tag-dispatch的技巧，通过类型去match不同的重载函数，最终再经重载决议确定，也算是模板元编程里一种经典if-else手法了。

我们以传入左值作为例子，看一下在构造一个`ScopeGuardImpl`对象时，经历的path：

对于不会抛出异常的copy ctor，流程是一目了然的：

```
=> 匹配构造器：ScopeGuardImpl(const FunctionType& fn) : ScopeGuardImpl(fn, makeFailsafe(std::is_nothrow_copy_constructible<FunctionType>{}, &fn)) { }
    => 转发到private构造器：template<typename Fn> ScopeGuardImpl(Fn&& fn, ScopeGuardImplBase&& failsafe)，其中fn是转发引用，failsafe是右值引用
    => 由于makeFailsafe匹配到std::true_type的版本，返回一个默认构造的ScopeGuardImplBase{}临时对象，作为failsafe参数传递
    => 此时Fn类型被推导为const FunctionType&，形参fn由于引用折叠也被推导为const FunctionType&
      => : ScopeGuardImplBase{}, function_(std::forward<Fn>(fn)) {failsafe.dismiss();}
      => 成员初始化列表：基类默认构造，function_成员通过完美转发接收fn，此时会触发FunctionType的copy ctor来初始化对象
      => 构造出的临时对象failsafe调用dismiss()，避免析构时执行execute
    => ScopeGuardImpl对象构建完毕，大功告成
```

而对于可能会抛出异常的copy ctor，则要复杂得多：

```
=> 匹配构造器：ScopeGuardImpl(const FunctionType& fn) : ScopeGuardImpl(fn, makeFailsafe(std::is_nothrow_copy_constructible<FunctionType>{}, &fn)) { }
  => 转发到private构造器：template<typename Fn> ScopeGuardImpl(Fn&& fn, ScopeGuardImplBase&& failsafe)，其中fn是转发引用，failsafe是右值引用
  => 由于makeFailsafe会匹配到std::false_type的版本，返回的是另一个全新构建的ScopeGuardImpl临时对象：
  => return ScopeGuardImpl<decltype(std::ref(*fn))>{std::ref(*fn)};
    => 通过std::ref包裹*fn，来构建出一个std::reference_wrapper<FunctionType>临时对象
    => 使用该临时对象显式实例化构建另一个ScopeGuardImpl<std::reference_wrapper<FunctionType>>对象
      => 匹配调用的是右值引用版本的ctor：ScopeGuardImpl(FunctionType&& fn) : ScopeGuardImpl(std::move_if_noexcept(fn), makeFailsafe(std::is_nothrow_move_constructbile<FunctionType>){}, &fn) { }
      => 此时的ScopeGuardImpl类模板实例化的是一个全新版本，其模板参数为std::reference_wrapper<原始FunctionType>
      => 由于std::reference_wrapper的移动构造器在设计上已标记成了nothrow，因此，当再进行构造器转发时，makeFailsafe匹配的就是std::true_type的版本：
        => 转发到private构造器：template<typename Fn> ScopeGuardImpl(Fn&& fn, ScopeGuardImplBase&& failsafe)，其中fn是转发引用，failsafe是右值引用
          => failsafe又是一个默认构造的ScopeGuardImplBase{}临时对象
          => Fn被推导为std::reference_wrapper<原始FunctionType>，参数fn被推导为std::reference_wrapper<原始FunctionType>&&右值引用型
          => : ScopeGuardImplBase{}, function_(std::forward<Fn>(fn)) {failsafe.dismiss();}
            => 此时的function_实际上是一个std::reference_wrapper<原始FunctionType>类型，这里经过std::reference_wrapper的move ctor最终构造了一个接收了包裹原始Functor的std::reference_wrapper
            => 临时对象failsafe（此时是一个ScopeGuardImplBase）调用dismiss()，避免析构时执行execute
  => 到此，临时对象ScopeGuardImpl<std::reference_wrapper<FunctionType>>构建完毕，并返回给上游作为failsafe参数
  => 用fn和临时对象调用private构造器：
    => template<typename Fn> ScopeGuardImpl(Fn&& fn, ScopeGuardImplBase&& failsafe) : ScopeGuardImplBase{}, function_(std::forward<Fn>(fn)) { failsafe.dismiss(); }
    => 此时的Fn被推导为原始FunctionType，fn是const FunctionType &, failsafe是包裹了Functor对象的临时对象。
    => 若function_(std::forward<Fn>(fn))在执行时没有抛异常，则临时对象的使命完成，它会调用到failsafe.dismiss()，即在其析构时不会调用我们的Functor对象。
      => 此时Functor是否要调用的权利，正式交棒给了外层对象ScopeGuardImpl<FunctionType>。
    => 若function_(std::forward<Fn>(fn))在执行时其copy ctor确实抛了异常，那么最外层的ScopeGuardImpl<FunctionType>对象就不会构造成功，构造器内的failsafe.dismiss()也得不到执行。
      => 此时，尽管我们并没能成功构造出ScopeGuardImpl<FunctionType>对象，但最开始传入的Functor还是能够被执行。
      => 这是由于没能执行failsafe.dismiss()操作的内层临时对象的dismiss_仍然是false，而在其析构时Functor依然得到了执行。
```

可以看到代码的实现非常的绕，但实际上核心思想就是借助`std::reference_wrapper`这个引用包裹器具有nothrow的move ctor这一特点来处理那些原本有可能会抛异常的copy/move ctor的`FunctionType`。

#### log补齐代码：
我们打印一些log，并分别用左值和右值两个案例来测试一下。

```cpp
// scope.h
#include <iostream>
#include <type_traits>
#include <functional>
#include "type_name.h"

class ScopeGuardImplBase {
  public:
    void dismiss() noexcept { dismissed_ = true; }
  protected:
    ScopeGuardImplBase(bool dismissed = false) noexcept : dismissed_(dismissed) { std::cout << "construct ScopeGuardImplBase, dismissed: " << dismissed << std::endl;}

    static ScopeGuardImplBase makeEmptyScopeGuard() noexcept {
      return ScopeGuardImplBase{};
    }
  protected:
    bool dismissed_;
};

template<typename FunctionType>
class ScopeGuardImpl : public ScopeGuardImplBase {
  public:
    explicit ScopeGuardImpl(FunctionType& fn)
        : ScopeGuardImpl(std::as_const(fn), makeFailsafe(std::is_nothrow_copy_constructible<FunctionType>{}, &fn)) {
            std::cout << "const lvalue ref ctor, FunctionType: " << type_name<FunctionType>() << ", fn type: "
                << type_name<decltype(fn)>() << std::endl;
        }

    explicit ScopeGuardImpl(const FunctionType& fn)
        : ScopeGuardImpl(fn, makeFailsafe(std::is_nothrow_copy_constructible<FunctionType>{}, &fn)) {
            std::cout << "lvalue ref ctor, FunctionType: " << type_name<FunctionType>() << ", fn type: "
                << type_name<decltype(fn)>() << std::endl;
        }

    explicit ScopeGuardImpl(FunctionType&& fn)
        : ScopeGuardImpl(std::move(fn), makeFailsafe(std::is_nothrow_move_constructible<FunctionType>{}, &fn)) {
            std::cout << "rvalue ref ctor, FunctionType: " << type_name<FunctionType>() << ", fn type: "
                << type_name<decltype(fn)>() << std::endl;
        }

    ScopeGuardImpl(ScopeGuardImpl&& other) : function_(std::move(other.function_)) {
      dismissed_ = std::exchange(other.dismissed_, true);
    }

    ~ScopeGuardImpl() {
      std::cout << "dtor, FunctionType: " << type_name<FunctionType>() << ", function_ type: " << type_name<decltype(function_)>() << std::endl;
      if (!dismissed_) {
        function_();
      }
    }
  private:
    static ScopeGuardImplBase makeFailsafe(std::true_type, const void*) noexcept {
      std::cout << "true_type makeFailsafe" << std::endl;
      return makeEmptyScopeGuard();
    }

    template <typename Fn>
    static auto makeFailsafe(std::false_type, Fn* fn) noexcept
        -> ScopeGuardImpl<decltype(std::ref(*fn))> {
      std::cout << "false_type makeFailsafe, Fn type: " << type_name<Fn>() << ", fn type: " << type_name<decltype(fn)>() << std::endl;
      return ScopeGuardImpl<decltype(std::ref(*fn))>{std::ref(*fn)};
    }

    template <typename Fn>
    explicit ScopeGuardImpl(Fn&& fn, ScopeGuardImplBase&& failsafe)
      : ScopeGuardImplBase{}, function_(std::forward<Fn>(fn)) {
      std::cout << "private ctor, Fn type: " << type_name<Fn>() << ", fn type: " << type_name<decltype(fn)>() << std::endl;
      failsafe.dismiss();
    }
  private:
    FunctionType function_;
};

template <typename F>
ScopeGuardImpl<std::decay_t<F>> makeGuard(F&& f) {
  return ScopeGuardImpl<std::decay_t<F>>(std::forward<F>(f));
}

// type_name.h
template <class T>
constexpr
std::string_view
type_name()
{
    using namespace std;
#ifdef __clang__
    string_view p = __PRETTY_FUNCTION__;
    return string_view(p.data() + 34, p.size() - 34 - 1);
#elif defined(__GNUC__)
    string_view p = __PRETTY_FUNCTION__;
#  if __cplusplus < 201402
    return string_view(p.data() + 36, p.size() - 36 - 1);
#  else
    return string_view(p.data() + 49, p.find(';', 49) - 49);
#  endif
#elif defined(_MSC_VER)
    string_view p = __FUNCSIG__;
    return string_view(p.data() + 84, p.size() - 84 - 7);
#endif
}
```

这里附带一个C++最好用的类型名称打印器：`type_name`

#### 用例1：左值引用
```cpp
#include <iostream>
#include <vector>
#include <type_traits>
#include <functional>
#include "scope.h"

class Functor {
  public:
    explicit Functor(std::vector<int>& friends) : friends_(friends) {}

    Functor(const Functor &rhs) : friends_(rhs.friends_) { throw std::runtime_error("throw exception in Functor copy ctor..."); }

    void operator()() { friends_.pop_back(); }

  private:
    std::vector<int>& friends_;
};

void func(std::vector<int> &friends) {
    friends.push_back(1);
    Functor f(friends);
    auto guard = makeGuard(f);
    // some code that will not throw ...
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

输出结果：

```shell
false_type makeFailsafe, Fn type: Functor, fn type: Functor *
true_type makeFailsafe
construct ScopeGuardImplBase, dismissed: 0
construct ScopeGuardImplBase, dismissed: 0
private ctor, Fn type: std::reference_wrapper<Functor>, fn type: std::reference_wrapper<Functor> &&
rvalue ref ctor, FunctionType: std::reference_wrapper<Functor>, fn type: std::reference_wrapper<Functor> &&
construct ScopeGuardImplBase, dismissed: 0
dtor, FunctionType: std::reference_wrapper<Functor>, function_ type: std::reference_wrapper<Functor>
exception: throw exception in Functor copy ctor...
size=0
```

可以看到结果完全符合预期，我们的copy ctor在runtime期间确实抛出了异常，在构造外层的`ScopeGuardImpl<Functor>`对象时遭到了异常，但由于内层`ScopeGuardmpl<std::reference_wrapper<Functor>>`对象的析构还是成功执行了垃圾清理。

我们把拷贝构造器抛出的异常干掉，但依然让其保持默认的`noexcept(false)`，再看看执行log：
```cpp

class Functor {
  public:
    explicit Functor(std::vector<int>& friends) : friends_(friends) {}

    Functor(const Functor &rhs) : friends_(rhs.friends_) { }

    void operator()() { friends_.pop_back(); }

  private:
    std::vector<int>& friends_;
};

void func(std::vector<int> &friends) {
    friends.push_back(1);
    Functor f(friends);
    auto guard = makeGuard(f);
    // some code that will not throw ...
    guard.dismiss();
}
```

```shell
false_type makeFailsafe, Fn type: Functor, fn type: Functor *
true_type makeFailsafe
construct ScopeGuardImplBase, dismissed: 0
construct ScopeGuardImplBase, dismissed: 0
private ctor, Fn type: std::reference_wrapper<Functor>, fn type: std::reference_wrapper<Functor> &&
rvalue ref ctor, FunctionType: std::reference_wrapper<Functor>, fn type: std::reference_wrapper<Functor> &&
construct ScopeGuardImplBase, dismissed: 0
private ctor, Fn type: const Functor &, fn type: const Functor &
dtor, FunctionType: std::reference_wrapper<Functor>, function_ type: std::reference_wrapper<Functor>
const lvalue ref ctor, FunctionType: Functor, fn type: Functor &
dtor, FunctionType: Functor, function_ type: Functor
size=1
```
此时可以看到成功构造出了两个`ScopeGuardImpl`对象，一个模板实参为`Functor`，一个模板实参为`std::reference_wrapper<Functor>`。由于我们的代码在执行期间没有抛异常，故垃圾清理函数没有被执行，符合预期。

我们再让执行期间随便抛一个异常来试试：

```cpp
class Functor {
  public:
    explicit Functor(std::vector<int>& friends) : friends_(friends) {}

    Functor(const Functor &rhs) : friends_(rhs.friends_) { }

    void operator()() { friends_.pop_back(); }

  private:
    std::vector<int>& friends_;
};

void unexcepted() { throw std::runtime_error("oops..."); }

void func(std::vector<int> &friends) {
    friends.push_back(1);
    Functor f(friends);
    auto guard = makeGuard(f);
    throw std::runtime_error("oops...");
    guard.dismiss();
}
```

```shell
false_type makeFailsafe, Fn type: Functor, fn type: Functor *
true_type makeFailsafe
construct ScopeGuardImplBase, dismissed: 0
construct ScopeGuardImplBase, dismissed: 0
private ctor, Fn type: std::reference_wrapper<Functor>, fn type: std::reference_wrapper<Functor> &&
rvalue ref ctor, FunctionType: std::reference_wrapper<Functor>, fn type: std::reference_wrapper<Functor> &&
construct ScopeGuardImplBase, dismissed: 0
private ctor, Fn type: const Functor &, fn type: const Functor &
dtor, FunctionType: std::reference_wrapper<Functor>, function_ type: std::reference_wrapper<Functor>
const lvalue ref ctor, FunctionType: Functor, fn type: Functor &
dtor, FunctionType: Functor, function_ type: Functor
exception: oops...
size=0
```
垃圾函数还是得到了正确的执行，此时如果在dtor中打印`dismissed_`字段，就会发现是`ScopeGuardImpl<Functor>`的析构函数执行了垃圾清理。

我们把`Functor`的copy ctor标记为`noexcept`试试：
```cpp
    Functor(const Functor &rhs) noexcept : friends_(rhs.friends_) { }//throw std::runtime_error("throw exception in Functor copy ctor..."); }
```

```shell
true_type makeFailsafe
construct ScopeGuardImplBase, dismissed: 0
construct ScopeGuardImplBase, dismissed: 0
private ctor, Fn type: const Functor &, fn type: const Functor &
const lvalue ref ctor, FunctionType: Functor, fn type: Functor &
dtor, FunctionType: Functor, function_ type: Functor
exception: oops...
size=0
```

通过log可以看出，此时走得就是简单路线，匹配的依然是const lvalue reference参数版本的ctor。

#### 用例2：右值引用
```cpp
class Functor {
  public:
    explicit Functor(std::vector<int>& friends) : friends_(friends) {}

    Functor(const Functor &rhs) noexcept : friends_(rhs.friends_) {
        std::cout << "Functor copy ctor" << std::endl;
    }
    Functor(Functor &&rhs) : friends_(rhs.friends_) {
        std::cout << "Functor move ctor" << std::endl;
        throw std::runtime_error("oops, runtime error in Functor move ctor");
    }
    void operator()() { friends_.pop_back(); }

  private:
    std::vector<int>& friends_;
};

void func(std::vector<int> &friends) {
    friends.push_back(1);
    Functor f(friends);
    auto guard = makeGuard(std::move(f));
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

执行log：

```shell
false_type makeFailsafe, Fn type: Functor, fn type: Functor *
true_type makeFailsafe
construct ScopeGuardImplBase, dismissed: 0
construct ScopeGuardImplBase, dismissed: 0
private ctor, Fn type: std::reference_wrapper<Functor>, fn type: std::reference_wrapper<Functor> &&
rvalue ref ctor, FunctionType: std::reference_wrapper<Functor>, fn type: std::reference_wrapper<Functor> &&
construct ScopeGuardImplBase, dismissed: 0
Functor move ctor
dtor, FunctionType: std::reference_wrapper<Functor>, function_ type: std::reference_wrapper<Functor>
exception: oops, runtime error in Functor move ctor
size=0
```

在构造外层`ScopeGuardImpl`对象时，由于`Functor`的move ctor抛出了异常，最终垃圾清理函数实际上是内层的`ScopeGuardImpl`对象所执行。

若移除move ctor的异常，且标记为noexcept，则最终走的是情景一的简单路径：

```cpp
class Functor {
  public:
    explicit Functor(std::vector<int>& friends) : friends_(friends) {}

    Functor(const Functor &rhs) noexcept : friends_(rhs.friends_) {
        std::cout << "Functor copy ctor" << std::endl;
    }
    Functor(Functor &&rhs) noexcept : friends_(rhs.friends_) {
        std::cout << "Functor move ctor" << std::endl;
    }
    void operator()() { friends_.pop_back(); }

  private:
    std::vector<int>& friends_;
};

void unexcepted() { throw std::runtime_error("oops..."); }

void func(std::vector<int> &friends) {
    friends.push_back(1);
    Functor f(friends);
    auto guard = makeGuard(std::move(f));
    unexcepted();
    guard.dismiss();
}
```

执行log：

```shell
true_type makeFailsafe
construct ScopeGuardImplBase, dismissed: 0
construct ScopeGuardImplBase, dismissed: 0
Functor move ctor
private ctor, Fn type: Functor, fn type: Functor &&
rvalue ref ctor, FunctionType: Functor, fn type: Functor &&
dtor, FunctionType: Functor, function_ type: Functor
exception: oops...
size=0
```

### 登峰造极，精益求精
到此，`ScopeGuard`的改良已经解决了我们的第一个顽疾，现在我们要面对第二个问题：如果垃圾清理器在执行过程中可能会抛出异常，我们应该怎么办？

其实这个问题反倒好处理，因为这单纯就是一个设计问题，毕竟我们总不能一直递归套娃下去。Folly库给出的解法也非常简单：概不负责，自行买单！

尽管是自行买单，但Folly在设计上为了精优化性能，还是允许用户传递一个非类型模板参数`InvokeNoexcept`给到`ScopeGuardImpl`，若其为true，则表示用户已承诺绝不会抛出异常，此时，如果清理器还是执行中抛了异常，那么会在程序终止之前，打印一些关键的错误信息，以示友好。

> C++的noexcept只是个标记，他告诉编译期它不会抛异常，但不代表它真的不会抛异常，只不过它一旦抛了异常，就会导致进程终止。

另一方面，为了精优化性能，我们还可以选择性的设计各个成员函数的noexcept标记：

```cpp
#include <iostream>
#include <type_traits>
#include <functional>
#include "type_name.h"

class ScopeGuardImplBase {
  public:
    void dismiss() noexcept { dismissed_ = true; }
  protected:
    ScopeGuardImplBase(bool dismissed = false) noexcept : dismissed_(dismissed) { }

    static void terminate() noexcept {
      std::ios_base::Init ioInit;
      std::cerr << "This program will now terminate because a ScopeGuard callback threw an \nexception.\n";
      std::rethrow_exception(std::current_exception());
    }

    static ScopeGuardImplBase makeEmptyScopeGuard() noexcept {
      return ScopeGuardImplBase{};
    }
  protected:
    bool dismissed_;
};

template<typename FunctionType, bool InvokeNoexcept>
class ScopeGuardImpl : public ScopeGuardImplBase {
  public:
    explicit ScopeGuardImpl(FunctionType& fn) noexcept(std::is_nothrow_copy_constructible<FunctionType>::value)
        : ScopeGuardImpl(std::as_const(fn), makeFailsafe(std::is_nothrow_copy_constructible<FunctionType>{}, &fn)) {
        }

    explicit ScopeGuardImpl(const FunctionType& fn) noexcept(std::is_nothrow_copy_constructible<FunctionType>::value)
        : ScopeGuardImpl(fn, makeFailsafe(std::is_nothrow_copy_constructible<FunctionType>{}, &fn)) {
        }

    explicit ScopeGuardImpl(FunctionType&& fn) noexcept(std::is_nothrow_move_constructible<FunctionType>::value)
        : ScopeGuardImpl(std::move_if_noexcept(fn), makeFailsafe(std::is_nothrow_move_constructible<FunctionType>{}, &fn)) {
        }

    ScopeGuardImpl(ScopeGuardImpl&& other) noexcept(std::is_nothrow_move_constructible<FunctionType>::value)
        : function_(std::move_if_noexcept(other.function_)) {
      dismissed_ = std::exchange(other.dismissed_, true);
    }

    ~ScopeGuardImpl() noexcept(InvokeNoexcept) {
      if (!dismissed_) {
        execute();
      }
    }

  private:
    static ScopeGuardImplBase makeFailsafe(std::true_type, const void*) noexcept {
      return makeEmptyScopeGuard();
    }

    template <typename Fn>
    static auto makeFailsafe(std::false_type, Fn* fn) noexcept
        -> ScopeGuardImpl<decltype(std::ref(*fn)), InvokeNoexcept> {
      return ScopeGuardImpl<decltype(std::ref(*fn)), InvokeNoexcept>{std::ref(*fn)};
    }

    template <typename Fn>
    explicit ScopeGuardImpl(Fn&& fn, ScopeGuardImplBase&& failsafe)
      : ScopeGuardImplBase{}, function_(std::forward<Fn>(fn)) {
      failsafe.dismiss();
    }

    void execute() noexcept(InvokeNoexcept) {
      if (InvokeNoexcept) {
        try {
          function_();
        } catch(std::exception &e) {
          terminate();
        }
      }else {
        function_();
      }
    }

  private:
    FunctionType function_;
};

template <typename F, bool INE>
using ScopeGuardImplDecay = ScopeGuardImpl<std::decay_t<F>, INE>;

// makeGuard在设计上使用InvokeNoexcept=true，这也符合使用者的期望
template <typename F>
ScopeGuardImplDecay<F, true> makeGuard(F&& f) noexcept(noexcept(ScopeGuardImplDecay<F, true>(
                static_cast<F&&>(f)))) {
  return ScopeGuardImplDecay<std::decay_t<F>, true>(std::forward<F>(f));
}
```
可以看到代码里又补充了很多细节调优，诸如对`noexcept`的处理、将`std::move`换成更保守的`std::move_if_noexcept`等等。

现在让我们来当一次猪比，故意传一个会抛异常的清理器进去：
```cpp
class Functor {
  public:
    explicit Functor(std::vector<int>& friends) : friends_(friends) {}

    Functor(const Functor &rhs) noexcept : friends_(rhs.friends_) {
    }
    Functor(Functor &&rhs) noexcept : friends_(rhs.friends_) {
    }
    void operator()() { friends_.pop_back(); throw std::runtime_error("oops, runtime error in cleaner..."); }

  private:
    std::vector<int>& friends_;
};

void unexcepted() { throw std::runtime_error("oops..."); }

void func(std::vector<int> &friends) {
    friends.push_back(1);
    Functor f(friends);
    auto guard = makeGuard(std::move(f));
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
```cpp
This program will now terminate because a ScopeGuard callback threw an
exception.
libc++abi: terminating due to uncaught exception of type std::runtime_error: oops, runtime error in cleaner...
[1]    11720 abort      ./scope
```

可以看到进程确实终止了，也如愿打印了`terminate()`中的友好信息。

## 倒行逆施：makeDismissedGuard
我们还可以对当前的设计做一些补充，比如支持反向操作：

```cpp
struct ScopeGuardDismissed {};

class ScopeGuardImplBase {
 public:
  void dismiss() noexcept { dismissed_ = true; }
  // 增加一个rehire函数，和dismiss作用相反
  void rehire() noexcept { dismissed_ = false; }

 protected:
  ScopeGuardImplBase(bool dismissed = false) noexcept : dismissed_(dismissed) {}

  static ScopeGuardImplBase makeEmptyScopeGuard() noexcept {
    return ScopeGuardImplBase{};
  }

  bool dismissed_;
};

template <typename FunctionType, bool InvokeNoexcept>
class ScopeGuardImpl : public ScopeGuardImplBase { 
  public:
    ...
    // 增加一个构造器，初始化dismissed_=true
    explicit ScopeGuardImpl(FunctionType&& fn, ScopeGuardDismissed) noexcept(
      std::is_nothrow_move_constructible<FunctionType>::value)
      : ScopeGuardImplBase{true}, function_(std::forward<FunctionType>(fn)) {}
    ...
};

template <typename F>
ScopeGuardImplDecay<F, true>
makeDismissedGuard(F&& f) noexcept(
    noexcept(ScopeGuardImplDecay<F, true>(std::forward<F>(f), ScopeGuardDismissed{}))) {
  return ScopeGuardImplDecay<F, true>(std::forward<F>(f), ScopeGuardDismissed{});
}
```

配合`rehire`接口，`makeDismissedGuard`就可以实现反向操作：如果遭遇了异常，则不执行传入的Functor；若一切顺利，反而会执行。

## defer：优雅，永不过时
`makeGuard()`+`dismiss()`和`makeDismissedGuard()`+`rehire()`的结对编程法已经很漂亮了，但Folly库的实现还不止于此，我们还可以用ScopeGuard来实现类似golang的defer：

```cpp
//  Example: 
/* open scope */ {
  some_resource_t resource;
  some_resource_init(resource);
  SCOPE_EXIT { some_resource_fini(resource); };
  // 执行过SOPE_EXIT宏过后，无论后面的语句是否会抛出异常，在离开末尾作用域后都会执行后接的块语句
  if (!cond)
    throw 0; // the cleanup happens at end of the scope
  else
    return; // the cleanup happens at end of the scope

  use_some_resource(resource); // may throw; cleanup will happen
} /* close scope */
```

怎样设计才能实现这一自制语法糖呢？且看Folly的实现：

```cpp
namespace detail {
enum class ScopeGuardOnExit {};

template <typename FunctionType>
ScopeGuardImpl<std::decay_t<FunctionType>, true> operator+(ScopeGuardOnExit, FunctionType&& fn) {
  return ScopeGuardImpl<std::decay_t<FunctionType>, true>(std::forward<FunctionType>(fn));
}
} // namespace detail

#define CONCATENATE(s1, s2) s1##s2

#define ANONYMOUS_VARIABLE(str) \
  CONCATENATE(CONCATENATE(CONCATENATE(str, __COUNTER__), _), __LINE__)

#define SCOPE_EXIT      \
  auto ANONYMOUS_VARIABLE(SCOPE_EXIT_STATE) = \
    detail::ScopeGuardOnExit() + [&]() noexcept
```

可以看到`SCOPE_EXIT`宏的实现借用了运算符重载，通过在内部定义一个新类型`ScopeGuardOnExit`并重载它的`operator+`操作符，用右操作数`fn`来最终构造出`ScopeGuardImpl`对象。而右操作数实际上是一个lambda对象，它使用引用捕获，示例代码中编写的花括号体实际上是这个匿名lambda对象的body。实际上这里重载哪个运算符是自由的，不一定非要选择`operator+`，只要能和宏定义衔接上即可。

另一方面，SCOPE_EXIT可以有复数个，也可以嵌套定义。对于前者来说，由于离开作用域后局部变量对象的析构顺序与定义时是逆序的，因此它也达成了像是go中多个defer的逆序执行效果；对于后者来说，嵌套本身就是支持的，因为局部变量的生命期只和它所在的外层作用域绑定。

### 其他的变种实现
略加变化，我们还可以实现其他变种，比如`SCOPE_FAIL`, `SCOPE_SUCCESS`:

```cpp
//  SCOPE_FAIL
//
//  May be useful in situations where the caller requests a resource where
//  initializations of the resource is multi-step and may fail.
//
//  Example:
// 
{
      some_resource_t resource;
      some_resource_init(resource);
      SCOPE_FAIL { some_resource_fini(resource); };

      if (do_throw)
        throw 0; // the cleanup happens at the end of the scope
      else
        return resource; // the cleanup does not happen
}

//  SCOPE_SUCCESS
//
//  In a sense, the opposite of SCOPE_FAIL.
//
//  Example:
//
{
      folly::stop_watch<> watch;
      SCOPE_FAIL { log_failure(watch.elapsed(); };
      SCOPE_SUCCESS { log_success(watch.elapsed(); };

      if (do_throw)
        throw 0; // the cleanup does not happen; log failure
      else
        return; // the cleanup happens at the end of the scope; log success
}
```

相比于`makeGuard()`+`dismiss()`和`makeDismissedGuard()`+`rehire()`的结对编程，这两位更是极尽优雅。

在实现上也与`SCOPE_EXIT`类似，只不过它需要额外捕获当前上下文内exception处理的状态，这是`ScopeGuardImpl`所不具备的能力，因此需要再封装一层：

```cpp
namespace detail {
/**
 * ScopeGuard used for executing a function when leaving the current scope
 * depending on the presence of a new uncaught exception.
 *
 * If the executeOnException template parameter is true, the function is
 * executed if a new uncaught exception is present at the end of the scope.
 * If the parameter is false, then the function is executed if no new uncaught
 * exceptions are present at the end of the scope.
 *
 * Used to implement SCOPE_FAIL and SCOPE_SUCCESS below.
 */
template <typename FunctionType, bool ExecuteOnException>
class ScopeGuardForNewException {
 public:
  explicit ScopeGuardForNewException(const FunctionType& fn) : guard_(fn) {}

  explicit ScopeGuardForNewException(FunctionType&& fn)
      : guard_(std::move(fn)) {}

  ScopeGuardForNewException(ScopeGuardForNewException&& other) = default;

  // 在dtor中对比初始化期间和当前的异常处理情况，来决定是否对组合的ScopeGuardImpl对象执行dismiss
  // ExecuteOnException是个mask
  ~ScopeGuardForNewException() noexcept(ExecuteOnException) {
    if (ExecuteOnException != (exceptionCounter_ < std::uncaught_exceptions())) {
      guard_.dismiss();
    }
  }

 private:
  void* operator new(std::size_t) = delete;
  void operator delete(void*) = delete;

  ScopeGuardImpl<FunctionType, ExecuteOnException> guard_;
  int exceptionCounter_{std::uncaught_exceptions()};
};

enum class ScopeGuardOnFail {};

// 设置mask：ExecuteOnException = true
template <typename FunctionType>
ScopeGuardForNewException<std::decay_t<FunctionType>, true>
operator+(detail::ScopeGuardOnFail, FunctionType&& fn) {
  return ScopeGuardForNewException<
      typename std::decay<FunctionType>::type,
      true>(std::forward<FunctionType>(fn));
}

enum class ScopeGuardOnSuccess {};

// 设置mask：ExecuteOnException = false
template <typename FunctionType>
ScopeGuardForNewException<std::decay_t<FunctionType>, false>
operator+(ScopeGuardOnSuccess, FunctionType&& fn) {
  return ScopeGuardForNewException<
      std::decay_t<FunctionType>,
      false>(std::forward<FunctionType>(fn));
}

} // namespace detail

#define SCOPE_FAIL                               \
  auto ANONYMOUS_VARIABLE(SCOPE_FAIL_STATE) = \
      detail::ScopeGuardOnFail() + [&]() noexcept

#define SCOPE_SUCCESS                               \
  auto ANONYMOUS_VARIABLE(SCOPE_SUCCESS_STATE) = \
      detail::ScopeGuardOnSuccess() + [&]()
```

一目了然的设计，伟大，无需多言。

注：在`ScopeGuardForNewException`设计中还可以看到设计者删除了operator new/delete的接口，这也是一个细节，因为ScopeGuardImpl在使用上就是作为一个栈空间临时变量对象而存在，它本就不应该被分配在堆空间上，为了避免误用，也就删除了这两个运算符。实际上，在ScopeGuardImpl的源代码里也做了一样的操作，只是前文中为了更紧凑的叙述才没写而已。

## 总结
到此，ScopeGuard源码的剖析就圆满结束了，奇怪的知识又增加了！值得注意的是：行文中的代码和Folly源代码的实现会有些细微差别，一些平台/语言标准版本的兼容代码也被我简化了，一些细节处也统一用了C++17的惯用法。

拜读大师级作品，掌握C++黑魔法，~~以雷霆、击碎黑暗！~~

## 参考链接

- [facebook/folly](https://github.com/facebook/folly)
- [Generic: Change the Way You Write Exception-Safe Code — Forever](http://drdobbs.com/184403758)