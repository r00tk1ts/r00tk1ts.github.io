---
title: Linux内核学习——内存管理之进程地址空间
date: 2018-08-13 22:05:11
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
2. `kmem_cache_alloc`或`kmalloc`使用slab分配器为专用或通用对象分配块。
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

![](20180807_1.jpg)

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

![](20180807_2.jpg)

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

## 缺页异常处理程序

缺页异常有两种情况：编程错误引发异常；引用进程地址空间但尚未分配物理页框。

![](20180813_1.jpg)

80x86的缺页异常处理程序非常复杂：

```c
/*
 * This routine handles page faults.  It determines the address,
 * and the problem, and then passes it off to one of the appropriate
 * routines.
 *
 * error_code:
 *	bit 0 == 0 means no page found, 1 means protection fault
 *	bit 1 == 0 means read, 1 means write
 *	bit 2 == 0 means kernel, 1 means user-mode
 */
/**
 * 缺页中断服务程序。
 * regs-发生异常时寄存器的值
 * error_code-当异常发生时，控制单元压入栈中的错误代码。
 *			  当第0位被清0时，则异常是由一个不存在的页所引起的。否则是由无效的访问权限引起的。
 *			  如果第1位被清0，则异常由读访问或者执行访问所引起，如果被设置，则异常由写访问引起。
 *			  如果第2位被清0，则异常发生在内核态，否则异常发生在用户态。
 */
fastcall void do_page_fault(struct pt_regs *regs, unsigned long error_code)
{
	struct task_struct *tsk;
	struct mm_struct *mm;
	struct vm_area_struct * vma;
	unsigned long address;
	unsigned long page;
	int write;
	siginfo_t info;

	/* get the address */
	/**
	 * 读取引起异常的线性地址。CPU控制单元把这个值存放在cr2控制寄存器中。
	 */
	__asm__("movl %%cr2,%0":"=r" (address));

	if (notify_die(DIE_PAGE_FAULT, "page fault", regs, error_code, 14,
					SIGSEGV) == NOTIFY_STOP)
		return;
	/* It's safe to allow irq's after cr2 has been saved */
	/**
	 * 只在保存了cr2就可以打开中断了。
	 * 如果中断发生前是允许中断的，或者运行在虚拟8086模式，就打开中断。
	 */
	if (regs->eflags & (X86_EFLAGS_IF|VM_MASK))
		local_irq_enable();

	tsk = current;

	info.si_code = SEGV_MAPERR;

	/*
	 * We fault-in kernel-space virtual memory on-demand. The
	 * 'reference' page table is init_mm.pgd.
	 *
	 * NOTE! We MUST NOT take any locks for this case. We may
	 * be in an interrupt or a critical region, and should
	 * only copy the information from the master page table,
	 * nothing more.
	 *
	 * This verifies that the fault happens in kernel space
	 * (error_code & 4) == 0, and that the fault was not a
	 * protection error (error_code & 1) == 0.
	 */
	/**
	 * 根据异常地址，判断是访问内核态地址还是用户态地址发生了异常。
	 * xie.baoyou注：这并不代表异常发生在用户态还是内核态。
	 */
	if (unlikely(address >= TASK_SIZE)) { 
        /*
         * !(第0位 = 1，由无效的访问权限引起的； 第2位==1，异常发生在用户态。) ==> 内核态访问了一个不存在的页框
         */
		if (!(error_code & 5))
			/**
			 * 内核态访问了一个不存在的页框，这可能是由于内核态访问非连续内存区而引起的。
			 * 注:vmalloc可能打乱了内核页表，而进程切换后，并没有随着修改这些页表项。这可能会引起异常，而这种异常其实不是程序逻辑错误。
			 */
			goto vmalloc_fault;
		/* 
		 * Don't take the mm semaphore here. If we fixup a prefetch
		 * fault we could otherwise deadlock.
		 */
		/*
		 *  第0位 = 1，由无效的访问权限引起的； 或者 第2位==1，异常发生在用户态。
         */		 
		/**
		 * 否则，第0位或者第2位设置了，可能是没有访问权限或者用户态程序访问了内核态地址。
		 */
		goto bad_area_nosemaphore;
	} 

	mm = tsk->mm;

	/*
	 * If we're in an interrupt, have no user context or are running in an
	 * atomic region then we must not take the fault..
	 */
	/**
	 * 内核是否在执行一些关键例程，或者是内核线程出错了。
	 * in_atomic表示内核现在禁止抢占，一般是中断处理程序、可延迟函数、临界区或内核线程中。
	 * 一般来说，这些程序不会去访问用户空间地址。因为访问这些地址总是可能造成导致阻塞。
	 * 而这些地方是不允许阻塞的。
	 * 总之，问题有点严重。
	 */
	if (in_atomic() || !mm)
		goto bad_area_nosemaphore;

	/* When running in the kernel we expect faults to occur only to
	 * addresses in user space.  All other faults represent errors in the
	 * kernel and should generate an OOPS.  Unfortunatly, in the case of an
	 * erroneous fault occuring in a code path which already holds mmap_sem
	 * we will deadlock attempting to validate the fault against the
	 * address space.  Luckily the kernel only validly references user
	 * space from well defined areas of code, which are listed in the
	 * exceptions table.
	 *
	 * As the vast majority of faults will be valid we will only perform
	 * the source reference check when there is a possibilty of a deadlock.
	 * Attempt to lock the address space, if we cannot we then validate the
	 * source.  If this is invalid we can skip the address space check,
	 * thus avoiding the deadlock.
	 */
	/**
	 * 缺页没有发生在中断处理程序、可延迟函数、临界区、内核线程中
	 * 那么，就需要检查进程所拥有的线性区，以决定引起缺页的线性地址是否包含在进程的地址空间中
	 * 为此，必须获得进程的mmap_sem读写信号量。
	 */

	/**
	 * 既然不是内核BUG，也不是硬件故障，那么缺页发生时，当前进程就还没有为写而获得信号量mmap_sem.
	 * 但是为了稳妥起见，还是调用down_read_trylock确保mmap_sem没有被其他地方占用。
	 */
	if (!down_read_trylock(&mm->mmap_sem)) {
		/**
		 * 一般不会运行到这里来。
		 * 运行到这里表示:信号被关闭了。
		 */
		if ((error_code & 4) == 0 &&
		    !search_exception_tables(regs->eip)) /* 第2位被清0，则异常发生在内核态; 且在异常处理表中又没有对应的处理函数。*/
		    /**
		     * 内核态异常，在异常处理表中又没有对应的处理函数。
		     * 转到bad_area_nosemaphore，它会再检查一下：是否是使用作为系统调用参数被传递给内核的线性地址。
		     * 请回想一下,access_ok只是作了简单的检查，并不确保线性地址空间真的存在（只要是用户态地址就行了）
		     * 也就是说：用户态程序在调用系统调用的时候，可能传递了一个非法的用户态地址给内核。
		     * 而内核试图读写这个地址的时候，就出错了
		     * 的确，这里就会处理这个情况。
		     */
			goto bad_area_nosemaphore;
		/**
		 * 否则，不是内核异常或者严重的硬件故障。并且信号量被其他线程占用了，等待其他线程释放信号量后继续。
		 */
		down_read(&mm->mmap_sem);
	}

	/**
	 * 运行到此，就已经获得了mmap_sem信号量。
	 * 可以放心的操作mm了。
	 * 现在开始搜索出错地址所在的线性区。
	 */
	vma = find_vma(mm, address); /*★*/

	/**
	 * 如果vma为空，说明在出错地址后面没有线性区了，说明错误的地址肯定是无效的。
	 */ 
	if (!vma)
		goto bad_area;
	/**
	 * vma在address后面，并且它的起始地址在address前面，说明线性区包含了这个地址。
	 * 谢天谢地，这很可能不是真的错误，可能是COW机制起作用了，也可能是需要调页了。
	 */
	if (vma->vm_start <= address) /*★*/
		goto good_area;

	/**
	 * 运行到此，说明地址并不在线性区中。
	 * 但是我们还要再判断一下，有可能是push指令引起的异常。和vma==NULL还不太一样。
	 * 直接转到bad_area是不正确的。
	 */
	if (!(vma->vm_flags & VM_GROWSDOWN)) /*★*/
		goto bad_area;
	/**
	 * 运行到此，说明address地址后面的vma有VM_GROWSDOWN标志，表示它是一个堆栈区
	 * 请注意，如果是内核态访问用户态的堆栈空间，就应该直接扩展堆栈，而不判断if (address + 32 < regs->esp)
	 * 注意，如果是内核态在访问用户态堆栈空间，没有32的距离限制，都应该expand_stack
	 */
	if (error_code & 4) {
		/*
		 * accessing the stack below %esp is always a bug.
		 * The "+ 32" is there due to some instructions (like
		 * pusha) doing post-decrement on the stack and that
		 * doesn't show up until later..
		 */
		/**
		 * 虽然下一个线性区是堆栈，可是离非法地址太远了，不可能是操作堆栈引起的错误
		 * xie.baoyou注：32而不是4是考虑到pusha的存在。
		 */
		if (address + 32 < regs->esp) /*★*/
			goto bad_area;
	}
	/**
	 * 线程堆栈空间不足，就扩展一下，一般会成功的，不会运行到bad_area.
	 * 注意:如果异常发生在内核态，说明内核正在访问用户态的栈，就直接扩展用户栈。
	 * 注意这里只是扩展了vma，但是并没有分配新的也
	 */
	if (expand_stack(vma, address)) /*★*/
		goto bad_area;
/*
 * Ok, we have a good vm_area for this memory access, so
 * we can handle it..
 */
/**
 * 处理地址空间内的错误地址。其实可能并不是错误地址。
 */
good_area:
	info.si_code = SEGV_ACCERR;
	write = 0;

	switch (error_code & 3) {/* 错误是由写访问引起的 */ /*★*/
		/**
		 * 无权写？？
		 */
		default:	/* 3: write, present */ /*异常是由写访问,无效的访问权限引起的。*/
#ifdef TEST_VERIFY_AREA
			if (regs->cs == KERNEL_CS)
				printk("WP fault at %08lx\n", regs->eip);
#endif
			/* fall through */
		/**
		 * 写访问出错。
		 */
		case 2:		/* write, not present */ /*异常由由一个不存在的页, 写访问引起。*/
			/**
			 * 但是线性区不允许写，难道是想写只读数据区或者代码段？？？
			 * 注意，当errcode==3也会到这里
			 */
			if (!(vma->vm_flags & VM_WRITE))
				goto bad_area;
			/**
			 * 线性区可写，但是此时发生了写访问错误。
			 * 说明可以启动写时复制或请求调页了。将write++其实就是将它置1
			 */
			write++; /*★*/
			break;
		/**
		 * 没有读权限？？
		 * 可能是由于进程试图访问一个有特权的页框。
		 */
		case 1:		/* read, present */ /*异常由无效的访问权限, 读访问或者执行访问所引起*/
			goto bad_area;
		/**
		 * 要读的页不存在，检查是否真的可读或者可执行
		 */
		case 0:		/* read, not present */ /*异常由一个不存在的页, 读访问或者执行访问所引起*/
			/**
			 * 要读的页不存在，也不允许读和执行，那也是一种错误
			 */
			if (!(vma->vm_flags & (VM_READ | VM_EXEC)))
				goto bad_area;
			/**
			 * 运行到这里，说明要读的页不存在，但是线性区允许读，说明是需要调页了。
			 */
	}

/**
 * 幸免于难，可能不是真正的错误。
 * 呵呵，找块毛巾擦擦汗。
 */
 survive:
	/*
	 * If for any reason at all we couldn't handle the fault,
	 * make sure we exit gracefully rather than endlessly redo
	 * the fault.
	 */
	/**
	 * 线性区的访问权限与引起异常的类型相匹配，调用handle_mm_fault分配一个新页框。
	 * handle_mm_fault中会处理请求调页和写时复制两种机制。
	 */
	switch (handle_mm_fault(mm, vma, address, write)) {
		/**
		 * VM_FAULT_MINOR表示没有阻塞当前进程，即次缺页。
		 */
		case VM_FAULT_MINOR: /*正常*/
			tsk->min_flt++;
			break;
		/**
		 * VM_FAULT_MAJOR表示阻塞了当前进程，即主缺页。
		 * 很可能是由于当用磁盘上的数据填充所分配的页框时花费了时间。
		 */			
		case VM_FAULT_MAJOR: /*正常*/
			tsk->maj_flt++;
			break;
		/**
		 * VM_FAULT_SIGBUS表示其他错误。
		 * do_sigbus会向进程发送SIGBUS信号。
		 */
		case VM_FAULT_SIGBUS: /*不正常*/
			goto do_sigbus;
		/**
		 * VM_FAULT_OOM表示没有足够的内存
		 * 如果不是init进程，就杀死它，否则就调度其他进程运行，等待内存被释放出来。
		 */
		case VM_FAULT_OOM: /*不正常*/
			goto out_of_memory;
		default:
			BUG();
	}

	/*
	 * Did it hit the DOS screen memory VA from vm86 mode?
	 */
	if (regs->eflags & VM_MASK) {
		unsigned long bit = (address - 0xA0000) >> PAGE_SHIFT;
		if (bit < 32)
			tsk->thread.screen_bitmap |= 1 << bit;
	}
	up_read(&mm->mmap_sem);
	return;

/*
 * Something tried to access memory that isn't in our memory map..
 * Fix it, but check if it's kernel or user first..
 */
/**
 * 处理地址空间以外的错误地址。
 * 当要访问的地址不在进程的地址空间内时，执行到此。
 */
bad_area:
	up_read(&mm->mmap_sem);

/**
 * 用户态程序访问了内核态地址，或者访问了没有权限的页框。
 * 或者是内核态线程出错了，也或者是当前有很紧要的操作
 * 总之，运行到这里可不是什么好事。
 */
bad_area_nosemaphore:
	/* User mode accesses just cause a SIGSEGV */
	/**
	 * 第2位 == 1, 异常发生在用户态。
	 * 发生在用户态的错误地址。
	 * 就发生一个SIGSEGV信号给current进程，并结束函数。
	 */
	if (error_code & 4) {
		/* 
		 * Valid to do another page fault here because this one came 
		 * from user space.
		 */
		if (is_prefetch(regs, address, error_code))
			return;

		tsk->thread.cr2 = address;
		/* Kernel addresses are always protection faults */
		tsk->thread.error_code = error_code | (address >= TASK_SIZE);
		tsk->thread.trap_no = 14;
		info.si_signo = SIGSEGV; /*SIGSEGV信号*/
		info.si_errno = 0;
		/* info.si_code has been set above */
		info.si_addr = (void __user *)address;
		/**
		 * force_sig_info确信进程不忽略或阻塞SIGSEGV信号
		 * SEGV_MAPERR或SEGV_ACCERR已经被设置在info.si_code中。
		 */
		force_sig_info(SIGSEGV, &info, tsk); /*★*/
		return;
	}

    /*从这里往下，异常发生在内核态*/
#ifdef CONFIG_X86_F00F_BUG
	/*
	 * Pentium F0 0F C7 C8 bug workaround.
	 */
	if (boot_cpu_data.f00f_bug) {
		unsigned long nr;
		
		nr = (address - idt_descr.address) >> 3;

		if (nr == 6) {
			do_invalid_op(regs, 0);
			return;
		}
	}
#endif
	/**
	 * 异常发生在内核态。
	 */
no_context:
	/* Are we prepared to handle this kernel fault?  */
	/**
	 * 异常的引起是因为把某个线性地址作为系统调用的参数传递给内核。
	 * 调用fixup_exception判断这种情况，如果是这样的话，那谢天谢地，还有修复的可能。
	 * 典型的：fixup_exception可能会向进程发送SIGSEGV信号或者用一个适当的出错码终止系统调用处理程序。
	 */
	if (fixup_exception(regs)) /*★*/
		return;

	/* 
	 * Valid to do another page fault here, because if this fault
	 * had been triggered by is_prefetch fixup_exception would have 
	 * handled it.
	 */
 	if (is_prefetch(regs, address, error_code))
 		return;

/*
 * Oops. The kernel tried to access some bad page. We'll have to
 * terminate things with extreme prejudice.
 */

	bust_spinlocks(1);

#ifdef CONFIG_X86_PAE
	if (error_code & 16) {
		pte_t *pte = lookup_address(address);

		if (pte && pte_present(*pte) && !pte_exec_kernel(*pte))
			printk(KERN_CRIT "kernel tried to execute NX-protected page - exploit attempt? (uid: %d)\n", current->uid);
	}
#endif
	/**
	 * 但愿程序不要运行到这里来，著名的oops出现了^-^
	 * 不过oops算什么呢，真正的内核高手在等着解决这些错误呢
	 */
	if (address < PAGE_SIZE)
		printk(KERN_ALERT "Unable to handle kernel NULL pointer dereference");
	else
		printk(KERN_ALERT "Unable to handle kernel paging request");
	printk(" at virtual address %08lx\n",address);
	printk(KERN_ALERT " printing eip:\n");
	printk("%08lx\n", regs->eip);
	asm("movl %%cr3,%0":"=r" (page));
	page = ((unsigned long *) __va(page))[address >> 22];
	printk(KERN_ALERT "*pde = %08lx\n", page);
	/*
	 * We must not directly access the pte in the highpte
	 * case, the page table might be allocated in highmem.
	 * And lets rather not kmap-atomic the pte, just in case
	 * it's allocated already.
	 */
#ifndef CONFIG_HIGHPTE
	if (page & 1) {
		page &= PAGE_MASK;
		address &= 0x003ff000;
		page = ((unsigned long *) __va(page))[address >> PAGE_SHIFT];
		printk(KERN_ALERT "*pte = %08lx\n", page);
	}
#endif
	die("Oops", regs, error_code); /*★*/
	bust_spinlocks(0);
	do_exit(SIGKILL);   /*★*/

/*
 * We ran out of memory, or some other thing happened to us that made
 * us unable to handle the page fault gracefully.
 */
/**
 * 缺页了，但是没有内存了，就杀死进程（init除外）
 */
out_of_memory:
	up_read(&mm->mmap_sem);
	if (tsk->pid == 1) { /*如果是init, 则调度其他进行*/
		yield(); /*★*/
		down_read(&mm->mmap_sem);
		goto survive;
	}
	printk("VM: killing process %s\n", tsk->comm);
	if (error_code & 4) /*★*/ /*如果不是init,就杀死进程*/
		do_exit(SIGKILL);
	goto no_context;

/**
 * 缺页了，但是分配页时出现了错误，就向进程发送SIGBUS信号。
 */
do_sigbus:
	up_read(&mm->mmap_sem);

	/* Kernel mode? Handle exceptions or die */
	if (!(error_code & 4))
		goto no_context;

	/* User space => ok to do another page fault */
	if (is_prefetch(regs, address, error_code))
		return;

	tsk->thread.cr2 = address;
	tsk->thread.error_code = error_code;
	tsk->thread.trap_no = 14;
	info.si_signo = SIGBUS;
	info.si_errno = 0;
	info.si_code = BUS_ADRERR;
	info.si_addr = (void __user *)address;
	force_sig_info(SIGBUS, &info, tsk); /*★*/
	return;

/**
 * 内核访问了不存在的页框。
 * 内核在更新非连续内存区对应的页表项时是非常懒惰的。实际上，vmalloc和vfree函数只把自己限制在更新主内核页表（即全局目录和它的子页表）。
 * 但是，如果内核真的访问到了vmalloc的空间，就需要把页表项补充完整了。
 */
vmalloc_fault:
	{
		/*
		 * Synchronize this task's top level page-table
		 * with the 'reference' page table.
		 *
		 * Do _not_ use "tsk" here. We might be inside
		 * an interrupt in the middle of a task switch..
		 */
		int index = pgd_index(address);
		unsigned long pgd_paddr;
		pgd_t *pgd, *pgd_k;
		pud_t *pud, *pud_k;
		pmd_t *pmd, *pmd_k;
		pte_t *pte_k;

		/**
		 * 把cr3中当前进程页全局目录的物理地址赋给局部变量pgd_paddr。
		 * 注：内核不使用current->mm->pgd导出当前进程的页全局目录地址。因为这种缺页可能在任何时刻发生，甚至在进程切换期间发生。
		 */
		asm("movl %%cr3,%0":"=r" (pgd_paddr));
		pgd = index + (pgd_t *)__va(pgd_paddr);
		/**
		 * 把主内核页全局目录的线性地址赋给pgd_k
		 */
		pgd_k = init_mm.pgd + index;

		/**
		 * pgd_k对应的主内核页全局目录项为空。说明不是非连续内存区产生的错误。
		 * 因为非连续内存区产生的缺页仅仅是没有页表项，而不会缺少目录项。
		 */
		if (!pgd_present(*pgd_k))
			goto no_context;

		/*
		 * set_pgd(pgd, *pgd_k); here would be useless on PAE
		 * and redundant with the set_pmd() on non-PAE. As would
		 * set_pud.
		 */

		/**
		 * 检查了全局目录项，还必须检查主内核页上级目录项和中间目录项。
		 * 如果它们中有一个为空，也转到no_context
		 */
		pud = pud_offset(pgd, address); /*直接用，没有分配，因为共享了内核主页表的PUD*/
		pud_k = pud_offset(pgd_k, address);
		if (!pud_present(*pud_k))
			goto no_context;
		
		pmd = pmd_offset(pud, address); /*直接用，没有分配，因为共享了内核主页表的PMD*/
		pmd_k = pmd_offset(pud_k, address);
		if (!pmd_present(*pmd_k))
			goto no_context;
		/**
		 * 目录项不为空，说明真的是在访问非连续内存区。就把主目录项复制到进程页中间目录的相应项中。
		 */
		set_pmd(pmd, *pmd_k); /*这里共享了内核主页表的PT，因为PMD,PUD都是共享的，所以vmalloc只可能是因为pgd中的present没有置位引发的*/

		pte_k = pte_offset_kernel(pmd_k, address);
		if (!pte_present(*pte_k))
			goto no_context;
		return;
	}
}

/*
 * By the time we get here, we already hold the mm semaphore
 */
/**
 * 当程序缺页时，调用此过程分配新的页框。
 * mm-异常发生时，正在CPU上运行的进程的内存描述符
 * vma-指向引起异常的线性地址所在线性区的描述符。
 * address-引起异常的地址。
 * write_access-如果tsk试图向address写，则为1，否则为0。为1时候，表示COW
 */
int handle_mm_fault(struct mm_struct *mm, struct vm_area_struct * vma,
		unsigned long address, int write_access)
{
	pgd_t *pgd;
	pud_t *pud;
	pmd_t *pmd;
	pte_t *pte;

	__set_current_state(TASK_RUNNING);

	inc_page_state(pgfault);

	if (is_vm_hugetlb_page(vma))
		return VM_FAULT_SIGBUS;	/* mapping truncation does this. */

	/*
	 * We need the page table lock to synchronize with kswapd
	 * and the SMP-safe atomic PTE updates.
	 */
	/**
	 * pgd_offset和pud_alloc检查映射address的页中间目录和页表是否存在。
	 */
	pgd = pgd_offset(mm, address);
	spin_lock(&mm->page_table_lock);

	pud = pud_alloc(mm, pgd, address);
	if (!pud)
		goto oom;

	pmd = pmd_alloc(mm, pud, address);
	if (!pmd)
		goto oom;

	pte = pte_alloc_map(mm, pmd, address); 
	if (!pte)
		goto oom;

    /*
     * 至此，从pgd到pte的页表已经建立好了，就差新分配一个页面并填写到pte中了
     */

	/**
	 * handle_pte_fault函数检查address地址所对应的页表项。并决定如何为进程分配一个新页框。
	 */
	return handle_pte_fault(mm, vma, address, write_access, pte, pmd);

 oom:
	spin_unlock(&mm->page_table_lock);
	return VM_FAULT_OOM;
}
```

