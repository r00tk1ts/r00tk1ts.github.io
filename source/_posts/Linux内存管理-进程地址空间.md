---
title: Linux内核学习——内存管理之进程地址空间
date: 2018-08-07 22:05:11
categories: Linux
tags:
	- linux-kernel
	- ulk
---
Linux内存管理非常复杂，从物理内存的分门别类，到内核用户态进程的索取，可以说是包罗万象。这一篇透过源码理解一下进程地址空间的管理。

<!--more-->
# Linux内存管理—进程地址空间

内核获取动态内存的方式有3种：

1. 直接调用buddy system接口`__get_free_pages()`或`alloc_pages()`从zone的页框分配器中获取页框。
2. `kmem_cache_alloc`或`kmalloc`使用slab分配器为专用或通用对象分配块
3. `vmalloc()`或`vmalloc_32()`获取非连续内存区。

上面的机制都是内核自身使用的，内核为用户态进程分配内存时，情况就完全不同了。用户态内存请求被认为是不紧迫的，内核总是尽量推迟给用户态进程分配动态内存 ，同时由于用户进程不可信任，所以内核还需要实时捕获用户态进程引起的寻址错误。

那么我们平时用户态进程请求内存时难道还需要等吗？等到内核推迟后再分配的时点？显然不是的。内核使用一种叫“线性区”的资源分配给用户进程，用户进程请求内存分配时，并没有立即获得页框，而仅仅得到了一个线性区，也就是一片线性地址空间的使用权（线性地址空间是进程地址空间0~3GB的一部分）。而真正的页框分配被推迟到了缺页异常处理程序中。

## 内存描述符

每个进程的进程地址空间相互之间没什么联系，在内核中由不同的线性区表示，每个进程都有对应的一些线性区资源。

每个进程的地址空间都对应一个内存描述符结构—`mm_struct`。`task_struct.mm`字段指向这个结构。

```c
/**
 * 内存描述符。task_struct的mm字段指向它。
 * 它包含了进程地址空间有关的全部信息。
 */
struct mm_struct {
	/**
	 * 指向线性区对象的链表头。
	 */
	struct vm_area_struct * mmap;		/* list of VMAs */
	/**
	 * 指向线性区对象的红-黑树的根
	 */
	struct rb_root mm_rb;
	/**
	 * 指向最后一个引用的线性区对象。
	 */
	struct vm_area_struct * mmap_cache;	/* last find_vma result */
	/**
	 * 在进程地址空间中搜索有效线性地址区的方法。
	 */
	unsigned long (*get_unmapped_area) (struct file *filp,
				unsigned long addr, unsigned long len,
				unsigned long pgoff, unsigned long flags);
	/**
	 * 释放线性地址区间时调用的方法。
	 */
	void (*unmap_area) (struct vm_area_struct *area);
	/**
	 * 标识第一个分配的匿名线性区或文件内存映射的线性地址。
	 */
	unsigned long mmap_base;		/* base of mmap area */
	/**
	 * 内核从这个地址开始搜索进程地址空间中线性地址的空间区间。
	 */
	unsigned long free_area_cache;		/* first hole */
	/**
	 * 指向页全局目录。
	 */
	pgd_t * pgd;
	/**
	 * 次使用计数器。存放共享mm_struct数据结构的轻量级进程的个数。
	 */
	atomic_t mm_users;			/* How many users with user space? */
	/**
	 * 主使用计数器。每当mm_count递减时，内核都要检查它是否变为0,如果是，就要解除这个内存描述符。
	 */
	atomic_t mm_count;			/* How many references to "struct mm_struct" (users count as 1) */
	/**
	 * 线性区的个数。
	 */
	int map_count;				/* number of VMAs */
	/**
	 * 内存描述符的读写信号量。
	 * 由于描述符可能在几个轻量级进程间共享，通过这个信号量可以避免竞争条件。
	 */
	struct rw_semaphore mmap_sem;
	/**
	 * 线性区和页表的自旋锁。
	 */
	spinlock_t page_table_lock;		/* Protects page tables, mm->rss, mm->anon_rss */

	/**
	 * 指向内存描述符链表中的相邻元素。
	 */
	struct list_head mmlist;		/* List of maybe swapped mm's.  These are globally strung
						 * together off init_mm.mmlist, and are protected
						 * by mmlist_lock
						 */

	/**
	 * start_code-可执行代码的起始地址。
	 * end_code-可执行代码的最后地址。
	 * start_data-已初始化数据的起始地址。
	 * end_data--已初始化数据的结束地址。
	 */
	unsigned long start_code, end_code, start_data, end_data;
	/**
	 * start_brk-堆的超始地址。
	 * brk-堆的当前最后地址。
	 * start_stack-用户态堆栈的起始地址。
	 */
	unsigned long start_brk, brk, start_stack;
	/**
	 * arg_start-命令行参数的起始地址。
	 * arg_end-命令行参数的结束地址。
	 * env_start-环境变量的起始地址。
	 * env_end-环境变量的结束地址。
	 */
	unsigned long arg_start, arg_end, env_start, env_end;
	/**
	 * rss-分配给进程的页框总数
	 * anon_rss-分配给匿名内存映射的页框数。s
	 * total_vm-进程地址空间的大小(页框数)
	 * locked_vm-锁住而不能换出的页的个数。
	 * shared_vm-共享文件内存映射中的页数。
	 */
	unsigned long rss, anon_rss, total_vm, locked_vm, shared_vm;
	/**
	 * exec_vm-可执行内存映射的页数。
	 * stack_vm-用户态堆栈中的页数。
	 * reserved_vm-在保留区中的页数或在特殊线性区中的页数。
	 * def_flags-线性区默认的访问标志。
	 * nr_ptes-this进程的页表数。
	 */
	unsigned long exec_vm, stack_vm, reserved_vm, def_flags, nr_ptes;

	/**
	 * 开始执行elf程序时使用。
	 */
	unsigned long saved_auxv[42]; /* for /proc/PID/auxv */

	/**
	 * 表示是否可以产生内存信息转储的标志。
	 */
	unsigned dumpable:1;
	/**
	 * 懒惰TLB交换的位掩码。
	 */
	cpumask_t cpu_vm_mask;

	/* Architecture-specific MM context */
	/**
	 * 特殊体系结构信息的表。
	 * 如80X86平台上的LDT地址。
	 */
	mm_context_t context;

	/* Token based thrashing protection. */
	/**
	 * 进程有资格获得交换标记的时间。
	 */
	unsigned long swap_token_time;
	/**
	 * 如果最近发生了主缺页。则设置该标志。
	 */
	char recent_pagein;

	/* coredumping support */
	/**
	 * 正在把进程地址空间的内容卸载到转储文件中的轻量级进程的数量。
	 */
	int core_waiters;
	/**
	 * core_startup_done-指向创建内存转储文件时的补充原语。
	 * core_done-创建内存转储文件时使用的补充原语。
	 */
	struct completion *core_startup_done, core_done;

	/* aio bits */
	/**
	 * 用于保护异步IO上下文链表的锁。
	 */
	rwlock_t		ioctx_list_lock;
	/**
	 * 异步IO上下文链表。
	 * 一个应用可以创建多个AIO环境，一个给定进程的所有的kioctx描述符存放在一个单向链表中，该链表位于ioctx_list字段
	 */
	struct kioctx		*ioctx_list;

	/**
	 * 默认的异步IO上下文。
	 */
	struct kioctx		default_kioctx;

	/**
	 * 进程所拥有的最大页框数。
	 */
	unsigned long hiwater_rss;	/* High-water RSS usage */
	/**
	 * 进程线性区中的最大页数。
	 */
	unsigned long hiwater_vm;	/* High-water virtual memory usage */
};
```

