---
title: Linux内核学习——Linux进程概述
date: 2017-08-22 21:51:11
categories: Linux
tags:
	- linux-kernel
	- ulk
---
近年来陆陆续续对Linux内核各模块做些研究，本文是对Linux内核中进程基本概念的一点研究。主要参考ULK以及毛批，结合自己的一些深入的理解，以及从源代码中翻箱倒柜获取的心得。

<!--more-->
# Linux内核学习——Linux进程概述
## 活在OS理论中的进程与线程
进程与线程是OS理论中的基本概念。在理论中，进程通常被定义成一个正在运行的程序的实例。举个例子，你在Windows系统上运行一个notepad.exe，这个notepad.exe就会被拉到RAM中映射成一个应用程序，这个应用程序就是进程。同理，在Linux下，使用vim编辑某个文件，vim也被拉到了RAM中映射出一个进程。

进程由两部分组成：

- OS管理进程的内核对象。
- 自己的地址空间

线程和进程很相似，同样由两部分组成：

- 一个是线程的内核对象
- 自己专有的资源（如线程堆栈地址空间）

## Linux的进程、轻量级进程与线程
然而，理论毕竟是理论，各种OS真正在实现时，其抽象体与具象体的设计往往并不一致。

拿我比较熟悉的32位 Windows来说，Windows中进程是一个相对抽象的概念，进程本身是不干活的，进程包含了一个(主线程)或多个线程，这些线程实际上才是干活的主体对象。一个进程只是定义了内核对象（用于管理，状态标志等）和一片自己的地址空间(每个程序都告诉自己有4G的线性地址，其中高2G内核地址共享)。每个线程也有他自己的内核对象，以及它所属进程的地址空间中的一部分（主要是线程堆栈）。用Windows话来说，进程是不活泼的，从不执行任何东西，只是线程的容器。线程是活泼的，同属于一个进程的所有线程共享这片地址空间，但每个线程又有它自己的一亩三分地，这就使得同组线程的交互极为简单，而不同组线程又被相对隔离。

而Linux则大相径庭，Linux中进程和线程的概念有那么一点混淆不清，不似Windows般泾渭分明。纵观Linux源代码，各路Author常把进程称为task或thread（实际上叫Process更合适），各种结构的命名上可见一斑。

但无论你如何命名，如何设计，从内核观点来看，进程的目的就在于担当分配系统资源（CPU分片、地址空间）的实体。Linux中除了第一个pid为0的进程，所有的进程都是复制出来的。父子进程虽然可以共享程序代码页，但有自己独立的数据拷贝。子进程自身内存单元的修改对父进程是不可见的，反之亦然。

> 这和Windows的进程设计完全不同，Windows是可以由OS通过CreateProcess捏出一个无父无母的进程，Linux的每个进程却都是克隆出来的，Linux从未提供一个类似CreateProcess的方法来凭空捏一个进程，唯有通过fork(),clone()或vfork()复制。

Linux内核没有线程，POSIX兼容的pthread库在早期的Linux上，其多个执行流的创建、处理、调度整个都是在用户态进行的。因此，pthread引入的所谓多线程应用程序从内核角度来看，只是一个普通进程。

然而，Linux还是拗不过潮流，这种老式的设计在很多情况下的表现不尽人意。ULK的例子尤为生动，假设一个象棋程序使用两个线程：线程A负责GUI，等待人类棋手移动并显示计算机的移动；线程B思考下一手棋。尽管A等待人类选手时，B应当继续运行，把时间利用起来。但是，如果象棋程序是一个单独进程，A就不能简单的发出等待用户行为的阻塞系统调用，否则B也会阻塞。相反，A需要利用复杂的非阻塞技术确保进程仍然是可运行的（轮询，自旋）。

现代Linux对此妥协，对比Windows的设计来说，前者引入了一个不伦不类的概念：轻量级进程(lightweight process)。用于对多线程提供更好的支持。两个轻量级进程基本上可以共享一些资源，比如地址空间、打开的文件等。只要其中一个修改共享资源，那么另一个就立即查看这种修改。当然，两个线程访问共享资源时需要同步处理。

于是处理多线程时，可以把轻量级进程和每个线程关联起来。线程之间可以简单地共享同一内存地址空间、打开文件集等。每个线程也就由内核独立调度，不再出现一个发出阻塞系统调用，其他宕机的情况。

归根结底，Linux的基本单元是进程。

