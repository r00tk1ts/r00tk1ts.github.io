---
title: Linux内核学习——内存管理之非连续内存区管理
date: 2017-10-25 20:21:11
categories: Linux
tags:
	- linux-kernel
	- ulk
---
非连续内存区管理内容较少，也比较简单，无非就是连续的线性地址到非连续物理内存区的映射。与内核永久内存映射和内核临时内存映射并称三大物理地址与线性地址之间的映射机制。了解非连续内存区管理前，最好优先阅读页框管理和内存区管理。

<!--more-->
# Linux内存管理-非连续内存区管理
研究内存区管理时已经知道，内存区映射到一组连续的页框是非常好的做法，这可以充分利用高速缓存来提高效率。Linux还设计了一种机制来实现连续的线性地址映射到非连续的页框。这种方式避免了外碎片，但与此同时打乱了内核页表。Linux的非连续内存区必须是4K的倍数，IO驱动缓冲区，模块分配等都会用到非连续内存区。

## 线性地址
从PAGE_OFFSET也就是0xc0000000，第4GB的线性地址查找空闲区，这1GB的线性地址空间如图：
![](20171025_1.jpg)

1. 内存区开始部分包含的是对前896M的RAM进行映射的线性地址，这一部分直接映射的物理内存末尾所对应的线性地址在high_memory变量中。
2. 内存区结尾部分包含的是固定映射的线性地址。
3. PKMAP_BASE起始，查找用于高端内存页框的永久内核映射的线性地址。
4. 其余的线性地址可以用于非连续内存区。物理内存映射末尾到第一个内存区之间有个8M的安全区，为了捕获内存越界访问。每个内存区之间也有4K的安全区来隔离非连续的内存区。

对应的宏：
```cpp
/**
 * 第一个非连续映射的内存区与连续映射的线性区末尾之间的安全区大小。
 */
#define VMALLOC_OFFSET	(8*1024*1024)
/**
 * 为非连续内存区保留的线性地址的起始地址。
 */
#define VMALLOC_START	(((unsigned long) high_memory + vmalloc_earlyreserve + \
			2*VMALLOC_OFFSET-1) & ~(VMALLOC_OFFSET-1))
/**
 * 为非连续内存区保留的线性地址的结束地址。
 */
#ifdef CONFIG_HIGHMEM
# define VMALLOC_END	(PKMAP_BASE-2*PAGE_SIZE)
#else
# define VMALLOC_END	(FIXADDR_START-2*PAGE_SIZE)
#endif
```

## vm_struct
每个非连续内存区都对应一个vm_struct的描述符：
```cpp

/**
 * 非连续内存区的描述符
 */
struct vm_struct {
	/**
	 * 内存区内第一个内存单元的线性地址。
	 */
	void			*addr;
	/**
	 * 内存区大小加4096(内存区之间的安全区的大小)
	 */
	unsigned long		size;
	/**
	 * 非连续内存区映射的内存的类型。
	 * VM_ALLOC表示使用vmalloc得到的页.
	 * VM_MAP表示使用vmap映射的已经被分配的页。
	 * VM_IOREMAP表示使用ioremap映射的硬件设备的板上内存。
	 */
	unsigned long		flags;
	/**
	 * 指向nr_pages数组的指针，该数组由指向页描述符的指针组成。
	 */
	struct page		**pages;
	/**
	 * 内存区填充的页的个数。
	 */
	unsigned int		nr_pages;
	/**
	 * 一般为0,除非内存已经被创建来映射一个硬件设备IO共享内存。
	 */
	unsigned long		phys_addr;
	/**
	 * 指向下一个vm_struct结构的指针。
	 */
	struct vm_struct	*next;
};
```
描述符通过next形成链，链表第一个元素地址存放在vmlist中。
flags标志非连续区映射的内存类型，VM_ALLOC表示使用vmalloc()得到页，VM_MAP表示使用vmap()得到页，VM_IOREMAP表示ioremap()得到页。

