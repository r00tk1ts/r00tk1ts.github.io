---
title: Linux内核学习——内存管理之页框管理
date: 2017-10-20 20:10:11
categories: Linux
tags:
	- linux-kernel
	- ulk
---
Linux内存管理非常复杂，从物理内存的分门别类，到内核用户态进程的索取，可以说是包罗万象。这一篇学习页框管理机制的设计以及透过源码了解一些技术细节。

<!--more-->

# Linux内存管理——页框管理

在“内存寻址”一节中，我们知道了描述一个物理页的单位叫页框(page frame)。页框的尺寸大小和硬件和Linux系统配置有关。对Intel来说，Linux采用4KB页框尺寸作为标准内存分配单元。

> 1. 开启PAE的情况下，为2MB。
> 2. 如果开启了大页，那么页框尺寸为4MB。

页框之所以选择4KB大小，ULK给出了两方面原因：

1. 分页系统造成的缺页异常很好解释，要么是页存在但process不被允许访问，要么是页不存在。如果不存在，内存分配器需要找一个free 4KB页框给process。（这样一来管理虚拟地址空间的分页系统的4KB单位与页框大小一致，处理起来也就省事。此外，关于缺页异常实际上也没这里描摹的那么简单，ULK这里的说法是高度抽象的。）
2. 尽管4K和4M都是磁盘块尺寸的倍数，但内存和磁盘间往往传递小块更为的高效。（这是考虑到页的换入换出机制，磁盘IO比内存寻址慢的多，页越大那么单次执行换页操作就越耗时，这不难理解。）

物理内存可以分成静态+动态两部分，其中静态部分永久保留，用于硬件和Linux内核代码/静态数据结构。动态内存部分则是Linux系统RT所使用和管理。

物理内存的动静态布局大致如此：

![](20171020_1.jpg)

## 内存模型

动态内存是如何管理的呢？这将牵扯到各个层面的管理模块，每个模块都足够复杂。在探索之前，我们先来搞清楚Linux在设计上如何抽象动态内存。

### 平坦模式

所谓平坦模式，是指物理地址空间连续没有空洞。这是最为理想化的模型，管理也简单。虚拟地址空间位于线性映射区域的页与物理地址空间的页帧一一对应，利用内核的某种映射机制来相互转换。

### 非连续模式

cpu访问物理内存，地址空间往往有着空洞。这种模式本质上是平坦模式的扩展，它相当于用空洞做间隔，把物理内存拆分成多个平坦空间，而虚拟地址空间就多了一个索引转换层，每个内存页需要去索引属于哪一块物理内存区。

> 为什么设计上要自找麻烦呢？用平坦模式不好吗？当然是因为外因造成的，比如NUMA架构（详见下文）。

### 稀疏模式

这种模式就比较新潮了，是内核发展到现代所演变出来的一种模式。平坦模式是把物理页全部抽象成一片连续的地址空间，所有页page都在`mem_maps[]`中，而因为NUMA架构，物理内存空间不再连续，原本的单一`mem_maps[]`变成了`mem_maps[][]`，多了一个维度来索引某一块物理连续内存空间。而稀疏模式则在此基础上，引入了section的概念，每个section内部内存连续，而mem_map的page数组依附于section而不再依附于node(`struct pglist_data`)。



由于我们分析2.6.x内核，所以只会涉及到平坦和非连续模式，着重探索平坦模式，NUMA架构只是做一些介绍。

## 物理内存的三个层次

宏观上的设计大抵如此，但如果不了解微观上的各相关结构意义与耦合关系，读内核代码势必云里雾里。物理内存的管理上，有三个最为基础的数据结构：`struct page`, `struct pg_data_t`和`struct zone` 。

### page
内核记录了每个页框的状态：属于哪个进程；属于内核代码或数据结构；空闲等等。页框的状态保存在页描述符，其类型为`struct page`。内核把所有的页描述符都存在全局`mem_map`数组（通常存放在ZONE_NORMAL首部）。page是最为基本的存储单元，可以看成是字节数组，标准大小为4KB。

```c
/*
 * Each physical page in the system has a struct page associated with
 * it to keep track of whatever it is we are using the page for at the
 * moment. Note that we have no way to track which tasks are using
 * a page.
 */
struct page {
	/**
	 * 一组标志，也对页框所在的管理区进行编号
	 * 在不支持NUMA的机器上，flags中字段中管理索引占两位，节点索引占一位。
	 * 在支持NUMA的32位机器上，flags中管理索引占用两位。节点数目占6位。
	 * 在支持NUMA的64位机器上，64位的flags字段中，管理区索引占用两位，节点数目占用10位。
	 * page_flags_t实际上是个无符号整型数
	 */
	page_flags_t flags;		/* Atomic flags, some possibly
					 * updated asynchronously */
	/**
	 * 页框的引用计数。当小于0表示没有人使用。
	 * Page_count返回_count+1表示正在使用的人数。
	 */
	atomic_t _count;		/* Usage count, see below. */
	/**
	 * 页框中的页表项数目（没有则为-1）
	 *		-1:		表示没有页表项引用该页框。
	 *		0:		表明页是非共享的。
	 *		>0:		表示而是共享的。
	 */
	atomic_t _mapcount;		/* Count of ptes mapped in mms,
					 * to show when page is mapped
					 * & limit reverse map searches.
					 */
	/**
	 * 可用于正在使用页的内核成分（如在缓冲页的情况下，它是一个缓冲器头指针。）
	 * 如果页是空闲的，则该字段由伙伴系统使用。
	 * 当用于伙伴系统时，如果该页是一个2^k的空闲页块的第一个页，那么它的值就是k.
	 * 这样，伙伴系统可以查找相邻的伙伴，以确定是否可以将空闲块合并成2^(k+1)大小的空闲块。
	 */
	unsigned long private;		/* Mapping-private opaque data:
					 * usually used for buffer_heads
					 * if PagePrivate set; used for
					 * swp_entry_t if PageSwapCache
					 * When page is free, this indicates
					 * order in the buddy system.
					 */
	/**
	 * 当页被插入页高速缓存时使用或者当页属于匿名页时使用）。
	 * 		如果mapping字段为空，则该页属于交换高速缓存(swap cache)。
	 *		如果mapping字段不为空，且最低位为1，表示该页为匿名页。同时该字段中存放的是指向anon_vma描述符的指针。
	 *		如果mapping字段不为空，且最低位为0，表示该页为映射页。同时该字段指向对应文件的address_space对象。
	 */
	struct address_space *mapping;	/* If low bit clear, points to
					 * inode address_space, or NULL.
					 * If page mapped as anonymous
					 * memory, low bit is set, and
					 * it points to anon_vma object:
					 * see PAGE_MAPPING_ANON below.
					 */
	/**
	 * 作为不同的含义被几种内核成分使用。
	 * 在页磁盘映象或匿名区中表示存放在页框中的数据的位置。
	 * 或者它存放在一个换出页标志符。
	 * 表示所有者的地址空间中以页大小为单位的偏移量，
	 * 也就是磁盘映像中页中数据的位置
	 * page->index是区域内的页索引或是页的线性地址除以PAGE_SIZE
	 * 
	 * liufeng: 
	 * 不是页内偏移量，而是该页面相对于文件起始位置，以页面为大小的偏移量
	 * 如果减去vma->vm_pgoff，就表示该页面的虚拟地址相对于vma起始地址，以页面为大小的偏移量
	 */
	pgoff_t index;			/* Our offset within mapping. */
	/**
	 * 包含页的最近最少使用的双向链表的指针。
	 */
	struct list_head lru;		/* Pageout list, eg. active_list
					 * protected by zone->lru_lock !
					 */
	/*
	 * On machines where all RAM is mapped into kernel address space,
	 * we can simply calculate the virtual address. On machines with
	 * highmem some memory is mapped into kernel virtual memory
	 * dynamically, so we need a place to store that address.
	 * Note that this field could be 16 bits on x86 ... ;)
	 *
	 * Architectures with slow multiplication can define
	 * WANT_PAGE_VIRTUAL in asm/page.h
	 */
#if defined(WANT_PAGE_VIRTUAL)
	/**
	 * 如果进行了内存映射，就是内核虚拟地址。对存在高端内存的系统来说有意义。
	 * 否则是NULL
	 */
	void *virtual;			/* Kernel virtual address (NULL if
					   not kmapped, ie. highmem) */
#endif /* WANT_PAGE_VIRTUAL */
};
```

每个页框都有一个page描述符，内部的各个字段与耦合的各个模块有关，比如buddy system，lru pageout链等。另外也可以看出内核设计上对空间的取用近乎苛刻，虽然不似WRK中充斥着大量的union语法（较新的内核4.x版本已经用union重写了，为了增强可读性），但同一个成员在不同的场景下其代表意义不同，并非单一职责。

此外，在Linux内核中，有个非常重要的全局指针变量`mem_map`:

```c
/**
 * 内存映射数组。管理区描述符的zone_mem_map指向它的一个元素。
 * 用于伙伴系统。
 * 
 */
struct page *mem_map;
```

实际上是个flexible数组。

flags字段非常重要，取值非常多且意义深远：

```c
typedef enum page_buf_flags_e {		/* pb_flags values */
	PBF_READ = (1 << 0),	/* buffer intended for reading from device */
	PBF_WRITE = (1 << 1),	/* buffer intended for writing to device   */
	PBF_MAPPED = (1 << 2),  /* buffer mapped (pb_addr valid)           */
	PBF_PARTIAL = (1 << 3), /* buffer partially read                   */
	PBF_ASYNC = (1 << 4),   /* initiator will not wait for completion  */
	PBF_NONE = (1 << 5),    /* buffer not read at all                  */
	PBF_DELWRI = (1 << 6),  /* buffer has dirty pages                  */
	PBF_STALE = (1 << 7),	/* buffer has been staled, do not find it  */
	PBF_FS_MANAGED = (1 << 8),  /* filesystem controls freeing memory  */
	PBF_FS_DATAIOD = (1 << 9),  /* schedule IO completion on fs datad  */
	PBF_FORCEIO = (1 << 10),    /* ignore any cache state		   */
	PBF_FLUSH = (1 << 11),	    /* flush disk write cache		   */
	PBF_READ_AHEAD = (1 << 12), /* asynchronous read-ahead		   */

	/* flags used only as arguments to access routines */
	PBF_LOCK = (1 << 14),       /* lock requested			   */
	PBF_TRYLOCK = (1 << 15),    /* lock requested, but do not wait	   */
	PBF_DONT_BLOCK = (1 << 16), /* do not block in current thread	   */

	/* flags used only internally */
	_PBF_PAGE_CACHE = (1 << 17),/* backed by pagecache		   */
	_PBF_KMEM_ALLOC = (1 << 18),/* backed by kmem_alloc()		   */
	_PBF_RUN_QUEUES = (1 << 19),/* run block device task queue	   */
} page_buf_flags_t;
```

参考ULK的释义：
![](20171020_2.jpg)

flags包含了各个耦合模块上相对的意义，设计上也可以看得出来其枚举值是可以叠加而不交叉的。关于它们的意义，具体读到相关代码时再做阐述。

### node

如果所有内存都是均质的，那么物理地址整个都是平坦的，一气呵成也就不需要进行分类，但现实中往往不是所有的内存都是均质的，不同架构寻址内存的不同空间速度是不同的（Alpha,mips,etc.）。所以内存的使用布局就很有讲究，2.6支持NUMA，物理内存被瓜分成多个nodes。不同的node中的page同一CPU访问速度是相同的。所以一个node就表示一种物理内存。

