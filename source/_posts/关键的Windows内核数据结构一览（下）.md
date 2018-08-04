---
title: 关键的Windows内核数据结构一览（下）
date: 2018-01-14 09:25:11
categories: windows
tags:
	- windows-kernel
---
延续上篇的翻译，由于下篇中的一些数据结构本人也不甚相熟，所以难免有错误，海涵。

<!--more-->

## I/O管理器

### `nt!_IRP`

`IRP`表示一个I/O请求包结构体，它用来封装执行一个特定I/O操作所需要的所有参数以及I/O操作的状态。`IRP`的表现也类似于一个线程独立调用栈因此它可以从一个线程传递到另一个线程，也可以通过驱动实现的队列传递给一个DPC例程。IRPs是Windows异步I/O处理模型的关键所在，应用程序可以发出一连串的I/O请求并继续执行其他代码，而与此同时I/O请求会被驱动或硬件设备处理。这种异步模型允许相当大的吞吐量且可以实现最佳资源利用。IRPs通过I/O管理器组件分配，由Win32 I/O APIs进行操纵。在内核中IRPs也可以通过设备驱动为I/O请求而分配。IRPs流穿过驱动的栈，在这里驱动在传递IRP给下方的驱动之前会先增加它自身的值到IRP上，这是通过调用`IoCallDriver()`完成的。一般来说在最下方的驱动栈上会通过调用`IoCompleteRequest()`来完成该IRP，该调用会引起栈上的每个驱动被通知I/O已完成并给这些驱动一个在此IRP上执行后向处理操作的机会。

IRPs包含一个固定大小的头部，即IRP数据结构和一个可变的堆栈位置，存储在`StackCount`字段，其中每个堆栈位置都通过`IO_STACK_LOCATION`结构来表示。IRPs必须包含足够多的堆栈位置，该数量不应少于设备栈上互相叠罗汉的设备对象个数，这将用于处理IRP。驱动栈上的设备对象数量存储在`DEVICE_OBJECT.StackSize`字段。IRPs一般由支持固定大小分配器的前向链表分配。因此，由I/O管理器分配的IRPs有10或4个I/O堆栈位置，这取决于IRP目标所在的堆栈上的设备对象数量。

一些IRPs通过`ThreadListEntry`字段被链入到线程中，链表头为`ETHREAD.IrpList`。

`Tail.Overlay.ListEntry`字段被驱动用来保存内部队列的IRP，典型的固定在一个有`LIST_ENTRY`结构的设备扩展结构体中，它由驱动创建`DEVICE_OBJECT`时创建。

`Tail.CompletionKey`在IRP链入到一个I/O完成端口时被使用。

调试器命令`!irp`将显示一个IRP的细节描述。`!irpfind`命令可以通过扫描non-paged pool来找到系统中所有的或者特定的IRPs集合。

APIs：

- `IoAllocateIrp()`
- `IoBuildDeviceIoControlRequest()`
- `IoFreeIrp()`
- `IoCallDriver()`
- `IoCompleteRequest()`
- `IoBuildAsynchronousFsdRequest()`
- `IoBuildSynchronousFsdRequest()`
- `IoCancelIrp()`
- `IoForwardAndCatchIrp()`
- `IoForwardIrpSynchronously()`
- `IoIsOperationSynchronous()`
- `IoMarkIrpPending()`

### `nt!_IO_STACK_LOCATION`

`IO_STACK_LOCATION`包含有关一个I/O操作的信息，该操作要求在一组驱动中执行一个特定的驱动程序。IRP包含多个内嵌的I/O堆栈位置，它们都是在IRP被分配时分配的。设备栈上有多少个驱动IRP中就至少有多少个I/O堆栈位置。I/O堆栈位置归设备所有，它们逆序出现在设备堆栈上，也就是说堆栈上最顶层的设备拥有最底层的堆栈位置，反之亦然。I/O管理器负责填充最顶层设备的I/O堆栈位置，每个设备都负责填充链中下一个设备的I/O堆栈位置。

`Parameters`字段是一个多重结构体的共用体，每个结构代表了一个对应驱动必须执行的I/O操作。共用体`Parameters`中特定结构的选择依赖于`MajorFunction`字段。该字段可用的值由`IRP_MJ_XXX`在wdm.h中定义。具体的主函数(major function)还拥有和它们相关联的副函数(minor function)。这些副函数的数量存储在`MinorFunction`字段中。例如`IRP_MN_START_DEVICE`是一个和主函数`IRP_MJ_PNP`相关的副函数码。

APIs：

- `IoGetCurrentIrpStackLocation()`
- `IoGetNextIrpStackLocation()`
- `IoCopyCurrentIrpStackLocationToNext()`
- `IoSkipCurrentIrpStackLocation()`
- `IoMarkIrpPending()`

### `nt!_DRIVER_OBJECT`

