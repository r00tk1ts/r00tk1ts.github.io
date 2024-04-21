---
title: Linux内核学习——Linux进程切换
date: 2017-08-26 10:20:11
categories: Linux
tags:
	- linux-kernel
	- ulk
---
Linux内核的进程切换可谓精彩绝伦，遗憾的是，ULK对此的介绍不够详尽，而毛批中也是一带而过，所以一时云里雾里。本文是我参考了大量资料，反复研磨后对Linux内核中进程切换的一点理解。

<!--more-->

# Linux内核学习——Linux进程切换
对于Linux这种分时操作系统来说，内核必须有能力挂起CPU正在运行的进程，以复杂的策略调度另外一个进程。这就是进程切换。在Linux中，这也常被称为任务切换或上下文切换。

> Linux的进程和任务是一回事，按我的理解，一般内核进程叫任务，用户进程叫进程。

每个进程有自己的地址空间，相互隔离，而CPU是共享的。于是，进程切换时，CPU寄存器的L/S(Load and Save)大法需要内核来谨慎的完成。进一步说，折腾寄存器的这组数据叫硬件上下文，它是进程可执行上下文的一个子集。硬件上下文的一部分放在TSS段，剩余部分放在内核态堆栈。

<font color="red">进程切换只发生在内核态!</font>

## 任务状态段
TSS段在我另一篇文章中简单提到过(Linux内存寻址)，它用来存放硬件上下文。尽管Linux根本不使用硬件上下文切换，但还是强制它为系统中每个不同CPU创建一个TSS，原因有二：
- x86的一个CPU从用户态切换到内核态时，它就从TSS中获取内核态堆栈的地址(中断和异常中的sysenter->系统调用)。
- 用户态进程试图通过in或out指令访问一个I/O端口时，CPU需要访问存在TSS的I/O许可权位图以检查该进程是否有访问端口的权利。

TSS的结构定义成tss_struct：

```cpp
/**
 * 描述TSS的格式
 */
struct tss_struct {
	unsigned short	back_link,__blh;
	unsigned long	esp0;
	unsigned short	ss0,__ss0h;
	unsigned long	esp1;
	unsigned short	ss1,__ss1h;	/* ss1 is used to cache MSR_IA32_SYSENTER_CS */
	unsigned long	esp2;
	unsigned short	ss2,__ss2h;
	unsigned long	__cr3;
	unsigned long	eip;
	unsigned long	eflags;
	unsigned long	eax,ecx,edx,ebx;
	unsigned long	esp;
	unsigned long	ebp;
	unsigned long	esi;
	unsigned long	edi;
	unsigned short	es, __esh;
	unsigned short	cs, __csh;
	unsigned short	ss, __ssh;
	unsigned short	ds, __dsh;
	unsigned short	fs, __fsh;
	unsigned short	gs, __gsh;
	unsigned short	ldt, __ldth;
	unsigned short	trace, io_bitmap_base;
	/*
	 * The extra 1 is there because the CPU will access an
	 * additional byte beyond the end of the IO permission
	 * bitmap. The extra byte must be all 1 bits, and must
	 * be within the limit.
	 */
	unsigned long	io_bitmap[IO_BITMAP_LONGS + 1];
	/*
	 * Cache the current maximum and the last task that used the bitmap:
	 */
	unsigned long io_bitmap_max;
	struct thread_struct *io_bitmap_owner;
	/*
	 * pads the TSS to be cacheline-aligned (size is 0x100)
	 */
	unsigned long __cacheline_filler[35];
	/*
	 * .. and then another 0x100 bytes for emergency kernel stack
	 */
	unsigned long stack[64];
} __attribute__((packed));
```

每次进程切换时，内核更新TSS的某些字段以便相应CPU控制单元可以安全检索到它需要的信息。因此，TSS反映了CPU上当前进程的特权级，但对于没有运行的进程来说，并未保留TSS（TSS跟随CPU而不是进程，这和intel的初衷不同）。