## 进程描述符
有进程就得有管理的媒介。进程描述符用于提供内核进程的优先级、运行状态、地址空间等信息。Linux的进程描述符是`task_struct`结构，该字段极为庞大复杂，包含了进程相关的所有信息。

![](20170822_1.jpg)

```cpp
struct task_struct {
	/**
	 * 进程状态。
	 */
	volatile long state;	/* -1 unrunnable, 0 runnable, >0 stopped */
	/**
	 * 进程的基本信息。
	 */
	struct thread_info *thread_info;
	atomic_t usage;
	unsigned long flags;	/* per process flags, defined below */
	unsigned long ptrace;

	int lock_depth;		/* Lock depth */

	/**
	 * 进行的动态优先权和静态优先权
	 */
	int prio, static_prio;
	/**
	 * 进程所在运行队列。每个优先级对应一个运行队列。
	 */
	struct list_head run_list;
	/**
	 * 指向当前运行队列的prio_array_t
	 */
	prio_array_t *array;

	/**
	 * 进程的平均睡眠时间
	 */
	unsigned long sleep_avg;
	/**
	 * timestamp-进程最近插入运行队列的时间。或涉及本进程的最近一次进程切换的时间
	 * last_ran-最近一次替换本进程的进程切换时间。
	 */
	unsigned long long timestamp, last_ran;
	/**
	 * 进程被唤醒时所使用的代码。
	 *     0:进程处于TASK_RUNNING状态。
	 *     1:进程处于TASK_INTERRUPTIBLE或者TASK_STOPPED状态，而且正在被系统调用服务例程或内核线程唤醒。
	 *     2:进程处于TASK_INTERRUPTIBLE或者TASK_STOPPED状态，而且正在被ISR或者可延迟函数唤醒。
	 *     -1:表示从UNINTERRUPTIBLE状态被唤醒
	 */
	int activated;

	/**
	 * 进程的调度类型:sched_normal,sched_rr或者sched_fifo
	 */
	unsigned long policy;
	/**
	 * 能执行进程的CPU的位掩码
	 */
	cpumask_t cpus_allowed;
	/**
	 * time_slice-在进程的时间片中，还剩余的时钟节拍数。
	 * first_time_slice-如果进程肯定不会用完其时间片，就把该标志设置为1.
	 *            xie.baoyou注:原文如此,应该是表示任务是否是第一次执行。这样，如果是第一次执行，并且在开始运行
	 *                         的第一个时间片内就运行完毕，那么就将剩余的时间片还给父进程。主要是考虑到有进程
	 *                         会大量的动态创建子进程时，而子进程会立即退出这种情况。如果不还给父进程时间片，会对这种进程不公平。
	 */
	unsigned int time_slice, first_time_slice;

#ifdef CONFIG_SCHEDSTATS
	struct sched_info sched_info;
#endif

	/**
	 * 通过此链表把所有进程链接到一个双向链表中。
	 */
	struct list_head tasks;
	/*
	 * ptrace_list/ptrace_children forms the list of my children
	 * that were stolen by a ptracer.
	 */
	/**
	 * 链表的头。该链表包含所有被debugger程序跟踪的P的子进程。
	 */
	struct list_head ptrace_children;
	/**
	 * 指向所跟踪进程其实际父进程链表的前一个下一个元素。
	 */
	struct list_head ptrace_list;

	/**
	 * mm:指向内存区描述符的指针
	 * mm字段指向进程所拥有的内存描述符，而active_mm字段指向进程运行时所使用的内存描述符
	 * 对于普通进程而已，这两个字段存放相同的指针，但是，内核线程不拥有任何内存描述符，因此，他们的mm字段总是为NULL
	 * 当内核线程得以运行时，他的active_mm字段被初始化为前一个运行进程的active_mm值
	 */
	struct mm_struct *mm, *active_mm;

/* task state */
	struct linux_binfmt *binfmt;
	long exit_state;
	int exit_code, exit_signal;
	int pdeath_signal;  /*  The signal sent when the parent dies  */
	/* ??? */
	unsigned long personality;
	/**
	 * 进程发出execve系统调用的次数。
	 */
	unsigned did_exec:1;
	/**
	 * 进程PID
	 */
	pid_t pid;
	/**
	 * 线程组领头线程的PID。
	 */
	pid_t tgid;
	/* 
	 * pointers to (original) parent process, youngest child, younger sibling,
	 * older sibling, respectively.  (p->father can be replaced with 
	 * p->parent->pid)
	 */
	/**
	 * 指向创建进程的进程的描述符。
	 * 如果进程的父进程不再存在，就指向进程1的描述符。
	 * 因此，如果用户运行一个后台进程而且退出了shell，后台进程就会成为init的子进程。
	 */
	struct task_struct *real_parent; /* real parent process (when being debugged) */
	/**
	 * 指向进程的当前父进程。这种进程的子进程终止时，必须向父进程发信号。
	 * 它的值通常与real_parent一致。
	 * 但偶尔也可以不同。例如：当另一个进程发出监控进程的ptrace系统调用请求时。
	 */
	struct task_struct *parent;	/* parent process */
	/*
	 * children/sibling forms the list of my children plus the
	 * tasks I'm ptracing.
	 */
	/**
	 * 链表头部。链表指向的所有元素都是进程创建的子进程。
	 */
	struct list_head children;	/* list of my children */
	/**
	 * 指向兄弟进程链表的下一个元素或前一个元素的指针。
	 */
	struct list_head sibling;	/* linkage in my parent's children list */
	/**
	 * P所在进程组的领头进程的描述符指针。
	 */
	struct task_struct *group_leader;	/* threadgroup leader */

	/* PID/PID hash table linkage. */
	/**
	 * PID散列表。通过这四个表，可以方便的查找同一线程组的其他线程，同一会话的其他进程等等。
	 */
	struct pid pids[PIDTYPE_MAX];

	struct completion *vfork_done;		/* for vfork() */
	/**
	 * 子进程在用户态的地址。这些用户态地址的值将被设置或者清除。
	 * 在do_fork时记录这些地址，稍后再设置或者清除它们的值。
	 */
	int __user *set_child_tid;		/* CLONE_CHILD_SETTID */
	int __user *clear_child_tid;		/* CLONE_CHILD_CLEARTID */

	/**
	 * 进程的实时优先级。
	 */
	unsigned long rt_priority;
	/**
	 * 以下三对值用于用户态的定时器。当定时器到期时，会向用户态进程发送信号。
	 * 每一对值分别存放了两个信号之间以节拍为单位的间隔，及定时器的当前值。
	 */
	unsigned long it_real_value, it_real_incr;
	cputime_t it_virt_value, it_virt_incr;
	cputime_t it_prof_value, it_prof_incr;
	/**
	 * 每个进程的动态定时器。用于实现ITIMER_REAL类型的间隔定时器。
	 * 由settimer系统调用初始化。
	 */
	struct timer_list real_timer;
	/**
	 * 进程在用户态和内核态下经过的节拍数
	 */
	cputime_t utime, stime;
	unsigned long nvcsw, nivcsw; /* context switch counts */
	struct timespec start_time;
/* mm fault and swap info: this can arguably be seen as either mm-specific or thread-specific */
	unsigned long min_flt, maj_flt;
/* process credentials */
	uid_t uid,euid,suid,fsuid;
	gid_t gid,egid,sgid,fsgid;
	struct group_info *group_info;
	kernel_cap_t   cap_effective, cap_inheritable, cap_permitted;
	unsigned keep_capabilities:1;
	struct user_struct *user;
#ifdef CONFIG_KEYS
	struct key *session_keyring;	/* keyring inherited over fork */
	struct key *process_keyring;	/* keyring private to this process (CLONE_THREAD) */
	struct key *thread_keyring;	/* keyring private to this thread */
#endif
	int oomkilladj; /* OOM kill score adjustment (bit shift). */
	char comm[TASK_COMM_LEN];
/* file system info */
	/**
	 * 文件系统在查找路径时使用，避免符号链接查找深度过深，导致死循环。
	 * link_count是__do_follow_link递归调用的层次。
	 * total_link_count调用__do_follow_link的总次数。
	 */
	int link_count, total_link_count;
/* ipc stuff */
	struct sysv_sem sysvsem;
/* CPU-specific state of this task */
	struct thread_struct thread;
/* filesystem information */
	/**
	 * 与文件系统相关的信息。如当前目录。
	 */
	struct fs_struct *fs;
/* open file information */
	/**
	 * 指向文件描述符的指针
	 */
	struct files_struct *files;
/* namespace */
	struct namespace *namespace;
/* signal handlers */
	/**
	 * 指向进程的信号描述符的指针
	 */
	struct signal_struct *signal;
	/**
	 * 指向进程的信号处理程序描述符的指针
	 */
	struct sighand_struct *sighand;

	/**
	 * blocked-被阻塞的信号的掩码
	 * real_blocked-被阻塞信号的临时掩码（由rt_sigtimedwait系统调用使用）
	 */
	sigset_t blocked, real_blocked;
	/**
	 * 存放私有挂起信号的数据结构
	 */
	struct sigpending pending;

	/**
	 * 信号处理程序备用堆栈的地址
	 */
	unsigned long sas_ss_sp;
	/**
	 * 信号处理程序备用堆栈的大小
	 */
	size_t sas_ss_size;
	/**
	 * 指向一个函数的指针，设备驱动程序使用这个函数阻塞进程的某些信号
	 */
	int (*notifier)(void *priv);
	/**
	 * 指向notifier函数可能使用的数据
	 */
	void *notifier_data;
	/*
	 * 设备驱动程序通过notifier 函数所阻塞的信号的位掩码
	 */
	sigset_t *notifier_mask;
	
	void *security;
	struct audit_context *audit_context;

/* Thread group tracking */
   	u32 parent_exec_id;
   	u32 self_exec_id;
/* Protection of (de-)allocation: mm, files, fs, tty, keyrings */
	spinlock_t alloc_lock;
/* Protection of proc_dentry: nesting proc_lock, dcache_lock, write_lock_irq(&tasklist_lock); */
	spinlock_t proc_lock;
/* context-switch lock */
	spinlock_t switch_lock;

/* journalling filesystem info */
	/**
	 * 当前活动日志操作处理的地址。
	 */
	void *journal_info;

/* VM state */
	struct reclaim_state *reclaim_state;

	struct dentry *proc_dentry;
	struct backing_dev_info *backing_dev_info;

	struct io_context *io_context;

	unsigned long ptrace_message;
	siginfo_t *last_siginfo; /* For ptrace use.  */
/*
 * current io wait handle: wait queue entry to use for io waits
 * If this thread is processing aio, this points at the waitqueue
 * inside the currently handled kiocb. It may be NULL (i.e. default
 * to a stack based synchronous wait) if its doing sync IO.
 */
	wait_queue_t *io_wait;
/* i/o counters(bytes read/written, #syscalls */
	u64 rchar, wchar, syscr, syscw;
#if defined(CONFIG_BSD_PROCESS_ACCT)
	u64 acct_rss_mem1;	/* accumulated rss usage */
	u64 acct_vm_mem1;	/* accumulated virtual memory usage */
	clock_t acct_stimexpd;	/* clock_t-converted stime since last update */
#endif
#ifdef CONFIG_NUMA
  	struct mempolicy *mempolicy;
	short il_next;
#endif
};
```