`DRIVER_OBJECT`表示一个加载到内存中的驱动映像。I/O管理器会在驱动加载到内存前创建驱动对象，此后`DriverEntry()`例程会收到一个指向该驱动对象的指针。驱动被卸载出内存后驱动对象的释放与此相似。

`MajorFunction`字段是个数组，每个元素指向一个由驱动所知的分发入口指针提供的函数。这些入口指针被I/O管理器用来分发IRPs给驱动处理。

`DriverName`字段包含对象管理器命名空间内驱动对象的名称。

`DriverStart`字段是内核虚拟地址空间中驱动加载的地址，`DriverSize`包含驱动映射的字节数，按最近页边界向上取整。

`FastIoDispatch`指向了`FAST_IO_DISPATCH`类型的结构，它包含文件系统驱动提供的例程指针。

`DriverSection`指向一个`LDR_DATA_TABLE_ENTRY`类型的数据结构，供加载器在内存中找到驱动映像。

调试器命令`!drvobj`显示了一个驱动对象的信息。

APIs：

- `IoAllocateDriverObjectExtension()`
- `IoGetDriverObjectExtension()`

### `nt!_DEVICE_OBJECT`

`DEVICE_OBJECT`用于表示一个系统中的逻辑或物理设备。和驱动对象不同的是，驱动对象在驱动加载前由I/O管理器创建，而设备对象则是由驱动本身来创建。I/O请求的目标总是设备对象而不是驱动对象。一个指向设备对象的指针会传递给驱动的分发例程以识别哪个设备对象是此I/O请求的目标。

驱动对象在创建时怀有一个设备链表。该链表固定在`DRIVER_OBJECT.DeviceObject`中，使用`NextDevice`字段来把所有的设备驱动链在一起。设备对象，同样的，有一个`DriverObject`字段用以指回拥有它的驱动对象。尽管设备对象是系统定义的数据结构它们也可以拥有驱动特定的扩展结构。该扩展数据结构也一起放在设备对象中，它在non-paged pool中分配，大小是一个特定的声明的尺寸，而用于指向扩展结构的指针就是设备对象的`DeviceExtension`字段。

设备对象可以互相叠罗汉来形成一组设备对象。`StackSize`字段标志了有多少个设备对象在该设备对象的下方。该字段也被I/O管理器用来为IRPs分配合适数量的I/O堆栈位置。`CurrentIrp`和`DeviceQueue`字段只有当驱动为设备对象使用系统管理的I/O时才有用，这非常罕见，因此`CurrentIrp`字段在大多数情况下常常被设置成NULL。`AttachedDevice`指向了下一个设备栈上更高层的设备对象。

调试器命令`!devobj`可以显示一个设备对象的信息。

APIs：

- `IoCreateDevice()`
- `IoDeleteDevice()`
- `IoCreateDeviceSecure()`
- `IoCreateSymbolicLink()`
- `IoCallDriver()`
- `IoCreateFileSpecifyDeviceObjectHint()`
- `IoAttachDevice()`
- `IoAttachDeviceToDeviceStack()`
- `IoDetachDevice()`
- `IoGetAttachedDevice()`
- `IoGetAttachedDeviceReference()`
- `IoGetLowerDeviceObject()`
- `IoGetDeviceObjectPointer()`
- `IoGetDeviceNumaNode()`

### `nt!_DEVICE_NODE`

`DEVICE_NODE`用于表示一个已经被PnP管理器枚举到的物理或逻辑设备。设备节点(nodes)是电源管理和PnP操作的目标。系统中完整的硬件设备树源于具有层级结构的设备节点。设备节点有一个父子与兄弟的关系网。用户模式在CfgMgr32.h/CfgMgr32.lib定义的配置管理器APIs用于处理这些设备节点。

`DEVICE_NODE`结构体的`Child`,`Sibling`,`Parent`以及`LastChild`字段用于存放系统中所有的设备节点，形成一个层级结构。设备节点包含至少一个由`PhysicalDeviceObject`字段指向的物理设备对象（PDO）和一个功能设备对象（FDO），以及一到多个过滤设备对象（FDOs）。`ServiceName`字段指向了一个字符串，标志了创建FDO并驱动该设备的功能驱动。`InstancePath`指向了一个字符串，该字符串唯一的特定标志一个设备实例，系统中可以有多个相同的设备实例。`State`,`PreviousState`,`StateHistory`字段的组合用来确定设备节点在其当前状态设定之前所经历的状态。

调试器命令`!devnode`可以显示`DEVICE_NODE`结构体的信息。`!devstack`命令显示了所有的设备对象，它们是单一devnode的一部分。

### `nt!_FILE_OBJECT`