```cpp
/*
 * The pg_data_t structure is used in machines with CONFIG_DISCONTIGMEM
 * (mostly NUMA machines?) to denote a higher-level memory zone than the
 * zone denotes.
 *
 * On NUMA machines, each NUMA node would have a pg_data_t to describe
 * it's memory layout.
 *
 * Memory statistics and page replacement data structures are maintained on a
 * per-zone basis.
 */
struct bootmem_data;
/**
 * 内存管理结点描述符。每个描述符中的物理内存，对CPU来说，访问是一致的。
 * 但是每个管理结点又包含了不同的管理区。
 */
typedef struct pglist_data {
	/**
	 * 节点管理区描述符数组
	 */
	struct zone node_zones[MAX_NR_ZONES];
	/**
	 * 页分配器使用的zonelist数据结构的数组。
	 * 实际上这个东西对NUMA来说意义非凡，它可以承载备用节点，在当前节点没有空间时使用备用
	 */
	struct zonelist node_zonelists[GFP_ZONETYPES];
	/**
	 * 节点中管理区的个数
	 */
	int nr_zones;
	/**
	 * 节点中页描述符的数组
	 */
	struct page *node_mem_map;
	/**
	 * 用在内核初始化阶段
	 * 内存管理子系统初始化前，内核页需要使用内存
	 * 此结构用于这个阶段的内存管理，内核使用自举内存分配器
	 */
	struct bootmem_data *bdata;
	/**
	 * 节点中第一个页框的下标。
	 * 系统中所有的页帧是依次编号的，每个页帧的号码都是全局唯一的
	 */
	unsigned long node_start_pfn;
	/**
	 * 内存结点的大小，不包含空洞（以页为单位）
	 */
	unsigned long node_present_pages; /* total number of physical pages */
	/**
	 * 节点的大小，包括空洞
	 */
	unsigned long node_spanned_pages; /* total size of physical page
					     range, including holes */
	/**
	 * 节点标识符
	 */
	int node_id;
	/**
	 * 内存节点链表的下一项。该字段构成node单链表。
	 */
	struct pglist_data *pgdat_next;
	/**
	 * Kswapd页换出守护进程使用的等待队列
	 */
	wait_queue_head_t kswapd_wait;
	/**
	 * 指针指向kswapd内核线程的进程描述符。
	 */
	struct task_struct *kswapd;
	/**
	 * Kswapd将要创建的空闲块大小取对数的值。
	 */
	int kswapd_max_order;
} pg_data_t;
```

每个node又可以分成多个zones，每个node有个类型为`pg_data_t`的描述符，所有node描述符都存储在单一链，第一个元素由`pgdat_list`变量所指向。

```c
/**
 * 内核将物理内存分为几个结点。
 * 每个结点上的内存，对当前CPU来说，其访问时间是相等的。
 * pgdat_list是这些结点的单向列表。
 * 对x86来说，不支持NUMA，所以这个链表只有一个结点。这个结点保存在contig_page_data中。
 */
struct pglist_data *pgdat_list;
```

对于x86这种不支持NUMA的架构，node实际上只有一个，它在全局数组变量`contig_page_data`中：

```c
/**
 * 支持NUMA。
 * 对IBM来说，虽然并不真正需要NUMA支持，但是：即使NUMA的支持没有编译进内核，LINUX还是使用结点管理NUMA。
 * 不过，这是一个单独的结点。它包含了系统中所有的物理内存。
 * 这个元素由contig_page_data表示。它包含在一个只有一个结点的链表中，这个链表被pgdat_list指向。
 */
struct pglist_data contig_page_data = { .bdata = &contig_bootmem_data };
```

此外，`mem_map`在`node_alloc_mem_map`中被赋值，指向了`contig_page_data.node_mem_map`。

```c
void __init node_alloc_mem_map(struct pglist_data *pgdat)
{
	unsigned long size;

	size = (pgdat->node_spanned_pages + 1) * sizeof(struct page);
	pgdat->node_mem_map = alloc_bootmem_node(pgdat, size);
#ifndef CONFIG_DISCONTIGMEM
	mem_map = contig_page_data.node_mem_map;
#endif
}
```

NUMA架构比UMA架构复杂得多，整个系统的内存由`node_data`这个`pg_data_t`数组管理，而UMA只有一个简单的`contig_page_data`。

可以用`NODE_DATA(node_id)`来查找编号为`node_id`的节点，对于UMA架构，node只有一个，所以该宏总是返回全局的`contig_page_data`。

本文只讨论UMA架构，在UMA系统中, 内存就相当于一个只使用一个NUMA节点来管理整个系统的内存。而内存管理的其他地方则认为他们就是在处理一个(伪)NUMA系统。

### zone

理想架构中，页框作为内存存储单元可以存储内核和用户数据，磁盘数据缓存等等。任何页数据都可以存在任何页框中，没有任何限制。然而真实的架构有着硬件限制。

Linux内核必须处理80x86架构的两个硬件限制：
1. 旧ISA总线的DMA处理器有一个严格的限制：它们只能寻址RAM的前16MB空间。
2. 现代32位计算机有很多RAM，CPU不能直接访问所有的物理内存，因为线性地址空间太小了。(内核线性地址只有3G到4G这1GB，而RAM当前比这大得多，设计成一对一肯定是不行的，所以映射机制是个复杂的大杂烩。)

为了应付这两个限制，Linux在80x86 UMA架构中分割物理内存node成3个zones。
- ZONE_DMA
  - 包含低16MB的内存页框，用于兼容旧ISA总线DMA处理器
- ZONE_NORMAL
  - 包含16MB到896MB的内存页框
- ZONE_HIGHMEM
  - 包含高于896MB的内存页框

ZONE_DMA给旧ISA总线设备用。ZONE_DMA和ZONE_NORMAL包含内存的常规页框，通过线性映射到4GB的线性地址空间，它们可以被内核直接使用。ZONE_HIGHMEM包含不能够直接访问的内存页框。此外，因为x64虚拟地址空间的膨胀，ZONE_HIGHMEN zone在64bit机器上总是空的（没有存在的必要）。

不同的硬件架构node分成的zone类别也有所不同，但大体上来讲都有两个非常重要的zone，一个是直接线性映射的NORMAL区，比如x86上的16M到896M这一部分偏移量的物理内存可以直接映射到3G+16M到3G+896M的虚拟线性地址上。而HIGHMEM则比较复杂，它被各种机制安排到剩余的128MB（1024MB-896MB）（不同架构可能不一样）线性空间上（也就是靠近4G的最后128MB），这些机制我们过后再探索。

```cpp
/**
 * 内存管理区描述符
 */
struct zone {
	/* Fields commonly accessed by the page allocator */
	/**
	 * 管理区中空闲页的数目
	 */
	unsigned long		free_pages;
	/**
	 * Pages_min-管理区中保留页的数目
	 * Page_low-回收页框使用的下界。同时也被管理区分配器为作为阈值使用。
	 * pages_high-回收页框使用的上界，同时也被管理区分配器作为阈值使用。
	 */
	unsigned long		pages_min, pages_low, pages_high;
	/*
	 * We don't know if the memory that we're going to allocate will be freeable
	 * or/and it will be released eventually, so to avoid totally wasting several
	 * GB of ram we must reserve some of the lower zone memory (otherwise we risk
	 * to run OOM on the lower zones despite there's tons of freeable ram
	 * on the higher zones). This array is recalculated at runtime if the
	 * sysctl_lowmem_reserve_ratio sysctl changes.
	 */
	/**
	 * 为内存不足保留的页框，分别为各种内存域指定了若干页
	 * 用于一些无论如何都不能失败的关键性内存分配
	 */
	unsigned long		lowmem_reserve[MAX_NR_ZONES];
	/**
	 * 用于实现单一页框的特殊高速缓存。
	 * 每内存管理区对每CPU都有一个。包含热高速缓存和冷高速缓存。
	 * 内核使用这些列表来保存可用于满足实现的“新鲜”页。
	 * 有些页帧很可能在CPU高速缓存中，因此可以快速访问，称之为热。
	 * 未缓存的页帧称之为冷的。
	 */
	struct per_cpu_pageset	pageset[NR_CPUS];

	/*
	 * free areas of different sizes
	 */
	/**
	 * 保护该描述符的自旋锁
	 */
	spinlock_t		lock;
	/**
	 * 标识出管理区中的空闲页框块。
	 * 包含11个元素，被伙伴系统使用。分别对应大小的1,2,4,8,16,32,128,256,512,1024连续空闲块的链表。
	 * 第k个元素标识所有大小为2^k的空闲块。free_list字段指向双向循环链表的头。
	 * free_list是free_area的内部结构，是个双向环回链表节点。
	 */
	struct free_area	free_area[MAX_ORDER];

	/* 
	 * 为了cache line对齐加的pad
	 */
	ZONE_PADDING(_pad1_)

	/* Fields commonly accessed by the page reclaim scanner */
	/**
	 * 活动以及非活动链表使用的自旋锁。
	 */
	spinlock_t		lru_lock;	
	/**
	 * 管理区中的活动页链表
	 */
	struct list_head	active_list;
	/**
	 * 管理区中的非活动页链表。
	 */
	struct list_head	inactive_list;
	/**
	 * 回收内存时需要扫描的活动页数。
	 */
	unsigned long		nr_scan_active;
	/**
	 * 回收内存时需要扫描的非活动页数目
	 */
	unsigned long		nr_scan_inactive;
	/**
	 * 管理区的活动链表上的页数目。
	 */
	unsigned long		nr_active;
	/**
	 * 管理区的非活动链表上的页数目。
	 */
	unsigned long		nr_inactive;
	/**
	 * 管理区内回收页框时使用的计数器。
	 */
	unsigned long		pages_scanned;	   /* since last reclaim */
	/**
	 * 在管理区中填满不可回收页时此标志被置位
	 */
	int			all_unreclaimable; /* All pages pinned */

	/*
	 * prev_priority holds the scanning priority for this zone.  It is
	 * defined as the scanning priority at which we achieved our reclaim
	 * target at the previous try_to_free_pages() or balance_pgdat()
	 * invokation.
	 *
	 * We use prev_priority as a measure of how much stress page reclaim is
	 * under - it drives the swappiness decision: whether to unmap mapped
	 * pages.
	 *
	 * temp_priority is used to remember the scanning priority at which
	 * this zone was successfully refilled to free_pages == pages_high.
	 *
	 * Access to both these fields is quite racy even on uniprocessor.  But
	 * it is expected to average out OK.
	 */
	/**
	 * 临时管理区的优先级。
	 */
	int temp_priority;
	/**
	 * 管理区优先级，范围在12和0之间。
	 */
	int prev_priority;


	ZONE_PADDING(_pad2_)
	/* Rarely used or read-mostly fields */

	/*
	 * wait_table		-- the array holding the hash table
	 * wait_table_size	-- the size of the hash table array
	 * wait_table_bits	-- wait_table_size == (1 << wait_table_bits)
	 *
	 * The purpose of all these is to keep track of the people
	 * waiting for a page to become available and make them
	 * runnable again when possible. The trouble is that this
	 * consumes a lot of space, especially when so few things
	 * wait on pages at a given time. So instead of using
	 * per-page waitqueues, we use a waitqueue hash table.
	 *
	 * The bucket discipline is to sleep on the same queue when
	 * colliding and wake all in that wait queue when removing.
	 * When something wakes, it must check to be sure its page is
	 * truly available, a la thundering herd. The cost of a
	 * collision is great, but given the expected load of the
	 * table, they should be so rare as to be outweighed by the
	 * benefits from the saved space.
	 *
	 * __wait_on_page_locked() and unlock_page() in mm/filemap.c, are the
	 * primary users of these fields, and in mm/page_alloc.c
	 * free_area_init_core() performs the initialization of them.
	 */
	/**
	 * 进程等待队列的散列表。这些进程正在等待管理区中的某页。
	 */
	wait_queue_head_t	* wait_table;
	/**
	 * 等待队列散列表的大小。
	 */
	unsigned long		wait_table_size;
	/**
	 * 等待队列散列表数组的大小。值为2^order
	 */
	unsigned long		wait_table_bits;

	/*
	 * Discontig memory support fields.
	 */
	/**
	 * 内存节点。
	 */
	struct pglist_data	*zone_pgdat;
	/** 
	 * 指向管理区的第一个页描述符的指针。这个指针是数组mem_map的一个元素。
	 */
	struct page		*zone_mem_map;
	/* zone_start_pfn == zone_start_paddr >> PAGE_SHIFT */
	/**
	 * 管理区的第一个页框的下标。
	 */
	unsigned long		zone_start_pfn;

	/**
	 * 以页为单位的管理区的总大小，包含空洞。
	 */
	unsigned long		spanned_pages;	/* total size, including holes */
	/**
	 * 以页为单位的管理区的总大小，不包含空洞。
	 */
	unsigned long		present_pages;	/* amount of memory (excluding holes) */

	/*
	 * rarely used fields:
	 */
	/**
	 * 指针指向管理区的传统名称：DMA、NORMAL、HighMem
	 */
	char			*name;
} ____cacheline_maxaligned_in_smp;
```