包罗万象！

### 进程状态
state描述进程状态，0表示正在运行或准备执行(runnable)，-1表示unrunnable，而大于0的情况都是stop态，此时要根据具体的值来判断其所处状态：

以下代码展示了各种状态值：
```cpp
/**
 * 进程要么在CPU上执行，要么准备执行。
 */
#define TASK_RUNNING		0
/**
 * 可中断的等待状态。
 * 进程被挂起，直到某个条件变为真。产生一个硬件中断，释放进程正等待的系统资源，或传递一个信号都是可以唤醒进程的条件
 */
#define TASK_INTERRUPTIBLE	1
/**
 * 不可中断的等待状态。
 * 这种情况很少，但是有时也有用：比如进程打开一个设备文件，其相应的驱动程序在探测硬件设备时，就是这种状态。
 * 在探测完成前，设备驱动程序如果被中断，那么硬件设备的状态可能会处于不可预知状态。
 */
#define TASK_UNINTERRUPTIBLE	2
/**
 * 暂停状态。当收到SIGSTOP,SIGTSTP,SIGTTIN或者SIGTTOU信号后，会进入此状态。
 */
#define TASK_STOPPED		4
/**
 * 被跟踪状态。当进程被另外一个进程监控时，任何信号都可以把这个置于该状态
 */
#define TASK_TRACED		8
/**
 * 僵死状态。进程的执行被终止，但是，父进程还没有调用完wait4和waitpid来返回有关
 * 死亡进程的信息。在此时，内核不能释放相关数据结构，因为父进程可能还需要它。
 */
#define EXIT_ZOMBIE		16
/**
 * 在父进程调用wait4后，删除前，为避免其他进程在同一进程上也执行wait4调用
 * 将其状态由EXIT_ZOMBIE转为EXIT_DEAD，即僵死撤销状态。
 */
#define EXIT_DEAD		32
```

