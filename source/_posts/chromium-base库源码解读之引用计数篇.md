---
title: chromium-base库源码解读之引用计数篇
date: 2020-10-22 20:59:11
categories: code-anatomy
tags:
	- cpp
	- cpp-templates
---
chromium-base库作为业界标杆实现，值得每一位开发者细细品读。
本系列将围绕“基本概念、设计哲学、代码技巧”三个关键点来对chromium-base库进行深度剖析，本篇拆解“引用计数”。

<!--more-->
# chromium-base库源码解读之引用计数篇

引用计数，应是开发者耳熟能详的概念，不做赘述。

封装一个引用计数类，无非就是内部维护一个计数器，然后提供一对Add/Release接口，在计数器归零时，将对象释放。引用计数类一般由开发者定义的子类继承，然后配合智能指针类一起使用，达到开发者无需关心对象的生命周期、通过智能指针将对象传来传去的目的（~~妈妈再也不用担心内存泄漏了（然而并不一定）~~）。

chromium-base的引用计数类封装在`base/memory/ref_counted.h`，分别命名为`RefCounted`和`RefCountedThreadSafe`，顾名思义，后者是线程安全的。

本文结合《chromium-base库源码解读之智能指针篇》一起阅读效果最佳。

## `RefCounted`

这是个类模板，形如：

```cpp
template <class T, typename Traits = DefaultRefCountedTraits<T>>
class RefCounted : public subtle::RefCountedBase {
    //...
};
```

其他类如果想要使用`RefCounted`类，一般会通过公有继承`RefCounted<T>`来实现，`T`就是类本身的名字。

```cpp
class MyFoo : public RefCounted<MyFoo> {
  //...
  private:
    // 引用计数对象的消亡无需（且不能由）外部接管，故析构器为private/protected权限
    // 对象的析构由RefCounted<>在合适的时点(引用计数归0)执行，所以要声明为友元类
    friend class RefCounted<MyFoo>;  // Allow destruction by RefCounted<>.
    ~MyFoo();                        // Destructor must be private/protected.
};
```

另外它公有继承了`subtle::RefCountedBase`这么个基类，subtle是base内部的命名空间，对外曝光的类去公有继承一个`XXXBase`的基类，这是老套路了。

### `subtle::RefCountedBase`

我们先看看基类（移除了调试用的各种脚手架）：

```cpp
class RefCountedBase {
 public:
  // 两个对外曝光的接口，用于判断引用计数个数
  bool HasOneRef() const { return ref_count_ == 1; }
  bool HasAtLeastOneRef() const { return ref_count_ >= 1; }

 protected:
  // StartRefCountFromZeroTag和StartRefCountFromOneTag是两种枚举类型
  // 使用枚举类型做构造器参数，主要是为了表意（算是见名知意）
  // explicit禁用了危险的隐式转换
  explicit RefCountedBase(StartRefCountFromZeroTag) {}

  explicit RefCountedBase(StartRefCountFromOneTag) : ref_count_(1) {}

  ~RefCountedBase() {}

  // AddRef和Release就是核心的add/release语义接口，实际上是个wrapper
  void AddRef() const {
    AddRefImpl();
  }

  // Returns true if the object should self-delete.
  bool Release() const {
    ReleaseImpl();
    return ref_count_ == 0;
  }

 private:
  // 该友元函数用于RefCountedBase(StartRefCountFromOneTag)构造的场景
  // 返回对象的智能指针对象scoped_refptr<U>
  // scoped_refptr的解析请参考《chromium-base库源码解读之智能指针拆解篇》
  template <typename U>
  friend scoped_refptr<U> base::AdoptRef(U*);

// 这里才是add/release语义的真正实现
#if defined(ARCH_CPU_64_BITS)
  // 对64位来说，位数充足的溢出会导致安全漏洞，所以不能简单的++/--
  void AddRefImpl() const;
  void ReleaseImpl() const;
#else
  void AddRefImpl() const { ++ref_count_; }
  void ReleaseImpl() const { --ref_count_; }
#endif

  // 经典mutable+const函数限定，以支持const对象
  mutable uint32_t ref_count_ = 0;
  // 直接用了标准库的型别萃取，确保编译环境下uint32_t是种无符号类型
  static_assert(std::is_unsigned<decltype(ref_count_)>::value,
                "ref_count_ must be an unsigned type.");

  // 禁用拷贝构造器和赋值操作符
  DISALLOW_COPY_AND_ASSIGN(RefCountedBase);
}
```