每个TSS有自己的8字节任务状态段描述符(TSSD)，放在GDT中，CPU的tr寄存器包含相应TSS的TSSD，同时包含两个隐藏的非编程字段：TSSD的Base和Limit域。关于这些的设计，可以参考我之前写的“Linux内存寻址”。

所以，Linux的TSS跟着CPU走，那么进程切换时，进程的硬件上下文就必须保存在别处。于是，Linux的每个进程描述符包含一个类型为thread_struct的thread字段，只要进程被换出，就把硬件上下文存于此。

```cpp
/**
 * 进程被切换出去后，内核把它的硬件上下文保存在这个结构中。
 * 它包含大部分CPU寄存器，但是不包含eax、ebx这样的通用寄存器,他们的值保留在内核堆栈中
 */
struct thread_struct {
/* cached TLS descriptors. */
	struct desc_struct tls_array[GDT_ENTRY_TLS_ENTRIES];
	unsigned long	esp0;
	unsigned long	sysenter_cs;
	unsigned long	eip;
	unsigned long	esp;
	unsigned long	fs;
	unsigned long	gs;
/* Hardware debugging registers */
	unsigned long	debugreg[8];  /* %%db0-7 debug registers */
/* fault info */
	unsigned long	cr2, trap_no, error_code;
/* floating point info */
	/**
	 * 为支持选择性装入FPU、MMX和XMM寄存器，引入此结构。
	 * 当切换进程时，将进程的这些寄存器保存在i387结构中。
	 */
	union i387_union	i387;
/* virtual 86 mode info */
	struct vm86_struct __user * vm86_info;
	unsigned long		screen_bitmap;
	unsigned long		v86flags, v86mask, saved_esp0;
	unsigned int		saved_fs, saved_gs;
/* IO permissions */
	unsigned long	*io_bitmap_ptr;
/* max allowed port in the bitmap, in bytes: */
	unsigned long	io_bitmap_max;
};

#define INIT_THREAD  {							\
	.vm86_info = NULL,						\
	.sysenter_cs = __KERNEL_CS,					\
	.io_bitmap_ptr = NULL,						\
}
```

可以看到诸如eip，esp，fs，gs等寄存器放在了这里，但是一些通用的eax等寄存器却不在这里（他们在内核堆栈）。

## 进程切换的入口
进程切换只可能发生在schedule()函数中，这个函数涉及了复杂的调度策略，在选出一个合适的待换入进程后，执行进程切换。
本节，我们只关心进程切换。

进程切换本质上由两步组成：切换页全局目录以安装一个新的地址空间；切换内核态堆栈和硬件上下文。

忽略调度策略，schedule()对进程切换的处理：

