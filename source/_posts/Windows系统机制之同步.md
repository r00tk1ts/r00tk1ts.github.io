---
title: Windows系统机制之同步
date: 2018-03-22 17:10:11
categories: Windows
tags:
	- wrk
	- windows-kernel
---
根据wrk1.2源码，结合毛批&潘老师的《Windows内核原理与实现》&《Windows Internals》对Windows内核的设计进行归纳。 

<!--more-->

# Windows系统机制之同步

DPC级别以上叫高IRQL级别，这一级别不能依赖会导致缺页异常或重新调度的同步机制，所以DPC是个分水岭，高低级别的同步大相径庭。

## 高IRQL同步

单处理器上，如果内核更新全局数据结构时，恰好遇到中断的处理例程也要修改，那么可以简单的解决——处理器IRQL提升到任何有可能访问该全局数据的中断源所用的IRQL级别。显然，多处理器就不行了，使用中的处理器IRQL提升了，但可能有另一个处理器发生中断去访问全局数据。

### 互锁操作

最简单的方式就是直接依赖硬件的多处理器安全操作支持，如InterlockedIncrement, InterlockedDecrement, InterlockedExchange, InterlockedCompareExchange等函数。这些函数执行时会用lock指令锁住多处理器总线，这些函数称为intrinsic。

### 自旋锁

多处理器互斥的一种机制，它与某个全局数据结构相关联。

![](/images/wrk/20180322_1.jpg)

自旋锁非空闲时，内核会一直尝试获取该锁，直到成功。

获取了自旋锁后会提升到锁对应的DPC或更高级别，这就意味着拿到了自旋锁的线程不会被抢占，因为线程分发是在DPC级别处理的。因为自旋锁比较霸道，所以一般持有自旋锁的过程中执行的指令都尽可能的短。

获取了自旋锁后，虽然线程不会被调度出去，但更高级别的中断依然是可以抢占的，所以如果该中断服务例程和临界区内的数据有共享的话，在使用自旋锁时，比如一个设备驱动程序，还是要考虑二者的同步，比如关闭中断等手法。

自旋锁很危险，错误的用法比如持有自旋锁的代码企图让调度器执行分发操作，或者引发了缺页异常，那系统就崩了。

### 排队自旋锁

非标准自旋锁，在自旋锁基础上给了先来先得（FIFO）的顺序。额外的好处是降低处理器总线流量的繁重（自选自己的标记，而非全局锁）。

通过PRCB的字段找到。

### 栈内排队自旋锁

动态分配的，而不是全局的排队自旋锁。句柄时栈变量而非全局变量，保证了调用者线程的局部性。

### 执行体互锁操作

建立在自旋锁基础上的同步函数，支持更高级操作，比如单向链表和双向链表的插入删除。ExInterlockedPopEntryList和ExInterlockedPushEntryList等。

## 低IRQL同步

自旋锁使用场景受限，不合适时考虑这些同步机制：

- 内核分发器对象
- 快速互斥体和守护互斥体
- 推锁
- 执行体资源

低IRQL执行的用户模式代码也必须具有自己的锁原语。windows支持各种用户模式的同步语义：

- 条件变量(CondVars)
- Slim读写锁(SRWs)
- 一次运行初始化(InitOnce)
- 临界区(critical sections)

|             | 是否暴露给设备驱动程序使用 | 禁止常规内核模式APC | 禁止特殊内核模式APC | 支持递归获取操作 | 支持共享的和独占的获取操作 |
| ----------- | ------------- | ----------- | ----------- | -------- | ------------- |
| 内核分发器互斥体    | 是             | 是           | 否           | 是        | 否             |
| 内核分发器信号量或事件 | 是             | 否           | 否           | 否        | 否             |
| 快速互斥体       | 是             | 是           | 是           | 否        | 否             |
| 守护互斥体       | 是             | 是           | 是           | 否        | 否             |
| 推锁          | 否             | 否           | 否           | 否        | 是             |
| 执行体资源       | 是             | 否           | 否           | 是        | 是             |

### 内核分发器对象

以内核对象的形式向执行体提供同步机制，它们称为内核分发器对象。API函数用的可见同步对象，就是从这里获取的同步能力。执行体资源提供了独占访问能力（类似互斥体），也提供了共享读访问能力（多方共享只读）。

这些只能用于内核模式代码，WIN API并未暴露。

线程可以与一个分发器对象同步，做法就是等待该对象的句柄。此时该线程进入等待态。

![](/images/wrk/20180322_2.jpg)