`FILE_OBJECT`表示了一个打开的设备对象实例。文件对象会在一个用户模式进程调用`CreateFile()`或本地(native)API`NtCreateFile()`或内核模式驱动调用`ZwCreateFile()`时被创建。多个文件对象可以指向一个设备对象除非该设备被标记被设置为排他属性，这是通过设置`DEVICE_OBJECT`标记中的`DO_EXCLUSIVE`位实现的。

`DeviceObject`字段指向了打开该文件对象实例的设备对象。`Event`字段包含一个内嵌的事件结构体，它用于阻塞一个在设备对象上已经请求过异步I/O操作的线程以便于其所有的驱动程序执行异步IO。

`FsContext`和`FsContext2`被文件系统驱动(FSDs)所使用，它们保存文件对象特定的上下文信息。当被一个文件系统驱动使用时，`FsContext`字段指向了一个类型为`FSRTL_COMMON_FCB_HEADER`或`FSRTL_ADVANCED_FCB_HEADER`的结构体，它包含一个文件或流的信息。多重`FILE_OBJECT`的`FsContext`字段表示相同的文件（或流）的打开实例指向同一个文件控制块（FCB）。`FsContext2`字段指向了一个缓存控制块，FSD用它来存储有关文件或流的特定实例信息。

`CompletionContext`,`IrpList`以及`IrpListLock`用于文件对象和I/O完成端口相关联的场景。`CompletionContext`字段由`NtSetInformationFile()`初始化，调用参数为信息类`FileCompletionInformation`。`CompletionContext.Port`字段指向一个KQUEUE类型的结构体，它包含一个已完成并等待被取回的IRP链表。`IoCompleteRequest()`通过字段`IRP.Tail.Overlay.ListEntry`查询该链表中的IRP。

调试命令`!fileobj`显示了一个文件对象的信息。

APIs：

- `IoCreateFile()`
- `IoCreateFileEx()`
- `IoCreateFileSpecifyDeviceObjectHint()`
- `IoCreateStreamFileObject()`
- `IoCreateStreamFileObjectEx()`
- `ZwCreateFile()`
- `ZwReadFile()`
- `ZwWriteFile()`
- `ZwFsControlFile()`
- `ZwDeleteFile()`
- `ZwDeviceIoControlFile()`
- `ZwFlushBuffersFile()`
- `ZwOpenFile()`
- `ZwFsControlFile()`
- `ZwLockFile()`
- `ZwQueryDirectoryFile()`
- `ZwQueryEaFile()`
- `ZwCancelIoFile()`
- `ZwQueryFullAttributesFile()`
- `ZwQueryInformationFile()`
- `ZwQueryVolumeInformationFile()`
- `ZwSetEaFile()`
- `ZwSetInformationFile()`
- `ZwSetQuotaInformationFile()`
- `ZwSetVolumeInformationFile()`
- `ZwUnlockFile()`
- `ZwWriteFile()`

## 对象和句柄

### `nt!_OBJECT_HEADER`

Windows内核中的对象（Object）数据结构用来表示通用的设施比如文件、注册表键、进程、线程、设备等等。它由对象管理器管理，对象管理器是Windows内核的一个组件。所有这样的对象内部都有一个先驱的`OBJECT_HEADER`结构体，它包含对象相关的信息并用来维护对象的生命周期，允许对象有一个专有名字，通过应用访问控制来保护对象，调用对象特定类型方法以及追踪分配器的配额使用。

对象中部署在`OBJECT_HEADER`后方的数据结构与`OBJECT_HEADER`存在着部分的重叠。实际上对象的数据存放位置是从`OBJECT_HEADER`的`Body`字段开始而不是从其尾部开始。

对象头部包含了引用计数`HandleCount`和`PointerCount`，它们被对象管理器用来保存对象直到没有外部的引用指向该对象。`HandleCount`是句柄的数量，`PointerCount`是句柄和内核模式对象引用的数量。

对象头可以由一个可选的对象头引导，类似`OBJECT_HEADER_PROCESS_INFO`,`OBJECT_HEADER_QUOTA_INFO`,`OBJECT_HEADER_HANDLE_INFO`,`OBJECT_HEADER_NAME_INFO`以及`OBJECT_HEADER_CREATOR_INFO`结构，它们描述了关于对象额外的属性。`InfoMask`字段是一个位掩码，它决定了当前是哪一种前面描述的可选头结构。

`SecurityDescriptor`字段指向一个类型为`SECURITY_DESCRRIPTOR`的结构体，它包含了任意访问控制列表（DACL）和系统访问控制列表（SACL）。DACL用于检查进程的tokens是否有访问对象的权限。SACL用于审计访问对象的权限。

内核不能在IRQL高于PASSIVE_LEVEL的级别上删除对象。`ObpDeferObjectDeletion()`把对象链在一起，其删除操作需要被延迟到一个链表中，该链表在内核变量`ObpRemoveObjectList`中。`NextToFree`字段为此而生。