根据`mmlist`成员可以看出设计上所有内存描述符都链成一个表。链表第一个元素是`init_mm`的`mmlist`，`init_mm`是初始化阶段进程0所用。

### 内核线程

内核线程比较特别，它没有自己的进程空间，运行在内核态使用的是TASK_SIZE以上的地址，所以不用线性区。

内核线程使用一组最近运行的普通进程的页表（大于TASK_SIZE线性地址空间页表项都是相同的，内核线程使用什么页表集根本没有关系）。所以，每个进程描述符中包含了两种内存描述符指针：`mm`和`active_mm`。

`mm`指向的是进程拥有的内存描述符，而`active_mm`指向进程运行时使用的内存描述符。对普通进程来说，两个字段值相同。然而内核线程没有内存描述符，所以`mm`一直是NULL。内核线程得以运行时，`active_mm`字段会被初始化为上一个运行进程的`active_mm`值（参考schedule()调度函数）。

## 线性区

每个线性区都对应一个`vm_area_struct`对象。

```c
/*
 * This struct defines a memory VMM memory area. There is one of these
 * per VM-area/task.  A VM area is any part of the process virtual memory
 * space that has a special rule for the page-fault handlers (ie a shared
 * library, the executable area etc).
 */
/**
 * 线性区描述符。
 */
struct vm_area_struct {
	/**
	 * 指向线性区所在的内存描述符。
	 */
	struct mm_struct * vm_mm;	/* The address space we belong to. */
	/**
	 * 线性区内的第一个线性地址。
	 */
	unsigned long vm_start;		/* Our start address within vm_mm. */
	/**
	 * 线性区之后的第一个线性地址。
	 */
	unsigned long vm_end;		/* The first byte after our end address
					   within vm_mm. */

	/* linked list of VM areas per task, sorted by address */
	/**
	 * 进程链表中的下一个线性区。
	 */
	struct vm_area_struct *vm_next;

	/**
	 * 线性区中页框的访问许可权。
	 */
	pgprot_t vm_page_prot;		/* Access permissions of this VMA. */
	/**
	 * 线性区的标志。
	 */
	unsigned long vm_flags;		/* Flags, listed below. */

	/**
	 * 用于红黑树的数据。
	 */
	struct rb_node vm_rb;

	/*
	 * For areas with an address space and backing store,
	 * linkage into the address_space->i_mmap prio tree, or
	 * linkage to the list of like vmas hanging off its node, or
	 * linkage of vma in the address_space->i_mmap_nonlinear list.
	 */
	/**
	 * 链接到反映射所使用的数据结构。
	 */
	union {
		/**
		 * 如果在优先搜索树中，存在两个节点的基索引、堆索引、大小索引完全相同，那么这些相同的节点会被链接到一个链表，而vm_set就是这个链表的元素。
		 */
		struct {
			struct list_head list;
			void *parent;	/* aligns with prio_tree_node parent */
			struct vm_area_struct *head;
		} vm_set;

		/**
		 * 如果是文件映射，那么prio_tree_node用于将线性区插入到优先搜索树中。作为搜索树的一个节点。
		 */
		struct raw_prio_tree_node prio_tree_node;
	} shared;

	/*
	 * A file's MAP_PRIVATE vma can be in both i_mmap tree and anon_vma
	 * list, after a COW of one of the file pages.  A MAP_SHARED vma
	 * can only be in the i_mmap tree.  An anonymous MAP_PRIVATE, stack
	 * or brk vma (with NULL file) can only be in an anon_vma list.
	 */
	/**
	 * 指向匿名线性区链表的指针(参见"映射页的反映射")。
	 * 页框结构有一个anon_vma指针，指向该页的第一个线性区，随后的线性区通过此字段链接起来。
	 * 通过此字段，可以将线性区链接到此链表中。
	 */
	struct list_head anon_vma_node;	/* Serialized by anon_vma->lock */
	/**
	 * 指向anon_vma数据结构的指针(参见"映射页的反映射")。此指针也存放在页结构的mapping字段中。
	 */
	struct anon_vma *anon_vma;	/* Serialized by page_table_lock */

	/* Function pointers to deal with this struct. */
	/**
	 * 指向线性区的方法。
	 */
	struct vm_operations_struct * vm_ops;

	/* Information about our backing store: */
	/**
	 * 在映射文件中的偏移量(以页为单位)。对匿名页，它等于0或vm_start/PAGE_SIZE
	 */
	unsigned long vm_pgoff;		/* Offset (within vm_file) in PAGE_SIZE
					   units, *not* PAGE_CACHE_SIZE */
	/**
	 * 指向映射文件的文件对象(如果有的话)
	 */
	struct file * vm_file;		/* File we map to (can be NULL). */
	/**
	 * 指向内存区的私有数据。
	 */
	void * vm_private_data;		/* was vm_pte (shared mem) */
	/**
	 * 释放非线性文件内存映射中的一个线性地址区间时使用。
	 */
	unsigned long vm_truncate_count;/* truncate_count or restart_addr */

#ifndef CONFIG_MMU
	atomic_t vm_usage;		/* refcount (VMAs shared if !MMU) */
#endif
#ifdef CONFIG_NUMA
	struct mempolicy *vm_policy;	/* NUMA policy for the VMA */
#endif
};
```

线性区从不重叠，内核尽力把新分配的线性区与紧邻的现有线性区合并。

![](/images/ulk/20180807_1.jpg)

`vm_ops`结构存放了线性区的方法：