无论何时，同步对象只能处于两个状态之一：有信号态和无信号态。下才能在等待条件满足前无法恢复执行。如果线程等待一个分发器对象的句柄，且分发器对象经历一次状态改变（无信号到有信号），则线程等待条件满足。

有信号状态的卡入时机：

| 类型            | 何时signaled        | 对等待线程的影响                     |
| ------------- | ----------------- | ---------------------------- |
| 进程            | 最后一个线程终止          | 全部被解除                        |
| 线程            | 线程终止              | 全部被解除                        |
| 事件（通知类型）      | 线程设置了该事件          | 全部被解除                        |
| 事件（同步类型）      | 线程设置了该事件          | 一个线程被解除，事件对象被重置              |
| 门对象（Gate，锁类型） | 线程置门对象为有信号状态      | 第一个等待线程被解除，并且接收一次优先级提升       |
| 门对象（信号类型）     | 线程置门对象为有信号状态      | 第一个等待线程被解除                   |
| 带键的事件         | 线程用一个键设置了事件       | 正在等待该键且与设置该事件的线程同属一个进程的线程被解除 |
| 信号量           | 信号量计数递增1          | 一个线程被解除                      |
| 定时器（通知类型）     | 设置的时间到了，或者时间间隔到期了 | 全部被解除                        |
| 定时器（同步类型）     | 设置的时间到了，或者时间间隔到期了 | 一个线程被解除                      |
| 互斥体           | 线程释放了互斥体          | 一个线程被解除，并且获得该互斥体的所属权         |
| 队列            | 队列中放入了项目          | 一个线程被解除                      |

关于这些东西的上层API表现，参考Jeffrey在《Windows核心编程》。

从代码的角度看，等待机制是通过KWAIT_BLOCK对象将分发器对象与分发器对象关联。

```cpp
typedef struct _KWAIT_BLOCK {
	//所在的线程分发器等待链表
    LIST_ENTRY WaitListEntry;
    struct _KTHREAD *Thread;
    PVOID Object;
	//哪些线程在等待它
    struct _KWAIT_BLOCK *NextWaitBlock;
    USHORT WaitKey;
    UCHAR WaitType;
    UCHAR SpareByte;

#if defined(_AMD64_)

    LONG SpareLong;

#endif

} KWAIT_BLOCK, *PKWAIT_BLOCK, *PRKWAIT_BLOCK;
```

分发器对象头部结构，它相当于等待机制这一基类：

```cpp
typedef struct _DISPATCHER_HEADER {
    union {
        struct {
            UCHAR Type;	//决定了分发器对象的类型，接相应的body，泛型思想
            union {
                UCHAR Absolute;
                UCHAR NpxIrql;
            };

            union {
                UCHAR Size;
                UCHAR Hand;
            };

            union {
                UCHAR Inserted;
                BOOLEAN DebugActive;
            };
        };

        volatile LONG Lock;
    };

    LONG SignalState;
    LIST_ENTRY WaitListHead;
} DISPATCHER_HEADER;
```

#### 事件对象KEVENT

通知对象和同步对象的差别体现在KeSetEvent操作。上表已描述行为差异。

```cpp
//
// Event object
//
//无需额外的数据属性
typedef struct _KEVENT {
    DISPATCHER_HEADER Header;
} KEVENT, *PKEVENT, *PRKEVENT;
```

#### 互斥对象KMUTEX

```cpp
//
// Mutant object
//
typedef struct _KMUTANT {
    DISPATCHER_HEADER Header;
    LIST_ENTRY MutantListEntry;
    struct _KTHREAD *OwnerThread;
    BOOLEAN Abandoned;
    UCHAR ApcDisable;
} KMUTANT, *PKMUTANT, *PRKMUTANT, KMUTEX, *PKMUTEX, *PRKMUTEX;
```

当前无信号，则一定被一个线程占有。

#### 信号量KSEMAPHORE

```cpp
//
//
// Semaphore object
//
// N.B. The limit field must be the last member of this structure.
//

typedef struct _KSEMAPHORE {
    DISPATCHER_HEADER Header;
    LONG Limit;
} KSEMAPHORE, *PKSEMAPHORE, *PRKSEMAPHORE;
```

这个就一目了然了。

#### 队列对象KQUEUE

windows内核中用于支持线程池的机制。

```cpp
typedef struct _KQUEUE {
    DISPATCHER_HEADER Header;
    LIST_ENTRY EntryListHead;//待处理的项
    ULONG CurrentCount;	//当前活动线程
    ULONG MaximumCount;	//最多活动线程上限
    LIST_ENTRY ThreadListHead;//已加入该队列对象的线程
} KQUEUE, *PKQUEUE, *PRKQUEUE;
```