`QuotaBlockCharged`字段指向了`EPROCESS.QuotaBlock`的`EPROCESS_QUOTA_BLOCK`结构体，它被`PsChargeSharedPoolQuota()`和`PsReturnSharedPoolQuota()`函数用来追踪一个使用Non-Paged Pool和Paged Pool的特定进程。分配对象时配额总是会被分配。

调试器命令`!object`显示了存储在对象头的信息。`!obtrace`命令显示了根据对象引用trace到的数据。如果对象引用追踪在一个对象上可用，`!obja`命令可以显示任何对象的属性信息。

APIs：

- `ObReferenceObject()`
- `ObReferenceObjectByPointer()`
- `ObDereferenceObject()`

### `nt!_OBJECT_TYPE`

对对象管理器管理的每种类型的对象来说，都有一个“类型对象”结构体用于存储该类型对象的通用属性。这种“类型对象”结构体由`OBJECT_TYPE`表示。Windows 7上有大概42种不同的对象类型结构体。内核变量`ObTypeIndexTable`是一个指针数组，它的每个成员都指向一种对象类型的`OBJECT_TYPE`结构体。对于每种对象类型内核也会保存一个指向相关对象类型结构体的全局变量。例如，变量`nt!IoDeviceObjectType`指向了`DEVICE_OBJECTS`的`OBJECT_TYPE`结构体。

`OBJECT_TYPE`的`TypeInfo`字段指向了一个`OBJECT_TYPE_INITIALIZER`结构体，该结构体包含了对象类型特定的函数，它们被对象管理器用来在对象上执行各种各样的操作。

`CallbackList`字段一个特定对象类型的驱动安装的回调函数列表的头。当前只有进程和线程对象支持回调函数，它们由`TypeInfo.SupportsObjectCallbacks`字段指定。

关键字段包含了池标签(pool tag)，它用于分配该类型的对象。

调试器命令`!object ObjectTypes`用于显示系统中所有的类型对象。

### `nt!_HANDLE_TABLE_ENTRY`

Windows中的进程有它们自己的私有句柄表，它保存在内核虚拟地址空间中。`HANDLE_TABLE_ENTRY`表示了一个进程句柄表中 个体表项。句柄表在分页内存池中分配。当一个进程终止时，函数`ExSweepHandleTable()`关闭该进程句柄表中所有的句柄。

`Object`字段指向了一个对象结构体，比如File, Key, Kevent等，它们的句柄均已创建。

`GrantedAccess`字段是一个类型为`ACCESS_MASK`的位掩码，它决定了对象上特定句柄所允许的操作集合。该字段的值由`SeAccessCheck()`计算，基于调用者（可信访问）的访问请求以及对象安全描述符的DACL中的ACEs。

调试命令`!handle`可以用来查看任何进程的句柄表。`!htrace`命令可以用来显示堆栈上跟踪句柄的数据，如果句柄跟踪可用的话。

APIs：

- `ObReferenceObjectByHandle()`
- `ObReferenceObjectByHandleWithTag()`

## 内存管理

### `nt!_MDL`

`MDL`表示一个内存描述符列表结构，它描述那些已被锁定的用户或内核模式内存。它由一个固定长度的头和可变的页帧数（PFNs）组成。每一页都由MDL描述。

MDL结构包含了它描述的虚拟地址和缓冲区尺寸，对用户模式缓冲区来说它也指向了拥有该缓冲区的进程。设备驱动程序使用MDLs对硬件设备进行编程，执行DMA的传输，同时也映射用户模式缓冲区到内核模式，反之亦可。

某些类型的驱动程序，例如网络栈，在Windows支持的链式MDLs，多个MDL中描述了实质上分散的缓冲区，它们由`Next`字段链在一起。

对MDLs来说那描述了用户模式缓冲区，`Process`字段指向了进程的EPROCESS结构体，它的虚拟地址空间被MDL锁住。

如果被MDL描述的缓冲区被映射到了内核虚拟地址空间，`MappedSystemVa`就指向了内核模式的缓冲区地址。该字段仅在位`MDL_MAPPED_TO_SYSTEM_VA`或`MDL_SOURCE_IS_NONPAGED_POOL`在`MdlFlags`字段中被设置时，才是有效的。

`Size`字段包含MDL数据结构以及MDL后面跟随的整个PFN数组的尺寸。

`StartVa`字段和`ByteOffset`字段一起定义了MDL中被锁住的原始缓冲区的起始位置。`StartVa`指向了页起始，`ByteOffset`包含了从`StartVa`计算缓冲区实际起始地址的偏移字节。

`ByteCount`字段描述了MDL锁住的缓冲区尺寸。

APIs：