> 由于 `struct zone` 结构经常被访问到, 因此这个数据结构要求以 `L1 Cache` 对齐. 另外, 这里的 `ZONE_PADDING( )` 让 `zone->lock` 和 `zone_lru_lock` 这两个很热门的锁可以分布在不同的 `Cahe Line` 中. 一个内存 `node` 节点最多也就几个 `zone`, 因此 `zone` 数据结构不需要像 `struct page` 一样关心数据结构的大小, 因此这里的 `ZONE_PADDING( )` 可以理解为用空间换取时间(性能). 在内存管理开发过程中, 内核开发者逐渐发现有一些自选锁竞争会非常厉害, 很难获取. 像 `zone->lock` 和 `zone->lru_lock` 这两个锁有时需要同时获取锁. 因此保证他们使用不同的 `Cache Line` 是内核常用的一种优化技巧.

回过头看page，page中有到node和zone的链接，为了省空间，并没有设定单独的指针指向，而是编码成索引放在flags字段的高位(2.4.18前，page是有个指向zone的指针的，但page那么多，每个都浪费一个指针大小，浪费的内存还是相当可观的，对于linux这种性能癖，当然优化掉了)。源码中可以看到flags的标志数目是有限的，flags字段的高位可以用于编码特定node和zone号（对32位非NUMA，zone索引2位，node索引1位。（因为zone有3个而node只有1个））。

page_zone()接收一个page地址作为参数，读取flags中的最高位，通过查看zone_table来确定zone的地址：
```cpp
/**
 * 接收一个页描述符的地址作为它的参数，它读取页描述符的flags字段的高位，并通过zone_table数组来确定相应管理区描述符的地址
 */
static inline struct zone *page_zone(struct page *page)
{
	return zone_table[page->flags >> NODEZONE_SHIFT];
}

#define NODEZONE_SHIFT (sizeof(page_flags_t)*8 - MAX_NODES_SHIFT - MAX_ZONES_SHIFT)

/* There are currently 3 zones: DMA, Normal & Highmem, thus we need 2 bits */
#define MAX_ZONES_SHIFT		2

#ifndef CONFIG_DISCONTIGMEM

extern struct pglist_data contig_page_data;
#define NODE_DATA(nid)		(&contig_page_data)
#define NODE_MEM_MAP(nid)	mem_map
#define MAX_NODES_SHIFT		1
#define pfn_to_nid(pfn)		(0)

#else /* CONFIG_DISCONTIGMEM */
//只关心non-NUMA
```
内核调用内存分配函数时，必须指明请求page所在的zone。内核一般指明它想用的zone。内核使用`zonelist`数组指明首选的zone：
```cpp
/*
 * One allocation request operates on a zonelist. A zonelist
 * is a list of zones, the first one is the 'goal' of the
 * allocation, the other zones are fallback zones, in decreasing
 * priority.
 *
 * Right now a zonelist takes up less than a cacheline. We never
 * modify it apart from boot-up, and only a few indices are used,
 * so despite the zonelist table being relatively big, the cache
 * footprint of this construct is very small.
 */
struct zonelist {
	struct zone *zones[MAX_NUMNODES * MAX_NR_ZONES + 1]; // NULL delimited
};
```

可以看到`zonelist`不过是zone的一个集成，聚合了所有node的所有zone，它的内部究竟如何布局，具体分配时又如何使用等到阅读内存分配代码时再一探究竟。

## 保留的页框池

一方面内存有时会不够用，一些操作会被阻塞到有其他内存free。另一方面内核的某些操作是非常不想阻塞的，所以折中的办法就是留一个保留区。保留内存的数量由`min_free_kbytes`决定，单位为KB。这个值可以动态修改，通过sysctl系统调用或者直接写/proc文件系统下的内核变量。
```cpp
/**
 * 内核保留内存池大小。
 * 一般等于sqrt(16*内核直接映射内存大小)
 * 但是不能小于128也不能大于65536
 * 管理员可以通过写入/proc/sys/vm/min_free_kbytes来改变这个值。
 */
int min_free_kbytes = 1024;
```

zone的`pages_min`存储管理区内保留页框的数目。连同`pages_low`，`pages_high`在页框回收算法中会用到。

## Zoned Page Frame分配器

zoned page frame分配器是一个内核子系统，处理一组连续的页框分配请求。

**概要：**
分配器接收分配和释放请求。分配时，组件搜索包含连续page frames的zone来满足请求。在每个zone内，page frames由组件"buddy system"控制。为了得到更高的性能，少量的page frames放在缓存中以满足多次对单个页面的分配请求。

### 分配接口
分配页框可以使用6个不同的函数和宏，直接对照源码最易理解。

note：以下源码取非NUMA配置。
```cpp
/**
 * 分配2^order个连续的页框。它返回第一个所分配页框描述符的地址或者返回NULL
 */
#define alloc_pages(gfp_mask, order) \
		alloc_pages_node(numa_node_id(), gfp_mask, order)

/**
 * 用于获得一个单独页框的宏
 * 它返回所分配页框描述符的地址，如果失败，则返回NULL
 */
#define alloc_page(gfp_mask) alloc_pages(gfp_mask, 0)

/* 可以看出alloc_page就是alloc_pages的特例 */

/**
 * 类似于alloc_pages，但是它返回第一个所分配页的线性地址。
 */
fastcall unsigned long __get_free_pages(unsigned int gfp_mask, unsigned int order)
{
	struct page * page;
	page = alloc_pages(gfp_mask, order);
	if (!page)
		return 0;
  	// 实际上是page->virtual
	return (unsigned long) page_address(page);
}

/* 可以看到__get_free_pages内部也是调用了alloc_pages，只是返回的page由page_address转成了线性地址 */

/**
 * 用于获得一个单独页框的宏。
 */
#define __get_free_page(gfp_mask) \
		__get_free_pages((gfp_mask),0)
/* __get_free_page亦是__get_free_pages的特例 */


/**
 * 用于获取填满0的页框。
 * 返回的是页框对应的虚拟地址VA
 */
fastcall unsigned long get_zeroed_page(unsigned int gfp_mask)
{
	struct page * page;

	/*
	 * get_zeroed_page() returns a 32-bit address, which cannot represent
	 * a highmem page
	 */
	BUG_ON(gfp_mask & __GFP_HIGHMEM);

	page = alloc_pages(gfp_mask | __GFP_ZERO, 0);
	if (page)
		return (unsigned long) page_address(page);
	return 0;
}

/* 显然也是alloc_pages的特例，给定了__GFP_ZERO标志 */

/**
 * 用于获得适用于dma的页框。
 */
#define __get_dma_pages(gfp_mask, order) \
		__get_free_pages((gfp_mask) | GFP_DMA,(order))

/* 又是个特例，给定了GFP_DMA标志 */
```

可以看出这一族alloc函数都是对`alloc_pages`的包装，做了些善后工作，传递了不同的参数。

所以最后的谜底就在于`alloc_pages`中的`alloc_pages_node`函数：

```cpp
static inline struct page *alloc_pages_node(int nid, unsigned int gfp_mask,
						unsigned int order)
{
	if (unlikely(order >= MAX_ORDER))
		return NULL;

	return __alloc_pages(gfp_mask, order,
		NODE_DATA(nid)->node_zonelists + (gfp_mask & GFP_ZONEMASK));
}
```
`alloc_pages_node`只是做心智健全检查，实际工作交给了`__alloc_pages`，注意参数中`zonelist`是通过`NODE_DATA(nid)`取出。

```c
extern struct pglist_data *node_data[];
#define NODE_DATA(nid)		(node_data[nid])
```

转去看看`__alloc_pages`这个内存区PF分配器的总入口函数：