如果被访问的页不存在，内核分配一个新的页框并适当初始化，这种技术叫请求调页。而如果被访问的页存在但是只读，那么内核分配一个新的页框并把旧页框数据拷贝到新页框来初始化它的内容，这叫写时复制COW。

### 请求调页

本质上是一种推迟，直到第一次访问页时才借助缺页异常在RAM中分配。这种推迟很有意义，因为进程开始运行时并不会访问全部地址。这种策略节省了RAM开销但也增加了CPU开销（局部性原理使得缺页异常注定稀有）。

被访问的页不在ram中，有几种可能：要么进程从未访问过该页，要么该页对应页框被内核回收了。此时就需要分配新页框，如何初始化取决于是哪一种页，以前是否被进程访问过。

几种特殊情况：

1. 这个页从未被进程访问且没有映射磁盘文件，或者页映射了磁盘文件。内核可以识别这种情况，因为页表相应的表项被填充为0，`pte_none`会返回1。
2. 页属于非线性磁盘文件的映射。内核通过Present标志为0且Dirty标志为1来识别。`pte_file`返回1。
3. 进程已经访问过这个页，但是内容被临时保存在磁盘上。内核能够识别这种情况，这是因为相应表项没被填充为0，但是Present和Dirty标志被清0。

这三种情况的鉴别在上面的`handle_pte_fault`中皆已看到。