> 知识点：C++11之后，禁用默认生成的拷贝构造器和赋值操作符有了更表意的写法：
>
> ```cpp
> // ALL DISALLOW_xxx MACROS ARE DEPRECATED; DO NOT USE IN NEW CODE.
> // Use explicit deletions instead.  See the section on copyability/movability in
> // //styleguide/c++/c++-dos-and-donts.md for more information.
> 
> // Put this in the declarations for a class to be uncopyable.
> #define DISALLOW_COPY(TypeName) \
>   TypeName(const TypeName&) = delete
> 
> // Put this in the declarations for a class to be unassignable.
> #define DISALLOW_ASSIGN(TypeName) TypeName& operator=(const TypeName&) = delete
> 
> // Put this in the declarations for a class to be uncopyable and unassignable.
> #define DISALLOW_COPY_AND_ASSIGN(TypeName) \
>   DISALLOW_COPY(TypeName);                 \
>   DISALLOW_ASSIGN(TypeName)
> ```
>
> 利用`=delete`语法糖，相比C++11之前的仅声明不定义的迂回写法，更加易懂。

64位机器需要对上溢和下溢做检查：

```cpp
// For security and correctness, we check the arithmetic on ref counts.
//
// In an attempt to avoid binary bloat (from inlining the `CHECK`), we define
// these functions out-of-line. However, compilers are wily. Further testing may
// show that `NOINLINE` helps or hurts.
//
#if defined(ARCH_CPU_64_BITS)
void RefCountedBase::AddRefImpl() const {
  // An attacker could induce use-after-free bugs, and potentially exploit them,
  // by creating so many references to a ref-counted object that the reference
  // count overflows. On 32-bit architectures, there is not enough address space
  // to succeed. But on 64-bit architectures, it might indeed be possible.
  // Therefore, we can elide the check for arithmetic overflow on 32-bit, but we
  // must check on 64-bit.
  //
  // Make sure the addition didn't wrap back around to 0. This form of check
  // works because we assert that `ref_count_` is an unsigned integer type.
  CHECK(++ref_count_ != 0);
}

void RefCountedBase::ReleaseImpl() const {
  // Make sure the subtraction didn't wrap back around from 0 to the max value.
  // That could cause memory leaks, and may induce application-semantic
  // correctness or safety bugs. (E.g. what if we really needed that object to
  // be destroyed at the right time?)
  //
  // Note that unlike with overflow, underflow could also happen on 32-bit
  // architectures. Arguably, we should do this check on32-bit machines too.
  CHECK(--ref_count_ != std::numeric_limits<decltype(ref_count_)>::max());
}
#endif
```

对于上溢来说，32位无须检查的理由是：32位架构中这种通过疯狂创建引用的攻击手法会因为没有足够的地址空间而不奏效；但对下溢来说，实际上32位架构也可以奏效，目前chromium-base库并没对32位的`ReleaseImpl`做检查，这就为漏洞利用留了操作空间。

> 知识点：CHECK宏实现：
>
> ```cpp
> // Discard log strings to reduce code bloat.
> //
> // This is not calling BreakDebugger since this is called frequently, and
> // calling an out-of-line function instead of a noreturn inline macro prevents
> // compiler optimizations.
> #define CHECK(condition) \
>   UNLIKELY(!(condition)) ? IMMEDIATE_CRASH() : EAT_CHECK_STREAM_PARAMS()
> 
> // Macro for hinting that an expression is likely to be false.
> #if !defined(UNLIKELY)
> #if defined(COMPILER_GCC) || defined(__clang__)
> #define UNLIKELY(x) __builtin_expect(!!(x), 0)
> #else
> #define UNLIKELY(x) (x)
> #endif  // defined(COMPILER_GCC)
> #endif  // !defined(UNLIKELY)
> ```
>
> UNLIKELY宏的包装是为了提升gcc和clang对分支处理的效率，可以参考：
>
> https://www.jianshu.com/p/2684613a300f
>
> 后两个宏不展开了，要根据编译器和指令集来细分，前者会导致程序崩溃，后者返回0。

---

回头来看`RefCounted`类模板：