```c
/*
 * These are the virtual MM functions - opening of an area, closing and
 * unmapping it (needed to keep files on disk up-to-date etc), pointer
 * to the functions called when a no-page or a wp-page exception occurs. 
 */
/**
 * 线性区的方法。
 */
struct vm_operations_struct {
	/**
	 * 当把线性区增加到进程所拥有的线性区集合时调用。
	 */
	void (*open)(struct vm_area_struct * area);
	/**
	 * 当从进程所拥有的线性区集合删除线性区时调用。
	 */
	void (*close)(struct vm_area_struct * area);
	/**
	 * 当进程试图访问RAM中不存在的一个页，但该页的线性地址属于线性区时，由缺页异常处理程序调用。
	 */
	struct page * (*nopage)(struct vm_area_struct * area, unsigned long address, int *type);
	/**
	 * 设置线性区的线性地址(预缺页)所对应的页表项时调用。主要用于非线性文件内存映射。
	 */
	int (*populate)(struct vm_area_struct * area, unsigned long address, unsigned long len, pgprot_t prot, unsigned long pgoff, int nonblock);
#ifdef CONFIG_NUMA
	int (*set_policy)(struct vm_area_struct *vma, struct mempolicy *new);
	struct mempolicy *(*get_policy)(struct vm_area_struct *vma,
					unsigned long addr);
#endif
};
```

线性区的结构图：

![](/images/ulk/20180807_2.jpg)

链表遍历搜索线性区毕竟低效，Linux内核也使用了红黑树来存放进程线性区（红黑树太经典了，不科普了），内存描述符的`mm_rb`字段指向红黑树的根。

线性区管理的几个底层函数：

### `find_vma()`

```c
/* Look up the first VMA which satisfies  addr < vm_end,  NULL if none. */
/**
 * 查找给定地址的最邻近区。
 * 它查找线性区的vm_end字段大于addr的第一个线性区的位置。并返回这个线性区描述符的地址。
 * 如果没有这样的线性区存在，就返回NULL。
 * 由find_vma函数所选择的线性区并不一定要包含addr，因为addr可能位于任何线性区之外。
 *     mm-进程内存描述符地址
 *     addr-线性地址。
 */
struct vm_area_struct * find_vma(struct mm_struct * mm, unsigned long addr)
{
	struct vm_area_struct *vma = NULL;

	if (mm) {
		/* Check the cache first. */
		/* (Cache hit rate is typically around 35%.) */
		/**
		 * mmap_cache指向最后一个引用的线性区对象。
		 * 引入这个附加的字段是为了减少查找一个给定线性地址所在线性区而花费的时间。
		 * 程序中引用地址的局部性使这种情况出现的可能性非常大:
		 *    如果检查的最后一个线性地址属于某一给定的线性区。那么下一个要检查的线性地址也属于这一个线性区。
		 */
		vma = mm->mmap_cache;
		/**
		 * 首先检查mmap_cache指定的线性区是否包含addr，如果是就返回这个线性区描述符的指针。
		 */
		if (!(vma && vma->vm_end > addr && vma->vm_start <= addr)) {
			/**
			 * 进入这里，说明mmap_cache中没有包含addr。就扫描进程的线性区。并在红黑树中查找线性区。
			 */
			struct rb_node * rb_node;

			rb_node = mm->mm_rb.rb_node;
			vma = NULL;

			while (rb_node) {
				struct vm_area_struct * vma_tmp;

				/**
				 * rb_entry从指向红黑树中一个节点的指针导出相应线性区描述符的指针。
				 */
				vma_tmp = rb_entry(rb_node,
						struct vm_area_struct, vm_rb);

				/**
				 * 视情况在左右子树中查找。
				 */
				if (vma_tmp->vm_end > addr) {
					vma = vma_tmp;
					/**
					 * 当前线性区包含addr,退出循环并返回vma.
					 */
					if (vma_tmp->vm_start <= addr)
						break;
					/**
					 * 否则在左子树中继续
					 */
					rb_node = rb_node->rb_left;
				} else/* 在右子树中继续 */
					rb_node = rb_node->rb_right;
			}
			/**
			 * 如果有必要，记录下mmap_cache。这样，下次就从mmap_cache继续查找。
			 */
			if (vma)
				mm->mmap_cache = vma;
		}
	}
	return vma;
}
```

一目了然的红黑树搜索操作。

### `find_vma_intersection()`

```c
/* Look up the first VMA which intersects the interval start_addr..end_addr-1,
   NULL if none.  Assume start_addr < end_addr. */
/**
 * find_vma_intersection函数查找与给定的线性地址区间相重叠的第一个线性区。完全交叉才叫intersection
 * mm-进程的内存描述符。
 * start_addr-要查找的区间的起始地址。
 * end_addr-要查找的区间的结束地址。
 */
static inline struct vm_area_struct * find_vma_intersection(struct mm_struct * mm, unsigned long start_addr, unsigned long end_addr)
{
	struct vm_area_struct * vma = find_vma(mm,start_addr);

	/**
	 * 如果没有这样的线性区存在就返回NULL。
	 * 如果所找到的线性区是从地址区间的末尾开始的，也返回0.
	 */
	if (vma && end_addr <= vma->vm_start)
		vma = NULL;
	return vma;
}
```

内部复用了`find_vma`，如果没有找到就返回NULL。

### `get_unmapped_area()`

```c
/**
 * 搜索进程的地址空间以找到一个可以使用的线性地址区间。
 * len-区间的长度。
 * addr-指定必须从哪个地址开始进行查找。如果查找成功，函数返回这个新区间的起始地址。否则返回错误码-ENOMEM.
 */
unsigned long
get_unmapped_area(struct file *file, unsigned long addr, unsigned long len,
		unsigned long pgoff, unsigned long flags)
{
	if (flags & MAP_FIXED) {
		unsigned long ret;

		/**
		 * 如果addr不等于0,就检查指定的地址是否在用户态空间。
		 * 这是为了避免用户态程序绕过安全检查而影响内核地址空间。
		 */
		if (addr > TASK_SIZE - len)
			return -ENOMEM;
		/**
		 * 检查是否与页边界对齐。
		 */
		if (addr & ~PAGE_MASK)
			return -EINVAL;

		if (file && is_file_hugepages(file))  {
			/*
			 * Check if the given range is hugepage aligned, and
			 * can be made suitable for hugepages.
			 */
			ret = prepare_hugepage_range(addr, len);
		} else {
			/*
			 * Ensure that a normal request is not falling in a
			 * reserved hugepage range.  For some archs like IA-64,
			 * there is a separate region for hugepages.
			 */
			ret = is_hugepage_only_range(addr, len);
		}
		if (ret)
			return -EINVAL;
		return addr;
	}

	/**
	 * 检查线性地址区间是否应该用于文件内存映射或匿名内存映射。
	 * 分别调用文件和内存的get_unmapped_area操作。
	 */
	if (file && file->f_op && file->f_op->get_unmapped_area)
		return file->f_op->get_unmapped_area(file, addr, len,
						pgoff, flags);
    /*
     * 磁盘文件系统不会定义这个方法，那么就调用内存描述符的get_unmapped_area()方法
     */
	return current->mm->get_unmapped_area(file, addr, len, pgoff, flags);
}
```

心智健全检查后，进一步判断线性地址区间是否应该用于文件内存映射(`file->f_op->get_unmapped_area`)或匿名内存映射(`current->mm->get_unmapped_area`)。