线程调度机制为支持线程池做的一个扩展，所以队列对象主要逻辑都在线程调度器中完成。

#### 进程对象、线程对象

没啥好说的

#### 定时器对象KTIMER

分通知对象和同步对象。

```cpp
//
// Timer object
//
// N.B. The period field must be the last member of this structure.
//

typedef struct _KTIMER {
    DISPATCHER_HEADER Header;
    ULARGE_INTEGER DueTime;
    LIST_ENTRY TimerListEntry;
    struct _KDPC *Dpc;
    LONG Period;
} KTIMER, *PKTIMER, *PRKTIMER;
```

仅在时钟中断或DISPATCH_LEVEL软件中断发生时，处理到期的定时器对象。

#### 门等待

轻量的等待机制，对windows标准等待机制的简化，避免了线程从进入等待函数到离开等待函数过程中的许多步骤。

```cpp
typedef struct _KGATE {
    DISPATCHER_HEADER Header;
} KGATE, *PKGATE;

```

和事件一样都不用额外的数据，门对象是个同步类型的分发器对象，windows内核内部用。

#### 守护互斥体、快速互斥体

守护互斥体使用门等待来实现互斥同步能力。

快速互斥体提高了性能，在不存在竞争时，不需要等待事件对象。

#### 执行体资源、推锁

允许内核以共享或互斥方式保护一个资源。

执行体资源建立在内核的事件对象和信号量对象之上。利用事件对象来实现互斥控制，信号量对象来实现共享访问控制。

```cpp
typedef struct _ERESOURCE {
    LIST_ENTRY SystemResourcesList;//链入全局ExpSystemResourceList
    POWNER_ENTRY OwnerTable;//指向动态数组，等待执行体资源的线程及信息
    SHORT ActiveCount;	//记录多少个线程获取执行体资源对象
    USHORT Flag;	//当前执行体资源对象状态
    PKSEMAPHORE SharedWaiters;	//信号量
    PKEVENT ExclusiveWaiters;	//事件
    OWNER_ENTRY OwnerThreads[2];//0为独占的，1为共享的
    ULONG ContentionCount;	//发生竞争次数
    USHORT NumberOfSharedWaiters;//多少个线程等待共享获取对象
    USHORT NumberOfExclusiveWaiters;//多少个线程等待独占获取对象
    union {
        PVOID Address;
        ULONG_PTR CreatorBackTraceIndex;
    };

    KSPIN_LOCK SpinLock;
} ERESOURCE, *PERESOURCE;
```

本质上用于实现读写锁，内核代码读资源时可以申请共享方式占资源，需要写时则申请独占方式访问。当处于独占方式时，任何其他线程请求都要等。

本质上就是封装事件和信号量对象来实现读写锁语义。

再看推锁。

```cpp
//
// Push lock definitions
//
typedef struct _EX_PUSH_LOCK {

//
// LOCK bit is set for both exclusive and shared acquires
//
#define EX_PUSH_LOCK_LOCK_V          ((ULONG_PTR)0x0)
#define EX_PUSH_LOCK_LOCK            ((ULONG_PTR)0x1)

//
// Waiting bit designates that the pointer has chained waiters
//

#define EX_PUSH_LOCK_WAITING         ((ULONG_PTR)0x2)

//
// Waking bit designates that we are either traversing the list
// to wake threads or optimizing the list
//

#define EX_PUSH_LOCK_WAKING          ((ULONG_PTR)0x4)

//
// Set if the lock is held shared by multiple owners and there are waiters
//

#define EX_PUSH_LOCK_MULTIPLE_SHARED ((ULONG_PTR)0x8)

//
// Total shared Acquires are incremented using this
//
#define EX_PUSH_LOCK_SHARE_INC       ((ULONG_PTR)0x10)
#define EX_PUSH_LOCK_PTR_BITS        ((ULONG_PTR)0xf)

    union {
        struct {
            ULONG_PTR Locked         : 1;
            ULONG_PTR Waiting        : 1;
            ULONG_PTR Waking         : 1;
            ULONG_PTR MultipleShared : 1;
            ULONG_PTR Shared         : sizeof (ULONG_PTR) * 8 - 4;
        };
        ULONG_PTR Value;
        PVOID Ptr;
    };
} EX_PUSH_LOCK, *PEX_PUSH_LOCK;
```

一图胜千言呐！

![](/images/wrk/20180322_3.jpg)

竞争和非竞争态的算法描述参考潘老师的《Windows内核原理与实现》