```cpp
template <class T, typename Traits>
class RefCounted;

template <typename T>
struct DefaultRefCountedTraits {
  static void Destruct(const T* x) {
    // 友元类可以调用RefCounted<T,Traits>的private静态函数
    RefCounted<T, DefaultRefCountedTraits>::DeleteInternal(x);
  }
};

template <class T, typename Traits = DefaultRefCountedTraits<T>>
class RefCounted : public subtle::RefCountedBase {
 public:
  // 定义一个静态枚举常量，这个枚举值用于从0引用计数起始创建对象
  static constexpr subtle::StartRefCountFromZeroTag kRefCountPreference =
      subtle::kStartRefCountFromZeroTag;

  // 这里之所以写成T::kRefCountPreference是因为T会继承RefCounted<T>
  // 所以类T本身可以覆盖定义kRefCountPreference，如未定义，则继承上面的static变量
  RefCounted() : subtle::RefCountedBase(T::kRefCountPreference) {}

  // AddRef和Release只是个代理
  void AddRef() const {
    subtle::RefCountedBase::AddRef();
  }

  void Release() const {
    if (subtle::RefCountedBase::Release()) {
	  // 返回true表示引用计数归0，调用T的析构器
      // 指向本对象的指针实际上是继承了RefCounted<T>的T对象，所以cast即可
      // const修饰是为了同时兼容const和非const T对象
      // Traits默认实参是DefaultRefCountedTraits<T>，Destruct是其静态函数
      Traits::Destruct(static_cast<const T*>(this));
    }
  }

 protected:
  ~RefCounted() = default;

 private:
  // 声明为友元类，因为它需要调用DeleteInternal
  friend struct DefaultRefCountedTraits<T>;
  // 兜兜转转，实际上就是个delete
  template <typename U>
  static void DeleteInternal(const U* x) {
    delete x;
  }

  // 经典禁用拷贝构造器和赋值操作符
  DISALLOW_COPY_AND_ASSIGN(RefCounted);
};
```

这里的`kRefCountPreference`静态成员变量值得一提，内置默认的是`subtle::kStartRefCountFromZeroTag`，象征着`RefCountedBase`的`ref_count_`成员的0值，因此构造器的初始化列表中`subtle::RefCountedBase(T::kRefCountPreference)`默认会匹配到`RefCountedBase::RefCountedBase(StartRefCountFromZeroTag)`，从而最终构造出来的对象一开始引用计数是0；当外部需要从1值初始化时，就需要在类`T`内自己覆盖定义`kRefCountPreference`。

显然，从使用者视角来看，定义一个又臭又长且意义不明的`kRefCountPreference`非常奇怪，而针对这种情况最简单的处理就是提供一个封装好的宏定义，宏定义除了语句简洁以外，还有名称表意的效果：

```cpp
#define REQUIRE_ADOPTION_FOR_REFCOUNTED_TYPE()             \
  static constexpr ::base::subtle::StartRefCountFromOneTag \
      kRefCountPreference = ::base::subtle::kStartRefCountFromOneTag
```

> 实际上`kRefCountPreference`还会在智能指针`scoped_refptr`的相关操作中引用到，毕竟引用计数对象要搭配智能指针类来使用，引用计数负责对象的生命周期，智能指针负责把对象传来传去。
>
> 比如我们展开看上面的友元函数实现：
>
> ```cpp
> namespace subtle {
>   enum AdoptRefTag { kAdoptRefTag };
> }
> template <typename T>
> scoped_refptr<T> AdoptRef(T* obj) {
>   using Tag = std::decay_t<decltype(T::kRefCountPreference)>;
>   static_assert(std::is_same<subtle::StartRefCountFromOneTag, Tag>::value,
>                 "Use AdoptRef only if the reference count starts from one.");
> 
>   // subtle::kAdoptRefTag用于占位，去匹配其private构造器（所以它也是scoped_refptr<T>的友元）
>   // 	scoped_refptr<T>::scoped_refptr(T* p, base::subtle::AdoptRefTag);
>   // 如此就区分开了public构造器scoped_refptr<T>::scoped_refptr(T* p);
>   // 后者会调用p->AddRef()来为T对象增加引用计数，而前者不会
>   return scoped_refptr<T>(obj, subtle::kAdoptRefTag);
> }
> ```
>
> 静态断言确保`T::kRefCountPreference`是`subtle::StartRefCountFromOneTag`，即引用从1值起始。然后直接用`obj`对象和`subtle::kAdoptRefTag`构造一个`scoped_refptr<T>`智能指针对象并返回，该指针也就指向了对象`obj`。

## `RefCountedThreadSafe`