```c
/*
 * This is the 'heart' of the zoned buddy allocator.
 */
/**
 * 请求分配一组连续页框，它是管理区分配器的核心
 * gfp_mask：在内存分配请求中指定的标志
 * order：   连续分配的页框数量的对数(实际分配的是2^order个连续的页框)
 * zonelist: zonelist数据结构的指针。该结构按优先次序描述了适于内存分配的内存管理区
 */
struct page * fastcall
__alloc_pages(unsigned int gfp_mask, unsigned int order,
		struct zonelist *zonelist)
{
	const int wait = gfp_mask & __GFP_WAIT;
	struct zone **zones, *z;
	struct page *page;
	struct reclaim_state reclaim_state;
	struct task_struct *p = current;
	int i;
	int classzone_idx;
	int do_retry;
	int can_try_harder;
	int did_some_progress;

	might_sleep_if(wait);

	/*
	 * The caller may dip into page reserves a bit more if the caller
	 * cannot run direct reclaim, or is the caller has realtime scheduling
	 * policy
	 */
	can_try_harder = (unlikely(rt_task(p)) && !in_interrupt()) || !wait;

	zones = zonelist->zones;  /* the list of zones suitable for gfp_mask */

	if (unlikely(zones[0] == NULL)) {
		/* Should this ever happen?? */
		return NULL;
	}

	classzone_idx = zone_idx(zones[0]);

 restart:
	/* Go through the zonelist once, looking for a zone with enough free */
	/**
 	 * 扫描包含在zonelist数据结构中的每个内存管理区
	 */
  	/*
  	 * 这里可以看到对zonelist的扫描顺序，是按索引从前向后的。
  	 * 那么zonelist是如何布局的呢？如何为不同zone区分高低贵贱呢？
  	 * 对于UMA架构比较简单，因为只有单一node，在build_zonelists中可以一探究竟
  	 * 简单归纳来说，HIGHMEM最廉价、NORMAL次之，DMA最昂贵。
  	 */
	for (i = 0; (z = zones[i]) != NULL; i++) {
		/**
		 * 对于每个内存管理区，该函数将空闲页框的个数与一个阀值进行比较
		 * 该值取决于内存分配标志、当前进程的类型及管理区被函数检查的次数。
		 * 实际上，如果空闲内存不足，那么每个内存管理区一般会被检查几次。
		 * 每一次在所请求的空闲内存最低量的基础上使用更低的值进行扫描。
		 * 因此，这段循环代码会被复制几次，而变化很小。
		 */

		/**
		 * zone_watermark_ok辅助函数接收几个参数，它们决定内存管理区中空闲页框个数的阀值min。
		 * 这是对内存管理区的第一次扫描，在第一次扫描中，阀值设置为z->pages_low
		 */
		if (!zone_watermark_ok(z, order, z->pages_low,
				       classzone_idx, 0, 0))
			continue;
		/*
		 * 这个函数就是关键的分配函数，它杂糅了伙伴系统分配策略 + 本地CPU高速缓存
		 */
		page = buffered_rmqueue(z, order, gfp_mask);
		if (page)
			goto got_pg;
	}

	/**
	 * 一般来说，应当在上一次扫描时得到内存。
	 * 运行到此，表示内存已经紧张了（xie.baoyou注：没有连续的页框可供分配了）
	 * 就唤醒kswapd内核线程来异步的开始回收页框。
	 */
	for (i = 0; (z = zones[i]) != NULL; i++)
		wakeup_kswapd(z, order);

	/*
	 * Go through the zonelist again. Let __GFP_HIGH and allocations
	 * coming from realtime tasks to go deeper into reserves
	 */
	/**
	 * 执行对内存管理区的第二次扫描，将值z->pages_min作为阀值传入。这个值已经在上一步的基础上降低了（pages_low一般是pages_min的5/4，pages_high一般是pages_min的3/2）。
	 * 当然，实际的min值还是要由can_try_harder和gfp_high确定。z->pages_min仅仅是一个参考值而已。
	 */
	for (i = 0; (z = zones[i]) != NULL; i++) {
		if (!zone_watermark_ok(z, order, z->pages_min,
				       classzone_idx, can_try_harder,
				       gfp_mask & __GFP_HIGH))
			continue;
		/*
		 * 第二次扫描后，可能因为阈值的降低也可能因为异步的kswapd内核线程回收了页框
		 * 此时已经可以满足分配需求了
		 */
		page = buffered_rmqueue(z, order, gfp_mask);
		if (page)
			goto got_pg;
	}

	/* This allocation should allow future memory freeing. */
	/**
	 * 上一步都还没有获得内存，系统内存肯定是不足了。
	 */

	/**
	 * 如果产生内存分配的内核控制路径不是一个中断处理程序或者可延迟函数，
	 * 并且它试图回收页框（PF_MEMALLOC，TIF_MEMDIE标志被置位）,那么才对内存管理区进行第三次扫描。
	 */
	if (((p->flags & PF_MEMALLOC) || unlikely(test_thread_flag(TIF_MEMDIE))) && !in_interrupt()) {
		/* go through the zonelist yet again, ignoring mins */
		for (i = 0; (z = zones[i]) != NULL; i++) {
			/**
			 * 本次扫描就不调用zone_watermark_ok，它忽略阀值，这样才能从预留的页中分配页。
			 * 允许这样做，因为是这个进程想要归还页框，那就暂借一点给它吧（呵呵，舍不得孩子套不到狼）。
			 */
			page = buffered_rmqueue(z, order, gfp_mask);
			if (page)
				goto got_pg;
		}

		/**
		 * 老天保佑，不要运行到这里来，实在是没有内存了。
		 * 不论是高端内存区还是普通内存区、还是DMA内存区，甚至这些管理区中保留的内存都没有了。
		 * 意味着我们的家底都完了。
		 */
		goto nopage;
	}

	/* Atomic allocations - we can't balance anything */
	/**
	 * 如果gfp_mask的__GFP_WAIT标志没有被置位，函数就返回NULL。你又不能等，实在没辙。
	 */
	if (!wait)
		goto nopage;

rebalance:
	/**
	 * 如果当前进程能够被阻塞，调用cond_resched检查是否有其他进程需要CPU
	 */
	cond_resched();

	/* We now go into synchronous reclaim */
	/**
	 * 设置PF_MEMALLOC标志来表示进程已经准备好执行内存回收。
	 */
	p->flags |= PF_MEMALLOC;
	reclaim_state.reclaimed_slab = 0;
	/**
	 * 将reclaim_state数据结构指针存入reclaim_state。这个结构只包含一个字段reclaimed_slab，初始值为0
	 */
	p->reclaim_state = &reclaim_state;

	/**
	 * 调用try_to_free_pages寻找一些页框来回收。
	 * 这个函数可能会阻塞当前进程。一旦返回，就重设PF_MEMALLOC，并再次调用cond_resched
	 */
	did_some_progress = try_to_free_pages(zones, gfp_mask, order);

	p->reclaim_state = NULL;
	p->flags &= ~PF_MEMALLOC;

	cond_resched();

	/**
	 * 如果已经回收了一些页框，那么执行第二遍扫描类似的操作。
	 */
	if (likely(did_some_progress)) {
		/*
		 * Go through the zonelist yet one more time, keep
		 * very high watermark here, this is only to catch
		 * a parallel oom killing, we must fail if we're still
		 * under heavy pressure.
		 */
		for (i = 0; (z = zones[i]) != NULL; i++) {
			if (!zone_watermark_ok(z, order, z->pages_min,
					       classzone_idx, can_try_harder,
					       gfp_mask & __GFP_HIGH))
				continue;

			page = buffered_rmqueue(z, order, gfp_mask);
			if (page)
				goto got_pg;
		}
	} else if ((gfp_mask & __GFP_FS) && !(gfp_mask & __GFP_NORETRY)) {
		/*
		 * Go through the zonelist yet one more time, keep
		 * very high watermark here, this is only to catch
		 * a parallel oom killing, we must fail if we're still
		 * under heavy pressure.
		 */
		/**
		 * 没有释放任何页框，说明内核遇到很大麻烦了。因为内存少又不能释放页框。
		 * 如果允许杀死进程：__GFP_FS被置位并且__GFP_NORETRY标志为0。
		 * 那就开始准备杀死进程吧。
		 */

		/**
		 * 再扫描一次内存管理区。
		 * 这样做有点莫名其妙，既然申请少一点的内存都不行，为什么还要传入z->pages_high？？它看起来更不会成功。
		 * 其实这样做还是有道理的：实际上，只有另一个内核控制路径已经杀死一个进程来回收它的内存后，这步才会成功。
		 * 因此，这步避免了两个（而不是一个）无辜的进程被杀死。
		 */
		for (i = 0; (z = zones[i]) != NULL; i++) {
			if (!zone_watermark_ok(z, order, z->pages_high,
					       classzone_idx, 0, 0))
				continue;

			page = buffered_rmqueue(z, order, gfp_mask);
			if (page)
				goto got_pg;
		}

		/**
		 * 还是不行，就杀死一些进程再试吧。
		 */
		out_of_memory(gfp_mask);
		/**
		 * let's go on
		 */
		goto restart;
	}

	/*
	 * Don't let big-order allocations loop unless the caller explicitly
	 * requests that.  Wait for some write requests to complete then retry.
	 *
	 * In this implementation, __GFP_REPEAT means __GFP_NOFAIL for order
	 * <= 3, but that may not be true in other implementations.
	 */
	/**
	 * 如果内存分配请求不能被满足，那么函数决定是否应当继续扫描内存管理区。
	 * 如果__GFP_NORETRY被清除，并且内存分配请求跨越了多达8个页框或者__GFP_REPEAT被置位，或者__GFP_NOFAIL被置位。
	 */
	do_retry = 0;
	if (!(gfp_mask & __GFP_NORETRY)) {
		if ((order <= 3) || (gfp_mask & __GFP_REPEAT))
			do_retry = 1;
		if (gfp_mask & __GFP_NOFAIL)
			do_retry = 1;
	}
	/**
	 * 要重试，就调用blk_congestion_wait 使进程休眠一会。再跳到rebalance 重试。
	 */
	if (do_retry) {
		blk_congestion_wait(WRITE, HZ/50);
		goto rebalance;
	}
	/**
	 * 既然不用重试，那就执行到nopage 返回NULL 了。
	 */
nopage:
	if (!(gfp_mask & __GFP_NOWARN) && printk_ratelimit()) {
		printk(KERN_WARNING "%s: page allocation failure."
			" order:%d, mode:0x%x\n",
			p->comm, order, gfp_mask);
		dump_stack();
	}
	return NULL;
got_pg:
	zone_statistics(zonelist, z);
	return page;
}

/*
 * Return 1 if free pages are above 'mark'. This takes into account the order
 * of the allocation.
 */
/**
 * zone_watermark_ok辅助函数接收几个参数，它们决定内存管理区中空闲页框个数的阀值min。
 * 特别的，如果满足下列两个条件，则该函数返回1：
 *     1、除了被分配的页框外，在内存管理区中至少还有min个空闲页框，不包括为内存不足保留的页框（zone的lowmem_reserve字段）。
 *     2、除了被分配的页框外，这里在order至少为k的块中，起码还有min/2^k个空闲页框。其中对每个k，取值在1和order之间。
 *
 * 作为参数传递的基本值可以是内存管理区界值pages_min,pages_low,pages_high中的任意一个。
 */
int zone_watermark_ok(struct zone *z, int order, unsigned long mark,
		      int classzone_idx, int can_try_harder, int gfp_high)
{
	/* free_pages my go negative - that's OK */
	long min = mark, free_pages = z->free_pages - (1 << order) + 1; /*free_pages是除了要分配的页框(1<<order)后剩余的空闲页面*/
	int o;

	/**
	 * 如果gfp_high标志被置位。则base除2。
	 * 注意这里不是：min /= 2;
	 * 一般来说，如果gfp_mask的__GFP_HIGH标志被置位，那么这个标志就会为1
	 * 换句话说，就是指从高端内存中分配。
	 */
	if (gfp_high)
		min -= min / 2;
	/**
	 * 如果作为参数传递的can_try_harder标志被置位，这个值再减少1/4
	 * can_try_harder=1一般是当：gfp_mask中的__GFP_WAIT标志被置位，或者当前进程是一个实时进程并且在进程上下文中已经完成了内存分配。
	 */
	if (can_try_harder)
		min -= min / 4;

    /*
     * 除了被分配的页框外，在内存管理区中至少还有min个空闲页框，不包括为内存不足保留的页框（zone的lowmem_reserve字段）。
     */
	if (free_pages <= min + z->lowmem_reserve[classzone_idx])
		return 0;

    /*
     * 除了被分配的页框外，这里在order至少为k的块中，起码还有min/2^k个空闲页框。其中对每个k，取值在1和order之间。
     */
	for (o = 0; o < order; o++) {
		/* At the next order, this order's pages become unavailable */
		free_pages -= z->free_area[o].nr_free << o;

		/* Require fewer higher order pages to be free */
		min >>= 1;

		if (free_pages <= min)
			return 0;
	}
	return 1;
}
```

可以看到整个zone的扫描策略非常复杂，但无论哪种情形，一旦满足了分配水位需求，就会调用

`buffered_rmqueue`函数：