```cpp
switch_tasks:
	/**
	 * 运行到这里，开始进行进程切换了。
	 */
	if (next == rq->idle)
		schedstat_inc(rq, sched_goidle);
	/**
	 * prefetch提示CPU控制单元把next的进程描述符的第一部分字段的内容装入硬件高速缓存。
	 * 这改善了schedule的性能。
	 */
	prefetch(next);
	/**
	 * 清除TIF_NEED_RESCHED标志。
	 */
	clear_tsk_need_resched(prev);
	/**
	 * 记录CPU正在经历静止状态。主要与RCU相关。
	 */
	rcu_qsctr_inc(task_cpu(prev));

	/**
	 * 减少prev的平均睡眠时间
	 */
	prev->sleep_avg -= run_time;
	if ((long)prev->sleep_avg <= 0)
		prev->sleep_avg = 0;
	/**
	 * 更新进程的时间戳
	 */
	prev->timestamp = prev->last_ran = now;

	sched_info_switch(prev, next);
	if (likely(prev != next)) {/* prev和next不同，需要切换 */
		next->timestamp = now;
		rq->nr_switches++;
		rq->curr = next;
		++*switch_count;

		prepare_arch_switch(rq, next);
		/**
		 * context_switch执行真正的进程切换
		 */
		prev = context_switch(rq, prev, next);

		/**
		 * 当进程再次被切换进来后，以下代码被接着运行。
		 * 但是此时prev并不指向当前进程，而是指代码从哪一个进程切换到本进程。
		 * 由于此时已经进行了进程空间的切换，寄存器中缓存的变量等都不再有效，所以用barrier产生一个优化屏障。
		 */
		barrier();

		/**
		 * 对前一个进程进行一些收尾工作，比如减少它的mm_struct,task_struct的引用计数等。
		 */
		finish_task_switch(prev);
	} else/* 如果prev和next是同一个进程，就不做进程切换。当prev仍然是当前活动集合中的最高优先级进程时，这是有可能发生的。 */
		spin_unlock_irq(&rq->lock);

	/**
	 * 在前几句中(context_switch之后)，prev代表的是从哪个进程切换到本进程。
	 * 在继续进行调度之前(因此在context_switch中开了中断，可能刚切回本进程就来了中断，并需要重新调度)，将prev设置成当前进程。
	 */
	prev = current;
	/**
	 * 重新获得大内核锁。
	 */
	if (unlikely(reacquire_kernel_lock(prev) < 0))
		goto need_resched_nonpreemptible;
	/**
	 * 打开抢占，并检查是否需要重新调度。
	 */
	preempt_enable_no_resched();

	/*
	 * 同时检查其他进程是否设置了当前进程的TIF_NEED_RESCHED标志，
	 * 如果是，则整个schedule()函数重新开始执行，否则函数结束
	 */
	if (unlikely(test_thread_flag(TIF_NEED_RESCHED)))
		goto need_resched;
```

此时prev和next指向换出和换入的进程，进入context_switch函数：

```cpp
/*
 * context_switch - switch to the new MM and the new
 * thread's register state.
 */
/**
 * 建立next的地址空间
 */
static inline
task_t * context_switch(runqueue_t *rq, task_t *prev, task_t *next)
{
	struct mm_struct *mm = next->mm;
	struct mm_struct *oldmm = prev->active_mm;

	/**
	 * 如果是切换到一个内核线程，新进程就使用pre的地址空间，避免了TLB的切换
	 */
	if (unlikely(!mm)) {
		next->active_mm = oldmm;
		atomic_inc(&oldmm->mm_count);
		/**
		 * 作为更进一步的优化措施，如果新进程是内核线程，就将进程设置为懒惰TLB模式
		 * xie.baoyou注：请想一下，如果内核线程切换出去后，可能又会回到上一个进程，此时就根本不需要切换地址空间。
		 * 皆大欢喜，大家都省事了，这叫“懒人有懒福”
		 */
		enter_lazy_tlb(oldmm, next);
	} else/* 否则就需要切换内地址空间。 */
		switch_mm(oldmm, mm, next);

	/**
	 * 如果上一个线程是内核线程或正在退出的进程，就把prev内存描述符的指针保存到运行队列的prev_mm中。
	 * 并清空rq->prev_mm
	 */
	if (unlikely(!prev->mm)) {
		prev->active_mm = NULL;
		WARN_ON(rq->prev_mm);
		rq->prev_mm = oldmm;
	}

	/* Here we just switch the register state and the stack. */
	/**
	 * 终于可以真正的切换了。
	 */	
	switch_to(prev, next, prev);

	return prev;
}
```