第一种文件映射很复杂，以后再研究。而第二种则执行内存描述符本身的`get_unmapped_area`方法。根据进程的线性区类型，由函数`arch_get_unmapped_area()`或`arch_get_unmapped_area_topdown()`实现`get_unmapped_area`方法。

> 通过系统调用mmap，每个进程都可能获得两种不同形式的线性区：从线性地址0x40000000开始向高端地址增长；从用户态堆栈开始向低端地址增长。

```c
/**
 * 分配从低端地址向高端地址移动的线性区(xie.baoyou注：如堆而不是栈)
 */
unsigned long
arch_get_unmapped_area(struct file *filp, unsigned long addr,
		unsigned long len, unsigned long pgoff, unsigned long flags)
{
	struct mm_struct *mm = current->mm;
	struct vm_area_struct *vma;
	unsigned long start_addr;

	/**
	 * 当然了，只要是普通进程地址，就不能超过TASK_SIZE(一般就是3G)
	 */
	if (len > TASK_SIZE)
		return -ENOMEM;

	if (addr) {
		/**
		 * 如果地址不是0，就从addr处开始分配
		 * 当然，需要将addr按4K取整
		 */
		addr = PAGE_ALIGN(addr);
		vma = find_vma(mm, addr);
		/**
		 * TASK_SIZE - len >= addr而不是addr + len <= TASK_SIZE
		 */
		if (TASK_SIZE - len >= addr &&
		    (!vma || addr + len <= vma->vm_start))
			return addr;/* 没有被映射，这块区间可以使用。 */
	}

	/**
	 * 如果addr==0或者前面的搜索失败
	 * 从free_area_cache开始搜索，这个值初始为用户态空间的1/3处（即1G处），它也是为正文段、数据段、BSS段保留的
	 */
	start_addr = addr = mm->free_area_cache;

full_search:
	for (vma = find_vma(mm, addr); ; vma = vma->vm_next) {
		/* At this point:  (!vma || addr < vma->vm_end). */
		if (TASK_SIZE - len < addr) {
			/*
			 * Start a new search - just in case we missed
			 * some holes.
			 */
			/**
			 * 没有找到，从TASK_UNMAPPED_BASE(1G)处开始找
			 * 第二次再从1G开始搜索的时候，start_addr = TASK_UNMAPPED_BASE，就会直接返回错误 -ENOMEM
			 */
			if (start_addr != TASK_UNMAPPED_BASE) {
				start_addr = addr = TASK_UNMAPPED_BASE;
				goto full_search;
			}
			return -ENOMEM;
		}
		if (!vma || addr + len <= vma->vm_start) {
			/*
			 * Remember the place where we stopped the search:
			 */
			/**
			 * 找到了，记下本次找到的地方，下次从addr+len处开始找
			 */
			mm->free_area_cache = addr + len;
			return addr;
		}
		addr = vma->vm_end;
	}
}

unsigned long
arch_get_unmapped_area_topdown(struct file *filp, const unsigned long addr0,
			  const unsigned long len, const unsigned long pgoff,
			  const unsigned long flags)
{
	struct vm_area_struct *vma, *prev_vma;
	struct mm_struct *mm = current->mm;
	unsigned long base = mm->mmap_base, addr = addr0;
	int first_time = 1;

	/* requested length too big for entire address space */
	if (len > TASK_SIZE)
		return -ENOMEM;

	/* dont allow allocations above current base */
	if (mm->free_area_cache > base)
		mm->free_area_cache = base;

	/* requesting a specific address */
	if (addr) {
		addr = PAGE_ALIGN(addr);
		vma = find_vma(mm, addr);
		if (TASK_SIZE - len >= addr &&
				(!vma || addr + len <= vma->vm_start))
			return addr;
	}

try_again:
	/* make sure it can fit in the remaining address space */
	if (mm->free_area_cache < len)
		goto fail;

	/* either no address requested or cant fit in requested address hole */
	addr = (mm->free_area_cache - len) & PAGE_MASK;
	do {
		/*
		 * Lookup failure means no vma is above this address,
		 * i.e. return with success:
		 */
 	 	if (!(vma = find_vma_prev(mm, addr, &prev_vma)))
			return addr;

		/*
		 * new region fits between prev_vma->vm_end and
		 * vma->vm_start, use it:
		 */
		if (addr+len <= vma->vm_start &&
				(!prev_vma || (addr >= prev_vma->vm_end)))
			/* remember the address as a hint for next time */
			return (mm->free_area_cache = addr);
		else
			/* pull free_area_cache down to the first hole */
			if (mm->free_area_cache == vma->vm_end)
				mm->free_area_cache = vma->vm_start;

		/* try just below the current vma->vm_start */
		addr = vma->vm_start-len;
	} while (len <= vma->vm_start);

fail:
	/*
	 * if hint left us with no space for the requested
	 * mapping then try again:
	 */
	if (first_time) {
		mm->free_area_cache = base;
		first_time = 0;
		goto try_again;
	}
	/*
	 * A failed mmap() very likely causes application failure,
	 * so fall back to the bottom-up function here. This scenario
	 * can happen with large stack limits and large mmap()
	 * allocations.
	 */
	mm->free_area_cache = TASK_UNMAPPED_BASE;
	addr = arch_get_unmapped_area(filp, addr0, len, pgoff, flags);
	/*
	 * Restore the topdown base:
	 */
	mm->free_area_cache = base;

	return addr;
}
```

### `insert_vm_struct()`