## 查找空闲区域
通过`get_vm_area()`，用kmalloc分配内存区，在vmlist上找线性地址的空闲区域。
```cpp
/**
 * 在线性地址VMALLOC_START和VMALLOC_END之间查找一个空闲区域
 * size-将被创建的内存区的字节大小
 * flag-指定空闲区类型
 */
struct vm_struct *get_vm_area(unsigned long size, unsigned long flags)
{
	return __get_vm_area(size, flags, VMALLOC_START, VMALLOC_END);
}

struct vm_struct *__get_vm_area(unsigned long size, unsigned long flags,
				unsigned long start, unsigned long end)
{
	struct vm_struct **p, *tmp, *area;
	unsigned long align = 1;
	unsigned long addr;

	if (flags & VM_IOREMAP) {
		int bit = fls(size);

		if (bit > IOREMAP_MAX_ORDER)
			bit = IOREMAP_MAX_ORDER;
		else if (bit < PAGE_SHIFT)
			bit = PAGE_SHIFT;

		align = 1ul << bit;
	}
	addr = ALIGN(start, align);

	/**
	 * 调用kmalloc为vm_struct类型的新描述符获得一个内存区。
	 */
	area = kmalloc(sizeof(*area), GFP_KERNEL);
	if (unlikely(!area))
		return NULL;

	/*
	 * We always allocate a guard page.
	 */
	size += PAGE_SIZE;
	if (unlikely(!size)) {
		kfree (area);
		return NULL;
	}

	/**
	 * 为写获得vmlist_lock锁。
	 */
	write_lock(&vmlist_lock);
	/**
	 * 扫描vmlist链表，来查找线性地址的一个空闲区域。至少覆盖size+4096个地址(4096是安全区)
	 */
	for (p = &vmlist; (tmp = *p) != NULL ;p = &tmp->next) {
        /*
         * 起始地址addr落在了其他已经分配的vm_struct的VA地址空间中间
         */
		if ((unsigned long)tmp->addr < addr) {
            /*
             * addr 已经超过了前一个vm_struct所覆盖的真个VA地址空间，那么addr就更新到上一个vm_struct的VA地址空间最后面
             */
			if((unsigned long)tmp->addr + tmp->size >= addr)
				addr = ALIGN(tmp->size + 
					     (unsigned long)tmp->addr, align);
			continue;
		}

        /*
         * 到达这里的条件是:
         *  tmp->addr >= addr 
         * 也就是 addr 没有落在其他已经分配的vm_struct的VA地址空间中间，即下一个vm_struct地址空间之前
         */
		if ((size + addr) < addr)
			goto out;

        /*
         * addr+size 也没有落在其他已经分配的vm_struct的VA地址空间中间，即下一个vm_struct地址空间之前
         */
		if (size + addr <= (unsigned long)tmp->addr)
			goto found;

        /*
         * 到达这里是 addr+size 落在了下一个已经分配的vm_struct的VA地址空间中间
         * addr就更新到下一个vm_struct的VA地址空间最后面
         */
		addr = ALIGN(tmp->size + (unsigned long)tmp->addr, align);
		if (addr > end - size)
			goto out;
	}

found:
	/**
	 * 如果存在这样一个空闲区间，就初始化描述符的字段
	 */
	area->next = *p;
	*p = area;

	area->flags = flags;
	area->addr = (void *)addr;
	area->size = size;
	area->pages = NULL;
	area->nr_pages = 0;
	area->phys_addr = 0;
	/**
	 * 释放锁并返回内存区的起始地址。
	 */
	write_unlock(&vmlist_lock);

	return area;

out:
	/**
	 * 没有找到空闲区，就释放锁并释放先前得到的描述符，然后返回NULL。
	 */
	write_unlock(&vmlist_lock);
	kfree(area);
	if (printk_ratelimit())
		printk(KERN_WARNING "allocation failed: out of vmalloc space - use vmalloc=<size> to increase size.\n");
	return NULL;
}
```