state字段的值通常都是简单的赋值语句：`p->state = TASK_RUNNING`

> 虽然设计上可以看出每个状态都是一个二进制位，理论上可以置多位。但当前的Linux中，这些标志全部都是互斥的。

每个进程都有也必须有自己的进程描述符（包括轻量级进程，因为它也能独立调度每个执行上下文）。进程和描述符间存在一一对应的关系，进程描述符指针指向这些地址，内核对进程的大部分引用都是通过进程描述符指针进行的。

### PID
Linux的每个进程都有一个PID，这个PID放在task_struct的pid中。pid循环使用，内核通过一个pidmap-array位图来管理PID的分配。Linux把不同的PID与系统中每个进程相关联。这种方式十分灵活，因为系统中每个执行上下文都可以被唯一地识别。

而因为考虑到多线程的支持，Linux引入线程组的表示。一个线程组中所有线程使用和该线程组领头线程相同的PID，也就是该组中第一个轻量级进程的PID，它放入task_struct的tgid字段。多线程应用程序的所有线程共享相同的PID。绝大多数进程都属于一个线程组，包含单一的成员；线程组的领头线程其tgid的值与pid的值相同。

> getpid()返回的是tgid值而不是pid值。某种视角来看，Linux多线程的支持概念中，线程和进程的地位刚好和Windows相反。