- `IoAllocateMdl()`
- `IoBuildPartialMdl()`
- `IoFreeMdl()`
- `MmInitializeMdl()`
- `MmSizeOfMdl()`
- `MmPrepareMdlForReuse()`
- `MmGetMdlByteCount()`
- `MmGetMdlByteOffset()`
- `MmGetMdlVirtualAddress()`
- `MmGetSystemAddressForMdl()`
- `MmGetSystemAddressForMdlSafe()`
- `MmGetMdlPfnArray()`
- `MmBuildMdlForNonPagedPool()`
- `MmProbeAndLockPages()`
- `MmUnlockPages()`
- `MmMapLockedPages()`
- `MmMapLockedPagesSpecifyCache()`
- `MmUnmapLockedPages()`
- `MmAllocatePagesForMdl()`
- `MmAllocatePagesForMdlEx()`
- `MmFreePagesFromMdl()`
- `MmMapLockedPagesWithReservedMapping()`
- `MmUnmapReservedMapping()`
- `MmAllocateMappingAddress()`
- `MmFreeMappingAddress()`

### `nt!_MMPTE`

MMPTE用来表示内存管理的页表项（PTE），它被CPU的内存管理单元（MMU）用来转换虚拟地址（VA）到物理地址（PA）。映射一个VA到PA所需要的转换级别取决于CPU类型。对x86来说它使用2级转换（PDE和PTE），x86在PAE模式下使用3级转换（PPE，PDE和PTE），x64 CPU则使用4级转换（PXE，PPE，PDE，PTE）。不同级别结构的格式，诸如PXE，PPE，PDE和PTE都是相似的，MMPTE不仅可以用于表示PTE，还可以表示这些其他转换结构。

MMPTE结构是一个多重子结构的联合体，它们被Windows内存管理器的页错误处理机制用来搜索由PTE表示的页位置。例如，当一个PTE包含了一个页的有效的物理地址且MMU可以使用这个PTE完成地址转换，那么使用的子结构就是`u.Hard`字段。

当一个页从进程的工作集中被移除时，Windows内存管理器从一个硬件视角来看会标记这个页的PTE为无效的，然后，内存管理器重新试图用该PTE来存储OS关于页的特定信息。这会导致CPU内存管理单元（MMU）不能再使用该PTE来进行地址转换。当进程试图去访问这样一个页时，CPU生成一个页错误来调用Windows的页错误例程。PTE中编码的信息现在用来定位该页并将它返还给进程的工作集，这样就解决了页错误。有个这样的例子就是PTE转换，该PTE表示一个处于待命或修改状态的页。在这种情形下，`u.Transition`子结构用来存储该页的信息。

当物理页的内容被保存在页文件中时，Windows内存管理器会修改PTE来指向页文件中的页位置，此时使用的是`u.Soft`子结构。`u.Soft.PageFileLow`字段决定了Windows支持的16个页文件中哪一个包含了该页，而`u.Soft.PageFileHigh`则包含了在该页文件中页的索引。

调试器命令`!pte`可以显示给定虚拟地址的所有级别的页表内容。

### `nt!_MMPFN`

Windows内存管理持有系统中每个物理页的信息，放置在一个叫PFN数据库的数组中。MMPFN结构体表示了该数据库中的每一个独立条目，它包含了单一物理页的信息。

变量`nt!MmPfnDatabase`指向了MMPN结构体数组，它们组成了PFN数据库。PFN数据库的条目数量为`nt!MmPfnSize`，这其中有一些额外的条目来处理热插拔内存。为了保存内存，MMPFN结构被塞得很满。每个字段的解析都是不同的，而这取决于每个页的状态。

物理页的状态存储在`u3.e1.PageLocation`，由枚举类型`nt!_MMLISTS`中的一个条目标识。

`u2.ShareCount`字段包含进程指向该页的PTE数量，如果存在共享页则应该比1大。

`u3.e2.ReferenceCount`包含该页的引用数量，如果页被锁住则也包含锁的数量。该引用计数会在`u2.ShareCount`变成0时递减。

调试命令`!pfn`可以显示给定物理页的MMPFN结构体的所有内容。

### `nt!_MMPFNLIST`

内存管理器持有处于相同状态的链在一起的物理页。这可以加速查找某种给定状态的一个或多个页的过程，例如，查找空闲页或被零化换出（zeroed out）的页。MMPFNLIST结构体持有这些链表的头。系统中有多个MMPFNLIST结构体，它们中的每个都包含一个特定状态的页并被存储在内核变量`nt!Mm<PageState>ListHead`中，这里的PageState可以表示待命(Standby)、已修改(Modified)、无写修改（ModifiedNoWrite），空闲(Free)，只读(Rom)，损坏(Bad)，零化(Zeroed)。活跃态的页面例如当前属于一个进程工作集的页面不会再任何一个链表中。