```c
/**
 * 返回第一个被分配的页框的页描述符。如果内存管理区没有所请求大小的一组连续页框，则返回NULL。
 * 在指定的内存管理区中分配页框。它使用每CPU页框高速缓存来处理单一页框请求。
 * zone:内存管理区描述符的地址。
 * order：请求分配的内存大小的对数,0表示分配一个页框。
 * gfp_flags:分配标志，如果gfp_flags中的__GFP_COLD标志被置位，那么页框应当从冷高速缓存中获取，否则应当从热高速缓存中获取（只对单一页框请求有意义。）
 */
static struct page *
buffered_rmqueue(struct zone *zone, int order, int gfp_flags)
{
	unsigned long flags;
	struct page *page = NULL;
	int cold = !!(gfp_flags & __GFP_COLD);

	/**
	 * 如果order!=0，则每CPU页框高速缓存就不能被使用。
	 * 因为高速缓存仅限于单页分配，这是固有的设计，就是为了加速单页请求分配的效率。
	 */
	if (order == 0) {
		struct per_cpu_pages *pcp;

		/**
		 * 检查由__GFP_COLD标志所标识的内存管理区本地CPU高速缓存是否需要被补充。
		 * 其count字段小于或者等于low
		 */
		pcp = &zone->pageset[get_cpu()].pcp[cold];
		local_irq_save(flags);
		/**
		 * 当前缓存中的页框数低于low，需要从伙伴系统中补充页框。
		 * 调用rmqueue_bulk函数从伙伴系统中分配batch个单一页框
		 * rmqueue_bulk反复调用__rmqueue，直到缓存的页框达到low。
		 */
		if (pcp->count <= pcp->low)
			pcp->count += rmqueue_bulk(zone, 0,
						pcp->batch, &pcp->list);
		/**
		 * 如果count为正，函数从高速缓存链表中获得一个页框。
		 * count减1
		 */
		if (pcp->count) {
			page = list_entry(pcp->list.next, struct page, lru);
			list_del(&page->lru);
			pcp->count--;
		}
		local_irq_restore(flags);
		/**
		 * 没有和get_cpu配对使用呢？
		 * 这就是内核，外层一定调用了get_cpu。这种代码看起来头疼。
		 */
		put_cpu();
	}

	/**
	 * 内存请求没有得到满足，或者是因为请求跨越了几个连续页框，或者是因为被选中的页框高速缓存为空。
	 * 调用__rmqueue函数(因为已经保护了，直接调用__rmqueue即可)从伙伴系统中分配所请求的页框。
	 */
	if (page == NULL) {
		spin_lock_irqsave(&zone->lock, flags);
		page = __rmqueue(zone, order);
		spin_unlock_irqrestore(&zone->lock, flags);
	}

	/**
	 * 如果内存请求得到满足，函数就初始化（第一个）页框的页描述符
	 */
	if (page != NULL) {
		BUG_ON(bad_range(zone, page));
		/**
		 * 将第一个页清除一些标志，将private字段置0，并将页框引用计数器置1。
		 */
		mod_page_state_zone(zone, pgalloc, 1 << order);
		prep_new_page(page, order);

		/**
		 * 如果__GFP_ZERO标志被置位，则将被分配的区域填充0。
		 */
		if (gfp_flags & __GFP_ZERO)
			prep_zero_page(page, order, gfp_flags);

		if (order && (gfp_flags & __GFP_COMP))
			prep_compound_page(page, order);
	}
	return page;
}
```

除了每CPU页框高速缓存的分配以外，重点就是`__rmqueue`这个函数，该函数实际上就是通往伙伴系统的分配接口。另一方面，每CPU页框高速缓存在分配时也会检查当前预存量的水位，如果不足也会调用相应的到伙伴系统的接口去批量申请页框。

关于伙伴系统的`__rmqueue`，我们过后再深入分析。

####Appendix

`gfp_mask`flags:

```cpp
/*
 * GFP bitmasks..
 */
/* Zone modifiers in GFP_ZONEMASK (see linux/mmzone.h - low two bits) */
/**
 * 所请求的页框必须处于ZONE_DMA管理区。等价于GFP_DMA
 */
#define __GFP_DMA	0x01
/**
 * 所请求的页框处于ZONE_HIGHMEM管理区
 */
#define __GFP_HIGHMEM	0x02

/*
 * Action modifiers - doesn't change the zoning
 *
 * __GFP_REPEAT: Try hard to allocate the memory, but the allocation attempt
 * _might_ fail.  This depends upon the particular VM implementation.
 *
 * __GFP_NOFAIL: The VM implementation _must_ retry infinitely: the caller
 * cannot handle allocation failures.
 *
 * __GFP_NORETRY: The VM implementation must not retry indefinitely.
 */
/**
 * 允许内核对等待空闲页框的当前进程进行阻塞
 */
#define __GFP_WAIT	0x10	/* Can wait and reschedule? */
/**
 * 允许内核访问保留的页框池
 */
#define __GFP_HIGH	0x20	/* Should access emergency pools? */
/**
 * 允许内核在低端内存上执行IO传输以释放页框。
 */
#define __GFP_IO	0x40	/* Can start physical IO? */
/**
 * 如果清0,则不允许内核执行依赖于文件系统的操作。
 */
#define __GFP_FS	0x80	/* Can call down to low-level FS? */
/**
 * 所请求的页可能为"冷"的。即不在高速缓存中。
 */
#define __GFP_COLD	0x100	/* Cache-cold page required */
/**
 * 一次内存分配失败将不会产生警告信息
 */
#define __GFP_NOWARN	0x200	/* Suppress page allocation failure warning */
/**
 * 内核重试内存分配直到成功。
 */
#define __GFP_REPEAT	0x400	/* Retry the allocation.  Might fail */
/**
 * 与__GFP_REPEAT相同
 */
#define __GFP_NOFAIL	0x800	/* Retry for ever.  Cannot fail */
/**
 * 一次内存分配失败后不再重试。
 */
#define __GFP_NORETRY	0x1000	/* Do not retry.  Might fail */
/**
 * Slab分配器不允许增大slab高速缓存。
 */
#define __GFP_NO_GROW	0x2000	/* Slab internal usage */
/**
 * 属于扩展页的页框。
 */
#define __GFP_COMP	0x4000	/* Add compound page metadata */
/**
 * 任何返回的页框必须被填满0
 */
#define __GFP_ZERO	0x8000	/* Return zeroed page on success */
```

###释放接口

再看释放：

```cpp
/**
 * 首先检查page指向的页描述符。
 * 如果该页框未被保留，就把描述符的count字段减1
 * 如果count变为0,就假定从与page对应的页框开始的2^order个连续页框不再被使用。
 * 这种情况下，该函数释放页框。
 */
fastcall void __free_pages(struct page *page, unsigned int order)
{
  	/*
  	 * 如果是保留的页框或者引用计数-1不为0，则不用释放页框。
  	 * 前者是因为保留页框本来就不用释放（通过flags的PG_reserved位标识），
  	 * 后者是因为当前尚有进程引用该页框（比如经典的父子进程）
  	 */
	if (!PageReserved(page) && put_page_testzero(page)) {
		if (order == 0)
			free_hot_page(page);
		else
			__free_pages_ok(page, order);
	}
}

/**
 * 类似于__free_pages，但是它接收的参数为要释放的第一个页框的线性地址。
 */
fastcall void free_pages(unsigned long addr, unsigned int order)
{
	if (addr != 0) {
		BUG_ON(!virt_addr_valid((void *)addr));
		__free_pages(virt_to_page((void *)addr), order);
	}
}

/* 和alloc相反，此时用virt_to_page来从线性地址到page进行转换，本质上调用依然是__free_pages */

/* 衍生出的另两个order为0的特例 */

/**
 * 释放page指向的页框
 */
#define __free_page(page) __free_pages((page), 0)

/**
 * 释放线性地址addr对应的页框。
 */
#define free_page(addr) free_pages((addr),0)
```

所以还是看__free_pages，对于order为0的情况，最终cold参数为0进入free_hot_cold_page中：
```cpp
void fastcall free_hot_page(struct page *page)
{
  	// cold为0，表示是热高速缓存
	free_hot_cold_page(page, 0);
}

/**
 * 释放单个页框到页框高速缓存。
 * page-要释放的页框描述符地址。
 * cold-释放到热高速缓存还是冷高速缓存中
 */
static void fastcall free_hot_cold_page(struct page *page, int cold)
{
	/**
	 * page_zone从page->flag中，获得page所在的内存管理区描述符。
	 */
	struct zone *zone = page_zone(page);
	struct per_cpu_pages *pcp;
	unsigned long flags;

	arch_free_page(page, 0);

	kernel_map_pages(page, 1, 0);
	inc_page_state(pgfree);
	if (PageAnon(page))
		page->mapping = NULL;
	free_pages_check(__FUNCTION__, page);
	/**
	 * 冷高速缓存还是热高速缓存??
	 */
	pcp = &zone->pageset[get_cpu()].pcp[cold];
	local_irq_save(flags);
	/**
	 * 如果缓存的页框太多，就清除一些。
	 * 调用free_pages_bulk将这些页框释放给伙伴系统。
	 * 当然，需要更新一下count计数。
	 */
	if (pcp->count >= pcp->high)
		pcp->count -= free_pages_bulk(zone, pcp->batch, &pcp->list, 0);
	/**
	 * 将释放的页框加到高速缓存链表上。并增加count字段。
	 */
	list_add(&page->lru, &pcp->list);
	pcp->count++;
	local_irq_restore(flags);
	put_cpu();
}
```
对order为0也就是单页释放的情况，和分配时策略相同，都是最终释放到每CPU页框高速缓存。此外，释放时也会做相反的判断，如果当前预留量过多了，就还给伙伴系统一些。而释放给伙伴系统的核心在于`free_pages_bulk`，这涉及到伙伴系统的策略，过后再研究。

> 到这里，透过代码也就不难看出，每CPU页框高速缓存的管理类似zone，也有自己的阈值，判断水位高低的策略，这与zone分配器的扫描策略十分相似。
>
> 更多详细可以展开每CPU页框高速缓存的结构体与内部函数，这里限于篇幅不展开了。

对于order不为0的，核心也是`free_page_bulk`，说白了还是释放给伙伴系统：

```cpp
void __free_pages_ok(struct page *page, unsigned int order)
{
	LIST_HEAD(list);
	int i;

	arch_free_page(page, order);

	mod_page_state(pgfree, 1 << order);

#ifndef CONFIG_MMU
	if (order > 0)
		for (i = 1 ; i < (1 << order) ; ++i)
			__put_page(page + i);
#endif

	for (i = 0 ; i < (1 << order) ; ++i)
		free_pages_check(__FUNCTION__, page + i);
	list_add(&page->lru, &list);
	kernel_map_pages(page, 1<<order, 0);
	free_pages_bulk(page_zone(page), 1, &list, order);
}
```

#### Appendix 

`virt_to_page`:

```cpp
/**
 * 将内核逻辑地址转换为相应的page结构指针。
 */
#define virt_to_page(kaddr)	pfn_to_page(__pa(kaddr) >> PAGE_SHIFT)

#define __pa(x)			((unsigned long)(x)-PAGE_OFFSET)		//PAGE_OFFSET = 0xC0000000

#define pfn_to_page(pfn)	(mem_map + (pfn))	//由页框号+mem_map得到描述符page， mem_map就是存放page的数组，根据number索引
```

## ZONE_HIGHMEM内核映射
high_memory变量存放HIGHMEM起始地址，设置为896M。896MB边界以上的page frame无法直接映射在内核线性地址空间，因此内核无法直接访问。因此，`__get_free_pages`等函数对此情形并不适用（32位的系统中，因为`__get_free_pages()`无法返回一个不存在的线性地址（会溢出0xFFFFFFFF）转而返回NULL，会造成page frame的丢失）。

为了32位系统内核可以使用HIGHMEM，做了一种映射的机制。约定如下：
1. 只用`alloc_pages`,`alloc_page`分配HIGHMEM的内存页框，虽然线性地址不存在，但是页描述符page的线性地址存在，这个东西在一开始就分配到NON-HIGHMEM区了。
2. 内核线性地址空间的最后128MB的一部分用于映射HIGHMEM的页框。线性地址对页框是一对多的，映射是暂时性的（如果都是永久的，那肯定不够用）。

当内核想访问高于896M的物理地址内存时，会从0xF8000000~0xFFFFFFFF空间范围内找一段相应大小空闲的地址空间，暂借一会儿，用毕归还。这就相当于坑位就那么多，大家轮着用。

内核有3种机制映射高端内存的页框：永久内核映射、临时内核映射和非连续内存分配。目前只关心前两种。

> 实际上896M只是个上限值，不一定真的会分配这么多给NORMAL，毕竟有时候物理内存没有这么大，这个值会根据真实物理内存大小动态计算出来。

内核1GB线性地址空间划分如图：

![](20171020_4.jpg)

- PAGE_OFFSET ~ high_memory: 16~896MB直接映射
- VMALLOC_START ~ VMALLOC_END: 非连续内存分配
- KMAP_BASE ~ FIXADDR_START: 永久内核映射
- FIXADDR_START ~ 4GB:固定映射线性地址空间（FIX_KMAP区域为临时内核映射）