### 进程<->进程描述符
进程是动态的，内核显然不能把进程描述符放在永久分配的静态内存区，而应该放在动态内存中。对每个进程来说，Linux把两个不同的数据结构紧凑地存放在一个单独为进程分配的存储区域内：内核态堆栈+thread_info。

![](20170822_2.jpg)

共同占据了8K也就是两页的内存，栈向上增长，thread_info有个task指针指向了进程描述符，进程描述符也通过thread_info指针指向这一结构，形成互相引用的关系。

```cpp
/**
 * 内核栈与thread_info的联合体。
 */
union thread_union {
	struct thread_info thread_info;
	unsigned long stack[THREAD_SIZE/sizeof(long)];
};
```

> 考虑到效率因素，内核会让这8K占据连续的两个物理页框，第一个页框起始地址总是`1<<13`的倍数。另一方面因为连续页框会导致大量碎片，所以x86也提供了编译时的设置项，可以让内核栈和线程描述符跨越一个单独的页框。

因为esp的关系，内核想要获取thread_info非常简单，current_thread_info()来完成这一简单的操作：

```
# AT&T汇编产生的汇编指令
movl $0xffffe000, $ecx
andl $esp, $ecx
movl $ecx, p
```
实际上就是简单的把esp的低13位归零，自然就对应着thread_union的起始地址，也正是thread_info。
进一步拿到current则有:

```cpp
#define current get_current()

static inline struct task_struct * get_current(void)
{
	return current_thread_info()->task;
}
```

> 栈存放进程描述符的优点在于多处理器上，每个处理器只需要通过检查栈就可以获得当前正确的进程。

### 双向链表
Linux内核有个通用的结构list_head，本身只是个双向循环链表。但Linux常常把它嵌入到某个struct中，间接实现struct的链式管理：

![](20170822_3.jpg)

注意next和prev指针指向的不是某个struct，而是struct中list_head的首地址（显然是为了泛用型设计）。