## 晦涩的switch_to
这个switch_to是个宏，它很有意思，因为除了prev和next外，还有个last参数：
```cpp
/**
 * 进程切换时，切换内核态堆栈和硬件上下文。
 * prev-被替换的进程
 * next-新进程
 * last-在任何进程切换中，到三个进程而不是两个。假设内核决定暂停A而激活B，那么在schedule函数中，prev指向A而next指向B。
 *      当切换回A后，就必须暂停另外一个进程C。而LAST则指向C进程。
 */
#define switch_to(prev,next,last) do {					\
	unsigned long esi,edi;						\
	/**
	 * 在真正执行汇编代码前，已经将prev存入eax，next存入edx中了。
	 */

				/**
				 * 保存eflags和ebp到内核栈中。必须保存是因为编译器认为在switch_to结束前，
				 * 它们的值应当保持不变。
				 */
	asm volatile("pushfl\n\t"					\
		     "pushl %%ebp\n\t"					\
		     /**
		      * 把esp的内容保存到prev->thread.esp中
		      * 这样该字段指向prev内核栈的栈顶。
		      */
		     "movl %%esp,%0\n\t"	/* save ESP */		\
		     /**
		      * 将next->thread.esp装入到esp.
		      * 此时，内核开始在next的栈上进行操作。这条指令实际上完成了从prev到next的切换。
		      * 由于进程描述符的地址和内核栈的地址紧挨着，所以改变内核栈意味着改变当前进程。
		      */
		     "movl %5,%%esp\n\t"	/* restore ESP */	\
		     /**
		      * 将标记为1f的地址存入prev->thread.eip.
		      * 当被替换的进程重新恢复执行时，进程执行被标记为1f的那条指令。
		      */
		     "movl $1f,%1\n\t"		/* save EIP */		\
		     /**
		      * 将next->thread.eip的值保存到next的内核栈中。
		      * 这样，_switch_to调用ret返回时，就会跳转到next->thread.eip执行。
		      * 这个地址一般情况下就会是1f.
		      */
		     "pushl %6\n\t"		/* restore EIP */	\
		     /**
		      * 注意，这里不是用call，是jmp，这样，上一条语句中压入的eip地址就可以执行了。
		      */
		     "jmp __switch_to\n"				\
		     /**
		      * 到这里，进程A再次获得CPU。它从栈中弹出ebp和eflags。
		      */
		     "1:\t"						\
		     "popl %%ebp\n\t"					\
		     "popfl"						\
		     :"=m" (prev->thread.esp),"=m" (prev->thread.eip),	\
		     /* last被作为输出参数，它的值会由eax赋给它。 */
		      "=a" (last),"=S" (esi),"=D" (edi)			\
		     :"m" (next->thread.esp),"m" (next->thread.eip),	\
		      "2" (prev), "d" (next));				\
} while (0)
```

这个宏是用AT&T汇编语法写的，不是很好理解，虽然上述代码根据ULK的描述，已一一按桩插入，但我们还是展开成最终反汇编码说明。

> AT&T汇编语法的基础知识，可以参考毛批的第一章。

1. eax和edx分别保存prev和next:
```asm
movl prev, %eax
movl next, %edx
```
2. eflags和ebp保存到prev的内核堆栈：
```asm
pushfl
pushl %ebp
```
3. 把esp的内容保存到prev->thread.esp中使得该字段指向prev内核堆栈栈顶：
```asm
movl %esp, 484(%eax)
```
484(%eax)表示内存单元地址为eax+484，也就是prev->thread.esp。
4. 把next->thread.esp装入esp。esp指向了next的内核栈，就代表此时内核切换到了next的内核栈。
```asm
movl 484(%edx), %esp
```
5. 把标记为1的地址放入prev->thread.eip，这表示替换的进程再次恢复执行时，restore出eip，从1处继续执行：
```asm
movl $1f, 480(%eax)
```
480(%eax)就是prev->thread.eip
6. 对称的，把next->thread.eip的值（大概率就是上一次压入的1的地址，如果它未被换出过，那情况要复杂一些）压入next内核栈：
```asm
pushl 480(%edx)
```
7. 跳到__switch_to() C函数：
```asm
jmp __switch_to
```
8. 从__switch_to回来后，实际上已经变成了next进程，而__switch_to实际上内部是通过ret返回的，那么返回地址也就刚好是push的next->thread.eip。玩安全的想必非常熟悉，这是经典的push+jmp+ret伪造call指令。于是，next执行从1标签开始的指令，进行restore，而此时因为堆栈早已换成next自己的，所以pop出来的也就是上一次被换出时保存的ebp和eflags。
```asm
1:
popl %ebp
popfl
```
9. 拷贝eax的内容到switch_to的第三个参数last标记的内存区域中：
```asm
movl %eax, last
```
prev和next是进程堆栈中的两个局部变量，那么对于两个进程来说，我们把prev标志的称为A，next标志的称为B。A的进程堆栈中有着prev和next，在切换前夕，prev是A，next是B。当跳到__switch_to后，ret回来，此时就是进程B的堆栈，而B的堆栈中同样有上一次的prev和next，B的prev应该是B本身，而B的next应该是上一次B切换到C时的C。而我们设计上，希望从A切换到B时，不仅能够保证A的prev和next正确，还要保证切换到B后，B的prev也是正确的。
如果只是单纯的A->B->A在不会有问题，他们的prev和next都是正确的，但是如果A->B，C->A就会有问题:（注意，这里换入的进程的prev还没有被eax重置，这一忽略曾让我怀疑人生）