### 永久内核映射
在2.6内核上, 这个地址范围是4G-8M到4G-4M之间. 这个空间起叫”内核永久映射空间”或者”永久内核映射空间”, 这个空间和其它空间使用同样的页目录表，对于内核来说，就是 `swapper_pg_dir`，对普通进程来说，通过 CR3 寄存器指向。通常情况下，这个空间是 4M 大小，因此仅仅需要一个页表即可，内核通过来 `pkmap_page_table` 寻找这个页表。通过 `kmap()`，可以把一个 page 映射到这个空间来。由于这个空间是 4M 大小，最多能同时映射1024 个 page。因此，对于不使用的的 page，及应该时从这个空间释放掉（也就是解除映射关系），通过`kunmap()`，可以把一个 page 对应的线性地址从这个空间释放出来。

```cpp
/**
 * 用于建立永久内核映射的页表。
 * 这样，内核可以长期映射高端内存到内核地址空间中。
 * 页表中的表项数由LAST_PKMAP宏产生，取决于是否打开PAE，它的值可能是512或者1024，
 * 这样可能映射2MB或4MB的永久内核映射。
 */
pte_t * pkmap_page_table;

/**
 * Pkmap_count数组包含LAST_PKMAP个计数器，pkmap_page_table页表中每一项都有一个。
 * 它记录了永久内核映射使用了哪些页表项。
 * 它的值可能为：
 *	0：对应的页表项没有映射任何高端内存页框，并且是可用的。
 *	1：对应页表项没有映射任何高端内存，但是它仍然不可用。因为自从它最后一次使用以来，相应的TLB表还没有被刷新。
 *	>1：相应的页表项映射了一个高端内存页框。并且正好有n-1个内核正在使用这个页框。
 */
static int pkmap_count[LAST_PKMAP];
```

映射起始地址从PKMAP_BASE开始:
```cpp
/**
 * 永久内核映射的线性地址起始处。
 */
#define PKMAP_BASE ( (FIXADDR_BOOT_START - PAGE_SIZE*(LAST_PKMAP + 1)) & PMD_MASK )
```

内核使用`page_address_htable`散列表记录高端内存页框和线性地址的关系：
```cpp
/**
 * 本散列表记录了高端内存页框与永久内核映射映射包含的线性地址。
 */
static struct page_address_slot {
	struct list_head lh;			/* List of page_address_maps */
	spinlock_t lock;			/* Protect this bucket's list */
} ____cacheline_aligned_in_smp page_address_htable[1<<PA_HASH_ORDER];
```
典型的list_head嵌入链表结构。实际上每个节点是`page_address_map`：
```cpp
/*
 * Describes one page->virtual association
 */
struct page_address_map {
	struct page *page;
	void *virtual;
	struct list_head list;
};
```

####Appendix 

`page_address`函数:

```cpp
/**
 * page_address返回页框对应的线性地址。
 */
void *page_address(struct page *page)
{
	unsigned long flags;
	void *ret;
	struct page_address_slot *pas;

	/**
	 * 如果页框不在高端内存中(PG_highmem标志为0)，则线性地址总是存在的。
	 * 并且通过计算页框下标，然后将其转换成物理地址，最后根据物理地址得到线性地址。
	 */
	if (!PageHighMem(page))
		/**
		 * 本句等价于__va((unsigned long)(page - mem_map) << 12)
		 */
		return lowmem_page_address(page);

	/**
	 * 否则页框在高端内存中(PG_highmem标志为1)，则到page_address_htable散列表中查找。
	 */
	pas = page_slot(page);
	ret = NULL;
	spin_lock_irqsave(&pas->lock, flags);
	if (!list_empty(&pas->lh)) {
		struct page_address_map *pam;

		list_for_each_entry(pam, &pas->lh, list) {
			/**
			 * 在page_address_htable中找到，返回对应的物理地址。
			 */
			if (pam->page == page) {
				ret = pam->virtual;
				goto done;
			}
		}
	}
	/**
	 * 没有在page_address_htable中找到，返回默认值NULL。
	 */
done:
	spin_unlock_irqrestore(&pas->lock, flags);
	return ret;
}
```

`kmap/kunmap`: 一目了然的设计
```cpp
/**
 * 建立永久内核映射。
 */
void *kmap(struct page *page)
{
	/**
	 * kmap是允许睡眠的，意思是说不能在中断和可延迟函数中调用。
	 * 如果试图在中断中调用，那么might_sleep会触发异常。
	 */
	might_sleep();
	/**
	 * 如果页框不属于高端内存，则调用page_address直接返回线性地址。
	 */
	if (!PageHighMem(page))
		return page_address(page);
	/**
	 * 否则调用kmap_high真正建立永久内核映射。
	 */
	return kmap_high(page);
}

/**
 * 为高端内存建立永久内核映射。
 */
void fastcall *kmap_high(struct page *page)
{
	unsigned long vaddr;

	/*
	 * For highmem pages, we can't trust "virtual" until
	 * after we have the lock.
	 *
	 * We cannot call this from interrupts, as it may block
	 */
	/**
	 * 这个函数不会在中断中调用，也不能在中断中调用。
	 * 所以，在这里只需要获取自旋锁就行了。
	 */
	spin_lock(&kmap_lock);
	/**
	 * page_address有检查页框是否被映射的作用。
	 */
	vaddr = (unsigned long)page_address(page);
	/**
	 * 没有被映射，就调用map_new_virtual把页框的物理地址插入到pkmap_page_table的一个项中。
	 * 并在page_address_htable散列表中加入一个元素。
	 */
	if (!vaddr)
		vaddr = map_new_virtual(page);
	/**
	 * 使页框的线性地址所对应的计数器加1.
	 */
	pkmap_count[PKMAP_NR(vaddr)]++;
	/**
	 * 初次映射时,map_new_virtual中会将计数置为1,上一句再加1.
	 * 多次映射时,计数值会再加1.
	 * 总之,计数值决不会小于2.
	 */
	if (pkmap_count[PKMAP_NR(vaddr)] < 2)
		BUG();
	/**
	 * 释放自旋锁.
	 */
	spin_unlock(&kmap_lock);
	return (void*) vaddr;
}

/**
 * 为建立永久内核映射建立初始映射.
 */
static inline unsigned long map_new_virtual(struct page *page)
{
	unsigned long vaddr;
	int count;

start:
	count = LAST_PKMAP;
	/* Find an empty entry */
	/**
	 * 扫描pkmap_count中的所有计数器值,直到找到一个空值.
	 */
	for (;;) {
		/**
		 * 从上次结束的地方开始搜索.
		 */
		last_pkmap_nr = (last_pkmap_nr + 1) & LAST_PKMAP_MASK;
		/**
		 * 搜索到最后一位了.在从0开始搜索前,刷新计数为1的项.
		 * 当计数值为1表示页表项可用,但是对应的TLB还没有刷新.
		 */
		if (!last_pkmap_nr) {
			flush_all_zero_pkmaps();
			count = LAST_PKMAP;
		}
		/**
		 * 找到计数为0的页表项,表示该页空闲且可用.
		 */
		if (!pkmap_count[last_pkmap_nr])
			break;	/* Found a usable entry */
		/**
		 * count是允许的搜索次数.如果还允许继续搜索下一个页表项.则继续,否则表示没有空闲项,退出.
		 */
		if (--count)
			continue;

		/*
		 * Sleep for somebody else to unmap their entries
		 */
		/**
		 * 运行到这里,表示没有找到空闲页表项.先睡眠一下.
		 * 等待其他线程释放页表项,然后唤醒本线程.
		 */
		{
			DECLARE_WAITQUEUE(wait, current);

			__set_current_state(TASK_UNINTERRUPTIBLE);
			/**
			 * 将当前线程挂到pkmap_map_wait等待队列上.
			 */
			add_wait_queue(&pkmap_map_wait, &wait);
			spin_unlock(&kmap_lock);
			schedule();
			remove_wait_queue(&pkmap_map_wait, &wait);
			spin_lock(&kmap_lock);

			/* Somebody else might have mapped it while we slept */
			/**
			 * 在当前线程等待的过程中,其他线程可能已经将页面进行了映射.
			 * 检测一下,如果已经映射了,就退出.
			 * 注意,这里没有对kmap_lock进行解锁操作.关于kmap_lock锁的操作,需要结合kmap_high来分析.
			 * 总的原则是:进入本函数时保证关锁,然后在本句前面关锁,本句后面解锁.
			 * 在函数返回后,锁仍然是关的.则外层解锁.
			 * 即使在本函数中循环也是这样.
			 * 内核就是这么乱,看久了就习惯了.不过你目前可能必须得学着适应这种代码.
			 */
			if (page_address(page))
				return (unsigned long)page_address(page);

			/* Re-start */
			goto start;
		}
	}
	/**
	 * 不管何种路径运行到这里来,kmap_lock都是锁着的.
	 * 并且last_pkmap_nr对应的是一个空闲且可用的表项.
	 */
	vaddr = PKMAP_ADDR(last_pkmap_nr);
	/**
	 * 设置页表属性,建立虚拟地址和物理地址之间的映射.
	 */
	set_pte(&(pkmap_page_table[last_pkmap_nr]), mk_pte(page, kmap_prot));

	/**
	 * 1表示相应的项可用,但是TLB需要刷新.
	 * 但是我们这里明明建立了映射,为什么还是可用的呢,其他地方不会将占用么?
	 * 其实不用担心,因为返回kmap_high后,kmap_high函数会将它再加1.
	 */
	pkmap_count[last_pkmap_nr] = 1;
	set_page_address(page, (void *)vaddr);

	return vaddr;
}


/**
 * 撤销先前由kmap建立的永久内核映射
 */
void kunmap(struct page *page)
{
	/**
	 * kmap和kunmap都不允许在中断中使用。
	 */
	if (in_interrupt())
		BUG();
	/**
	 * 如果对应页根本就不是高端内存，当然就没有进行内核映射，也就不用调用本函数了。
	 */
	if (!PageHighMem(page))
		return;
	/**
	 * kunmap_high真正执行unmap过程
	 */
	kunmap_high(page);
}

/**
 * 解除高端内存的永久内核映射
 */
void fastcall kunmap_high(struct page *page)
{
	unsigned long vaddr;
	unsigned long nr;
	int need_wakeup;

	spin_lock(&kmap_lock);
	/**
	 * 得到物理页对应的虚拟地址。
	 */
	vaddr = (unsigned long)page_address(page);
	/**
	 * vaddr会==0，可能是内存越界等严重故障了吧。
	 * BUG一下
	 */
	if (!vaddr)
		BUG();
	/**
	 * 根据虚拟地址，找到页表项在pkmap_count中的序号。
	 */
	nr = PKMAP_NR(vaddr);

	/*
	 * A count must never go down to zero
	 * without a TLB flush!
	 */
	need_wakeup = 0;
	switch (--pkmap_count[nr]) {
	case 0:
		BUG();/* 一定是逻辑错误了，多次调用了unmap */
	case 1:
		/*
		 * Avoid an unnecessary wake_up() function call.
		 * The common case is pkmap_count[] == 1, but
		 * no waiters.
		 * The tasks queued in the wait-queue are guarded
		 * by both the lock in the wait-queue-head and by
		 * the kmap_lock.  As the kmap_lock is held here,
		 * no need for the wait-queue-head's lock.  Simply
		 * test if the queue is empty.
		 */
		/**
		 * 页表项可用了。need_wakeup会唤醒等待队列上阻塞的线程。
		 */
		need_wakeup = waitqueue_active(&pkmap_map_wait);
	}
	spin_unlock(&kmap_lock);

	/* do wake-up, if needed, race-free outside of the spin lock */
	/**
	 * 有等待线程，唤醒它。
	 */
	if (need_wakeup)
		wake_up(&pkmap_map_wait);
}
```