在Windows的新版本中，`nt!MmStandbyPageListHead`不再使用，取而代之的是一个具有优先级的8链表集合，存在`nt!MmStandbyPageListByPriority`。`nt!MmFreePageListHead`和`nt!MmModifiedPageListHead`也不再被使用，转而使用的是`nt!MmFreePagesByColor`和`nt!MmModifiedProcessListByColor`或`nt!MmModifiedPageListByColor`。

`MMPFN.u1.Flink`和`MMPFN.u2.Blink`字段对一个特定页来说，用于保存该页到一个双向链表中。这些链表的头存储在对应的`MMPFNLIST`结构体的`Flink`和`Blink`字段。

`ListName`字段是枚举类型MMLISTS中的一个，它标识了该链表中页的类型。

调试器命令`!memusage 8`命令显示了某个特定状态的页的数量。

### `nt!_MMWSL`

Windows中每个进程都有一个和它关联的工作集，它由那些进程不会触发页错误的页组成。工作集整理者（WST），是内存管理器的一个组件，它运行在`KeBalanceSetManager()`线程的上下文，尽力去移除进程不再使用的页并重新分配它们到有需求的其他进程。为了执行这一任务，WST需要存储关于系统中每个进程的工作集信息。该信息被保存在MMWSL结构体。每个进程的`EPROCESS.Vm.VmWorkingSetList`都指向了它的MMWSL结构。

系统中每个进程的MMWSL都在内核虚拟地址空间的超空间(HyperSpace)上一个完全相同的地址。超空间是内核虚拟地址空间的一部分，每个进程都有一个自己的进程映射，并不是所有进程共享一个单一的映射。因此，内存管理器，对任何给定的实例，只能访问当前进程的MMWSL即目前在CPU上运行进程的线程。

MMWSL结构体的`Wsle`字段指向进程工作集列表项数组的基址。数组中条目的有效值为`EPROCESS.Vm.WorkingSetSize`。

### `nt!_MMWSLE`

MMWSLE数据结构表示一个进程工作集中单一页面的工作集列表，因此每一个在进程工作集中的页都有一个MMWSLE结构。该结构被工作集整理者用来判定该特定页是否是一个潜在的可整理候选页，也就是说从进程的工作集中移除。

当一个进程试图访问一个在工作集中的页面时，CPU内存管理单元MMU会在该页对应的PTE中设置`MMPTE.Hard.Accessed`位。工作集整理者有规律的唤醒并扫描一个进程的WSLEs。在扫描期间他会检查从上一次来是否已访问过一个特定的页，这是通过检查PTE的访问位来实现的。如果该页从上次扫描以来从未被访问过，该页就通过递增`u1.e1.Age`字段逐渐老化。如果页被访问过，那么`u1.e1.Age`字段就被重置为0。当`u1.e1.Age`涨到7时，该页就被认为是一个可以被换出的候选页。

`u1.e1.VirtualPageNumber`是页虚拟地址的高20位(在x86)或52位（在x64）。

调试命令`!wsle`可以显示一个特定进程的工作集列表条目。

### `nt!_POOL_HEADER`

内核中动态内存分配是由非页池、分页池和会话池组成的。根据请求分配的大小，池分配分为小池分配（尺寸少于4K）和大池分配（尺寸大于等于4k）。池分配大小在x86系统上总是向上取整到8字节，在x64系统上向上取整到16字节。

每个小池分配都由一个池头、数据区域组成。数据区域用于存储数据，里面还有一些用于对其的padding。池头用`POOL_HEADER`结构表示，该结构包含了关于后面跟随的数据区的信息。

`BlockSize`字段包含了池块的尺寸，囊括头和任何padding字节。`PreviousSize`字段包含了前一块的尺寸（低地址毗邻的块）。这两个字段在x86上都是8的倍数，x64上都是16的倍数。`PoolTag`字段包含4字节标记，它用于标识池分配的所有者，常常用于调试。如果最高位（31位）池标签被设置，则该池分配被标记为受保护。

大池分配没有内嵌的`POOL_HEADER`，取而代之的是，池头部存储了一个分离的叫做`nt!PoolBigTable`的表。因为大池分配需要在一个页边界(4K)对齐，所以这是有必要的。

调试命令`!pool`可以显示一个给定任何地址所在的池页中的所有池块。`!vm`显示池消耗信息。`!poolused`显示消耗的字节数以及所有池标签的块数。`!poolfind`定位一个特定标签的池分配。`!poolval`检查池头是否被污染，注意到他不会检查实际的池中数据是否被污染。`!frag`显示非分页池的外部池碎片信息。输出显示了系统中所有累计可用的非分页池页空闲块（碎片）的个数以及他们占用的内存量。

APIs：

- `ExAllocatePoolWithTag()`
- `ExAllocatePoolWithQuotaTag()`
- `ExFreePool()`

### `nt!_MMVAD`