![](20170826_1.jpg)

由于A此前在切换到B时，堆栈上prev和next保存的分别是A和B，那么如果我们从C到A时，A的prev依然是A，这就有问题了。A的prev明明应该是C才对（这就是前面说的我们不仅要保证换出的C的prev和next正确，换入的A的prev也应该正确，至于A的next，who care?等它再次切换时自己就处理好了）。本质上来讲，这其实就是堆栈切换后，局部变量因为他的含义需要相应调整的故事。
所以switch_to宏多引入了一个last变量，last实际上就是prev，__switch_to返回值eax实际上就是A的prev，因为执行__switch_to时prev变量指向的还是A的prev，而ret回去后，prev变量就是B的堆栈上的prev。因此，借由__switch_to前传递A的prev，ret后再返回这个A的prev值给B，用于覆盖B的prev，那么B的prev由谁覆盖？答案就是last，eax值返回给last，用last覆盖B的prev，从而保持了prev的字面意义。

如果说prev和next是变化的状态量，随着进程切换而改变意义，那么这个一开始和prev一致的last就是个过程量，用于切换后以A的prev正确的重置B的prev。prev和next是输入型变量，last是输出型变量，用于指明切换后被覆盖的地址。
```asm
movl %eax, last
```

这一过程非常的复杂，我自己也是研究了好久才弄明白，但是当我捋清了整个流程，便惊叹于内核设计的绝伦精妙。

我在网上淘金时，挖到了描述这一过程不错的图例：

![](20170826_2.jpg)
![](20170826_3.jpg)