这个是`RefCounted`的线程安全版本。

```cpp
template <class T, typename Traits = DefaultRefCountedThreadSafeTraits<T> >
class RefCountedThreadSafe : public subtle::RefCountedThreadSafeBase {
	//...
};
```

老套路，先看基类。

### `subtle::RefCountedThreadSafeBase`

```cpp
class RefCountedThreadSafeBase {
 public:
  bool HasOneRef() const;
  bool HasAtLeastOneRef() const;

 protected:
  explicit constexpr RefCountedThreadSafeBase(StartRefCountFromZeroTag) {}
  explicit constexpr RefCountedThreadSafeBase(StartRefCountFromOneTag)
      : ref_count_(1) {}

  ~RefCountedThreadSafeBase() = default;

// Release and AddRef are suitable for inlining on X86 because they generate
// very small code sequences. On other platforms (ARM), it causes a size
// regression and is probably not worth it.
#if defined(ARCH_CPU_X86_FAMILY)
  // Returns true if the object should self-delete.
  bool Release() const { return ReleaseImpl(); }
  void AddRef() const { AddRefImpl(); }
  void AddRefWithCheck() const { AddRefWithCheckImpl(); }
#else
  // Returns true if the object should self-delete.
  bool Release() const;
  void AddRef() const;
  void AddRefWithCheck() const;
#endif

 private:
  template <typename U>
  friend scoped_refptr<U> base::AdoptRef(U*);

  ALWAYS_INLINE void AddRefImpl() const {
    ref_count_.Increment();
  }

  ALWAYS_INLINE void AddRefWithCheckImpl() const {
    CHECK(ref_count_.Increment() > 0);
  }

  ALWAYS_INLINE bool ReleaseImpl() const {
    if (!ref_count_.Decrement()) {
      return true;
    }
    return false;
  }

  mutable AtomicRefCount ref_count_{0};

  DISALLOW_COPY_AND_ASSIGN(RefCountedThreadSafeBase);
};
```

> 知识点：我们知道对现代编译器来说，inline并不一定真的能inline，`ALWAYS_INLINE`宏用于确保inline生效。
>
> ```cpp
> #if defined(COMPILER_GCC) && defined(NDEBUG)
> #define ALWAYS_INLINE inline __attribute__((__always_inline__))
> #elif defined(COMPILER_MSVC) && defined(NDEBUG)
> #define ALWAYS_INLINE __forceinline
> #else
> #define ALWAYS_INLINE inline
> #endif
> ```

可以看到与非线程安全的版本大致相同，只是在几个关键点上有所区别：

1. 引用计数器不再是个`uint32_t`，换成了使用原子操作的`AtomicRefCount`。

   线程安全嘛，理所应当。`AtomicRefCount`是chromium-base对标准库`std::atomic_int`的封装，后面展开。

2. 多了一个`AddRefWithCheck`接口。

   这个接口替代了`AddRef`，除了首次增加引用计数的场景，我们可以在`RefCountThreadSafe`类中看到。该接口增强了保护，对previous value做了`CHECK`。

3. 非X86架构因为"size regression"的成本而放弃了inline。

   ```cpp
   // 对于X86架构的，由于Release、AddRef和AddRefWithCheck都在类内定义，所以都是inline
   // 对非X86的就迂回一下，把这几个定义放到了类外
   #if !defined(ARCH_CPU_X86_FAMILY)
   bool RefCountedThreadSafeBase::Release() const {
     return ReleaseImpl();
   }
   void RefCountedThreadSafeBase::AddRef() const {
     AddRefImpl();
   }
   void RefCountedThreadSafeBase::AddRefWithCheck() const {
     AddRefWithCheckImpl();
   }
   #endif
   ```

   > 疑点：这里的知识点我并不清楚，以后懂了再做补充。

---

回头看`RefCountThreadSafe`:

```cpp
template <class T, typename Traits> class RefCountedThreadSafe;

// Default traits for RefCountedThreadSafe<T>.  Deletes the object when its ref
// count reaches 0.  Overload to delete it on a different thread etc.
template<typename T>
struct DefaultRefCountedThreadSafeTraits {
  static void Destruct(const T* x) {
    // Delete through RefCountedThreadSafe to make child classes only need to be
    // friend with RefCountedThreadSafe instead of this struct, which is an
    // implementation detail.
    RefCountedThreadSafe<T,
                         DefaultRefCountedThreadSafeTraits>::DeleteInternal(x);
  }
};

// Traits的默认实参实际上只是换了个壳，用以区分线程安全版本
// 内部还是通过调用RefCountedThreadSafe的static private方法来销毁对象
template <class T, typename Traits = DefaultRefCountedThreadSafeTraits<T> >
class RefCountedThreadSafe : public subtle::RefCountedThreadSafeBase {
 public:
  static constexpr subtle::StartRefCountFromZeroTag kRefCountPreference =
      subtle::kStartRefCountFromZeroTag;

  explicit RefCountedThreadSafe()
      : subtle::RefCountedThreadSafeBase(T::kRefCountPreference) {}

  void AddRef() const { AddRefImpl(T::kRefCountPreference); }

  void Release() const {
    if (subtle::RefCountedThreadSafeBase::Release()) {
      Traits::Destruct(static_cast<const T*>(this));
    }
  }

 protected:
  ~RefCountedThreadSafe() = default;

 private:
  friend struct DefaultRefCountedThreadSafeTraits<T>;
  template <typename U>
  static void DeleteInternal(const U* x) {
    delete x;
  }

  // 根据两种不同的枚举类型参数做了重载
  void AddRefImpl(subtle::StartRefCountFromZeroTag) const {
    subtle::RefCountedThreadSafeBase::AddRef();
  }

  void AddRefImpl(subtle::StartRefCountFromOneTag) const {
    subtle::RefCountedThreadSafeBase::AddRefWithCheck();
  }

  DISALLOW_COPY_AND_ASSIGN(RefCountedThreadSafe);
};
```

与非线程安全版本基本一致，只是对`AddRef`加强了保护。

## `AtomicRefCount`

这个类非常简单，包装了`std::atomic_int`。

> 这里用`int`应该是基于以下两种考虑的：
>
> 1. 引用计数一般不会太离谱，不会超过int的正数范围
> 2. 刚好配合`subtle::RefCountedThreadSafeBase::AddRefWithCheck`，用于在add之后检查是否溢出(`CHECK(ref_count_.Increment() > 0);`)。

```cpp
class AtomicRefCount {
 public:
  constexpr AtomicRefCount() : ref_count_(0) {}
  explicit constexpr AtomicRefCount(int initial_value)
      : ref_count_(initial_value) {}

  // Increment a reference count.
  // Returns the previous value of the count.
  int Increment() { return Increment(1); }

  // Increment a reference count by "increment", which must exceed 0.
  // Returns the previous value of the count.
  int Increment(int increment) {
    return ref_count_.fetch_add(increment, std::memory_order_relaxed);
  }

  // Decrement a reference count, and return whether the result is non-zero.
  // Insert barriers to ensure that state written before the reference count
  // became zero will be visible to a thread that has just made the count zero.
  bool Decrement() {
    // TODO(jbroman): Technically this doesn't need to be an acquire operation
    // unless the result is 1 (i.e., the ref count did indeed reach zero).
    // However, there are toolchain issues that make that not work as well at
    // present (notably TSAN doesn't like it).
    return ref_count_.fetch_sub(1, std::memory_order_acq_rel) != 1;
  }

  // Return whether the reference count is one.  If the reference count is used
  // in the conventional way, a refrerence count of 1 implies that the current
  // thread owns the reference and no other thread shares it.  This call
  // performs the test for a reference count of one, and performs the memory
  // barrier needed for the owning thread to act on the object, knowing that it
  // has exclusive access to the object.
  bool IsOne() const { return ref_count_.load(std::memory_order_acquire) == 1; }

  // Return whether the reference count is zero.  With conventional object
  // referencing counting, the object will be destroyed, so the reference count
  // should never be zero.  Hence this is generally used for a debug check.
  bool IsZero() const {
    return ref_count_.load(std::memory_order_acquire) == 0;
  }

  // Returns the current reference count (with no barriers). This is subtle, and
  // should be used only for debugging.
  int SubtleRefCountForDebug() const {
    return ref_count_.load(std::memory_order_relaxed);
  }

 private:
  std::atomic_int ref_count_;
};
```

这里考虑CPU架构通用性以及性能，没有使用C++默认的`memory_order_seq_cst`，而是精心设计了恰当的Acquire/Release语义。这些东西是为了防止多线程环境因指令重排序、CPU缓存而导致的依赖关系错乱，详细可以参考该文章：https://zhuanlan.zhihu.com/p/31386431，强烈推荐！