对第二种和第三种来说，会牵扯到文件映射和页高速磁盘缓存的内容，这些又自成话题了，暂时不做探索。我们关心`do_no_page()->do_anonymous_page()`这一脉：

```c
/*
 * do_no_page() tries to create a new page mapping. It aggressively
 * tries to share with existing pages, but makes a separate copy if
 * the "write_access" parameter is true in order to avoid the next
 * page fault.
 *
 * As this is called only for pages that do not currently exist, we
 * do not need to flush old virtual caches or the TLB.
 *
 * This is called with the MM semaphore held and the page table
 * spinlock held. Exit with the spinlock released.
 */
/**
 * 当被访问的页不在主存中时，如果页从没有访问过，或者映射了磁盘文件
 * 那么pte_none宏会返回1，handle_pte_fault函数会调用本函数装入所缺的页。
 * 也就是还从来没有使用过这个页面
 * 执行对请求调页的所有类型都通用的操作。
 */
static int
do_no_page(struct mm_struct *mm, struct vm_area_struct *vma,
	unsigned long address, int write_access, pte_t *page_table, pmd_t *pmd)
{
	struct page * new_page;
	struct address_space *mapping = NULL;
	pte_t entry;
	unsigned int sequence = 0;
	int ret = VM_FAULT_MINOR;
	int anon = 0;

	/**
	 * vma->vm_ops || !vma->vm_ops->nopage,这是判断线性区是否映射了一个磁盘文件。
	 * 这两个值只要某一个为空，说明没有映射磁盘文件。也就是说：它是一个匿名映射。
	 * nopage指向装入页的函数。
	 * 当没有映射时，就调用do_anonymous_page获得一个新的页框。
	 */
	if (!vma->vm_ops || !vma->vm_ops->nopage)
		/**
		 * do_anonymous_page获得一个新的页框。分别处理写请求和读讨还。
		 */
		return do_anonymous_page(mm, vma, page_table,
					pmd, write_access, address); /*★*/
	/**
	 * 否则，就是一个文件映射。进行请求调页处理。
	 */
	pte_unmap(page_table);
	spin_unlock(&mm->page_table_lock);

	if (vma->vm_file) {
		mapping = vma->vm_file->f_mapping;
		sequence = mapping->truncate_count;
		smp_rmb(); /* serializes i_size against truncate_count */
	}
retry:
	cond_resched();
	/**
	 * 线性区定义了nopage方法，则回调此方法以返回所请求页的页框的地址。
	 * 调用filemap_nopage
	 */
	new_page = vma->vm_ops->nopage(vma, address & PAGE_MASK, &ret); /*★*/
	/*
	 * No smp_rmb is needed here as long as there's a full
	 * spin_lock/unlock sequence inside the ->nopage callback
	 * (for the pagecache lookup) that acts as an implicit
	 * smp_mb() and prevents the i_size read to happen
	 * after the next truncate_count read.
	 */

	/* no page was available -- either SIGBUS or OOM */
	if (new_page == NOPAGE_SIGBUS)
		return VM_FAULT_SIGBUS;
	if (new_page == NOPAGE_OOM)
		return VM_FAULT_OOM;

	/*
	 * Should we do an early C-O-W break?
	 */
	/**
	 * 进程试图对页进行写入，而该内存映射是私有的，需要取消内存映射。
	 */
	// TODO: 这是什么情况?
	if (write_access && !(vma->vm_flags & VM_SHARED)) {
		struct page *page;

		if (unlikely(anon_vma_prepare(vma)))
			goto oom;
		/**
		 * 分配一个新页。并将读取的页拷贝一份到新页中。。
		 */
		page = alloc_page_vma(GFP_HIGHUSER, vma, address); /*★*/
		if (!page)
			goto oom;
		copy_user_highpage(page, new_page, address); /*★*/
		page_cache_release(new_page); /*★*/
		/**
		 * 在后面的步骤中，使用新页而不是nopage方法返回的页，这样，后者就不会被用户态进程修改。
		 */
		new_page = page; /*★*/
		anon = 1;
	}

	spin_lock(&mm->page_table_lock);
	/*
	 * For a file-backed vma, someone could have truncated or otherwise
	 * invalidated this page.  If unmap_mapping_range got called,
	 * retry getting the page.
	 */
	/**
	 * 如果某个其他进程删改或者作废了该页(truncate_count用于此种检查),则跳转回去，尝试再次获得该页。
	 */
	if (mapping && unlikely(sequence != mapping->truncate_count)) {
		sequence = mapping->truncate_count;
		spin_unlock(&mm->page_table_lock);
		page_cache_release(new_page);
		goto retry;
	}
    /*返回address对应的页表项pte*/
	page_table = pte_offset_map(pmd, address);

	/*
	 * This silly early PAGE_DIRTY setting removes a race
	 * due to the bad i386 page protection. But it's valid
	 * for other architectures too.
	 *
	 * Note that if write_access is true, we either now have
	 * an exclusive copy of the page, or this is a shared mapping,
	 * so we can make it writable and dirty to avoid having to
	 * handle that later.
	 */
	/* Only go through if we didn't race with anybody else... */
	if (pte_none(*page_table)) {
		if (!PageReserved(new_page))
			++mm->rss;/* 增加进程的rss字段，以表示一个新页框已经分配给进程。 */
		acct_update_integrals();
		update_mem_hiwater();

		flush_icache_page(vma, new_page);
		/**
		 * 用新页框 的地址以及线性区的vm_page_prot字段中所包含的页访问权来设置缺页所在的地址对应的页表项。
		 */
		entry = mk_pte(new_page, vma->vm_page_prot);
		/**
		 * 如果进程试图对这个页进行写入，则把页表项的read/write和dirty设置为1.
		 */
		if (write_access)
			entry = maybe_mkwrite(pte_mkdirty(entry), vma);
        /*设定pte*/
		set_pte(page_table, entry); /*★*/
		if (anon) {
			lru_cache_add_active(new_page);
			page_add_anon_rmap(new_page, vma, address);
		} else
			page_add_file_rmap(new_page);
		pte_unmap(page_table);
	} else {
		/* One of our sibling threads was faster, back out. */
		pte_unmap(page_table);
		page_cache_release(new_page);
		spin_unlock(&mm->page_table_lock);
		goto out;
	}

	/* no need to invalidate: a not-present page shouldn't be cached */
	update_mmu_cache(vma, address, entry);
	spin_unlock(&mm->page_table_lock);
out:
	return ret;
oom:
	page_cache_release(new_page);
	ret = VM_FAULT_OOM;
	goto out;
}

/*
 * We are called with the MM semaphore and page_table_lock
 * spinlock held to protect against concurrent faults in
 * multithreaded programs. 
 */
/**
 * 获得一个新的页框。
 */
static int
do_anonymous_page(struct mm_struct *mm, struct vm_area_struct *vma,
		pte_t *page_table, pmd_t *pmd, int write_access,
		unsigned long addr)
{
	pte_t entry;
	struct page * page = ZERO_PAGE(addr); /*★*/

	/* Read-only mapping of ZERO_PAGE. */
	/**
	 * 对读访问时，页的内容是无关紧要的。
	 * 但是，第一次分给进程的页，最好还是填上0。以免旧页的信息被黑客利用。
	 * 没有必要立即分配这样的页框。只需要把empty_zero_page页映射给进程就行了。
	 * 并且将页标记为只读，当进程试图写这个页时，再启动写时复制机制。
	 */
	entry = pte_wrprotect(mk_pte(ZERO_PAGE(addr), vma->vm_page_prot)); /*★*/

	/* ..except if it's a write access */
	if (write_access) {
		/* Allocate our own private page. */
		/**
		 * 释放临时内核映射。
		 * 在调用handle_pte_fault函数之前，由pte_offset_map所建立页表项的高端内存物理地址。
		 * pte_offset_map宏是和pte_unmap配对使用的。
		 * pte_unmap必须在alloc_page前释放。因为alloc_page可能阻塞当前进程？？
		 * 答: 在alloc_zeroed_user_highpage时(可能睡眠)，其他线程可能更新的执行分配页面和设定页表，所指这里需要先unmap，防止阻塞其他线程
		 */
		pte_unmap(page_table);
		spin_unlock(&mm->page_table_lock);

		if (unlikely(anon_vma_prepare(vma)))
			goto no_mem;
		page = alloc_zeroed_user_highpage(vma, addr); /*★*/
		if (!page)
			goto no_mem;

		spin_lock(&mm->page_table_lock);
		page_table = pte_offset_map(pmd, addr);

		if (!pte_none(*page_table)) {
			pte_unmap(page_table);
			page_cache_release(page);
			spin_unlock(&mm->page_table_lock);
			goto out;
		}
		/**
		 * 递增rss字段，它记录了分配给进程的页框总数。
		 */
		mm->rss++;
		acct_update_integrals();
		update_mem_hiwater();
		/**
		 * 标记页框为既脏又可写。
		 * 如果调试程序试图往被跟踪进程只读线性区中的页中写数据。内核不会设置相关标志。
		 * maybe_mkwrite会处理这种特殊情况。
		 */
		entry = maybe_mkwrite(pte_mkdirty(mk_pte(page,
							 vma->vm_page_prot)),
				      vma);
		/**
		 * lru_cache_add_active把新页框插入与交换相关的数据结构中。
		 */
		lru_cache_add_active(page);
		SetPageReferenced(page);
		page_add_anon_rmap(page, vma, addr);
	}

    /*设定页表项*/
	set_pte(page_table, entry); /*★*/
	pte_unmap(page_table);

	/* No need to invalidate - it was non-present before */
	update_mmu_cache(vma, addr, entry);
	spin_unlock(&mm->page_table_lock);
out:
	return VM_FAULT_MINOR;
no_mem:
	return VM_FAULT_OOM;
}
```

### COW

傻老帽的fork方式：

- 为子进程的页表和页分配页框
- 初始化子进程页表
- 把父进程的页复制到子进程相应的页中

笨重至极！

现代Linux采用COW，父进程和子进程一开始共享页框不进行复制。一旦开始共享，那么二者都不能去修改，一旦父进程或子进程其中一个想要写共享页框，就产生异常。内核此时才会进行复制，独立出一个可写的页框给他。这也是`handle_pte_fault()`完成的（这函数非单一职责，当然复杂得很）。