```c
/* Insert vm structure into process list sorted by address
 * and into the inode's i_mmap tree.  If vm_file is non-NULL
 * then i_mmap_lock is taken here.
 */
/**
 * 在线性区对象链表和内存描述符的红黑树中插入一个vm_area_struct结构。
 * mm-指定进程内存描述符的地址。
 * vmp-指定要插入的vm_area_struct对象的地址。要求线性区对象的vm_start和vm_end字段必须被初始化。
 */
int insert_vm_struct(struct mm_struct * mm, struct vm_area_struct * vma)
{
	struct vm_area_struct * __vma, * prev;
	struct rb_node ** rb_link, * rb_parent;

	/*
	 * The vm_pgoff of a purely anonymous vma should be irrelevant
	 * until its first write fault, when page's anon_vma and index
	 * are set.  But now set the vm_pgoff it will almost certainly
	 * end up with (unless mremap moves it elsewhere before that
	 * first wfault), so /proc/pid/maps tells a consistent story.
	 *
	 * By setting it to reflect the virtual start address of the
	 * vma, merges and splits can happen in a seamless way, just
	 * using the existing file pgoff checks and manipulations.
	 * Similarly in do_mmap_pgoff and in do_brk.
	 */
	if (!vma->vm_file) {
		BUG_ON(vma->anon_vma);
		vma->vm_pgoff = vma->vm_start >> PAGE_SHIFT;
	}
	/**
	 * 调用find_vma_prepare确定在红黑树中vma应该位于何处。
	 */
	__vma = find_vma_prepare(mm,vma->vm_start,&prev,&rb_link,&rb_parent);
	if (__vma && __vma->vm_start < vma->vm_end)
		return -ENOMEM;
	/**
	 * 调用vma_link执行以下操作:
	 *     在mm->mmap所指向的链表中插入线性区。
	 *     在红黑树中插入线性区。
	 *     如果线性区是匿名的，就把它插入相应的anon_vma数据结构作为头节点的链表中。
	 *     递增mm->map_count计数器。
	 *     如果线性区包含一个内存映射文件，则vma_link执行其他与内存映射文件相关的任务。
	 */
	vma_link(mm, vma, prev, rb_link, rb_parent);
	return 0;
}

/**
 * 调用vma_link执行以下操作:
 *     在mm->mmap所指向的链表中插入线性区。
 *     在红黑树中插入线性区。
 *     如果线性区是匿名的，就把它插入相应的anon_vma数据结构作为头节点的链表中。
 *     递增mm->map_count计数器。
 *     如果线性区包含一个内存映射文件，则vma_link执行其他与内存映射文件相关的任务。
 */
static void vma_link(struct mm_struct *mm, struct vm_area_struct *vma,
			struct vm_area_struct *prev, struct rb_node **rb_link,
			struct rb_node *rb_parent)
{
	struct address_space *mapping = NULL;

	if (vma->vm_file)
		mapping = vma->vm_file->f_mapping;

	if (mapping) {
		spin_lock(&mapping->i_mmap_lock);
		vma->vm_truncate_count = mapping->truncate_count;
	}
	anon_vma_lock(vma);

    /*
     * VMA上树，上链
     */
	__vma_link(mm, vma, prev, rb_link, rb_parent);
    /*
     * 如果线性区包含一个内存映射文件，则vma_link执行其他与内存映射文件相关的任务。
     */
	__vma_link_file(vma);

	anon_vma_unlock(vma);
	if (mapping)
		spin_unlock(&mapping->i_mmap_lock);

    /*
     * 递增mm->map_count计数器。
     */
	mm->map_count++;
    /*
     * 校验mm的合法性并调试打印
     */
	validate_mm(mm);
}
...
```

不展开了，脑补都知道干了些什么。

## 分配线性地址空间