### 临时内核映射
内核在 FIXADDR_START 到 FIXADDR_TOP 之间保留了一些线性空间用于特殊需求。这个空间称为”固定映射空间”在这个空间中，有一部分用于高端内存的临时映射。临时内核映射可以用在中断处理程序和可延迟函数内部，它们从不阻塞当前进程。

这块空间具有如下特点：

1. 每个 CPU 占用一块空间
2. 在每个 CPU 占用的那块空间中，又分为多个小空间，每个小空间大小是 1 个 page，每个小空间用于一个目的，这些目的定义在 kmap_types.h 中的 `km_type` 中。

任一页框都可以通过一个窗口映射到内核地址空间，这些窗口非常少。

每个CPU有自己的13个窗口集合：
```cpp
#ifdef CONFIG_DEBUG_HIGHMEM
# define D(n) __KM_FENCE_##n ,
#else
# define D(n)
#endif

/**
 * 系统为每个CPU预留了13个临时内核映射页表项。这是它们在线性地址表中的下标。
 */
enum km_type {
D(0)	KM_BOUNCE_READ,
D(1)	KM_SKB_SUNRPC_DATA,
D(2)	KM_SKB_DATA_SOFTIRQ,
D(3)	KM_USER0,
D(4)	KM_USER1,
D(5)	KM_BIO_SRC_IRQ,
D(6)	KM_BIO_DST_IRQ,
D(7)	KM_PTE0,
D(8)	KM_PTE1,
D(9)	KM_IRQ0,
D(10)	KM_IRQ1,
D(11)	KM_SOFTIRQ0,
D(12)	KM_SOFTIRQ1,
D(13)	KM_TYPE_NR
};
```
同一窗口永不会被两个不同的控制路径同时使用。km_type的每个符号都只能由一种内核成分使用，并以此而命名。KM_TYPE_NR不是线性地址，而是CPU产生可用窗口的个数。

这些符号都只是线性地址的一个下标:
```cpp
enum fixed_addresses {
	FIX_HOLE,
	FIX_VSYSCALL,
#ifdef CONFIG_X86_LOCAL_APIC
	FIX_APIC_BASE,	/* local (CPU) APIC) -- required for SMP or not */
#endif
#ifdef CONFIG_X86_IO_APIC
	FIX_IO_APIC_BASE_0,
	FIX_IO_APIC_BASE_END = FIX_IO_APIC_BASE_0 + MAX_IO_APICS-1,
#endif
#ifdef CONFIG_X86_VISWS_APIC
	FIX_CO_CPU,	/* Cobalt timer */
	FIX_CO_APIC,	/* Cobalt APIC Redirection Table */ 
	FIX_LI_PCIA,	/* Lithium PCI Bridge A */
	FIX_LI_PCIB,	/* Lithium PCI Bridge B */
#endif
#ifdef CONFIG_X86_F00F_BUG
	FIX_F00F_IDT,	/* Virtual mapping for IDT */
#endif
#ifdef CONFIG_X86_CYCLONE_TIMER
	FIX_CYCLONE_TIMER, /*cyclone timer register*/
#endif 
#ifdef CONFIG_HIGHMEM
	FIX_KMAP_BEGIN,	/* reserved pte's for temporary kernel mappings */
	FIX_KMAP_END = FIX_KMAP_BEGIN+(KM_TYPE_NR*NR_CPUS)-1,
#endif
#ifdef CONFIG_ACPI_BOOT
	FIX_ACPI_BEGIN,
	FIX_ACPI_END = FIX_ACPI_BEGIN + FIX_ACPI_PAGES - 1,
#endif
#ifdef CONFIG_PCI_MMCONFIG
	FIX_PCIE_MCFG,
#endif
	__end_of_permanent_fixed_addresses,
	/* temporary boot-time mappings, used before ioremap() is functional */
#define NR_FIX_BTMAPS	16
	FIX_BTMAP_END = __end_of_permanent_fixed_addresses,
	FIX_BTMAP_BEGIN = FIX_BTMAP_END + NR_FIX_BTMAPS - 1,
	FIX_WP_TEST,
	__end_of_fixed_addresses
};

```
所以每个CPU都有KM_TYPE_NR个固定的映射线性地址。

当要进行一次临时映射的时候，需要指定映射的目的，根据映射目的，可以找到对应的小空间，然后把这个空间的地址作为映射地址。这意味着一次临时映射会导致以前的映射被覆盖。通过 `kmap_atomic()`可实现临时映射:

```cpp
/**
 * 建立临时内核映射
 * type和CPU共同确定用哪个固定映射的线性地址映射请求页。
 */
void *kmap_atomic(struct page *page, enum km_type type)
{
	enum fixed_addresses idx;
	unsigned long vaddr;

	/* even !CONFIG_PREEMPT needs this, for in_atomic in do_page_fault */
	inc_preempt_count();
	/**
	 * 如果被映射的页不属于高端内存，当然用不着映射。直接返回线性地址就行了。
	 */
	if (!PageHighMem(page))
		return page_address(page);

	/**
	 * 通过type和CPU确定线性地址。
	 */
	idx = type + KM_TYPE_NR*smp_processor_id();
	vaddr = __fix_to_virt(FIX_KMAP_BEGIN + idx);
#ifdef CONFIG_DEBUG_HIGHMEM
	if (!pte_none(*(kmap_pte-idx)))
		BUG();
#endif
	/**
	 * 将线性地址与页表项建立映射。
	 */
	set_pte(kmap_pte-idx, mk_pte(page, kmap_prot));
	/**
	 * 当然，最后必须刷新一下TLB。然后才能返回线性地址。
	 */
	__flush_tlb_one(vaddr);

	return (void*) vaddr;
}

#define __fix_to_virt(x)	(FIXADDR_TOP - ((x) << PAGE_SHIFT))

#define FIXADDR_TOP	((unsigned long)__FIXADDR_TOP)

#define __FIXADDR_TOP	0xfffff000
```
撤销`kunmap_atomic`:
```cpp
/**
 * 撤销内核临时映射
 */
void kunmap_atomic(void *kvaddr, enum km_type type)
{
#ifdef CONFIG_DEBUG_HIGHMEM
	unsigned long vaddr = (unsigned long) kvaddr & PAGE_MASK;
	enum fixed_addresses idx = type + KM_TYPE_NR*smp_processor_id();

	if (vaddr < FIXADDR_START) { // FIXME
		dec_preempt_count();
		preempt_check_resched();
		return;
	}

	if (vaddr != __fix_to_virt(FIX_KMAP_BEGIN+idx))
		BUG();

	/*
	 * force other mappings to Oops if they'll try to access
	 * this pte without first remap it
	 */
	/**
	 * 取消映射并刷新TLB
	 */
	pte_clear(kmap_pte-idx);
	__flush_tlb_one(vaddr);
#endif
	/**
	 * 允许抢占，并检查调度点。
	 */
	dec_preempt_count();
	preempt_check_resched();
}
```

高端内存是物理内存的概念，无论何种映射，最终它映射到的是内核线性地址空间，和用户进程没有什么关系，不要混淆。

## Buddy System（伙伴系统）Algorithm

内存管理的经典问题：碎片化
两种解决方案：
- 非连续空闲页框映射到连续的线性地址
- 定制一套体系处理空闲连续页框块，分配与回收

出于种种原因，Linux内核使用第二种方案（详见ULK，实际上一言以蔽之——第二种方案更契合Linux）。这个方案就是buddy system。空闲页框分组为11个双向循环链表，每个链表存储大小为1,2,4,8,16,32,64,128,256,512,1024个连续的页框，形成块。每个块的第一个页框的首地址都是该块大小的整数倍。

涉及的数据结构主要是两个，前面提到的所有page数组mem_map以及free_area。
关于mem_map，前面初始化的代码中可以看到它是整个node的page集合。而对于每个zone的page，都是node的page也就是mem_map的子集。zone的zone_mem_map成员就是在mem_map中的起始成员。
另一方面，zone的free_area数组存储了这11个链(MAX_ORDER=11)。看一下free_area结构：
```cpp
struct free_area {
	struct list_head	free_list;
	unsigned long		nr_free;
};
```
经典的list_head双向循环链表，该链表包含每个空闲页框块(2^k)的起始页框的page。指向链表中相邻元素的指针存放在page的lru字段中（lru在页非空闲时用于其它目的）。nr_free表示空闲块的个数。page中的private存放了块的order（private在非空闲时也用于其它目的）。
```cpp
struct page{
	...
	/**
	 * 可用于正在使用页的内核成分（如在缓冲页的情况下，它是一个缓冲器头指针。）
	 * 如果页是空闲的，则该字段由伙伴系统使用。
	 * 当用于伙伴系统时，如果该页是一个2^k的空闲页块的第一个页，那么它的值就是k.
	 * 这样，伙伴系统可以查找相邻的伙伴，以确定是否可以将空闲块合并成2^(k+1)大小的空闲块。
	 */
	unsigned long private;		/* Mapping-private opaque data:
					 * usually used for buffer_heads
					 * if PagePrivate set; used for
					 * swp_entry_t if PageSwapCache
					 * When page is free, this indicates
					 * order in the buddy system.
					 */
	...
};
```

### alloc
`__rmqueue()`负责在zone中找空闲块，顺着上次看到的`__alloc_pages`看下去，`__alloc_pages()=>buffered_rmqueue()=>__rmqueue()`：
```cpp
/**
 * 在管理区中找到一个空闲块。
 * 它需要两个参数：管理区描述符的地址和order。Order表示请求的空闲页块大小的对数值。
 * 如果页框被成功分配，则返回第一个被分配的页框的页描述符。否则返回NULL。
 * 本函数假设调用者已经禁止和本地中断并获得了自旋锁。
 */
static struct page *__rmqueue(struct zone *zone, unsigned int order)
{
	struct free_area * area;
	unsigned int current_order;
	struct page *page;

	/**
	 * 从所请求的order开始，扫描每个可用块链表进行循环搜索。
	 */
	for (current_order = order; current_order < MAX_ORDER; ++current_order) {
		area = zone->free_area + current_order;
		/**
		 * 对应的空闲块链表为空，在更大的空闲块链表中进行循环搜索。
		 */
		if (list_empty(&area->free_list))
			continue;

		/**
		 * 运行到此，说明有合适的空闲块。
		 */
		page = list_entry(area->free_list.next, struct page, lru);
		/**
		 * 首先在空闲块链表中删除第一个页框描述符。
		 */
		list_del(&page->lru);
		rmv_page_order(page);
		area->nr_free--;
		/**
		 * 并减少空闲管理区的空闲页数量。
		 */
		zone->free_pages -= 1UL << order;
		/**
		 * 如果2^order空闲块链表中没有合适的空闲块，那么就是从更大的空闲链表中分配的。
		 * 将剩余的空闲块分散到合适的链表中去。
		 */
		return expand(zone, page, order, current_order, area);
	}

	/**
	 * 直到循环结束都没有找到合适的空闲块，就返回NULL。
	 */
	return NULL;
}
```
由此也可以看到buddy system的策略算法。优先在等尺寸链上找，如果找不到则采用类似glibc的“small first，best fit”策略，在更大的链上找，并进行切割。切割手法也很简单,简单来说，如果切割1024给256，那么切割后前512链到k=9的链，剩下的256链到k=8的链（都是基于page frame对齐的，所以切割比glibc容易管理得多）：
```cpp
static inline struct page *
expand(struct zone *zone, struct page *page,
 	int low, int high, struct free_area *area)
{
	unsigned long size = 1 << high;

	while (high > low) {
		area--;
		high--;
		size >>= 1;
		BUG_ON(bad_range(zone, &page[size]));
        /*
         * 后半部分(page[size])加入free_area->free_list中，并设定order
         * 前半部分(page)，继续进行分裂或者返回
         */
		list_add(&page[size].lru, &area->free_list);
		area->nr_free++;
		set_page_order(&page[size], high);
	}
	return page;
}
```