## 分配非连续内存区
```cpp
/**
 * 给内核分配一个非连续内存区。
 * size-所请求分配的内存区的大小。
 */
void *vmalloc(unsigned long size)
{
       return __vmalloc(size, GFP_KERNEL | __GFP_HIGHMEM, PAGE_KERNEL);
}

void *__vmalloc(unsigned long size, int gfp_mask, pgprot_t prot)
{
	struct vm_struct *area;
	struct page **pages;
	unsigned int nr_pages, array_size, i;

	/**
	 * 首先将参数size设为4096的整数倍。
	 */
	size = PAGE_ALIGN(size);
	if (!size || (size >> PAGE_SHIFT) > num_physpages)
		return NULL;

	/**
	 * 通过调用get_vm_area来创建一个新的描述符。并返回分配给这个内存区的线性地址。
	 * 描述符的flags字段被初始化为VM_ALLOC，这意味着通过使用vmalloc函数，非连续页框将被映射到一个线性地址空间。
	 */
	area = get_vm_area(size, VM_ALLOC);
	if (!area)
		return NULL;

	nr_pages = size >> PAGE_SHIFT;
	array_size = (nr_pages * sizeof(struct page *));

	area->nr_pages = nr_pages;
	/* Please note that the recursion is strictly bounded. */
	/**
	 * 为页描述符指针数组分配页框。
	 */
	if (array_size > PAGE_SIZE)
		pages = __vmalloc(array_size, gfp_mask, PAGE_KERNEL);
	else
		pages = kmalloc(array_size, (gfp_mask & ~__GFP_HIGHMEM));
	area->pages = pages;
	if (!area->pages) {
		remove_vm_area(area->addr);
		kfree(area);
		return NULL;
	}
	memset(area->pages, 0, array_size);

	/**
	 * 重复调用alloc_page，为内存区分配nr_pages个页框。并把对应的页描述符放到area->pages中。
	 * 必须使用area->pages数组是因为:页框可能属于ZONE_HIGHMEM内存管理区，此时它们不一定映射到一个线性地址上。
	 */
	for (i = 0; i < area->nr_pages; i++) {
		area->pages[i] = alloc_page(gfp_mask);
		if (unlikely(!area->pages[i])) {
			/* Successfully allocated i pages, free them in __vunmap() */
			area->nr_pages = i;
			goto fail;
		}
	}

	/**
	 * 现在已经得到了一个连续的线性地址空间，并且分配了一组非连续的页框来映射这些地址。
	 * 需要修改内核页表项，将二者对应起来。这是map_vm_area的工作。
	 */
	if (map_vm_area(area, prot, &pages))
		goto fail;
	return area->addr;

fail:
	vfree(area->addr);
	return NULL;
}
```
最后的对应关系修正需要展开`map_vm_area`:
```cpp

/**
 * 将线性地址和页框对应起来
 * area-指向内存区的vm_struct描述符的指针
 * prot-已分配页框的保护位，它总是被置为0x63，对应着present,accessed,read/write及dirty.
 * pages-指向一个指针数组的变量的地址。该指针数组的指针指向页描述符。
 */
int map_vm_area(struct vm_struct *area, pgprot_t prot, struct page ***pages)
{
	/**
	 * 首先将内存区的开始和末尾的线性地址分配给局部变量address和end
	 */
	unsigned long address = (unsigned long) area->addr;
	unsigned long end = address + (area->size-PAGE_SIZE);
	unsigned long next;
	pgd_t *pgd;
	int err = 0;
	int i;

	/**
	 * 使用pgd_offset_k来获得主内核页全局目录中的目录项。该目录项对应于内存区起始线性地址。
	 */
	/*
	 * 更新根目录在swapper_pg_dir主内核页全局目录中的常规页表集合，
	 * 这个页全局目录由主内存描述符的pgd字段所指向，而主内存描述符存放于init_mm遍历
     */
	pgd = pgd_offset_k(address);
	/**
	 * 获得内核页表自旋锁。
	 */
	spin_lock(&init_mm.page_table_lock);
	/**
	 * 此循环为每个页框建立页表项。
	 */
	for (i = pgd_index(address); i <= pgd_index(end-1); i++) {
		/**
		 * 调用pud_alloc来为新内存区创建一个页上级目录。并把它的物理地址写入内核页全局目录的合适表项。
		 * 建立了PGD和PUD的联系(addr对应部分)
		 */
		pud_t *pud = pud_alloc(&init_mm, pgd, address);
		if (!pud) {
			err = -ENOMEM;
			break;
		}

        /*
         * next是本PUD项映射范围的结束地址
         */
		next = (address + PGDIR_SIZE) & PGDIR_MASK;
		if (next < address || next > end)
			next = end;
		/**
		 * map_area_pud函数为页上级目录所指向的所有页表建立对应关系。
		 * 建立PUD以下映射关系
		 */
		if (map_area_pud(pud, address, next, prot, pages)) {
			err = -ENOMEM;
			break;
		}

		address = next;
		pgd++;
	}

	spin_unlock(&init_mm.page_table_lock);
	flush_cache_vmap((unsigned long) area->addr, end);
	return err;
}
```
还有一个vmap()，它会映射非连续内存区中已经分配的页框：
```cpp
/**
 * 它将映射非连续内存区中已经分配的页框。本质上，该函数接收一组指向页描述符的指针作为参数，
 * 调用get_vm_area得到一个新的vm_struct描述符。然后调用map_vm_area来映射页框。因此该函数与vmalloc类似，但是不分配页框。
 * 即页框pages已经在函数调用者中分配好了，只是还没有映射VA而已，这里完成映射
 */
void *vmap(struct page **pages, unsigned int count,
		unsigned long flags, pgprot_t prot)
{
	struct vm_struct *area;

	if (count > num_physpages)
		return NULL;

	area = get_vm_area((count << PAGE_SHIFT), flags);
	if (!area)
		return NULL;
	if (map_vm_area(area, prot, &pages)) {
		vunmap(area->addr);
		return NULL;
	}

	return area->addr;
}
//显然map_vm_area是核心
/**
 * 将线性地址和页框对应起来
 * area-指向内存区的vm_struct描述符的指针
 * prot-已分配页框的保护位，它总是被置为0x63，对应着present,accessed,read/write及dirty.
 * pages-指向一个指针数组的变量的地址。该指针数组的指针指向页描述符。
 */
int map_vm_area(struct vm_struct *area, pgprot_t prot, struct page ***pages)
{
	/**
	 * 首先将内存区的开始和末尾的线性地址分配给局部变量address和end
	 */
	unsigned long address = (unsigned long) area->addr;
	unsigned long end = address + (area->size-PAGE_SIZE);
	unsigned long next;
	pgd_t *pgd;
	int err = 0;
	int i;

	/**
	 * 使用pgd_offset_k来获得主内核页全局目录中的目录项。该目录项对应于内存区起始线性地址。
	 */
	/*
	 * 更新根目录在swapper_pg_dir主内核页全局目录中的常规页表集合，
	 * 这个页全局目录由主内存描述符的pgd字段所指向，而主内存描述符存放于init_mm遍历
     */
	pgd = pgd_offset_k(address);
	/**
	 * 获得内核页表自旋锁。
	 */
	spin_lock(&init_mm.page_table_lock);
	/**
	 * 此循环为每个页框建立页表项。
	 */
	for (i = pgd_index(address); i <= pgd_index(end-1); i++) {
		/**
		 * 调用pud_alloc来为新内存区创建一个页上级目录。并把它的物理地址写入内核页全局目录的合适表项。
		 * 建立了PGD和PUD的联系(addr对应部分)
		 */
		pud_t *pud = pud_alloc(&init_mm, pgd, address);
		if (!pud) {
			err = -ENOMEM;
			break;
		}

        /*
         * next是本PUD项映射范围的结束地址
         */
		next = (address + PGDIR_SIZE) & PGDIR_MASK;
		if (next < address || next > end)
			next = end;
		/**
		 * map_area_pud函数为页上级目录所指向的所有页表建立对应关系。
		 * 建立PUD以下映射关系
		 */
		if (map_area_pud(pud, address, next, prot, pages)) {
			err = -ENOMEM;
			break;
		}

		address = next;
		pgd++;
	}

	spin_unlock(&init_mm.page_table_lock);
	flush_cache_vmap((unsigned long) area->addr, end);
	return err;
}
```