```c
/**
 * 为当前进程创建并初始化一个新的线性区。
 * 分配成功后，可以把这个新的线性区与进程已有的其他线性区进行合并。
 * file,offset-如果新的线性区将把一个文件映射到内存，则使用文件描述符指针file和文件偏移量offset.当不需要内存映射时，file和offset都会为空
 * addr-这个线性地址指定从何处开始查找一个空闲的区间。
 * len-线性地址区间的长度。
 * prot-这个参数指定这个线性区所包含页的访问权限。可能的标志有PROT_READ,PROT_WRITE,PROT_EXEC和PROT_NONE.前三个标志与VM_READ,VM_WRITE,WM_EXEC一样。PROT_NONE表示没有以上权限中的任意一个
 * flag-这个参数指定线性区的其他标志。MAP_GROWSDOWN,MAP_LOCKED,MAP_DENYWRITE,MAP_EXECURABLE
 */
static inline unsigned long do_mmap(struct file *file, unsigned long addr,
	unsigned long len, unsigned long prot,
	unsigned long flag, unsigned long offset)
{
	unsigned long ret = -EINVAL;
	/**
	 * 首先检查是否溢出。
	 */
	if ((offset + PAGE_ALIGN(len)) < offset)
		goto out;
	/**
	 * 检查是否对齐页。
	 */
	if (!(offset & ~PAGE_MASK))
		ret = do_mmap_pgoff(file, addr, len, prot, flag, offset >> PAGE_SHIFT);
out:
	return ret;
}

/*
 * The caller must hold down_write(current->mm->mmap_sem).
 */
/**
 * do_mmap的辅助函数。
 */
unsigned long do_mmap_pgoff(struct file * file, unsigned long addr,
			unsigned long len, unsigned long prot,
			unsigned long flags, unsigned long pgoff)
{
	struct mm_struct * mm = current->mm;
	struct vm_area_struct * vma, * prev;
	struct inode *inode;
	unsigned int vm_flags;
	int correct_wcount = 0;
	int error;
	struct rb_node ** rb_link, * rb_parent;
	int accountable = 1;
	unsigned long charged = 0;

	if (file) {
		if (is_file_hugepages(file))
			accountable = 0;

		/**
		 * 检查是否为要映射的文件定义了mmap文件操作。如果没有，就返回一个错误码。
		 * 如果文件操作表的mmap为NULL说明相应的文件不能被映射(例如，这是一个目录).
		 */
		if (!file->f_op || !file->f_op->mmap)
			return -ENODEV;

		if ((prot & PROT_EXEC) &&
		    (file->f_vfsmnt->mnt_flags & MNT_NOEXEC))
			return -EPERM;
	}
	/*
	 * Does the application expect PROT_READ to imply PROT_EXEC?
	 *
	 * (the exception is when the underlying filesystem is noexec
	 *  mounted, in which case we dont add PROT_EXEC.)
	 */
	if ((prot & PROT_READ) && (current->personality & READ_IMPLIES_EXEC))
		if (!(file && (file->f_vfsmnt->mnt_flags & MNT_NOEXEC)))
			prot |= PROT_EXEC;

	/**
	 * 检查长度为0
	 */
	if (!len)
		return addr;

	/* Careful about overflows.. */
	/**
	 * 检查包含的地址大于TASK_SIZE
	 */
	len = PAGE_ALIGN(len);
	if (!len || len > TASK_SIZE)
		return -EINVAL;

	/* offset overflow? */
	/**
	 * 是否越界了
	 */
	if ((pgoff + (len >> PAGE_SHIFT)) < pgoff)
		return -EINVAL;

	/* Too many mappings? */
	/**
	 * 进程映射了过多的线性区。
	 */
	if (mm->map_count > sysctl_max_map_count)
		return -ENOMEM;

	/* Obtain the address to map to. we verify (or select) it and ensure
	 * that it represents a valid section of the address space.
	 */
	/**
	 * get_unmapped_area获得新线性区的线性地址区间。
	 * 这个函数可能会调用文件对象的get_unapped_area方法。
	 */
	addr = get_unmapped_area(file, addr, len, pgoff, flags); /*★*/
	if (addr & ~PAGE_MASK)
		return addr;

	/* Do simple checking here so the lower-level routines won't have
	 * to. we assume access permissions have been handled by the open
	 * of the memory object, so we don't do any here.
	 */
	/**
	 * 通过prot和flag计算新线性区描述符的标志。
	 * mm->def_flags是线性区的默认标志。它只能由mlockall系统调用修改。这个调用可以设置VM_LOCKED标志。由此锁住以后申请的RAM中的有页。
	 */
	vm_flags = calc_vm_prot_bits(prot) | calc_vm_flag_bits(flags) |
			mm->def_flags | VM_MAYREAD | VM_MAYWRITE | VM_MAYEXEC; /*★*/

	/**
	 * flag参数指定新线性区地址区间的页必须被锁在RAM中，但不允许进程创建上锁的线性区
	 */
	if (flags & MAP_LOCKED) {
		if (!can_do_mlock())
			return -EPERM;
		vm_flags |= VM_LOCKED;
	}
	/* mlock MCL_FUTURE? */
	/**
	 * 或者进程加锁页的总数超过了保存在进程描述符的rlim[RLIMIT_MEMLOCK].rlim_cur字段中的值。
	 * 也直接返回错误。
	 */
	if (vm_flags & VM_LOCKED) {
		unsigned long locked, lock_limit;
		locked = mm->locked_vm << PAGE_SHIFT;
		lock_limit = current->signal->rlim[RLIMIT_MEMLOCK].rlim_cur;
		locked += len;
		if (locked > lock_limit && !capable(CAP_IPC_LOCK))
			return -EAGAIN;
	}

	inode = file ? file->f_dentry->d_inode : NULL;

	if (file) { /*★*/
		switch (flags & MAP_TYPE) {
        /*
         * 如果请求一个共享的内存映射
         */
		case MAP_SHARED:
			/**
			 * 如果请求一个共享可写的内存映射，就检查文件是否为写入而打开的。而不是以追加模式打开的(O_APPEND标志)
			 */
			if ((prot&PROT_WRITE) && !(file->f_mode&FMODE_WRITE))
				return -EACCES;

			/*
			 * Make sure we don't allow writing to an append-only
			 * file..
			 */
			/**
			 * 如果节点仅仅允许追加写，但是文件以写方式打开，则返回错误。
			 */
			if (IS_APPEND(inode) && (file->f_mode & FMODE_WRITE))
				return -EACCES;

			/*
			 * Make sure there are no mandatory locks on the file.
			 */
			/**
			 * 如果请求一个共享内存映射，就检查文件上没有强制锁。
			 */
			if (locks_verify_locked(inode))
				return -EAGAIN;

			vm_flags |= VM_SHARED | VM_MAYSHARE;
			/**
			 * 如果文件没有写权限，那么相应的线性区也不会有写权限。
			 */
			if (!(file->f_mode & FMODE_WRITE))
				vm_flags &= ~(VM_MAYWRITE | VM_SHARED);

			/* fall through */
			/**
			 * 此处没有调用break，也就是说，即便是共享映射，也要进行下面的映射。
			 *
			 * 对于任何形式的内存映射，都要检查文件是为读操作而打开的
			 */
		case MAP_PRIVATE:
			/**
			 * 不论是共享映射还是私有映射，都要检查文件的读权限。
			 */
			if (!(file->f_mode & FMODE_READ))
				return -EACCES;
			break;

		default:
			return -EINVAL;
		}
	} else {
		switch (flags & MAP_TYPE) {
		case MAP_SHARED:
			vm_flags |= VM_SHARED | VM_MAYSHARE;
			break;
		case MAP_PRIVATE:
			/*
			 * Set pgoff according to addr for anon_vma.
			 */
			pgoff = addr >> PAGE_SHIFT;
			break;
		default:
			return -EINVAL;
		}
	}

	error = security_file_mmap(file, prot, flags);
	if (error)
		return error;
		
	/* Clear old maps */
	error = -ENOMEM;
munmap_back:
	/**
	 * find_vma_prepare确定处于新区间前的线性区对象的位置，以及在红黑树中新线性区的位置。
	 */
	vma = find_vma_prepare(mm, addr, &prev, &rb_link, &rb_parent);
	/**
	 * 检查是否还存在与新区间重叠的线性区。
	 */
	if (vma && vma->vm_start < addr + len) {
		/**
		 * 重叠了，就调用do_munmap删除新的线性区，然后重复整个步骤。
		 */
		if (do_munmap(mm, addr, len))
			return -ENOMEM;
		goto munmap_back;
	}

	/* Check against address space limit. */
	/**
	 * 检查进程地址空间的大小mm->total_vm << PAGE_SHIFT) + len是否超过允许的值。
	 * 此检查不能被提前，因为上一步分配的线性区可能和已有线性区重叠，不能被加入线性区链表。
	 */
	if ((mm->total_vm << PAGE_SHIFT) + len
	    > current->signal->rlim[RLIMIT_AS].rlim_cur)
		return -ENOMEM;

	/**
	 * 没有MAP_NORESERVE表示需要检查空闲页框的数目。
	 */
	if (accountable && (!(flags & MAP_NORESERVE) ||
			    sysctl_overcommit_memory == OVERCOMMIT_NEVER)) {
		if (vm_flags & VM_SHARED) {
			/* Check memory availability in shmem_file_setup? */
			vm_flags |= VM_ACCOUNT;
		} else if (vm_flags & VM_WRITE) {
			/*
			 * Private writable mapping: check memory availability
			 */
			/**
			 * 需要检查空闲页框数目，并且新的线性区是私有可写页
			 */
			charged = len >> PAGE_SHIFT;
			/**
			 * 如果没有足够的空闲页框，就返回ENOMEM
			 */
			if (security_vm_enough_memory(charged))
				return -ENOMEM;
			vm_flags |= VM_ACCOUNT;
		}
	}

	/*
	 * Can we just expand an old private anonymous mapping?
	 * The VM_SHARED test is necessary because shmem_zero_setup
	 * will create the file object for a shared anonymous map below.
	 */
	/**
	 * 不是文件映射，并且新区间是私有的，那么就调用vma_merge
	 * 它会检查前一个线性区是否可扩展，以包含新区间。
	 * 当新区间正好是两个区间之间的空洞时，它会将三个区间合并起来。
	 * vma_merge成功，返回值为新merged的新的vma，如果不成功，返回NULL
	 */
	if (!file && !(vm_flags & VM_SHARED) &&
	    vma_merge(mm, prev, addr, addr + len, vm_flags,
					NULL, NULL, pgoff, NULL))
		goto out;

	/*
	 * Determine the object being mapped and call the appropriate
	 * specific mapper. the address has already been validated, but
	 * not unmapped, but the maps are removed from the list.
	 */
	/**
	 * 运行到此，说明没有发生线性区合并。
	 * 首先调用kmem_cache_alloc为新的线性区分配一个vm_area_struct
	 */
	vma = kmem_cache_alloc(vm_area_cachep, SLAB_KERNEL);
	if (!vma) {
		error = -ENOMEM;
		goto unacct_error;
	}
	/**
	 * 初始化新vma对象。
	 */
	memset(vma, 0, sizeof(*vma));

	vma->vm_mm = mm;
	vma->vm_start = addr;
	vma->vm_end = addr + len;
	vma->vm_flags = vm_flags;
	vma->vm_page_prot = protection_map[vm_flags & 0x0f];
	vma->vm_pgoff = pgoff;

	if (file) {
		error = -EINVAL;
		if (vm_flags & (VM_GROWSDOWN|VM_GROWSUP))
			goto free_vma;
		if (vm_flags & VM_DENYWRITE) {
			error = deny_write_access(file);
			if (error)
				goto free_vma;
			correct_wcount = 1;
		}
		/**
		 * 用文件对象的地址初始化线性区描述符的vm_file字段，并增加文件的引用计数。
		 */
		vma->vm_file = file;
		get_file(file);
		/**
		 * 对映射的文件调用mmap方法，对于大多数文件系统，该方法由generic_file_mmap实现。它执行以下步骤:
		 *     将当前时间赋给文件索引节点对象的i_atime字段，并将该索引节点标记为脏。
		 *     用generic_file_vm_ops表的地址初始化线性区描述符的vm_ops字段，在这个表中的方法，除了nopage和populate方法外，其他所有都为空。
		 *          其中nopage方法由filemap_nopage实现，而populate方法由filemap_populate实现。
		 * 对于大多数文件系统，这里调用generic_file_mmap
		 */
		error = file->f_op->mmap(file, vma); /*★*/
		if (error)
			goto unmap_and_free_vma;
	} else if (vm_flags & VM_SHARED) {
		/**
		 * 新线性区有VM_SHARED标志，又不是映射磁盘上的文件。则该线性区是一个共享匿名区。
		 * shmem_zero_setup对它进行初始化。共享匿名区主要用于进程间通信。
		 */
		error = shmem_zero_setup(vma);
		if (error)
			goto free_vma;
	}

	/* We set VM_ACCOUNT in a shared mapping's vm_flags, to inform
	 * shmem_zero_setup (perhaps called through /dev/zero's ->mmap)
	 * that memory reservation must be checked; but that reservation
	 * belongs to shared memory object, not to vma: so now clear it.
	 */
	if ((vm_flags & (VM_SHARED|VM_ACCOUNT)) == (VM_SHARED|VM_ACCOUNT))
		vma->vm_flags &= ~VM_ACCOUNT;

	/* Can addr have changed??
	 *
	 * Answer: Yes, several device drivers can do it in their
	 *         f_op->mmap method. -DaveM
	 */
	addr = vma->vm_start;
	pgoff = vma->vm_pgoff;
	vm_flags = vma->vm_flags;

	if (!file || !vma_merge(mm, prev, addr, vma->vm_end,
			vma->vm_flags, NULL, file, pgoff, vma_policy(vma))) {
		file = vma->vm_file;
		/**
		 * vma_link将新线性区插入到线性区链表和红黑树中。
		 */
		vma_link(mm, vma, prev, rb_link, rb_parent);
		if (correct_wcount)
			atomic_inc(&inode->i_writecount);
	} else {
		if (file) {
			/**
			 * 增加文件索引节点对象的i_writecount
			 */
			if (correct_wcount)
				atomic_inc(&inode->i_writecount); /*★*/
			fput(file);
		}
		mpol_free(vma_policy(vma));
		kmem_cache_free(vm_area_cachep, vma);
	}
out:	
	/**
	 * 增加进程地址空间的大小。
	 */
	mm->total_vm += len >> PAGE_SHIFT;
	__vm_stat_account(mm, vm_flags, file, len >> PAGE_SHIFT);
	/**
	 * VM_LOCKED标志被设置，就调用make_pages_present连续分配线性区的所有页，并将所有页锁在RAM中。
	 */
	if (vm_flags & VM_LOCKED) {
		mm->locked_vm += len >> PAGE_SHIFT;
		/**
		 * make_pages_present在所有页上循环，对其中每个页，调用follow_page检查当前页表中是否有到物理页的映射。
		 * 如果没有这样的页存在，就调用handle_mm_fault。这个函数分配一个页框并根据内存描述符的vm_flags字段设置它的页表项。
		 */
		make_pages_present(addr, addr + len);
	}
	if (flags & MAP_POPULATE) {
		up_write(&mm->mmap_sem);
		sys_remap_file_pages(addr, len, 0,
					pgoff, flags & MAP_NONBLOCK);
		down_write(&mm->mmap_sem);
	}
	acct_update_integrals();
	update_mem_hiwater();
    /*
     * 最后，函数通过返回新线性区的线性地址而终止
     */
	return addr;

unmap_and_free_vma:
	if (correct_wcount)
		atomic_inc(&inode->i_writecount);
	vma->vm_file = NULL;
	fput(file);

	/* Undo any partial mapping done by a device driver. */
	zap_page_range(vma, vma->vm_start, vma->vm_end - vma->vm_start, NULL);
free_vma:
	kmem_cache_free(vm_area_cachep, vma);
unacct_error:
	if (charged)
		vm_unacct_memory(charged);
	return error;
}
```