### free
释放使用`__free_pages_bulk()`：
```cpp
/**
 * 按照伙伴系统的策略释放页框。
 * page-被释放块中所包含的第一个页框描述符的地址。
 * zone-管理区描述符的地址。
 * order-块大小的对数。
 * base-纯粹由于效率的原因而引入。其实可以从其他三个参数计算得出。
 * 该函数假定调用者已经禁止本地中断并获得了自旋锁。
 */
static inline void __free_pages_bulk (struct page *page, struct page *base,
		struct zone *zone, unsigned int order)
{
	/**
	 * page_idx包含块中第一个页框的下标。
	 * 这是相对于管理区中的第一个页框而言的。
	 */
	unsigned long page_idx;
	struct page *coalesced;
	/**
	 * order_size用于增加管理区中空闲页框的计数器。
	 */
	int order_size = 1 << order;

	if (unlikely(order))
		destroy_compound_page(page, order);

	page_idx = page - base;

	/**
	 * 大小为2^k的块，它的线性地址都是2^k * 2 ^ 12的整数倍。
	 * 相应的，它在管理区的偏移应该是2^k倍。
	 */
	BUG_ON(page_idx & (order_size - 1));
	BUG_ON(bad_range(zone, page));

	/**
	 * 增加管理区的空闲页数
	 */
	zone->free_pages += order_size;
	/**
	 * 最多循环10 - order次。每次都将一个块和它的伙伴进行合并。
	 * 每次从最小的块开始，向上合并。
	 */
	while (order < MAX_ORDER-1) {
		struct free_area *area;
		struct page *buddy;
		int buddy_idx;

		/**
		 * 最小块的下标。它是要合并的块的伙伴。
		 * 注意异或操作的用法，是如何用来寻找伙伴的。
		 * 相当于在page_idx加上或者减去1 << order的距离就是buddy_idx
		 */
		buddy_idx = (page_idx ^ (1 << order));
		/**
		 * 通过伙伴的下标找到页描述符的地址。
		 */
		buddy = base + buddy_idx;
		if (bad_range(zone, buddy))
			break;
		/**
		 * 判断伙伴块是否是大小为order的空闲页框的第一个页。
		 * 首先，伙伴的第一个页必须是空闲的(_count == -1)
		 * 同时，必须属于动态内存(PG_reserved被清0,PG_reserved为1表示留给内核或者没有使用)
		 * 最后，其private字段必须是order
		 */
		if (!page_is_buddy(buddy, order))
			break;
		/**
		 * 运行到这里，说明伙伴块可以与当前块合并。
		 */
		/* Move the buddy up one level. */
		/**
		 * 伙伴将被合并，将它从现有链表中取下。
		 */
		list_del(&buddy->lru);
		area = zone->free_area + order;
		area->nr_free--;
		rmv_page_order(buddy);
        /*
         * 计算parent_idx
         */
		page_idx &= buddy_idx;
		/**
		 * 将合并了的块再与它的伙伴进行合并。
		 */
		order++;
	}

	/**
	 * 伙伴不能与当前块合并。
	 * 将块插入适当的链表，并以块大小的order更新第一个页框的private字段。
	 */
	coalesced = base + page_idx;
	set_page_order(coalesced, order);
	list_add(&coalesced->lru, &zone->free_area[order].free_list);
	zone->free_area[order].nr_free++;
}
```
理解了buddy system的策略，代码也就一目了然了。
调用链：`__free_pages()=>__free_pages_ok()=>free_pages_bulk()=>__free_pages_bulk()`。

## CPU页框高速缓存
内核经常请求或释放单个页框（对应order=0的情况），为了提升性能，内存管理区定义了一个每CPU页框高速缓存用于包含一些预先分配的页框。我们先前在Zoned Page Frame分配器中的分配和释放操作代码中已经看过了。
Linux为每个内存管理区和每CPU提供了两个高速缓存：热高速和冷高速。两种类型针对不同的请求而设计。

zone的`pageset`成员保存了两个高速缓存：
```cpp
/**
 * 管理区每CPU页框高速缓存描述符
 */
struct per_cpu_pageset {
	/**
	 * 热高速缓存和冷高速缓存。
	 */
	struct per_cpu_pages pcp[2];	/* 0: hot.  1: cold */
} ____cacheline_aligned_in_smp;

/**
 * 内存管理区页框高速缓存描述符
 */
struct per_cpu_pages {
	/**
	 * 高速缓存中的页框个数
	 */
	int count;		/* number of pages in the list */
	/**
	 * 下界，低于此就需要补充高速缓存。
	 */
	int low;		/* low watermark, refill needed */
	/**
	 * 上界，高于此则向伙伴系统释放页框。
	 */
	int high;		/* high watermark, emptying needed */
	/**
	 * 当需要增加或者减少高速缓存页框时，操作的页框个数。
	 */
	int batch;		/* chunk size for buddy add/remove */
	/**
	 * 高速缓存中包含的页框描述符链表。
	 */
	struct list_head list;	/* the list of pages */
};
```

### alloc
回顾一下处理`order==0`时使用CPU页框高速缓存分配的场景：
```cpp
/**
 * 返回第一个被分配的页框的页描述符。如果内存管理区没有所请求大小的一组连续页框，则返回NULL。
 * 在指定的内存管理区中分配页框。它使用每CPU页框高速缓存来处理单一页框请求。
 * zone:内存管理区描述符的地址。
 * order：请求分配的内存大小的对数,0表示分配一个页框。
 * gfp_flags:分配标志，如果gfp_flags中的__GFP_COLD标志被置位，那么页框应当从冷高速缓存中获取，否则应当从热高速缓存中获取（只对单一页框请求有意义。）
 */
static struct page *
buffered_rmqueue(struct zone *zone, int order, int gfp_flags)
{
	unsigned long flags;
	struct page *page = NULL;
	int cold = !!(gfp_flags & __GFP_COLD);

	/**
	 * 如果order!=0，则每CPU页框高速缓存就不能被使用。
	 */
	if (order == 0) {
		struct per_cpu_pages *pcp;

		/**
		 * 检查由__GFP_COLD标志所标识的内存管理区本地CPU高速缓存是否需要被补充。
		 * 其count字段小于或者等于low
		 */
		pcp = &zone->pageset[get_cpu()].pcp[cold];
		local_irq_save(flags);
		/**
		 * 当前缓存中的页框数低于low，需要从伙伴系统中补充页框。
		 * 调用rmqueue_bulk函数从伙伴系统中分配batch个单一页框
		 * rmqueue_bulk反复调用__rmqueue，直到缓存的页框达到low。
		 */
		if (pcp->count <= pcp->low)
			pcp->count += rmqueue_bulk(zone, 0,
						pcp->batch, &pcp->list);
		/**
		 * 如果count为正，函数从高速缓存链表中获得一个页框。
		 * count减1
		 */
		/* 这就是分配的核心了，直接操作就行了，哪那么多__rmqueue，所以快 */
		if (pcp->count) {
			page = list_entry(pcp->list.next, struct page, lru);
			list_del(&page->lru);
			pcp->count--;
		}
		local_irq_restore(flags);
		/**
		 * 没有和get_cpu配对使用呢？
		 * 这就是内核，外层一定调用了get_cpu。这种代码看起来头疼。
		 */
		put_cpu();
	}

	/**
	 * 内存请求没有得到满足，或者是因为请求跨越了几个连续页框，或者是因为被选中的页框高速缓存为空。
	 * 调用__rmqueue函数(因为已经保护了，直接调用__rmqueue即可)从伙伴系统中分配所请求的页框。
	 */
	if (page == NULL) {
		spin_lock_irqsave(&zone->lock, flags);
		page = __rmqueue(zone, order);
		spin_unlock_irqrestore(&zone->lock, flags);
	}

	/**
	 * 如果内存请求得到满足，函数就初始化（第一个）页框的页描述符
	 */
	if (page != NULL) {
		BUG_ON(bad_range(zone, page));
		/**
		 * 将第一个页清除一些标志，将private字段置0，并将页框引用计数器置1。
		 */
		mod_page_state_zone(zone, pgalloc, 1 << order);
		prep_new_page(page, order);

		/**
		 * 如果__GFP_ZERO标志被置位，则将被分配的区域填充0。
		 */
		if (gfp_flags & __GFP_ZERO)
			prep_zero_page(page, order, gfp_flags);

		if (order && (gfp_flags & __GFP_COMP))
			prep_compound_page(page, order);
	}
	return page;
}

//展开看补充页框的操作，实际上也是通过__rmqueue操作的，通过拿到的page->lru链到传入的pcp->list：
static int rmqueue_bulk(struct zone *zone, unsigned int order, 
			unsigned long count, struct list_head *list)
{
	unsigned long flags;
	int i;
	int allocated = 0;
	struct page *page;
	
	spin_lock_irqsave(&zone->lock, flags);
	for (i = 0; i < count; ++i) {
		page = __rmqueue(zone, order);
		if (page == NULL)
			break;
		allocated++;
		list_add_tail(&page->lru, list);
	}
	spin_unlock_irqrestore(&zone->lock, flags);
	return allocated;
}
```

### free

再回顾一下free：

```cpp
/**
 * 首先检查page指向的页描述符。
 * 如果该页框未被保留，就把描述符的count字段减1
 * 如果count变为0,就假定从与page对应的页框开始的2^order个连续页框不再被使用。
 * 这种情况下，该函数释放页框。
 */
fastcall void __free_pages(struct page *page, unsigned int order)
{
	if (!PageReserved(page) && put_page_testzero(page)) {
		if (order == 0)
			free_hot_page(page);
		else
			__free_pages_ok(page, order);
	}
}
```

暂时只关心先对于`order==0`的情况处理(即高速缓存)：
```cpp
void fastcall free_hot_page(struct page *page)
{
	free_hot_cold_page(page, 0);
}
	
void fastcall free_cold_page(struct page *page)
{
	free_hot_cold_page(page, 1);
}
/* 这两个都是封装 */
/* core api */
/**
 * 释放单个页框到页框高速缓存。
 * page-要释放的页框描述符地址。
 * cold-释放到热高速缓存还是冷高速缓存中
 */
static void fastcall free_hot_cold_page(struct page *page, int cold)
{
	/**
	 * page_zone从page->flag中，获得page所在的内存管理区描述符。
	 */
	struct zone *zone = page_zone(page);
	struct per_cpu_pages *pcp;
	unsigned long flags;

	arch_free_page(page, 0);

	kernel_map_pages(page, 1, 0);
	inc_page_state(pgfree);
	if (PageAnon(page))
		page->mapping = NULL;
	free_pages_check(__FUNCTION__, page);
	/**
	 * 冷高速缓存还是热高速缓存??
	 */
	pcp = &zone->pageset[get_cpu()].pcp[cold];
	local_irq_save(flags);
	/**
	 * 如果缓存的页框太多，就清除一些。
	 * 调用free_pages_bulk将这些页框释放给伙伴系统。
	 * 当然，需要更新一下count计数。
	 */
	if (pcp->count >= pcp->high)
		pcp->count -= free_pages_bulk(zone, pcp->batch, &pcp->list, 0);
	/**
	 * 将释放的页框加到高速缓存链表上。并增加count字段。
	 */
	list_add(&page->lru, &pcp->list);
	pcp->count++;
	local_irq_restore(flags);
	put_cpu();
}
```

## 总结
到此，关于页框的管理策略，从zoned alloc/free system到((per CPU cache)+buddy system)，一系列的算法已经了然于胸。物理内存的层层架构、支配与回收也就这么回事儿。

ULK的这张图很透彻：
![](20171020_3.jpg)