## __switch_to函数
```cpp
/**
 * __switch_to函数执行大多数进程切换的工作。
 * 进程切换的工作开始于switch_to宏，但是它的主要工作还是由__switch_to完成。
 * 这个函数是寄存器传参的函数。在switch_to宏中，参数已经保存在eax和edx中了.
 */
struct task_struct fastcall * __switch_to(struct task_struct *prev_p, struct task_struct *next_p)
{
	struct thread_struct *prev = &prev_p->thread,
				 *next = &next_p->thread;
	/**
	 * 通过读取current_thread_info()->cpu,获得当前进程在哪个CPU上运行.
	 * 因为在schedule函数中已经调用了禁用抢占,所以这里可以直接使用smp_processor_id()
	 */
	int cpu = smp_processor_id();
	struct tss_struct *tss = &per_cpu(init_tss, cpu);

	/* never put a printk in __switch_to... printk() calls wake_up*() indirectly */
	/**
	 * __unlazy_fpu宏有选择的保存FPU\MMX\XMM寄存器的内容.
	 * 它可能会延后保存这些寄存器的内容.
	 */
	__unlazy_fpu(prev_p);

	/*
	 * Reload esp0, LDT and the page table pointer:
	 */
	/**
	 * 把next_p->thread.esp0装入本地CPU的TSS的esp0字段.
	 * 任何由sysenter汇编指令产生的从用户态到内核态的特权级转换将把这个地址复制到esp寄存器.
	 */
	load_esp0(tss, next);

	/*
	 * Load the per-thread Thread-Local Storage descriptor.
	 */
	/**
	 * 将next_p进程使用的线程局部存储(TLS)段装入本地CPU的全局描述符表.
	 */
	load_TLS(next, cpu);

	/*
	 * Save away %fs and %gs. No need to save %es and %ds, as
	 * those are always kernel segments while inside the kernel.
	 */
	/**
	 * 把fs和gs段寄存器的内容分别存放在prev_p->thread.fs和prev_p->thread.gs中.
	 */
	asm volatile("movl %%fs,%0":"=m" (*(int *)&prev->fs));
	asm volatile("movl %%gs,%0":"=m" (*(int *)&prev->gs));

	/*
	 * Restore %fs and %gs if needed.
	 */
	/**
	 * 不管是prev还是next,只要他们使用了fs和gs,那么,都需要将next中的fs,gs更新到段寄存器.
	 * 即使next并不使用fs,但是只要prev使用了,也需要更新.这样可以防止next通过fs,gs访问prev的数据.
	 */
	if (unlikely(prev->fs | prev->gs | next->fs | next->gs)) {
		/**
		 * loadsegment可能会装载一个无效的段寄存器.CPU可能会产生一个异常.
		 * 但是loadsegment会采用代码修正技术来处理这种情况.
		 */
		loadsegment(fs, next->fs);
		loadsegment(gs, next->gs);
	}

	/*
	 * Now maybe reload the debug registers
	 */
	/**
	 * 用debugreg数组的内容dr0..dr7中的6个调试寄存器.这允许定义四个断点区域.
	 */
	if (unlikely(next->debugreg[7])) {
		loaddebug(next, 0);
		loaddebug(next, 1);
		loaddebug(next, 2);
		loaddebug(next, 3);
		/* no 4 and 5 */
		loaddebug(next, 6);
		loaddebug(next, 7);
	}

	/**
	 * 如果必要,更新TSS中的IO位图.当next或者prev有其自己的定制IO权限位图时必须这么做.
	 */
	if (unlikely(prev->io_bitmap_ptr || next->io_bitmap_ptr))
		/**
		 * handle_io_bitmap并不立即更新位图,而是采用一种懒模式的方法.
		 */
		handle_io_bitmap(next, tss);

	/**
	 * return产生的汇编指令是movl %edi, %eax,ret.
	 * 这里有保护eax和返回地址的问题.请仔细理解.
	 * 除了需要理解switch_to宏中的jmp指令外,对于没有产生切换,而是第一次开始执行的进程.
	 * 它并不会跳回switch_to,而是找到ret_from_fork函数的超始地址.
	 */
	return prev_p;
}
```
可以看到，进程切换中硬件上下文的实质性工作都是在这里处理的，而__switch_to返回值就是传入的prev_p，也就是switch_to中的prev局部变量，这个变量在执行jmp __switch_to之前代表着A的prev。

值得一提的是，__switch_to的函数参数不是通过栈传递的，而是指定了eax和edx寄存器，__switch_to什么都没有做，只是在对应Makefile中写上了`KBUILD_CFLAGS += -msoft-float -mregparm=3 -freg-struct-return`，-mregparm=3表示默认使用3个寄存器传参。

<font color="red">一言以蔽之，进程切换的关键操作无非就是切换地址空间、切换内核堆栈、切换内核控制流程以及必要寄存器的现场保护与还原。</font>

## 参考文献
- [x86体系结构下Linux-2.6.26的进程调度和切换](http://home.ustc.edu.cn/~hchunhui/linux_sched.html)
- [linux 进程调度switch_to宏浅析+系统执行过程总结](http://blog.csdn.net/titer1/article/details/45289159)
- [【内核】进程切换 switch_to 与 __switch_to](http://www.cnblogs.com/visayafan/archive/2011/12/10/2283660.html)
- 《深入理解Linux内核》
- 《Linux内核源代码情景分析(上)》