```cpp
/*
 * Simple doubly linked list implementation.
 *
 * Some of the internal functions ("__xxx") are useful when
 * manipulating whole lists rather than single entries, as
 * sometimes we already know the next/prev entries and we can
 * generate better code by using them directly rather than
 * using the generic single-entry routines.
 */

struct list_head {
	struct list_head *next, *prev;
};

#define LIST_HEAD_INIT(name) { &(name), &(name) }

/**
 * 创建一个新的链表。是新链表头的占位符，并且是一个哑元素。
 * 同时初始化prev和next字段，让它们指向list_name变量本身。
 */
#define LIST_HEAD(name) \
	struct list_head name = LIST_HEAD_INIT(name)

#define INIT_LIST_HEAD(ptr) do { \
	(ptr)->next = (ptr); (ptr)->prev = (ptr); \
} while (0)

/*
 * Insert a new entry between two known consecutive entries.
 *
 * This is only for internal list manipulation where we know
 * the prev/next entries already!
 */
static inline void __list_add(struct list_head *new,
			      struct list_head *prev,
			      struct list_head *next)
{
	next->prev = new;
	new->next = next;
	new->prev = prev;
	prev->next = new;
}

/**
 * list_add - add a new entry
 * @new: new entry to be added
 * @head: list head to add it after
 *
 * Insert a new entry after the specified head.
 * This is good for implementing stacks.
 */
/**
 * 把元素插入特定元素之后
 */
static inline void list_add(struct list_head *new, struct list_head *head)
{
	__list_add(new, head, head->next);
}

/**
 * list_add_tail - add a new entry
 * @new: new entry to be added
 * @head: list head to add it before
 *
 * Insert a new entry before the specified head.
 * This is useful for implementing queues.
 */
/**
 * 把元素插到特定元素之前。
 */
static inline void list_add_tail(struct list_head *new, struct list_head *head)
{
	__list_add(new, head->prev, head);
}

/*
 * Insert a new entry between two known consecutive entries.
 *
 * This is only for internal list manipulation where we know
 * the prev/next entries already!
 */
static inline void __list_add_rcu(struct list_head * new,
		struct list_head * prev, struct list_head * next)
{
	new->next = next;
	new->prev = prev;
	smp_wmb();
	next->prev = new;
	prev->next = new;
}

...

/*
 * Delete a list entry by making the prev/next entries
 * point to each other.
 *
 * This is only for internal list manipulation where we know
 * the prev/next entries already!
 */
static inline void __list_del(struct list_head * prev, struct list_head * next)
{
	next->prev = prev;
	prev->next = next;
}

/**
 * list_del - deletes entry from list.
 * @entry: the element to delete from the list.
 * Note: list_empty on entry does not return true after this, the entry is
 * in an undefined state.
 */
/**
 * 删除特定元素
 */
static inline void list_del(struct list_head *entry)
{
	__list_del(entry->prev, entry->next);
	entry->next = LIST_POISON1;
	entry->prev = LIST_POISON2;
}

...

/**
 * list_empty - tests whether a list is empty
 * @head: the list to test.
 */
/**
 * 检查指定的链表是否为空
 */
static inline int list_empty(const struct list_head *head)
{
	return head->next == head;
}

...

/**
 * list_entry - get the struct for this entry
 * @ptr:	the &struct list_head pointer.
 * @type:	the type of the struct this is embedded in.
 * @member:	the name of the list_struct within the struct.
 */
/** 
 * 返回链表所在结构
 */
#define list_entry(ptr, type, member) \
	container_of(ptr, type, member)

/**
 * list_for_each	-	iterate over a list
 * @pos:	the &struct list_head to use as a loop counter.
 * @head:	the head for your list.
 */
/**
 * 扫描指定的链表
 */
#define list_for_each(pos, head) \
	for (pos = (head)->next; prefetch(pos->next), pos != (head); \
        	pos = pos->next)

/**
 * __list_for_each	-	iterate over a list
 * @pos:	the &struct list_head to use as a loop counter.
 * @head:	the head for your list.
 *
 * This variant differs from list_for_each() in that it's the
 * simplest possible list iteration code, no prefetching is done.
 * Use this for code that knows the list to be very short (empty
 * or 1 entry) most of the time.
 */
#define __list_for_each(pos, head) \
	for (pos = (head)->next; pos != (head); pos = pos->next)

/**
 * list_for_each_prev	-	iterate over a list backwards
 * @pos:	the &struct list_head to use as a loop counter.
 * @head:	the head for your list.
 */
#define list_for_each_prev(pos, head) \
	for (pos = (head)->prev; prefetch(pos->prev), pos != (head); \
        	pos = pos->prev)

/**
 * list_for_each_safe	-	iterate over a list safe against removal of list entry
 * @pos:	the &struct list_head to use as a loop counter.
 * @n:		another &struct list_head to use as temporary storage
 * @head:	the head for your list.
 */
#define list_for_each_safe(pos, n, head) \
	for (pos = (head)->next, n = pos->next; pos != (head); \
		pos = n, n = pos->next)

/**
 * list_for_each_entry	-	iterate over list of given type
 * @pos:	the type * to use as a loop counter.
 * @head:	the head for your list.
 * @member:	the name of the list_struct within the struct.
 */
/**
 * 与list_for_each相似，但是返回每个链表结点所在结构
 */
#define list_for_each_entry(pos, head, member)				\
	for (pos = list_entry((head)->next, typeof(*pos), member);	\
	     prefetch(pos->member.next), &pos->member != (head); 	\
	     pos = list_entry(pos->member.next, typeof(*pos), member))

/**
 * list_for_each_entry_reverse - iterate backwards over list of given type.
 * @pos:	the type * to use as a loop counter.
 * @head:	the head for your list.
 * @member:	the name of the list_struct within the struct.
 */
#define list_for_each_entry_reverse(pos, head, member)			\
	for (pos = list_entry((head)->prev, typeof(*pos), member);	\
	     prefetch(pos->member.prev), &pos->member != (head); 	\
	     pos = list_entry(pos->member.prev, typeof(*pos), member))

/**
 * list_prepare_entry - prepare a pos entry for use as a start point in
 *			list_for_each_entry_continue
 * @pos:	the type * to use as a start point
 * @head:	the head of the list
 * @member:	the name of the list_struct within the struct.
 */
#define list_prepare_entry(pos, head, member) \
	((pos) ? : list_entry(head, typeof(*pos), member))

/**
 * list_for_each_entry_continue -	iterate over list of given type
 *			continuing after existing point
 * @pos:	the type * to use as a loop counter.
 * @head:	the head for your list.
 * @member:	the name of the list_struct within the struct.
 */
#define list_for_each_entry_continue(pos, head, member) 		\
	for (pos = list_entry(pos->member.next, typeof(*pos), member);	\
	     prefetch(pos->member.next), &pos->member != (head);	\
	     pos = list_entry(pos->member.next, typeof(*pos), member))

/**
 * list_for_each_entry_safe - iterate over list of given type safe against removal of list entry
 * @pos:	the type * to use as a loop counter.
 * @n:		another type * to use as temporary storage
 * @head:	the head for your list.
 * @member:	the name of the list_struct within the struct.
 */
#define list_for_each_entry_safe(pos, n, head, member)			\
	for (pos = list_entry((head)->next, typeof(*pos), member),	\
		n = list_entry(pos->member.next, typeof(*pos), member);	\
	     &pos->member != (head); 					\
	     pos = n, n = list_entry(n->member.next, typeof(*n), member))
```

