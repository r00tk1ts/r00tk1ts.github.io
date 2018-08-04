---
title: 关键的Windows内核数据结构一览（上）
date: 2018-01-08 21:56:11
categories: windows
tags:
	- windows-kernel
---
近期一直在整理归纳知识体系，译几叠好文，写数篇心得，旨在温故而知新。放眼四壁，一时浩如烟海，月迷津渡。在翻译fuzzySecurity的[Windows exploit开发系列教程第十部分](https://bbs.pediy.com/thread-223812.htm)时，觅得此文，甚佳，不敢独酌。

<!--more-->
# 关键的Windows内核数据结构一览（上）

在我们的“Windows internals and debugging”课程中，学生经常会问我们这样的一些问题：Windows内核使用哪种数据结构来实现互斥量？。。本文试图通过描述Windows内核和设备驱动所使用的一些关键数据结构来回答这样的问题。

本文重点强调了系统中各种数据结构的关系，帮助读者在内核调试中进行导航。当阅读本文时，读者应该使用一个可读的内核调试器来尝试调试命令、进行数据结构的实验。本文仅仅是一个参考，而并非新手向导。

对每种结构来说，本文提供了数据结构的一种高层次描述，同时也描述了数据结构中指向其他结构的关键字段的细节。合适的话，可以对该结构使用调试命令，实现对所提供的数据结构的巧妙操纵。大部分文中提到的数据结构均由内核的paged抑或non-paged pool分配空间，这也是内核虚拟地址空间的一部分。

下列数据结构会在文中进行描述，单击以查看详情。

| 描述      | 数据结构                                     |
| ------- | ---------------------------------------- |
| 双链表     | LIST_ENTRY                               |
| 进程和线程   | EPROCESS, KPROCESS, ETHREAD, KTHREAD     |
| 内核&HAL  | KPCR, KINTERRUPT, CONTEXT, KTRAP_FRAME, KDPC, KAPC, KAPC_STATE |
| 异步对象    | DISPATCHER_HEADER, KEVENT, KSEMAPHORE, KMUTANT, KTIMER, KGATE, KQUEUE |
| 执行体&RTL | IO_WORKITEM                              |
| I/O管理器  | IRP, IO_STACK_LOCATION, DRIVER_OBJECT, DEVICE_OBJECT, DEVICE_NODE, FILE_OBJECT |
| 对象和句柄   | OBJECT_HEADER, OBJECT_TYPE, HANDLE_TABLE_ENTRY |
| 内存管理器   | MDL, MMPTE, MMPFN, MMPFNLIST, MMWSL, MMWSLE, POOL_HEADER, MMVAD |
| 缓存管理器   | VACB, VACB_ARRAY_HEADER, SHARED_CACHE_MAP, PRIVATE_CACHE_MAP, SECTION_OBJECT_POINTERS |

## 双链表

### `nt!_LIST_ENTRY`

Windows内核中的大部分数据结构都保存在链表中，在链表头中保存着指向链表元素项的指针。`LIST_ENTRY`结构用于实现这些循环双链表。`LIST_ENTRY`结构既可用于链表头也可以用作保存单个链表元素的表项结构。`LIST_ENTRY`结构较为典型的通过嵌入到大数据结构中，维持着链表中元素的关系。

![](/images/wrk/20180108_1.jpg)

调试命令`dt -l`命令会步过任何内嵌了该双链表的数据结构并显示链表中的所有元素。`dl`和`db`命令会向前和向后步过双链表，也可以使用`!dflink`和`!dblink`。

APIs:

- `InitializeListHead()`
- `IsListEmpty()`
- `InsertHeadList()`
- `InsertTailList()`
- `RemoveHeadList()`
- `RemoveTailList()`
- `RemoveEntryList()`


- `ExInterlockedInsertHeadList()`
- `ExInterlockedInsertTailList()`
- `ExInterlockedRemoveHeadList()`

## 进程和线程

### `nt!_EPROCESS`

Windows内核使用`ERPROCESS`结构体来表示一个进程，其包含了所有内核需要去保存关乎该进程的信息。对每一个运行在系统中的进程包括System Process和System Idle Process来说，都有一个对应的`EPROCESS`结构，System Process和System Idle Process运行在内核中。

`EPROCESS`结构属于内核的执行体层，包含了进程的资源相关信息诸如句柄表、虚拟内存、安全、调试、异常、创建信息、I/O转移统计以及进程计时等。

指向System Process的`EPROCESS`结构的指针保存在`nt!PsInitialSystemProcess`，而System Idle Process的`EPROCESS`指针保存在`nt!PsIdleProcess`。

任何进程都可以同时隶属于多个集合或组。例如，一个进程总是在系统中active进程列表中，一个进程可以属于内部运行着一个会话的进程集合，一个进程也可以是某个job的一部分。为了实现这些集合或组，`EPROCESS`结构通过不同的字段持有数个列表项。

`ActiveProcessLink`字段用于将该EPROCESS结构链入系统中active进程链表，该链表的头保存在内核变量中`nt!PsActiveProcessHead`。类似的，`SessionProcessLinks`字段用于将该EPROCESS结构链入到一个会话链表，链表头在`MM_SESSION_SPACE.ProcessList`。`JobLinks`字段用于将该EPROCESS结构链入到所属的job链表中，链表头在`EJOB.ProcessListHead`。内存管理器全局变量`MmProcessList`通过`MmProcessLinks`字段链入了一个进程链表。该链表可以通过`MiReplicatePteChange()`横贯以更新内核模式中关于进程虚拟地址空间的那部分。

属于进程的所有线程链表保存在`ThreadListHead`中，线程通过`ETHREAD.ThreadListEntry`排队。

内核变量`ExpTimerResolutionListHead`持有一个进程链表，使用`NtSetTimerResolution()`来改变定时器间隔。该链表被`ExpUpdateTimerResolution()`函数使用来更新时间分辨率到所有进程需求值中最小的那个。

`!process`命令从`EPROCESS`结构展示信息。`.process`命令切换调试器的虚拟地址空间上下文到特定的进程，当在一个完全的内核转储中或现场使用内核调试器时进行用户模式虚拟地址的实验时，这是一个非常危险的操作。

APIs:

- `ZwOpenProcess()`
- `ZwTerminateProcess()`
- `ZwQueryInformationProcess()`
- `ZwSetInformationProcess()`
- `PsSuspendProcess()`
- `PsResumeProcess()`
- `PsGetProcessCreateTimeQuadPart()`
- `PsSetCreateProcessNotifyRoutineEx()`
- `PsLookupProcessByProcessId()`
- `PsGetCurrentProcess()`
- `PsGetCurrentProcessId()`
- `PsGetCurrentProcessSessionId()`
- `PsGetCurrentProcessWin32Process()`
- `PsGetCurrentProcessWow64Process()`
- `PsGetCurrentThreadProcess()`
- `PsGetCurrentThreadProcessId()`
- `PsGetProcessCreateTimeQuadPart()`
- `PsGetProcessDebugPort()`
- `PsGetProcessExitProcessCalled()`
- `PsGetProcessExitStatus()`
- `PsGetProcessExitTime()`
- `PsGetProcessId()`
- `PsGetProcessImageFileName()`
- `PsGetProcessInheritedUniqueProcessId()`
- `PsGetProcessJob()`
- `PsGetProcessPeb()`
- `PsGetProcessPriorityClass()`
- `PsGetProcessSectionBaseAddress()`
- `PsGetProcessSecurityPort()`
- `PsGetProcessSessionIdEx()`
- `PsGetProcessWin32Process()`
- `PsGetProcessWin32WindowStation()`
- `PsGetProcessWow64Process()`

### `nt!_KPROCESS`

`KPROCESS`结构内嵌在`EPROCESS`结构体中，保存在`EPROCESS.Pcb`字段，它被执行体层下一级的微内核层使用，包含了线程的调度、配额、优先级以及执行时间等信息。

`ProfileListHead`字段包含了为该进程创建的性能对象链表。该链表被性能中断所使用来记录相关性能的说明。

`ReadyListHead`字段是一个线程链表，保存的是进程中出于就绪状态的线程。只有当进程不在内存中时，该链表才是非空的。链表中每项都是指向`KTHREAD`对象的`WaitListEntry`域的地址。

> 译者注：这里我扩展解释一下该字段：记录了这个进程中处于就绪状态但尚未被加入全局就绪链表的线程。当进程被换出内存后，他所属的线程一旦就绪，则被挂入到此链表，并要求换入该进程；此后当进程被换入内存时，`ReadyListHead`中的所有线程被加入到系统全局的就绪线程链表中。

`ThreadListHead`字段是进程所有线程的链表。`KTHREAD`数据结构通过`KTHREAD.ThreadListEntry`链入。内核用它来遍历进程中所有的线程。

`JobLinks`字段是同属于一个job的进程链表，链表头在`EJOB.ProcessListHead`。

### `nt!_ETHREAD`

Windows内核使用`ETHREAD`结构来表示一个线程，每个线程都有一个`ETHREAD`结构，这也包括在System Idle Process中的线程。

`ETHREAD`结构属于内核的执行体层，它包含了其他执行体组件诸如I/O管理器、安全引用监视器、内存管理、ALPC管理器等需要保存的线程相关信息。

`Tcb`字段包含了`KTHREAD`结构体，它嵌入到`ETHREAD`中并被用来存储线程调度相关信息。

每个进程都存储了一个`ETHREAD`结构体链表，代表了在进程中执行的线程，它在`EPROCESS`结构体的`ThreadListHead`字段中。

`ETHREAD`结构体通过`ThreadListEntry`字段链入到链表。

`KeyedWaitChain`字段用于保存那些正在等待一个特定事件的线程。

`IrpList`是一个IRP链表，用于表示该线程生成的I/O请求，在系统中这些请求在各驱动中正在处理但尚未完成。当线程终止时，这些IRP请求需要被取消。

`CallbackListHead`字段用于保存一个注册表回调函数的链表，它们会被调用以便于通知注册表过滤驱动关于该线程正在执行的注册操作。该字段对前向和后向通知注册表回调函数都是有效的。

`Win32StartAddress`字段是线程顶层函数的地址。该函数会通过`CreateThread()`传递给用户模式线程，通过`PsCreateSsytemThread()`给内核模式线程。

`ActiveTimerListHead`字段是个链表头，链表中包含了所有的当前线程active定时器（在一个确定的间隔后超期）。`ActiveTimerListBlock`字段用于保护链表，函数`ExpSetTimer()`通过`ETIMER.ActiveTimerListEntry`字段将定时器对象插入此链表。

调试命令`!thread`展示了线程的信息。`.thread`命令切换调试器CPU寄存器上下文到一个特定的线程。

APIs:

- `PsCreateSystemThread()`
- `PsTerminateSystemThread()`
- `PsGetCurrentThreadId()`
- `PsSetCreateThreadNotifyRoutine()`
- `PsRemoveCreateThreadNotifyRoutine()`
- `PsLookupThreadByThreadId()`
- `PsIsSystemThread()`
- `PsIsThreadTerminating()`
- `ZwCurrentThread()`

### `nt!_KTHREAD`

`KTHREAD`结构体内嵌在`ETHREAD`结构体中，存储在`ETHREAD.Tcb`字段，它被执行体层的下层微内核层所使用，它包含了线程的堆栈、调度、APC、系统调用、优先级、执行时间等信息。

`QueueListEntry`字段用于将关联到一个`KQUEUE`数据结构的线程链入链表。`KQUEUE.ThreadListHead`是该链表的头。`KQUEUE`结构体用于实现执行体工作队列（`EX_WORK_QUEUE`）以及I/O完成端口。当当前工作线程在此队列中且处于等待状态时，像`KiCommitThreadWait()`这样的函数会使用它来激活工作队列中的其他线程。

`MutantListHead`字段用于保存一个线程所获取的所有互斥量的链表。函数`KiRundownMutants()`使用此链表来检测一个线程在终止前是否释放了所有的互斥量，如果未能释放，则它会使得系统崩溃，bugcheck为`THREAD_TERMINATE_HELD_MUTEX`。

`Win32Thread`字段指向了Win32K.sys结构体W32THREAD（指向由Win32子系统管理的区域）。当一个用模式线程转换到UI线程，它会发起一个到USER32或GDI32中API的调用。函数`PsConvertToGuiThread()`执行这一转换。Win32K.sys函数`AllocateW32Thread()`调用`PsSetThreadWin32Thread()`来设置`Win32Thread`字段的值。每个线程分配的结构体尺寸存储于Win32K.sys中的`W32ThreadSize`变量中。

`WaitBlock`字段是一个4个`KWAIT_BLOCK`数组结构，线程用来等待本地内核对象。`KWAIT_BLOCK`的其中之一是保留的，它用于实现超时等待，因此它只能指向KTIMER对象。

`WaitBlockList`字段指向了`KWAIT_BLOCK`数组结构，下才能用来等待一到多个对象。该字段由函数`KiCommitThreadWait()`于线程刚刚进入到它的等待状态时设置。如果我们的线程等待的对象数量少于`THREAD_WAIT_OBJECTS`(3)，`WaitBlockList`就应该指向内置的`WaitBlock[]`数组，如果等待对象的数量超过了`THREAD_WAIT_OBJECTS`，但少于`MAXIMUM_WAIT_OBJECTS`(64)，`WaitBlockList`应该指向一个外部分配的`KWAIT_BLOCKS`数组。`ObpWaitForMultipleObjects()`是用来分配带有标签'Wait'的`KWAIT_BLOCK`数组的其中一个函数。

`WaitListEntry`字段用于添加`KTHREAD`结构体到链表中，这些线程均已在特定CPU上进入了等待状态。内核进程控制区域结构体(KPRCB)的`WaitListHead`字段通过`KTHREAD.WaitListEntry`链接了这样的线程到一起。函数`KiCommitThreadWait()`添加线程，`KiSignalThread()`移除线程。

## 内核&HAL

### `nt!_KPCR`

`KPCR`表示内核进程控制区域。它包含了每个CPU的信息，被内核和HAL所共享。系统有几个CPU，就有几个`KPCR`。

当前CPU的`KPCR`总是可以通过`fs:[0]`在x86系统上访问，x64系统上则通过`gs:[0]`。通用的内核函数诸如`PsGetCurrentProcess()`和`KeGetCurrentThread()`会利用FS/GS相对访问来从KPCR中获取信息。

`Prcb`字段包含了一个内嵌的KPRCB结构体，用于表示内核进程控制块。

### nt!_KINTERRUPT

一旦一个中断或异常发生，中断服务例程(ISRs)就会在CPU上执行。中段描述符表（IDT）是一个CPU定义的数据结构，指向了内核注册的ISRs。当中断或异常发生时，IDT被CPU硬件所使用来查找ISR并进行分发。IDT有256个表项，每个都指向一个ISR。中断向量是IDT中每个特定槽的索引值。`KINTERRUPT`结构体表示一个驱动注册的某个中断向量的ISR。

字段`DispatchCode`是一个字节数组，它包含了中断服务码的一些说明。特定向量的IDT条目直接指向了`DispatchCode`数组，调用`DispatchAddress`指向的函数。该函数一般是`KiInterruptDispatch()`，它负责建立一个需要去调用驱动提供的ISR的环境，该ISR由`ServiceRoutine`字段提供。

对消息信号中断（MSIs）来讲，`ServiceRoutine`指向了内核包装器函数`KiInterruptMessageDispatch()`，它通过`MessageServiceRoutine`指向的驱动提供的MSI中断服务例程来调用。

`ActualLock`字段指向一个自旋锁，在调用驱动支持的ISR之前用于`SynchronizeIrql`字段获取IRQL。

因为中断共享PCI总线的多重`KINTERRUPT`数据结构可以被注册为一个单一中断请求线(IRQ)。每个共享的中断向量的IDT条目都指向了第一个`KINTERRUPT`结构体，其他的`KINTERRUPT`结构体通过字段`InterruptListEntry`形成链。

调试器`!idt -a`命令展示了全部的每CPU中断描述表。

APIs:

- `IoConnectInterrupt()`
- `IoConnectInterruptEx()`
- `IoDisconnectInterrupt()`
- `IoDisconnectInterruptEx()`
- `KeAcquireInterruptSpinLock()`
- `KeReleaseInterruptSpinLock()`
- `KeSynchronizeExecution()`

### `nt!_CONTEXT`

`CONTEXT`结构体存储了异常上下文依赖于CPU的部分，它由CPU寄存器组成并被`KiDispatchException()`这样的函数用来下发异常。

`CONTEXT`结构体的部分内容由`KeContextFromKframes`函数捕获的`KTRAP_FRAME`结构体组成。同样地，在异常被分发后，`CONTEXT`结构体中被修改的内容会被`KeContextToKframes()`改回原貌。这一机制用于实现结构化异常处理（SEH）。

`ContextFlags`字段是一个位掩码，决定了`CONTEXT`结构体的哪些字段包含有效的值。例如`CONTEXT_SEGMENTS`指示上下文结构体中段寄存器是有效的。

调试器的`.cxr`命令用于切换调试器当前寄存器上下文，加载存储的CONTEXT结构体的值。

API: 

- `RtlCaptureContext()`

### `nt!_KTRAP_FRAME`

`KTRAP_FRAME`用于在中断或异常发生时保存CPU寄存器的内容。`KTRAP_FRAME`结构体一般在线程的内核模式栈上分配。陷阱帧的一小部分由CPU组成，一部分由自身的中断和异常控制组成，剩下的那些由软件异常和硬件中断handler提供，Windows下诸如函数`KiTrap0E()`或`KiPageFault`，`KiInterruptDispatch()`。在64位CPU上，陷阱帧的某些包含非优化(non-volatile)寄存器值的字段不是由异常handler构成的。

调试器的`.trap`命令切换太欧式器当前寄存器上下文到存储的`KTRAP_FRAME`结构。

### `nt!_KDPC`

DPC例程用于延迟中断进程到IRQL的DISPATCH_LEVEL。它会降低特定CPU在高IRQL例如DIRQLx上的运行时间。DPC也被用来提醒内核组件超时的定时器。ISRs和定时器都需要DPC。

> 译者注：延迟过程调用是Windows下一个很重要的机制，这个东西不仅用于定时器超时处理，还用于实现类似linux中中断下半部分sortirq的机制。·

`KDPC`表示一个延迟过程调用（DPC）数据结构，包含一个指向了驱动提供的例程。它应该IRQL为DISPATCH_LEVEL优先级时在任意线程上下文中被调用。

和中断服务例程不同之处在于，中断服务例程在线程栈上执行，DPC例程在per-CPU DPC栈上执行，它存储在`KPCR.PrcbData.DpcStack`。

`DEVICE_OBJECT`结构有一个`KDPC`结构体内置在`Dpc`字段，用来从ISR请求DPC例程。

`KDPC`结构体持有一个per-CPU DPC队列。`KPCR`数据结构的`PrcbData.DpcData[0]`字段包含了链表头。`KDPC`的`DpcListEntry`字段用来保存链表中的DPC。

调试命令`!pcr`和`!dpcs`显示了单一进程的DPC例程。

APIs: 

- `IoRequestDpc()`
- `IoInitializeDpcRequest()`
- `KeInitializeDpc()`
- `KeInsertQueueDpc()`
- `KeRemoveQueueDpc()`
- `KeSetTargetProcessorDpcEx()`

### `nt!_KAPC`

异步过程调用例程在特定线程的上下文、PASSIVE_LEVEL或APC_LEVEL优先级上执行。这些例程被驱动用来执行特定进程上下文的行为，主要是访问进程的用户模式虚拟地址空间。Windows中具体的功能如附加和分离一个线程到进程以及线程悬挂都是基于APC实现。APC有3种类型：用户模式；普通内核模式；特殊内核模式。

`KAPC`表示了异步过程调用（APC）结构体，它包含一个指向驱动支持的例程的指针。当APC可以下发给该线程的时候，该例程会在此特定线程上下文的PASSIVE_LEVEL或APC_LEVEL优先级上执行。

`KTHREAD.ApcState.ApcListHead[]`数组的两个表项包含了用户模式和内核模式的线程悬挂的APC列表。`KAPC`结构通过字段`ApcListEntry`链入此结构。

设置`KTHREAD.SpecialApcDisable`为一个负数会引起线程的特殊和普通内核APC被禁用。

设置`KTHREAD.KernelApcDisable`位一个负数会引起线程的普通内核APC被禁用。

`NormalRoutine`字段对特殊内核APC来说是NULL。对普通内核APC来说，它指向的函数运行在PASSIVE_LEVEL。

`KernelRoutine`字段指向了在`APC_LEVEL`执行的函数。

`RundownRoutine`字段指向了一个函数，当APC在线程终止被丢弃时会被执行。

调试命令`!apc`用以扫描系统中所有线程的悬挂APC并显示。

APIs:

- `KeEnterGuardedRegion()`
- `KeLeaveGuardedRegion()`
- `KeEnterCriticalRegion()`
- `KeLeaveCriticalRegion()`
- `KeInitializeApc()`
- `KeInsertQueueApc()`
- `KeRemoveQueueApc()`
- `KeFlushQueueApc()`
- `KeAreApcsDisabled()`

### `nt!_KAPC_STATE`

Windows内核允许线程附加到不同的进程中而不必是原始创建它的那个进程。这允许线程去获取对另外的进程的用户模式虚拟地址空间的临时访问。当线程附加到其他进程时，`KAPC_STATE`用来保存的线程的APC列表。因为APC是线程（以及进程）特定的，当一个线程附加到一个不同于当前所在进程的进程时，它的APC状态数据需要被保存。这是必须的因为线程当前队列中的APC（需要知道原始进程的地址）不能被下发给新的进程上下文。`KTHREAD`结构有两个内置的`KAPC_STATE`对象：一个是线程原始进程，另一个是线程附加的进程。在线程执行堆栈附加事件中，调用者需要提供更多`KAPC_STATE`结构体的存储空间来保存当前APC状态变量并可以转移到新的APC环境上。

`ApcListHead`数组的两个成员分别是用户模式和内核模式的线程悬挂APC队列。`KAPC`结构体通过`ApcListEntry`字段链入到链表中。

APIs:

- `KeAttachProcess()`
- `KeDetachProcess()`
- `KeStackAttachProcess()`
- `KeUnstackDetachProcess()`

## 异步对象

### `nt!_DISPATCHER_HEADER`

Windows本地内核对象是一些数据结构。它们种类繁多，且可以被线程直接通过调用`KeWaitForSingleObject()`来等待。内核中有着这些结构的逻辑实现，大部分结构都通过用户模式应用程序的本地(Nt/Zw)Win32 API函数暴露出去。事件、信号量、互斥量、定时器、线程、进程以及队列都是本地内核对象的例子。

`DISPATCHER_HEADER`结构内嵌到每一个本地内核对象中，它是一个线程等待机制实现中非常关键的组件。

每一个`KTHREAD`结构体都包含一个内置的`KWAIT_BLOCK`数组结构，它用于让线程在本地内核对象上阻塞。`DISPATCHER_HEADER`的`WaitListHead`字段指向了一个`KWAIT_BLOCKS`链结构，链中每个成员表示了线程所等待的某个本地内核对象。`KWAIT_LOCK_WaitListEntry`字段用于保存处于链表中的`KWAIT_BLOCK`结构。当本地内核对象被通知时(Signaled)，一到多个`KWAIT_BLOCK`会从链表中移除，其包含的线程会被置入就绪态。

`Type`字段用于识别内嵌的`DISPATCHER_HEADER`所包含的对象类别。它是枚举类型`nt!_KOBJECTS`的前10个值中的一个。该字段决定了`DISPATCHER_HEADER`的其他字段应该如何被解析。

`Lock`字段(bit7)实现了一个对象特定自定义的自旋锁，用以保护`SignalState`和`WaitListFields`字段。`SignalState`字段决定了该对象是否已通知(Signaled)。

APIs：

- `KeWaitForSingleObject()`
- `KeWaitForMultipleObjects()`
- `KeWaitForMutexObject()`

### `nt!_KEVENT`

`KEVENT`表示了内核事件数据结构。事件有两种类型：异步（自动重置）和通知（手动重置）。当一个异步事件被某线程通知（Signaled）时，等待它的线程中仅有一个会进入就绪态；但是当一个通知事件被某个线程通知时，等待它的所有线程都会被置入就绪态。`KEVENTS`可以作为独立的数据结构存在，用`KeInitializeEvent()`初始化或者作为事件对象由`ZwCreateEvent()`，一个本地被内核API内部使用的函数`IoCreateSynchronizationEvent()`或`IoCreateNotificationEvent()`创建。

APIs:

- `KeInitializeEvent()`
- `KeSetEvent()`
- `KeClearEvent()`
- `KeResetEvent()`
- `KeReadStateEvent()`
- `KeWaitForSingleObject()`
- `KeWaitForMultipleObject()`
- `IoCreateSynchronizationEvent()`
- `IoCreateNotificationEvent()`

### `nt!_KSEMAPHORE`

`KSEMAPHORE`表示内核信号量数据结构。信号量由线程调用`KeWaitForSingleObject()`进行获取。如果已经有一定数量的线程获取了信号量使得信号量超出了使用限制，后续的线程调用`KeWaitForSingleObject()`时就会进入等待状态。一旦任何线程使用`KeReleaseSemaphore()`释放了信号量，一个等待的线程就进入准备执行的状态。

线程的数量也就是获取了信号量的个数存储在字段`Header.SignalState`。`Limit`字段用于存储可同时持有该信号量的最大数量的线程。

APIs：

- `KeInitializeSemaphore()`
- `KeReadStateSemaphore()`
- `KeReleaseSemaphore()`

### `nt!_KMUTANT`

`KMUTANT`表示一个内核互斥量数据结构。一个互斥量只能被单一线程在一个时间点拥有，但同一线程可以递归的多次获取这个互斥量。

`OwnerThread`字段指向了线程的`KTHREAD`结构，该线程持有互斥量。

每个线程持有一个获取的所有互斥量组成的链表，`KTHREAD.MutantListHead`是该链表的头，`MutantListEntry`字段用于链入该链表。

`ApcDisable`字段决定了互斥量是一个用户模式还是内核模式对象，值0表示用户模式互斥量，其他任何值都表示是一个内核模式互斥量。

`Abandoned`字段会在互斥量没有被释放而被删除时设置。这将抛出一个`STATUS_ABANDONED`异常。

APIs：

- `KeInitializeMutex`()
- `KeReadStateMutex()`
- `KeReleaseMutex()`
- `KeWaitForMutexObject()`

### `nt!_KTIMER`

`KTIMER`表示一个定时器数据结构。当一个线程睡眠或等待一个分发对象且持有一个超时值时，Windows内核内部使用`KTIMER`来实现这一等待。

内核持有一个数组`KiTimerListHead[]`，它包含512个链表头，每个头存储一个`KTIMER`对象链表，他们会在一个确定的时间超时。`TimerListEntry`字段用于将`KTIMER`链入队列。

当一个定时器超时时，它要么唤醒一个在该定时器等待的线程，要么调度一个DPC例程来通知一个驱动该定时器已超时，指向DPC数据结构的指针存放在`Dpc`字段。定时器可以是偶然发生（超时1次）或周期性发生（在被取消之前重复不断的超时）。

调试器的`!timer`命令用以显示系统中所有的活动`KTIMER`。

APIs：

- `KeInitializeTimer()`
- `KeSetTimer()`
- `KeSetTimerEx()`
- `KeCancelTimer()`
- `KeReadStateTimer()`

### `nt!_KGATE`

`KGATE`表示一个内核门对象。`KGATE`提供的功能和异步类型的`KEVENT`十分相似，然而`KGATE`比`KEVENT`更为高效。

当一个线程在等待诸如事件、信号量、互斥量等分发对象时，它会使用`KeWaitForSingleObject()`或其变形函数，这些函数都是通用函数，并且需要处理所有和线程等待相关的特殊情况，例如，警告(alerts), APCs, 工作线程唤醒等。另一方面，在`KGATE`上等待是通过一个特殊的函数`KiWaitForGate()`来完成的，它不满足所有特殊的情况，使得代码路径非常的高效。但是，使用专用API的缺点是，线程不能同时在`KGATE`对象和另一个分发对象上等待。

`KGATE` APIs被内核内部使用，不会导出给驱动调用。`KGATE`在内核的内部多出被使用，这其中包括实现的守卫互斥锁(guarded mutexes)。当互斥量不可用时，守卫互斥锁内部等待一个`KGATE`对象。

APIs:

- `KeWaitForGate()`
- `KeInitializeGate()`
- `KeSignalGateBoostPriority()`

### `nt!_KQUEUE`

`KQUEUE`表示一个内核队列数据结构。`KQUEUE`用于实现执行体工作队列、线程池以及I/O完成端口。

多个线程可以经调用`KeRemoveQueueEx()`函数同时等待一个队列。当一个队列条目（任何内嵌了`LIST_ENTRY`的数据结构）被插入到队列时，其中的一个等待线程会被唤醒并在从队列中取出条目后，得到一个指向该队列条目的指针。

通用内核等待函数诸如`KeWaitForSingleObject()`,`KeWaitForMultipleObjects()`中有特殊的逻辑来处理那些关联一个队列的线程。每当这样的线程在队列以外的分发对象上等待时，与队列关联的另一个线程将被唤醒，以处理来自队列的后续项目。这可以确保在队列中插入的条目能够尽快得到服务。

`EntryListHead`字段是使用内嵌的`LIST_ENTRY`字段插入到队列形成的链表的头。函数`KiAttemptFastInsertQueue()`负责插入条目，`KeRemoveQueueEx()`负责移除条目。

`ThreadListHead`字段指向关联到该队列的所有线程链表。对所有这样的线程，其`KTHREAD.Queue`字段指向了该队列。

`CurrentCount`字段包含了正在积极处理队列项的线程数量，该数量被`MaximumCount`字段值所限制，该值的设置会根据系统上CPU的数量。

APIs:

- `KeInitializeQueue()`
- `KeInsertQueue()`
- `KeRemoveQueue()`
- `KeInsertHeadQueue()`

## 执行体和运行时库

### `nt!_IO_WORKITEM`

驱动程序使用工作项将某些例程的执行延迟到内核工作线程，这些线程在PASSIVE_LEVEL优秀级上会调用驱动程序提供的例程。工作项包含指向驱动提供的工作例程的指针，这些工作例程由驱动排列成一个固定的内核工作队列集合。内核提供工作线程例如`ExpWorkerThread()`等通过出队列条目并调用工作例程的方式来服务这些工作队列。`IO_WORKITEM`结构用以表示一个工作项。

内核变量`nt!ExWorkerQueue`包含了一个含3个`EX_WORK_QUEUE`结构体成员的数组，分别表示系统中关键的（Critical也叫临界的），延迟的（Delayed）和超临界（HyperCritical）工作队列。`WorkItem`字段用于组织`IO_WORK_ITEM`结构成上面3个中的某一个工作队列。

函数`IoQueueWorkItemEx()`持有一个`IoObject`的引用，它是一个指向驱动或设备对象的指针，用以防止驱动在工作例程执行期间被卸载。

`WORK_QUEUE_ITEM`结构体内嵌的`WorkerRoutine`字段指向了I/O管理器，它提供了名为`IopProcessWorkItem()`的包装器函数，该函数调用驱动程序提供的工作例程，并降低IoObject的引用计数。

`Routine`字段指向了驱动提供的工作例程，它在PASSIVE_LEVEL优先级上执行。

调试器的`!exqueue`命令显示了关于工作队列和工作线程的详细信息。

APIs：

- `IoAllocateWorkItem()`
- `IoQueueWorkItem()`
- `IoInitializeWorkItem()`
- `IoQueueWorkItemEx()`
- `IoSizeofWorkItem()`
- `IoUninitializeWorkItem()`
- `IoFreeWorkItem()`

> 译者注：由于本文篇幅实在过巨，且译者英文较poor, 为了防止通篇又臭又长的译文劝退读者，故于此拦腰截断，近日将更新下半部分。