## 释放非连续内存区
```cpp
void vfree(void *addr)
{
	kfree(addr);
}

/**
 * 释放由kmalloc接口分配的内存
 */
void kfree (const void *objp)
{
	kmem_cache_t *c;
	unsigned long flags;

	if (!objp)
		return;
	local_irq_save(flags);
	kfree_debugcheck(objp);
	/**
	 * 通过((kmem_cache_t *)(pg)->lru.next)，可确定合适的高速缓存描述符。
	 */
	c = GET_PAGE_CACHE(virt_to_page(objp));
	/**
	 * __cache_free释放高速缓存中的对象。
	 */
	__cache_free(c, (void*)objp);
	local_irq_restore(flags);
}
```
vmap对应的是vunmap:
```cpp
/**
 * 释放vmap创建的内存区。
 */
void vunmap(void *addr)
{
	BUG_ON(in_interrupt());
	__vunmap(addr, 0);
}

/**
 * 被vfree或者vunmap调用，来释放非连续分配的内存区。
 * addr-要释放的内存区的起始地址。
 * deallocate_pages-如果被映射的页框需要释放到分区页框分配器，就置位(当vfree调用本函数时)。否则不置位(被vunmap调用时)
 */
void __vunmap(void *addr, int deallocate_pages)
{
	struct vm_struct *area;

	if (!addr)
		return;

	if ((PAGE_SIZE-1) & (unsigned long)addr) {
		printk(KERN_ERR "Trying to vfree() bad address (%p)\n", addr);
		WARN_ON(1);
		return;
	}

	/**
	 * 调用remove_vm_area得到vm_struct描述符的地址。
	 * 并清除非连续内存区中的线性地址对应的内核的页表项。
	 */
	area = remove_vm_area(addr);
	if (unlikely(!area)) {
		printk(KERN_ERR "Trying to vfree() nonexistent vm area (%p)\n",
				addr);
		WARN_ON(1);
		return;
	}

	/**
	 * 如果deallocate_pages被置位，扫描指向页描述符的area->nr_pages
	 */
	if (deallocate_pages) {
		int i;

		for (i = 0; i < area->nr_pages; i++) {
			/**
			 * 对每一个数组元素，调用__free_page函数释放页框到分区页框分配器。
			 */
			if (unlikely(!area->pages[i]))
				BUG();
			__free_page(area->pages[i]);
		}

		/**
		 * 释放area->pages数组本身。
		 */
		if (area->nr_pages > PAGE_SIZE/sizeof(struct page *))
			vfree(area->pages);
		else
			kfree(area->pages);
	}

	/**
	 * 释放vm_struct描述符。
	 */
	kfree(area);
	return;
}
```

释放永远要比分配简单。