篇幅太长，详细可以自己看`include/linux/List.h`

### 进程链表
进程链表是list_head的一个例子，进程链表把所有进程描述符链接起来。每个task_struct结构包含一个list_head类型的tasks字段，该字段的prev和next指向前后的task_struct元素（前面代码已给出）。

链表头是init_task描述符，就是0进程。init_task的prev指向链表最后插入的进程描述符的tasks。

### TASK_RUNNING态进程链表
CPU调度时，只需要考虑可运行进程(TASK_RUNNING态)即可。Linux2.6为此优化了数据结构，设计了多个优先级不等的可运行进程链表，通过task_struct中另外一个list_head字段run_list链入对应优先权为k的链中。每个CPU都有自己的运行队列（可运行进程链表集）。

```cpp
/**
 * 进程优先级数组。每个CPU对应一个此结构。
 */
struct prio_array {
	/**
	 * 链表中进程描述符的数量。
	 */
	unsigned int nr_active;
	/**
	 * 优先权数组。当且仅当某个优先权的进程链表不为空时设置相应的位标志。
	 */
	unsigned long bitmap[BITMAP_SIZE];
	/**
	 * 140个优先权队列的头结点。
	 */
	struct list_head queue[MAX_PRIO];
};

typedef struct prio_array prio_array_t;
```

### 进程间的关系
进程描述符中有4个描述进程关系的字段：

- real_parent	
	- 指向创建了P的进程的描述符，如果P父进程不再存在，则指向1的描述符(init进程)
- parent
	- 指向P的当前父进程（这种进程的子进程终止时，必须向父进程发信号）。它的值通常与real_parent一致，但偶尔也可以不同，例如，当另一个进程发出监控P的ptrace()系统调用请求时
- children
	- 链表头，链表中的所有元素都是P创建的子进程
- sibling
	- 指向兄弟进程链表中的下一个元素或前一个元素指针，这些兄弟进程的父进程都是P

进程关系的示意图(一图胜千言呐!)：

![](20170822_4.jpg)

建立非亲属关系的进程描述符字段：
- group_leader
	- P所在进程组的领头进程的描述符指针
- signal->pgrp
	- P所在进程组的领头进程的PID
- tgid	
	- P所在线程组的领头进程的PID
- signal->session
	- P的登录会话领头进程的PID
- ptrace_children
	- 链表的头，该链表包含所有被debugger程序跟踪的P的子进程
- ptrace_list
	- 指向所跟踪进程其实际父进程链表的前一个和下一个元素（用于P被跟踪的时候）

### pidhash表及链表
内核必须能从进程PID导出对应的进程描述符指针，比如kill()系统调用（P1调用kill()，参数为P2的PID）。顺序扫描进程链表并逐一检查效率太低了，Linux实际上在布局进程描述符时，不仅采用了内嵌的双向链表，还有4个散列表。他们分别是：