MMVAD结构代表虚拟地址描述符（vad）并用于描述一个进程用户模式虚拟地址空间中几乎连续的虚拟地址空间。每当进程虚拟地址的一部分通过`VirtualAlloc()`或`MapViewOfSection()`分配时，都有一个MMVAD结构被创建。MMVADs从非分页内存池分配并被组织成AVL树。每个进程都有它自己的VAD树，它仅仅用于描述用户模式虚拟地址空间，也就是说内核虚拟地址空间并没有VADs。

`StartingVpn`和`EndingVpn`字段包含高20位（x86）或高52位（x64）虚拟地址的起始和终止，通过VAD的区域描述。`LeftChild`和`RightChild`字段指向下一个VAD树底层的节点。

调试器命令`!vad`显示了进程的VAD结构体信息。

APIs：

- `ZwAllocateVirtualMemory()`
- `ZwMappedViewOfSection()`
- `MmMapLockedPagesSpecifyCache()`

## 缓存管理器

### `nt!_VACB`

系统缓存虚拟地址空间被分割成256K（由ntifs.h中常量`VACB_MAPPING_GRANULARITY`定义）大小的视图（views）块。这一数值也决定了文件被映射到系统缓存时的粒度和对齐。每个视图都持有一个包含关乎此视图信息的虚拟地址控制块(VACB)。

内核全局变量`CcNumberOfFreeVacbs`和`CcNumberOfFreeHighPriorityVacbs`一起决定了可以分配的VACB的数量。所有这样的VACB都被保存在一个`CcVacbFreeList`或`CcVacbFreeHighPriorityList`链表中。这一链接字段就是为此而准备的。

`BaseAddress`字段指向了系统缓存中视图的起始地址，描述它的VACB由函数`MmMapViewInSystemCache()`生成。

`SharedCacheMap`字段指向了共享缓存映射结构，它包含该VACB并描述了VACB映射到该视图中的文件映射的区段。

`ArrayHead`字段指向`VACB_ARRAY_HEADER`结构体，它包含了VACB。

调试器命令`!filecache`显示了使用中的VACB数据结构信息。

### `nt!_VACB_ARRAY_HEADER`

4095个VACB块是一起被分配的，同时分配了一个头`VACB_ARRAY_HEADER`，它用于管理VACB。`VACB_ARRAY_HEADER`结构附着在VACB数组的后面。

单个单元的VACB数组头分配尺寸为128K，它包含了`VACB_ARRAY_HEADER`以及跟随的4095个VACB结构体。因此每个单元可以映射最大1023MB的系统缓存虚拟地址空间（单个VACB映射256K）。`VACB_ARRAY_HEADER`结构以及嵌入的VACB的最大数量由系统全部的系统缓存虚拟地址空间限制，也就是说在x64上是1TB，在x86上是2GB。因此在x86系统上至多不会超过2个`VACB_ARRAY_HEADER`结构体。

内核变量`CcVacbArrays`指向了一个指针数组，这些指针指向`VACB_ARRAY_HEADER`结构体。`VacbArrayIndex`字段是特定`VACB_ARRAY_HEADER`结构体在数组中的索引值。变量`CcVacvArraysHighestUsedIndex`包含数组中上一次使用的那个索引值。该数组受`CcVacbSpinLock`这个队列自旋锁(queued spinlock)保护。

`VACB_ARRAY_HEADER`头结构的数量被存储在全局变量`CcVacbArraysAllocated`中，它包含了当前整个系统分配的头结构，它们由`CcVacbArrays`指向。

### `nt!_SHARED_CACHE_MAP`

`SHARED_CACHE_MAP`被缓存管理器用来存储关于文件当前被缓存到系统缓存虚拟地址空间的那些部分的信息。对一个缓存文件来说，有一个包含所有该文件已打开的实例的单一`SHARED_CACHE_MAP`结构体对象。所有的`FILE_OBJECT`表示了所有已打开的特定文件的实例，它们都通过`FILE_OBJECT`的`SectionObjectPointersSharedCacheMap`字段指向同一个`SHARED_CACHE_MAP`。

通过同一个`SHARED_CACHE_MAP`结构体可以访问相同文件流的所有映射的VACB。共享缓存映射结构保证了该文件的特定区段永远不会在缓存中除此之外的映射。

全局变量`CcDirtySharedCacheMapList`包含了所有的`SHARED_CACHE_MAP`结构体链表，它们包含携带脏数据的视图。链表中有一个特殊的项—全局变量`CcLazyWriterCursor`，以它开始的`SHARED_CACHE_MAP`结构体的子链都是懒回写（lazy written）。在每个懒回写完成后，`CcLazyWriterCursor`都在`CcDirtySharedCacheMapList`中移动。

包含了没有任何脏页的视图的`SHARED_CACHE_MAP`结构体被保存在全局链表`CcCleanSharedCacheMapList`中。`SharedCacheMapLinks`字段用于组织这两种队列（dirty or clean）。