##释放线性地址空间

```c
/* Munmap is split into 2 main parts -- this part which finds
 * what needs doing, and the areas themselves, which do the
 * work.  This now handles partial unmappings.
 * Jeremy Fitzhardinge <jeremy@goop.org>
 */
/**
 * 从当前进程的地址空间中删除一个线性地址区间。
 * 要删除的区间并不总是对应一个线性区。它或者是一个线性区的一部分，或者是多个线性区。
 * mm-进程内存描述符。
 * start-要删除的地址区间的起始地址。
 * len-要删除的长度。
 */
int do_munmap(struct mm_struct *mm, unsigned long start, size_t len)
{
	unsigned long end;
	struct vm_area_struct *mpnt, *prev, *last;

	/**
	 * 初步检查：线性区地址不能大于TASK_SIZE，start必须是4096的整数倍。
	 */
	if ((start & ~PAGE_MASK) || start > TASK_SIZE || len > TASK_SIZE-start)
		return -EINVAL;

	/**
	 * len也不能为0
	 */
	if ((len = PAGE_ALIGN(len)) == 0)
		return -EINVAL;

	/* Find the first overlapping VMA */
	/**
	 * mpnt是要删除的线性地址区间之后第一个线性区的位置。mpnt->end>start
	 */
	mpnt = find_vma_prev(mm, start, &prev);
	/**
	 * 没有这样的线性区
	 */
	if (!mpnt)
		return 0;
	/* we have  start < mpnt->vm_end  */

	/* if it doesn't overlap, we have nothing.. */
	end = start + len;
	/**
	 * 也没有与线性地址区间重叠的线性区，就不必做什么了。
	 */
	if (mpnt->vm_start >= end)
		return 0;

	/*
	 * If we need to split any vma, do it now to save pain later.
	 *
	 * Note: mremap's move_vma VM_ACCOUNT handling assumes a partially
	 * unmapped vm_area_struct will remain in use: so lower split_vma
	 * places tmp vma above, and higher split_vma places tmp vma below.
	 */
	/*
	 * 第一步: 分裂前半部分
     */ 
	/**
	 * 线性区的起始地址在mpnt内，就调用split_vma把线性区mpnt划分成两个较小的区：一个区在线性地址区间外部，而另一个区间内部
	 */
	if (start > mpnt->vm_start) {
		int error = split_vma(mm, mpnt, start, 0);
		if (error)
			return error;
		/**
		 * prev以前存储的是指向线性区mpnt前面一个线性区的指针。
		 * 现在它指向mpnt，即指向线性地址区间外部的那个新线性区。
		 * 这样，prev仍然指向要删除的那个线性区前面的那个线性区。
		 */
		prev = mpnt;
	}

	/*
	 * 第二步: 分裂后半部分
     */ 
	/* Does it split the last one? */
	last = find_vma(mm, end);
	/**
	 * 如果线性地址空间的结束地址在一个线性区内部，就再次调用split_vma把最后重叠的那个线性区划分成两个较小的区。
	 */
	if (last && end > last->vm_start) {
		int error = split_vma(mm, last, end, 1);
		if (error)
			return error;
	}
	/**
	 * 更新mpnt，使它指向线性地址区间的第一个线性区。
	 * 如果prev为空，即没有，就从mm->mmap获得第一个线性区的地址。
	 */
	mpnt = prev? prev->vm_next: mm->mmap;

	/*
	 * Remove the vma's, and unmap the actual pages
	 */
	/**
	 * detach_vmas_to_be_unmapped从进程的线性地址空间中删除位于线性地址区间中的线性区。
	 * 这可能是一个链表。
	 */
	detach_vmas_to_be_unmapped(mm, mpnt, prev, end);
	spin_lock(&mm->page_table_lock);
	/**
	 * unmap_region清除与线性地址区间对应的页表项并释放相应的页框。
	 */
	unmap_region(mm, mpnt, prev, start, end);
	spin_unlock(&mm->page_table_lock);

	/* Fix up all other VM information */
	/**
	 * 释放detach_vmas_to_be_unmapped收集的位于线性区间内的线性区描述符
	 */
	unmap_vma_list(mm, mpnt);

    /*
     * 返回0，成功
     */
	return 0;
}
```