| Hash表类型   | 字段    | 说明                |
| ------------ | ------- | ------------------- |
| PIDTYPE_PID  | pid     | 进程的PID           |
| PIDTYPE_TGID | tgid    | 进程组领头进程的PID |
| PIDTYPE_PGID | pgrp    | 进程组领头进程的PID |
| PIDTYPE_SID  | session | 会话领头进程的PID   |

4个散列表在内核初始化期间拿到空间，并把他们的地址存入pid_hash数组。

Linux内核的哈希对于冲突的处理采用桶式链处理：

![](20170822_5.jpg)

task_struct中struct pid成员pids数组即为四个散列表，看看struct结构：

```cpp
struct pid
{
	/* Try to keep pid_chain in the same cacheline as nr for find_pid */
	/**
	 * PID值。
	 */
	int nr;
	/**
	 * 链接散列表中下一个和前一个元素。
	 */
	struct hlist_node pid_chain;
	/* list of pids with the same nr, only one of them is in the hash */
	/**
	 * 每个PID的进程链表头。
	 */
	struct list_head pid_list;
};
```

所以，哈希的效果实现来看即如此：

![](20170822_6.jpg)

### 非TASK_RUNNING态进程
除了TASK_RUNNING态的进程组织成了链表，其他状态也有分别的处理：

- TASK_STOPPED,EXIT_ZOMBIE,EXIT_DEAD都是散户，没有组织成链表，这几种状态的进程访问或者通过PID散列，或者通过父进程的子进程链表。
- TASK_INTERRUPTIBLE，TASK_UNINTERRUPTIBLE状态被再细分成多个类，每一个对应一个特定的事件(event)。在这种情况下，进程状态不提供足够的信息来快速的追溯进程，所以有必要引入额外的进程链表。这被称作等待队列。

### 等待队列
进程必须经常等待某些事件的发生，例如，等待一个磁盘操作的终止，系统资源的释放，等待固定的间隔。等待队列实现了在事件上的条件等待：希望等待特定事件的进程把自己放入合适的等待队列，并放弃控制权（阻塞）。因此，等待队列表示的是一组睡眠的进程，当其对应的事件置True时，内核唤醒它们。

等待队列也是双向链表，使用list_head子结构，每个队列都有一个等待队列头，结构如下：

```cpp
/**
 * 等待队列的头
 */
struct __wait_queue_head {
	/**
	 * 由于等待队列可能由中断处理程序和内核函数修改，所以必须对双向链表进行保护，以免对其进行同时访问。
	 * 其同步是由lock自旋锁达到的。
	 */
	spinlock_t lock;
	/**
	 * 等待进程链表的头。
	 */
	struct list_head task_list;
};
typedef struct __wait_queue_head wait_queue_head_t;
```

lock为同步所用自旋锁。

等待队列链表中元素类型为wait_queue_t:

```cpp
typedef struct __wait_queue wait_queue_t;

/**
 * 等待队列中的元素，每个元素代表一个睡眠的进程。
 * 该进程等待某一个事件的发生。
 */
struct __wait_queue {
	/**
	 * 如果flags为1,表示等待进程是互斥的。等待访问临界资源的进程就是典型的互斥进程。
	 * 如果flags为0，表示等待进程是非互斥的。等待相关事件的进程是非互斥的。
	 */
	unsigned int flags;
#define WQ_FLAG_EXCLUSIVE	0x01
	/**
	 * 睡眠在队列上的进程描述符。
	 */
	struct task_struct * task;
	/**
	 * 等待队列中的睡眠进程以何种方式唤醒。
	 */
	wait_queue_func_t func;
	/**
	 * 通过此指针将睡眠进程链接起来。
	 */
	struct list_head task_list;
};
```

每个元素都是一个睡眠进程，等待某一事件发生；描述符地址在task字段，task_list包含的是指针，链接前后的等待相同事件的进程成员。

显然如果每次唤醒都是唤醒所有进程，则当多个进程申请互斥资源时，存在着废操作（雷鸣般兽群问题），此时仅唤醒一个即可。于是，互斥进程（等待队列元素的flags字段为1）由内核选择地唤醒，非互斥进程(flags为0)由内核在事件发生时唤醒。

## 参考文献

- 《深入理解Linux内核》
- 《Linux内核源代码情景分析(上)》
- 《Windows核心编程》
- [Linux内核中的hash与bucket](http://www.nowamagic.net/academy/detail/3008086)
- [ULK Chinese comments](https://github.com/sohu2000000/ULK)