`SectionSize`字段决定了通过`SHARED_CACHE_MAP`映射的区段的大小。

`InitialVacbs`字段是一个内建的4 VACB指针数组，它用于映射那些小于1MB的文件区段。如果区段大小超过了1MB，一个128 VACB的指针数组会被分配并存储在`Vacbs`字段中，它指向了当前可以描述文件到32MB(也就是`128*256K`)大小的VACBs。如果区段尺寸超过32MB，则指针数组的128个指针都用来指向另一个128 VACB指针数组。这种额外的层级设计使得区段大小可以达到4GB(`128*128*256K`)。一共可以有7层VACB，尺寸也就是(`128^7*256K`)，这将远高于缓存管理器提供的最大区段尺寸（2^63）。

函数`CcCreateVacbArray()`层级了VACB数组，所有的VACB数组都由一个推锁(`VacbLock`字段)保护。

`PrivateList`是一个链表的头，它用于保存和每个文件打开实例,也就是`FILE_OBJECT`相关联的`PRIVATE_CACHE_MAP`结构体。`PRIVATE_CACHE_MAP.PrivateLinks`字段用于形成链表。

调试命令`!fileobj`可以显示关于`SECTION_OBJECT_POINTER`结构体的信息，它包含一个指向`SHARED_CACHE_MAP`的指针。

### `nt!_PRIVATE_CACHE_MAP`

缓存管理器执行一个文件的智能预阅读来提升性能。这些预阅读在每个特定文件打开的实例上都独立执行。和每个文件打开实例相关联的`PRIVATE_CACHE_MAP`结构体，保存了一个上次文件阅读操作的历史记录，它被缓存管理器用于执行智能预阅读操作。

`FILE_OBJECT.PrivateCacheMap`指向了和文件打开实例相关联的`PRIVATE_CACHE_MAP`结构体。当该文件缓存被激活时该字段由`CcInitializeCacheMap()`初始化，同样当缓存被清掉时则通过`CcUninitializeCacheMap()`完成。

`FileObject`字段，指向了和`PRIVATE_CACHE_MAP`相关联的`FILE_OBJECT`。预阅读操作仅在`FILE_OBJECT.Flags`字段的`FO_RANDOM_ACCESS`位未置位时才会执行。

`SHARED_CACHE_MAP.PrivateList`指向所有打开的特定文件的实例的`PRIVATE_CACHE_MAP`结构体。特定文件的打开实例的`PRIVATE_CACHE_MAP`结构体通过`PrivateLinks`字段链在一起。

`FileOffset1`,`FileOffset2`,`BeyondLastBye`,`BeyondLastByte2`四个字段一起用于决定在特定`FILE_OBJECT`对应的文件上阅读的模式。缓存管理器函数`CcUpdateReadHistory()`用来更新这些读操作的计数。

`Flags.ReadAheadEnabled`字段决定了预读操作对该文件打开实例来说是否是需要的。`Flags.ReadAheadActive`字段由`CcScheduleReadAhead()`设置，标志了`ReadAheadWorkItem`字段的读工作例程当前是否是活跃的(active)。

调试命令`!fileobj`显示了关于`PRIVATE_CACHE_MAP`的信息。

APIs：

- `CcInitializeCacheMap()`
- `CcUninitializeCacheMap()`
- `CcIsFileCached()`

### `nt!_SECTION_OBJECT_POINTERS`

`SECTION_OBJECT_POINTERS`和文件对象相关联，它指向了文件映射以及该文件的缓存相关信息。一个单一文件可以有2个分离的映射，一个为可执行文件，另一个为数据文件。

`DataSectionObject`字段指向了控制区域，它是一种服务于内存管理器和文件系统之间的一个链接，它用于内存映射数据文件。

`ImageSectionObject`字段指向另一个控制区域结构，它用于内存映射可执行文件。一个数据映射由单一连续的具有相同保护属性的VA组成，一个映像映射则由可执行体不同的区段映射的多个不同保护属性的区域组成。

`SharedCacheMap`字段指向了文件的`SHARED_CACHE_MAP`结构，它描述了该文件被缓存在那里，缓存了哪些部分。

上面提到的`SECTION_OBJECT_POINTERS`结构体中的字段，是由内存管理器和文件系统驱动配合该文件对象的`SECTION_OBJECT_POINTERS`设置的。

APIs：

- `CcFlushCache()`
- `CcPurgeCacheSection()`
- `CcCoherencyFlushAndPurgeCache()`
- `MmFlushImageSection()`
- `MmForceSectionClosed()`
- `MmCanFileBeTruncated()`
- `MmDoesFileHaveUserWritableReferences()`
- `CcGetFileObjectFromSectionPtrsRef()`
- `CcSetFileSizes()`