## Appendix

### `split_vma()`

```c
/*
 * Split a vma into two pieces at address 'addr', a new vma is allocated
 * either for the first part or the the tail.
 */
/**
 * 把与线性地址区间交叉的线性区划分成两个较小的区。一个在线性地址区间外部，另一个在区间的内部。
 * mm-内存描述符指针。
 * vma-要被划分的线性区。
 * addr-区间与线性区之间交叉点的地址。
 * new_below-表示区间与线性区之间交叉点在区间超始处还是结束处的标志。
 */
int split_vma(struct mm_struct * mm, struct vm_area_struct * vma,
	      unsigned long addr, int new_below)
{
	struct mempolicy *pol;
	struct vm_area_struct *new;

	if (is_vm_hugetlb_page(vma) && (addr & ~HPAGE_MASK))
		return -EINVAL;

	if (mm->map_count >= sysctl_max_map_count)
		return -ENOMEM;

	/**
	 * 获得线性区描述符。如果没有可用的空闲空间，就返回-ENOMEM
	 */
	new = kmem_cache_alloc(vm_area_cachep, SLAB_KERNEL);
	if (!new)
		return -ENOMEM;

	/* most fields are the same, copy all, and then fixup */
	*new = *vma;

	if (new_below)
		/**
		 * 如果new_below为1,表示线性地址区间的结束地址在vma线性区的内部。
		 * 因此把新线性区放在vma线性区的前面。因此把new->vm_end和vma->vm_start设置成addr.
		 */
		new->vm_end = addr;
	else {
		/**
		 * 线性地址区间的起始地址在vma线性区的内部。因此必须把新线性区放在vma线性区之后。
		 * 因此把new->vm_start和vma->vm_end设置成addr.
		 */
		new->vm_start = addr;
		new->vm_pgoff += ((addr - vma->vm_start) >> PAGE_SHIFT);
	}

	pol = mpol_copy(vma_policy(vma));
	if (IS_ERR(pol)) {
		kmem_cache_free(vm_area_cachep, new);
		return PTR_ERR(pol);
	}
	vma_set_policy(new, pol);

	if (new->vm_file)
		get_file(new->vm_file);

	/**
	 * 如果定义了新线性区的open方法，就执行它。
	 */
	if (new->vm_ops && new->vm_ops->open)
		new->vm_ops->open(new);

	/**
	 * 把新线性区链接到线性区链表和红黑树中。
	 */
	if (new_below)
		vma_adjust(vma, addr, vma->vm_end, vma->vm_pgoff +
			((addr - new->vm_start) >> PAGE_SHIFT), new);
	else
		vma_adjust(vma, vma->vm_start, addr, vma->vm_pgoff, new);

	return 0;
}
```

### `unmap_region`

```c
/*
 * Get rid of page table information in the indicated region.
 *
 * Called with the page table lock held.
 */
/**
 * 遍历线性区链表并释放它们的页框。
 * mm-内存描述符指针
 * vma-指向第一个被删除线性区描述符的指针。
 * prev-指向vma前面的线性区的指针。
 * start,end-界定被删除线性地址区间的范围。
 */
static void unmap_region(struct mm_struct *mm,
	struct vm_area_struct *vma,
	struct vm_area_struct *prev,
	unsigned long start,
	unsigned long end)
{
	struct mmu_gather *tlb;
	unsigned long nr_accounted = 0;

	lru_add_drain();
	/**
	 * tlb_gather_mmu初始化每CPU变量mmu_gathers。mmu_gather的值依赖于体系结构。
	 * 通常该变量应该存放成功更新进程页表项所需要的所有信息。
	 * 在x86中，该函数只是简单地把内存描述符指针mm的值赋给本地CPU的mmu_gather变量
	 */
	tlb = tlb_gather_mmu(mm, 0);
	/**
	 * 调用unmap_vmas扫描线性地址空间的所有页表项。
	 * 如果只有一个CPU，就调用free_swap_and_cache反复释放相应的页。
	 */
	unmap_vmas(&tlb, mm, vma, start, end, &nr_accounted, NULL); /*★*/
	vm_unacct_memory(nr_accounted);

	/**
	 * free_pgtables回收已经清空的进程页表。
	 */
	if (is_hugepage_only_range(start, end - start))
		hugetlb_free_pgtables(tlb, prev, start, end);
	else
        //因为删除了一些映射，会造成一个页表空闲的情况，回收页表项所占的空间
		free_pgtables(tlb, prev, start, end);
	/**
	 * 刷新TLB
	 */
	tlb_finish_mmu(tlb, start, end);
}
```

