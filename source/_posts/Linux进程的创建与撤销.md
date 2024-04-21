---
title: Linux内核学习——Linux进程的创建与撤销
date: 2017-08-27 12:05:11
categories: Linux
tags:
	- linux-kernel
	- ulk
---
Linux的进程除了PID为0的第一个进程以外，所有的其他进程都是复制出来的，这和Windows凭空捏造不太一样。另一方面，Linux进程在复制时也有很多细节处理的设计，旨在更高效、更简洁。近来在研究Linux进程活动的整个生命周期，而本文仅仅对进程的出生与死亡进行了概略性描述，毕竟一个进程的前世今生绝不是一两篇文章可以hold住的。

<!--more-->

# Linux进程的创建与撤销
Linux进程的创建有三个函数。fork,vfork和clone。不要把exec系列的函数和这三个函数混为一谈，exec系列函数常用于三者之后，用于创建一个新的程序运行环境。

## clone(), fork(), vfork()
Linux既然有各种各样的机制，那么复制也有着多种手法。这也就对应了这三种创建进程的系统调用。

这三个函数实际上都是C库的封装，内部分别调用了sys_clone, sys_fork, sys_vfork三个系统调用。这三个系统调用又统一调用了do_fork函数，只是携带的参数和标志不同，从而在do_fork中有选择的完成某些任务。

便于理解，先大体来谈谈三者的区别：
- fork
	- 子进程复制父进程的所有资源
- clone
	- 轻量级进程使用clone
- vfork
	- 子进程和父进程共享数据段，子进程每次都优先于父进程执行(这很有意义，后面你会看到)

### fork()
既然Linux的进程是复制的，那么大多数人的第一想法一定是子进程复制整个父进程的资源。诸如最简单的fork，是否只要在调用时完全的深度拷贝即可？实际上深度拷贝所有的资源是没有必要的，Linux指定了3个机制，用于提升性能：
- COW，即写时复制。这一技术的引用就使得父子进程可以在一开始的时候共享物理页（一开始是相同的），当二者其一想要写该物理页时，再真正的copy出页的内容到新分配的物理页，然后进行修改。
- 轻量级进程允许父子进程共享内核的大部分数据结构（页表（用户态地址空间）、打开文件表和信号处理），所以如同在《Linux进程概述》中所说，它更像一个线程。
- vfork()系统调用创建的进程能共享父进程的内存地址空间。为了防止父进程重写子进程所需要的数据，阻塞父进程的执行，直到子进程推出或执行一个新的程序为止。

<font color="red">无论是哪一种机制，其根本的思想是一致的：能省则省，延迟拷贝。</font>

对于fork来说，进程A调用fork创建一个子进程B时，B和A拥有相同的物理页面，为了节约内存和提速，fork()会把A和B的物理页面设置成只读。此后，A或B想要执行写操作时，都会产生页面出错异常(page_fault int14)中断，此时CPU执行异常处理函数do_wp_page()解决该异常。do_wp_page()实际上非常简单，无非就是取消共享，为写进程复制一个新的物理页面（此时才真正进行了开辟和复制），设置权限。异常返回后，继续执行写操作。

### vfork()
比起fork()的延迟懒惰处理，vfork()更加粗暴。内核在复制子进程时，连子进程自身的虚拟地址空间都不创建了，干脆使用了父进程的虚拟空间。既然虚拟空间都霸占了，物理页面也当然共享了（这意味着修改子进程的变量值也会影响父进程）。
于是，vfork为了防止父进程overwrite，设计上会把父进程先挂起，子进程exit或execve时父进程才会被唤起。vfork创建的子进程不应该用return或exit()，但可以用_exit()退出（参考man vfork，exit是_exit()或_exitgroup()的封装，结束子进程，他不会修改函数栈，所以我在测试中exit()没有出错，这应该是一种严格的说法，当然vfork不接exec，要么脑子有坑，要么想干坏事。至于exit()封装的额外操作还有待研究）。

> vfork的子进程如果return就意味着父进程return（共享内存、栈），这看起来没问题，但是接下来父进程再次return就崩了，系统表示很困惑。（return后会自动接exit()，而此时栈已经被破坏了，有些系统会无限循环，再次调用main()，有些直接就segmentation fault）。

那么问题来了，我要这vfork有何用？实际上，vfork只是一个中间步骤，vfork的存在是为了exec的调用。exec是重新开辟空间，那么如果没有vfork，就只得用笨重的fork，而因为下一步想要exec，所以fork的复制过程就毫无意义。

### clone()
clone是给轻量级进程的，但clone本身的设计很强大，他可以有选择性的让子进程继承资源。

clone结构：`int clone(int (fn)(void ), void *child_stack, int flags, void *arg, ...); `

fn是函数指针，指向程序的指针，函数返回时子进程终止并返回一个退出码；child_stack是子进程堆栈；flags表示从父进程继承哪些资源；arg为传递给子进程的参数。

flags取值是一组宏，几个常见的标志：
- CLONE_VM
	- 共享内存描述符和所有的页表
- CLONE_FS
	- 共享根目录和当前工作目录所在的表，以及用于屏蔽新文件初始许可权的位掩码值
- CLONE_FILES
	- 共享打开文件表
- CLONE_SIGHAND
	- 共享信号处理程序的表、阻塞信号表和挂起信号表。如果标记为true，必须设置CLONE_VM
- CLONE_PTRACE
	- 如果父进程被跟踪，那么子进程也被跟踪。
- CLONE_VFORK
	- vfork()系统调用时设置
- CLONE_PARENT
	- 设置子进程的父进程为调用进程的父进程
- CLONE_THREAD
	- 把子进程插入到父进程的同一线程组中，使子进程共享父进程信号描述符。因此子进程的tgid和group_leader字段也被设置。如果该标记为true，必须设置CLONE_SIGHAND
- CLONE_NEWNS
	- 当clone需要自己的命名空间时设置这个标志。CLONE_NEWNS和CLONE_FS互斥。
- CLONE_PID
	- 子进程创建时PID和父进程一致

clone()是libc定义的封装函数，clone()系统调用的服务例程是sys_clone()，它没有fn和arg参数（clone()把fn指针放在了子进程堆栈的某个位置，arg在fn下面），clone()返回后，取出的地址就是fn，参数就是arg，顺理成章执行fn(arg)。

### 系统调用服务例程
实际上我在看glibc 2.19时发现整个流程非常的复杂，有兴趣的可以自己跟一下。但无论如何，我只需要知道，最终的系统调用对应的服务例程是sys_fork等函数就够了。

```cpp
asmlinkage int sys_fork(struct pt_regs regs)
{
	return do_fork(SIGCHLD, regs.esp, &regs, 0, NULL, NULL);
}

asmlinkage int sys_clone(struct pt_regs regs)
{
	unsigned long clone_flags;
	unsigned long newsp;
	int __user *parent_tidptr, *child_tidptr;

	clone_flags = regs.ebx;
	newsp = regs.ecx;
	parent_tidptr = (int __user *)regs.edx;
	child_tidptr = (int __user *)regs.edi;
	if (!newsp)
		newsp = regs.esp;
	return do_fork(clone_flags, newsp, &regs, 0, parent_tidptr, child_tidptr);
}

/*
 * This is trivial, and on the face of it looks like it
 * could equally well be done in user mode.
 *
 * Not so, for quite unobvious reasons - register pressure.
 * In user mode vfork() cannot have a stack frame, and if
 * done by calling the "clone()" system call directly, you
 * do not have enough call-clobbered registers to hold all
 * the information you need.
 */
asmlinkage int sys_vfork(struct pt_regs regs)
{
	return do_fork(CLONE_VFORK | CLONE_VM | SIGCHLD, regs.esp, &regs, 0, NULL, NULL);
}
```

可以看到vfork和fork最终也都是通过do_fork()进行的，只是提前给定了clone_flags。
do_fork()处理的事情：
```cpp
/**
 * 负责处理clone,fork,vfork系统调用。
 * clone_flags-与clone的flag参数相同
 * stack_start-与clone的child_stack相同
 * regs-指向通用寄存器的值。是在从用户态切换到内核态时被保存到内核态堆栈中的。
 * stack_size-未使用,总是为0
 * parent_tidptr,child_tidptr-clone中对应参数ptid,ctid相同
 */
long do_fork(unsigned long clone_flags,
	      unsigned long stack_start,
	      struct pt_regs *regs,
	      unsigned long stack_size,
	      int __user *parent_tidptr,
	      int __user *child_tidptr)
{
	struct task_struct *p;
	int trace = 0;
	/**
	 * 通过查找pidmap_array位图,为子进程分配新的pid参数.
	 */
	long pid = alloc_pidmap();

	if (pid < 0)
		return -EAGAIN;
	/**
	 * 如果父进程正在被跟踪,就检查debugger程序是否想跟踪子进程.并且子进程不是内核进程(CLONE_UNTRACED未设置)
	 * 那么就设置CLONE_PTRACE标志.
	 */
	if (unlikely(current->ptrace)) {
		trace = fork_traceflag (clone_flags);
		if (trace)
			clone_flags |= CLONE_PTRACE;
	}

	/**
	 * copy_process复制进程描述符.如果所有必须的资源都是可用的,该函数返回刚创建的task_struct描述符的地址.
	 * 这是创建进程的关键步骤.
	 */
	p = copy_process(clone_flags, stack_start, regs, stack_size, parent_tidptr, child_tidptr, pid);
	/*
	 * Do this prior waking up the new thread - the thread pointer
	 * might get invalid after that point, if the thread exits quickly.
	 */
	if (!IS_ERR(p)) {
		struct completion vfork;

		if (clone_flags & CLONE_VFORK) {
			p->vfork_done = &vfork;
			init_completion(&vfork);
		}

		/**
		 * 如果设置了CLONE_STOPPED,或者必须跟踪子进程.
		 * 就设置子进程为TASK_STOPPED状态,并发送SIGSTOP信号挂起它.
		 */
		if ((p->ptrace & PT_PTRACED) || (clone_flags & CLONE_STOPPED)) {
			/*
			 * We'll start up with an immediate SIGSTOP.
			 */
			sigaddset(&p->pending.signal, SIGSTOP);
			set_tsk_thread_flag(p, TIF_SIGPENDING);
		}

		/**
		 * 没有设置CLONE_STOPPED,就调用wake_up_new_task
		 * 它调整父进程和子进程的调度参数.
		 * 如果父子进程运行在同一个CPU上,并且不能共享同一组页表(CLONE_VM标志被清0).那么,就把子进程插入父进程运行队列.
		 * 并且子进程插在父进程之前.这样做的目的是:如果子进程在创建之后执行新程序,就可以避免写时复制机制执行不必要时页面复制.
		 * 否则,如果运行在不同的CPU上,或者父子进程共享同一组页表.就把子进程插入父进程运行队列的队尾.
		 */
		if (!(clone_flags & CLONE_STOPPED))
			wake_up_new_task(p, clone_flags);
		else/*如果CLONE_STOPPED标志被设置，就把子进程设置为TASK_STOPPED状态。*/
			p->state = TASK_STOPPED;

		/**
		 * 如果进程正被跟踪,则把子进程的PID插入到父进程的ptrace_message,并调用ptrace_notify
		 * ptrace_notify使当前进程停止运行,并向当前进程的父进程发送SIGCHLD信号.子进程的祖父进程是跟踪父进程的debugger进程.
		 * dubugger进程可以通过ptrace_message获得被创建子进程的PID.
		 */
		if (unlikely (trace)) {
			current->ptrace_message = pid;
			ptrace_notify ((trace << 8) | SIGTRAP);
		}

		/**
		 * 如果设置了CLONE_VFORK,就把父进程插入等待队列,并挂起父进程直到子进程结束或者执行了新的程序.
		 */
		if (clone_flags & CLONE_VFORK) {
			wait_for_completion(&vfork);
			if (unlikely (current->ptrace & PT_TRACE_VFORK_DONE))
				ptrace_notify ((PTRACE_EVENT_VFORK_DONE << 8) | SIGTRAP);
		}
	} else {
		free_pidmap(pid);
		pid = PTR_ERR(p);
	}
	return pid;
}
```
可以看到，实际上fork，clone和vfork的区别，仅仅是在do_fork中根据clone_flags来筛选的。对应不同的flags，实现效果和此前谈到的机制相同。

### copy_process()
这个函数创建了进程描述符以及子进程执行所要的所有其他数据结构。它的参数与do_fork()相同，外加子进程的PID。

```cpp
/**
 * 创建进程描述符以及子进程执行所需要的所有其他数据结构
 * 它的参数与do_fork相同。外加子进程的PID。
 */
static task_t *copy_process(unsigned long clone_flags,
				 unsigned long stack_start,
				 struct pt_regs *regs,
				 unsigned long stack_size,
				 int __user *parent_tidptr,
				 int __user *child_tidptr,
				 int pid)
{
	int retval;
	struct task_struct *p = NULL;

	/**
	 * 检查clone_flags所传标志的一致性。
	 */

	/**
	 * 如果CLONE_NEWNS和CLONE_FS标志都被设置，返回错误
	 */
	if ((clone_flags & (CLONE_NEWNS|CLONE_FS)) == (CLONE_NEWNS|CLONE_FS))
		return ERR_PTR(-EINVAL);

	/*
	 * Thread groups must share signals as well, and detached threads
	 * can only be started up within the thread group.
	 */
	/**
	 * CLONE_THREAD标志被设置，并且CLONE_SIGHAND没有设置。
	 * (同一线程组中的轻量级进程必须共享信号)
	 */
	if ((clone_flags & CLONE_THREAD) && !(clone_flags & CLONE_SIGHAND))
		return ERR_PTR(-EINVAL);

	/*
	 * Shared signal handlers imply shared VM. By way of the above,
	 * thread groups also imply shared VM. Blocking this case allows
	 * for various simplifications in other code.
	 */
	/**
	 * CLONE_SIGHAND被设置，但是CLONE_VM没有设置。
	 * (共享信号处理程序的轻量级进程也必须共享内存描述符)
	 */
	if ((clone_flags & CLONE_SIGHAND) && !(clone_flags & CLONE_VM))
		return ERR_PTR(-EINVAL);

	/**
	 * 通过调用security_task_create以及稍后调用security_task_alloc执行所有附加的安全检查。
	 * LINUX2.6提供扩展安全性的钩子函数，与传统unix相比，它具有更加强壮的安全模型。
	 */
	retval = security_task_create(clone_flags);
	if (retval)
		goto fork_out;

	retval = -ENOMEM;
	/**
	 * 调用dup_task_struct为子进程获取进程描述符。
	 */
	p = dup_task_struct(current);
	if (!p)
		goto fork_out;

	/**
	 * 检查存放在current->sigal->rlim[RLIMIT_NPROC].rlim_cur中的限制值，是否小于或者等于用户所拥有的进程数。
	 * 如果是，则返回错误码。当然，有root权限除外。
	 * p->user表示进程的拥有者，p->user->processes表示进程拥有者当前进程数
	 * xie.baoyou注：此处比较是用>=而不是>
	 */
	retval = -EAGAIN;
	if (atomic_read(&p->user->processes) >=
			p->signal->rlim[RLIMIT_NPROC].rlim_cur) {
		/**
		 * 当然，用户有root权限就另当别论了
		 */
		if (!capable(CAP_SYS_ADMIN) && !capable(CAP_SYS_RESOURCE) &&
				p->user != &root_user)
			goto bad_fork_free;
	}

	/**
	 * 递增user结构的使用计数器
	 */
	atomic_inc(&p->user->__count);
	/**
	 * 增加用户拥有的进程计数。
	 */
	atomic_inc(&p->user->processes);
	get_group_info(p->group_info);

	/*
	 * If multiple threads are within copy_process(), then this check
	 * triggers too late. This doesn't hurt, the check is only there
	 * to stop root fork bombs.
	 */
	/**
	 * 检查系统中的进程数量（nr_threads）是否超过max_threads
	 * max_threads的缺省值是由系统内存容量决定的。总的原则是：所有的thread_info描述符和内核栈所占用的空间
	 * 不能超过物理内存的1/8。不过，系统管理可以通过写/proc/sys/kernel/thread-max文件来改变这个值。
	 */
	if (nr_threads >= max_threads)
		goto bad_fork_cleanup_count;

	/**
	 * 如果新进程的执行域和可招待格式的内核函数都包含在内核中模块中，
	 * 就递增它们的使用计数器。
	 */
	if (!try_module_get(p->thread_info->exec_domain->module))
		goto bad_fork_cleanup_count;

	if (p->binfmt && !try_module_get(p->binfmt->module))
		goto bad_fork_cleanup_put_domain;

	/**
	 * 设置几个与进程状态相关的关键字段。
	 */

	/**
	 * did_exec是进程发出的execve系统调用的次数，初始为0
	 */
	p->did_exec = 0;
	/**
	 * 更新从父进程复制到tsk_flags字段中的一些标志。
	 * 首先清除PF_SUPERPRIV。该标志表示进程是否使用了某种超级用户权限。
	 * 然后设置PF_FORKNOEXEC标志。它表示子进程还没有发出execve系统调用。
	 */
	copy_flags(clone_flags, p);
	/**
	 * 保存新进程的pid值。
	 */
	p->pid = pid;
	retval = -EFAULT;
	/**
	 * 如果CLONE_PARENT_SETTID标志被设置，就将子进程的PID复制到参数parent_tidptr指向的用户态变量中。
	 * xie.baoyou:想想我们常常调用的pid = fork()语句吧。
	 */
	if (clone_flags & CLONE_PARENT_SETTID)
		if (put_user(p->pid, parent_tidptr))
			goto bad_fork_cleanup;

	p->proc_dentry = NULL;

	/**
	 * 初始化子进程描述符中的list_head数据结构和自旋锁。
	 * 并为挂起信号，定时器及时间统计表相关的几个字段赋初值。
	 */
	INIT_LIST_HEAD(&p->children);
	INIT_LIST_HEAD(&p->sibling);
	p->vfork_done = NULL;
	spin_lock_init(&p->alloc_lock);
	spin_lock_init(&p->proc_lock);

	clear_tsk_thread_flag(p, TIF_SIGPENDING);
	init_sigpending(&p->pending);

	p->it_real_value = 0;
	p->it_real_incr = 0;
	p->it_virt_value = cputime_zero;
	p->it_virt_incr = cputime_zero;
	p->it_prof_value = cputime_zero;
	p->it_prof_incr = cputime_zero;
	init_timer(&p->real_timer);
	p->real_timer.data = (unsigned long) p;

	p->utime = cputime_zero;
	p->stime = cputime_zero;
	p->rchar = 0;		/* I/O counter: bytes read */
	p->wchar = 0;		/* I/O counter: bytes written */
	p->syscr = 0;		/* I/O counter: read syscalls */
	p->syscw = 0;		/* I/O counter: write syscalls */
	acct_clear_integrals(p);

	/**
	 * 把大内核锁计数器初始化为-1
	 */
	p->lock_depth = -1;		/* -1 = no lock */
	do_posix_clock_monotonic_gettime(&p->start_time);
	p->security = NULL;
	p->io_context = NULL;
	p->io_wait = NULL;
	p->audit_context = NULL;
#ifdef CONFIG_NUMA
 	p->mempolicy = mpol_copy(p->mempolicy);
 	if (IS_ERR(p->mempolicy)) {
 		retval = PTR_ERR(p->mempolicy);
 		p->mempolicy = NULL;
 		goto bad_fork_cleanup;
 	}
#endif

	p->tgid = p->pid;
	if (clone_flags & CLONE_THREAD)
		p->tgid = current->tgid;

	if ((retval = security_task_alloc(p)))
		goto bad_fork_cleanup_policy;
	if ((retval = audit_alloc(p)))
		goto bad_fork_cleanup_security;
	/* copy all the process information */
	/**
	 * copy_semundo，copy_files，copy_fs，copy_sighand，copy_signal
	 * copy_mm，copy_keys，copy_namespace创建新的数据结构，并把父进程相应数据结构的值复制到新数据结构中。
	 * 除非clone_flags参数指出它们有不同的值。
	 */
	if ((retval = copy_semundo(clone_flags, p)))
		goto bad_fork_cleanup_audit;
	if ((retval = copy_files(clone_flags, p)))
		goto bad_fork_cleanup_semundo;
	if ((retval = copy_fs(clone_flags, p)))
		goto bad_fork_cleanup_files;
	if ((retval = copy_sighand(clone_flags, p)))
		goto bad_fork_cleanup_fs;
	if ((retval = copy_signal(clone_flags, p)))
		goto bad_fork_cleanup_sighand;
	if ((retval = copy_mm(clone_flags, p)))
		goto bad_fork_cleanup_signal;
	if ((retval = copy_keys(clone_flags, p)))
		goto bad_fork_cleanup_mm;
	if ((retval = copy_namespace(clone_flags, p)))
		goto bad_fork_cleanup_keys;
	/**
	 * 调用copy_thread，用发出clone系统调用时CPU寄存器的值（它们保存在父进程的内核栈中）
	 * 来初始化子进程的内核栈。不过，copy_thread把eax寄存器对应字段的值（这是fork和clone系统调用在子进程中的返回值）
	 * 强行置为0。子进程描述符的thread.esp字段初始化为子进程内核栈的基地址。ret_from_fork的地址存放在thread.eip中。
	 * 如果父进程使用IO权限位图。则子进程获取该位图的一个拷贝。
	 * 最后，如果CLONE_SETTLS标志被置位，则子进程获取由CLONE系统调用的参数tls指向的用户态数据结构所表示的TLS段。
	 */
	retval = copy_thread(0, clone_flags, stack_start, stack_size, p, regs);
	if (retval)
		goto bad_fork_cleanup_namespace;

	/**
	 * 如果clone_flags参数的值被置为CLONE_CHILD_SETTID或CLONE_CHILD_CLEARTID
	 * 就把child_tidptr参数的值分别复制到set_child_tid或clear_child_tid字段。
	 * 这些标志说明：必须改变子进程用户态地址空间的dhild_tidptr所指向的变量的值
	 * 不过实际的写操作要稍后再执行。
	 */
	p->set_child_tid = (clone_flags & CLONE_CHILD_SETTID) ? child_tidptr : NULL;
	/*
	 * Clear TID on mm_release()?
	 */
	p->clear_child_tid = (clone_flags & CLONE_CHILD_CLEARTID) ? child_tidptr: NULL;

	/*
	 * Syscall tracing should be turned off in the child regardless
	 * of CLONE_PTRACE.
	 */
	/**
	 * 清除TIF_SYSCALL_TRACE标志。使ret_from_fork函数不会把系统调用结束的消息通知给调试进程。
	 * 也不应该通知给调试进程，因为子进程并没有调用fork.
	 */
	clear_tsk_thread_flag(p, TIF_SYSCALL_TRACE);

	/* Our parent execution domain becomes current domain
	   These must match for thread signalling to apply */
	   
	p->parent_exec_id = p->self_exec_id;

	/* ok, now we should be set up.. */
	/**
	 * 用clone_flags参数低位的信号数据编码统建始化tsk_exit_signal字段。
	 * 如CLONE_THREAD标志被置位，就把exit_signal字段初始化为-1。
	 * 这样做是因为：当创建线程时，即使被创建的线程死亡，都不应该给领头进程的父进程发送信号。
	 * 而应该是领头进程死亡后，才向其领头进程的父进程发送信号。
	 */
	p->exit_signal = (clone_flags & CLONE_THREAD) ? -1 : (clone_flags & CSIGNAL);
	p->pdeath_signal = 0;
	p->exit_state = 0;

	/* Perform scheduler related setup */
	/**
	 * 调用sched_fork完成对新进程调度程序数据结构的初始化。
	 * 该函数把新进程的状态置为TASK_RUNNING，并把thread_info结构的preempt_count字段设置为1，
	 * 从而禁止抢占。
	 * 此外，为了保证公平调度，父子进程共享父进程的时间片。
	 */
	sched_fork(p);

	/*
	 * Ok, make it visible to the rest of the system.
	 * We dont wake it up yet.
	 */
	p->group_leader = p;
	INIT_LIST_HEAD(&p->ptrace_children);
	INIT_LIST_HEAD(&p->ptrace_list);

	/* Need tasklist lock for parent etc handling! */
	write_lock_irq(&tasklist_lock);

	/*
	 * The task hasn't been attached yet, so cpus_allowed mask cannot
	 * have changed. The cpus_allowed mask of the parent may have
	 * changed after it was copied first time, and it may then move to
	 * another CPU - so we re-copy it here and set the child's CPU to
	 * the parent's CPU. This avoids alot of nasty races.
	 */
	p->cpus_allowed = current->cpus_allowed;
	/**
	 * 初始化子线程的cpu字段。
	 */
	set_task_cpu(p, smp_processor_id());

	/*
	 * Check for pending SIGKILL! The new thread should not be allowed
	 * to slip out of an OOM kill. (or normal SIGKILL.)
	 */
	if (sigismember(&current->pending.signal, SIGKILL)) {
		write_unlock_irq(&tasklist_lock);
		retval = -EINTR;
		goto bad_fork_cleanup_namespace;
	}

	/* CLONE_PARENT re-uses the old parent */
	/**
	 * 初始化表示亲子关系的字段，如果CLONE_PARENT或者CLONE_THREAD被设置了
	 * 就用current->real_parent初始化，否则，当前进程就是初创建进程的父进程。
	 */
	if (clone_flags & (CLONE_PARENT|CLONE_THREAD))
		p->real_parent = current->real_parent;
	else
		p->real_parent = current;
	p->parent = p->real_parent;

	if (clone_flags & CLONE_THREAD) {
		spin_lock(&current->sighand->siglock);
		/*
		 * Important: if an exit-all has been started then
		 * do not create this new thread - the whole thread
		 * group is supposed to exit anyway.
		 */
		if (current->signal->flags & SIGNAL_GROUP_EXIT) {
			spin_unlock(&current->sighand->siglock);
			write_unlock_irq(&tasklist_lock);
			retval = -EAGAIN;
			goto bad_fork_cleanup_namespace;
		}
		p->group_leader = current->group_leader;

		if (current->signal->group_stop_count > 0) {
			/*
			 * There is an all-stop in progress for the group.
			 * We ourselves will stop as soon as we check signals.
			 * Make the new thread part of that group stop too.
			 */
			current->signal->group_stop_count++;
			set_tsk_thread_flag(p, TIF_SIGPENDING);
		}

		spin_unlock(&current->sighand->siglock);
	}

	/** 
	 * 把新进程加入到进程链表
	 */
	SET_LINKS(p);

	/**
	 * PT_PTRACED表示子进程必须被跟踪，就把current->parent赋给tsk->parent，并将子进程插入调试程序的跟踪链表中。
	 */
	if (unlikely(p->ptrace & PT_PTRACED))
		__ptrace_link(p, current->parent);

	/**
	 * 把新进程描述符的PID插入pidhash散列表中。
	 */
	attach_pid(p, PIDTYPE_PID, p->pid);
	attach_pid(p, PIDTYPE_TGID, p->tgid);

	/**
	 * 如果子进程是线程组的领头进程(CLONE_THREAD标志被清0)
	 */
	if (thread_group_leader(p)) {
		/**
		 * 将进程插入相应的散列表。
		 */
		attach_pid(p, PIDTYPE_PGID, process_group(p));
		attach_pid(p, PIDTYPE_SID, p->signal->session);
		if (p->pid)
			__get_cpu_var(process_counts)++;
	}

	/**
	 * 计数
	 */
	nr_threads++;
	total_forks++;
	write_unlock_irq(&tasklist_lock);
	retval = 0;

fork_out:
	if (retval)
		return ERR_PTR(retval);
	return p;

bad_fork_cleanup_namespace:
	exit_namespace(p);
bad_fork_cleanup_keys:
	exit_keys(p);
bad_fork_cleanup_mm:
	if (p->mm)
		mmput(p->mm);
bad_fork_cleanup_signal:
	exit_signal(p);
bad_fork_cleanup_sighand:
	exit_sighand(p);
bad_fork_cleanup_fs:
	exit_fs(p); /* blocking */
bad_fork_cleanup_files:
	exit_files(p); /* blocking */
bad_fork_cleanup_semundo:
	exit_sem(p);
bad_fork_cleanup_audit:
	audit_free(p);
bad_fork_cleanup_security:
	security_task_free(p);
bad_fork_cleanup_policy:
#ifdef CONFIG_NUMA
	mpol_free(p->mempolicy);
#endif
bad_fork_cleanup:
	if (p->binfmt)
		module_put(p->binfmt->module);
bad_fork_cleanup_put_domain:
	module_put(p->thread_info->exec_domain->module);
bad_fork_cleanup_count:
	put_group_info(p->group_info);
	atomic_dec(&p->user->processes);
	free_uid(p->user);
bad_fork_free:
	free_task(p);
	goto fork_out;
}
```

做完do_fork后，我们的子进程整装待发，等待CPU调度。后续进程调度时，调度程序会把进程描述符thread字段的值装入几个CPU寄存器（thread.esp装入esp寄存器，ret_from_fork()地址装入eip寄存器）。关于此，可以参考《Linux进程切换》一文，这一处理刚好和_switch_to的设计一致。

## 内核线程
Linux还有一组内核线程，它们只运行在内核态，因此也就不必受用户态上下文的拖累。内核线程只使用大于PAGE_OFFSET的内核地址空间。

创建内核线程使用kernel_thread()，实际上本质依然是do_fork():
```cpp
/**
 * 创建一个新的内核线程
 * fn-要执行的内核函数的地址。
 * arg-要传递给函数的参数
 * flags-一组clone标志
 */
int kernel_thread(int (*fn)(void *), void * arg, unsigned long flags)
{
	struct pt_regs regs;

	memset(&regs, 0, sizeof(regs));

	/**
	 * 内核栈地址，为其赋初值。
	 * do_fork将从这里取值来为新线程初始化CPU。
	 */
	regs.ebx = (unsigned long) fn;
	regs.edx = (unsigned long) arg;

	regs.xds = __USER_DS;
	regs.xes = __USER_DS;
	regs.orig_eax = -1;
	/**
	 * 把eip设置成kernel_thread_helper，这样，新线程将执行fn函数。如果函数结束，将执行do_exit
	 * fn的返回值作为do_exit的参数。
	 */
	regs.eip = (unsigned long) kernel_thread_helper;
	regs.xcs = __KERNEL_CS;
	regs.eflags = X86_EFLAGS_IF | X86_EFLAGS_SF | X86_EFLAGS_PF | 0x2;

	/* Ok, create the new process.. */
	/**
	 * CLONE_VM避免复制调用进程的页表。由于新的内核线程无论如何都不会访问用户态地址空间。
	 * 所以复制只会造成时间和空间的浪费。
	 * CLONE_UNTRACED标志保证内核线程不会被跟踪，即使调用进程被跟踪。
	 */
	return do_fork(flags | CLONE_VM | CLONE_UNTRACED, 0, &regs, 0, NULL, NULL);
}
```
不管你怎么折腾，说白了还是进程，是进程就得do_fork，设计用途不一致罢了。

## 进程0
一直在折腾clone，但总归有第一个进程，那就是进程0——idle或叫swapper。这是个Linux开天辟地时捏出来的内核线程。他使用的数据结构是静态分配的，换句话说，其他所有进程的数据结构都是动态分配的。
- init_task变量中存放进程描述符，INIT_TASK宏初始化。
```cpp
/**
 * 进程0的描述符, 也是进程链表的头
 */
struct task_struct init_task = INIT_TASK(init_task);

/**
 * 初始化进程0的任务描述符。
 */
#define INIT_TASK(tsk)	\
{									\
	.state		= 0,						\
	.thread_info	= &init_thread_info,				\
	.usage		= ATOMIC_INIT(2),				\
	.flags		= 0,						\
	.lock_depth	= -1,						\
	.prio		= MAX_PRIO-20,					\
	.static_prio	= MAX_PRIO-20,					\
	.policy		= SCHED_NORMAL,					\
	.cpus_allowed	= CPU_MASK_ALL,					\
	.mm		= NULL,						\
	.active_mm	= &init_mm,					\
	.run_list	= LIST_HEAD_INIT(tsk.run_list),			\
	.time_slice	= HZ,						\
	.tasks		= LIST_HEAD_INIT(tsk.tasks),			\
	.ptrace_children= LIST_HEAD_INIT(tsk.ptrace_children),		\
	.ptrace_list	= LIST_HEAD_INIT(tsk.ptrace_list),		\
	.real_parent	= &tsk,						\
	.parent		= &tsk,						\
	.children	= LIST_HEAD_INIT(tsk.children),			\
	.sibling	= LIST_HEAD_INIT(tsk.sibling),			\
	.group_leader	= &tsk,						\
	.real_timer	= {						\
		.function	= it_real_fn				\
	},								\
	.group_info	= &init_groups,					\
	.cap_effective	= CAP_INIT_EFF_SET,				\
	.cap_inheritable = CAP_INIT_INH_SET,				\
	.cap_permitted	= CAP_FULL_SET,					\
	.keep_capabilities = 0,						\
	.user		= INIT_USER,					\
	.comm		= "swapper",					\
	.thread		= INIT_THREAD,					\
	.fs		= &init_fs,					\
	.files		= &init_files,					\
	.signal		= &init_signals,				\
	.sighand	= &init_sighand,				\
	.pending	= {						\
		.list = LIST_HEAD_INIT(tsk.pending.list),		\
		.signal = {{0}}},					\
	.blocked	= {{0}},					\
	.alloc_lock	= SPIN_LOCK_UNLOCKED,				\
	.proc_lock	= SPIN_LOCK_UNLOCKED,				\
	.switch_lock	= SPIN_LOCK_UNLOCKED,				\
	.journal_info	= NULL,						\
}
```
- init_thread_union中存放thread_info描述符和内核堆栈，INIT_THREAD_INFO宏完成初始化。
```cpp
#define init_thread_info	(init_thread_union.thread_info)

#define INIT_THREAD_INFO(tsk)			\
{						\
	.task		= &tsk,			\
	.exec_domain	= &default_exec_domain,	\
	.flags		= 0,			\
	.cpu		= 0,			\
	.preempt_count	= 1,			\
	.addr_limit	= KERNEL_DS,		\
	.restart_block = {			\
		.fn = do_no_restart_syscall,	\
	},					\
}
```
- 由进程描述符指向的下列表，也有对应宏初始化：
	- init_mm 	INIT_MM
	- init_fs	INIT_FS
	- init_files	INIT_FILES
	- init_signals	INIT_SIGNALS
	- init_sighand	INIT_SIGHAND
- 主内核页全局目录放在swapper_pg_dir中。

start_kernel()初始化内核需要的所有数据结构，激活中断，创建另一个叫进程l的内核线程（一般叫init进程）：
`kernel_thread(init, NULL, CLONE_FS|CLONE_SIGHAND);`
新创建内核线程的PID为1，并与进程0共享每进程所有的内核数据结构。此外，当调度程序选择到它时，init进程开始执行init()函数。

创建init进程后，进程0执行cpu_idle()函数，该函数本质上是在开中断的情况下重复执行hlt汇编语言指令。只有当没有其他进程处于TASK_RUNNING状态时，调度程序才选择进程0。

## 进程1
0创建的内核线程执行init()，init()完成内核初始化。init()调用execve()系统调用装入可执行程序init。然后init内核线程变为一个普通进程，且拥有自己的每进程内核数据结构。系统关闭之前，init进程一直存活，因为它创建和监控在OS外层执行的所有进程的活动。

linux还有很多其他内核线程，到具体模块时再看。

## 撤销进程
进程终止时，必须通知内核释放进程的资源，包括内存、打开文件以及其他零碎的东西，比如信号量。进程终止一般通过调用exit()库函数，该函数释放C函数库所分配的资源，执行编程者所注册的每个函数，并结束从系统回收进程的那个系统调用。exit()可以显示的插入，C编译程序总是把exit()插入到main()的最后一条语句之后。

exit_group()和exit()都可以终止进程，前者终止的是整个线程组。do_group_exit()内核函数实现了这个系统调用，它对应C库的exit()。后者终止某一个线程，而不管线程所属线程组中的所有其他线程。do_exit()是实现这个系统调用的内核函数，它对应Linux线程库的pthread_exit()。

> 注意区分C库的exit()和系统调用exit()。

exit()的处理：
```cpp
asmlinkage long sys_exit(int error_code)
{
	do_exit((error_code&0xff)<<8);
}

/**
 * 所有进程的终止都是本函数处理的。
 * 它从内核数据结构中删除对终止进程的大部分引用（注：不是全部，进程描述符就不是）
 * 它接受进程的终止代码作为参数。
 */
fastcall NORET_TYPE void do_exit(long code)
{
	struct task_struct *tsk = current;
	int group_dead;

	profile_task_exit(tsk);

	if (unlikely(in_interrupt()))
		panic("Aiee, killing interrupt handler!");
	if (unlikely(!tsk->pid))
		panic("Attempted to kill the idle task!");
	if (unlikely(tsk->pid == 1))
		panic("Attempted to kill init!");
	if (tsk->io_context)
		exit_io_context();

	if (unlikely(current->ptrace & PT_TRACE_EXIT)) {
		current->ptrace_message = code;
		ptrace_notify((PTRACE_EVENT_EXIT << 8) | SIGTRAP);
	}

	/**
	 * PF_EXITING表示进程的状态：正在被删除。
	 */
	tsk->flags |= PF_EXITING;
	/**
	 * 从动态定时器队列中删除进程描述符。
	 */
	del_timer_sync(&tsk->real_timer);

	if (unlikely(in_atomic()))
		printk(KERN_INFO "note: %s[%d] exited with preempt_count %d\n",
				current->comm, current->pid,
				preempt_count());

	acct_update_integrals();
	update_mem_hiwater();
	group_dead = atomic_dec_and_test(&tsk->signal->live);
	if (group_dead)
		acct_process(code);

	/**
	 * exit_mm从进程描述符中分离出分页相关的描述符。
	 * 如果没有其他进程共享这些数据结构，就删除这些数据结构。
	 */
	exit_mm(tsk);

	/**
	 * exit_sem从进程描述符中分离出信号量相关的描述符
	 */
	exit_sem(tsk);
	/**
	 * __exit_files从进程描述符中分离出文件系统相关的描述符
	 */
	__exit_files(tsk);
	/**
	 * __exit_fs从进程描述符中分离出打开文件描述符相关的描述符
	 */
	__exit_fs(tsk);
	/**
	 * exit_namespace从进程描述符中分离出命名空间相关的描述符
	 */	
	exit_namespace(tsk);
	/**
	 * exit_thread从进程描述符中分离出IO权限位图相关的描述符
	 */		
	exit_thread();
	exit_keys(tsk);

	if (group_dead && tsk->signal->leader)
		disassociate_ctty(1);

	/**
	 * 如果实现了被杀死进程的执行域和可执行格式的内核函数在内核模块中
	 * 就递减它们的值。
	 * 注：这应该是为了防止意外的卸载模块。
	 */
	module_put(tsk->thread_info->exec_domain->module);
	if (tsk->binfmt)
		module_put(tsk->binfmt->module);

	/**
	 * 设置退出代码
	 */
	tsk->exit_code = code;
	/**
	 * exit_notify执行比较复杂的操作，更新了很多内核数据结构
	 */
	exit_notify(tsk);
#ifdef CONFIG_NUMA
	mpol_free(tsk->mempolicy);
	tsk->mempolicy = NULL;
#endif

	BUG_ON(!(current->flags & PF_DEAD));
	/**
	 * 完了，让其他线程运行吧
	 * 因为schedule会忽略处于EXIT_ZOMBIE状态的进程，所以进程现在是不会再运行了。
	 */
	schedule();
	/**
	 * 当然，谁还会让死掉的进程继续运行，说明内核一定是错了
	 * 注：难道schedule被谁改了，没有判断EXIT_ZOMBIE？？？
	 */
	BUG();
	/* Avoid "noreturn function does return".  */
	/**
	 * 仅仅为了防止编译器报警告信息而已，仅此而已。
	 */
	for (;;) ;
}
```

再看exit_group():
```cpp
asmlinkage void sys_exit_group(int error_code)
{
	do_group_exit((error_code & 0xff) << 8);
}

/**
 * 杀死属于current线程组的所有进程.它接受进程终止代码作为参数.
 * 这个参数可能是系统调用exit_group()指定的一个值,也可能是内核提供的一个错误代号.
 */
NORET_TYPE void
do_group_exit(int exit_code)
{
	BUG_ON(exit_code & 0x80); /* core dumps don't get here */

	/**
	 * 检查进程的SIGNAL_GROUP_EXIT,如果不为0,说明内核已经开始为线程组执行退出的过程.
	 */
	if (current->signal->flags & SIGNAL_GROUP_EXIT)
		exit_code = current->signal->group_exit_code;
	else if (!thread_group_empty(current)) {
		/**
		 * 设置进程的SIGNAL_GROUP_EXIT标志,并把终止代号放在sig->group_exit_code
		 */
		struct signal_struct *const sig = current->signal;
		struct sighand_struct *const sighand = current->sighand;
		read_lock(&tasklist_lock);
		spin_lock_irq(&sighand->siglock);
		if (sig->flags & SIGNAL_GROUP_EXIT)
			/* Another thread got here before we took the lock.  */
			exit_code = sig->group_exit_code;
		else {
			sig->flags = SIGNAL_GROUP_EXIT;
			sig->group_exit_code = exit_code;
			/**
			 * zap_other_threads杀死线程组中的其他线程.
			 * 它扫描PIDTYPE_TGID类型的散列表中的每个PID链表,向表中其他进程发送SIGKILL信号.
			 */
			zap_other_threads(current);
		}
		spin_unlock_irq(&sighand->siglock);
		read_unlock(&tasklist_lock);
	}

	/**
	 * 杀死当前进程,此过程不再返回.
	 */
	do_exit(exit_code);
	/* NOTREACHED */
}
```

## 进程删除
Linux的子进程可以通过查询内核获取父进程的PID，或者它的子进程的执行状态。考虑到这一设计的完整性，Linux不允许内核在进程终止后直接丢弃包含在进程描述符字段中的数据，而是当父进程发出了与被终止的进程相关的wait()系列系统调用后，才可以这样做。这就是引入僵死状态的原因（参考《Linux进程概述》）。

如果父进程死在子进程前，那么所有孤儿都交给init。init在用wait()系列系统调用检查到其终止时，撤销僵死的进程。

负责干活的是release_task()函数，他从僵死进程的描述符中分离出最后的数据结构；对僵死进程的处理有两种可能的方式：如果父进程不需要接收子进程的信号，就调用do_exit()；如果已经给父进程发了新号，就调用wait4()或waitpid()系统调用。后者中函数还会回收进程描述符所占用的内存空间，而前者中内存的回收由进程调度程序完成。

```cpp
/**
 * 释放进程描述符。如果进程已经是僵死状态，就会回收它占用的RAM。
 */
void release_task(struct task_struct * p)
{
	int zap_leader;
	task_t *leader;
	struct dentry *proc_dentry;

repeat: 
	/**
	 * 递减进程拥有者的进程个数。
	 */
	atomic_dec(&p->user->processes);
	spin_lock(&p->proc_lock);
	proc_dentry = proc_pid_unhash(p);
	write_lock_irq(&tasklist_lock);
	/**
	 * 如果进程正被跟踪，函数将它从调试程序的ptrace_children链表中删除，并让该进程重新属于初始的父进程。
	 */
	if (unlikely(p->ptrace))
		__ptrace_unlink(p);
	BUG_ON(!list_empty(&p->ptrace_list) || !list_empty(&p->ptrace_children));
	/**
	 * 删除所有的挂起信号并释放进程的signal_struct描述符。
	 * 如果该描述符不再被其他的轻量级进程使用，函数进一步删除这个数据结构。
	 * 它还会调用exit_itimers，删除所有POSIX时间间隔定时器。
	 */
	__exit_signal(p);
	/**
	 * 删除信号处理函数。
	 */
	__exit_sighand(p);
	/**
	 * __unhash_process将进程从各种hash表中摘除。它执行:
	 *    变量nr_threads减1
	 *    两次调用detach_pid，分别从PIDTYPE_PID和PIDTYPE_TGID类型的PID散列表中删除进程描述符。
	 *    如果进程是线程组的领头进程，那么再调用两次detach_pid，从PIDTYPE_PGID和PIDTYPE_SID类型的散列表中删除进程描述符。
	 *    用宏REMOVE_LINKS从进程链表中解除进程描述符的链接。
	 */
	__unhash_process(p);

	/*
	 * If we are the last non-leader member of the thread
	 * group, and the leader is zombie, then notify the
	 * group leader's parent process. (if it wants notification.)
	 */
	zap_leader = 0;
	leader = p->group_leader;
	/**
	 * 如果进程不是线程组的领头进程，领头进程处于僵死状态，并且进程是线程组的最后一个成员。
	 */
	if (leader != p && thread_group_empty(leader) && leader->exit_state == EXIT_ZOMBIE) {
		BUG_ON(leader->exit_signal == -1);
		/**
		 * 向领头进程的父进程发送一个信号，通知它:进程已经死亡。
		 */
		do_notify_parent(leader, leader->exit_signal);
		/*
		 * If we were the last child thread and the leader has
		 * exited already, and the leader's parent ignores SIGCHLD,
		 * then we are the one who should release the leader.
		 *
		 * do_notify_parent() will have marked it self-reaping in
		 * that case.
		 */
		zap_leader = (leader->exit_signal == -1);
	}

	/**
	 * sched_exit函数调整父进程的时间片。
	 */
	sched_exit(p);
	write_unlock_irq(&tasklist_lock);
	spin_unlock(&p->proc_lock);
	proc_pid_flush(proc_dentry);
	release_thread(p);
	/**
	 * 递减进程描述符的使用计数器。如果计数器变成0,则终止所有残留的对进程的引用。
	 *     递减进程所有者的user_struct数据结构的使用计数器。如果计数为0，就释放该结构。
	 *     释放进程描述符以及thread_info描述符和内核态堆栈所占用的内存区域。
	 */
	put_task_struct(p);

	p = leader;
	if (unlikely(zap_leader))
		goto repeat;
}
```

进程终止的处理函数：
```cpp
/**
 * 所有进程的终止都是本函数处理的。
 * 它从内核数据结构中删除对终止进程的大部分引用（注：不是全部，进程描述符就不是）
 * 它接受进程的终止代码作为参数。
 */
fastcall NORET_TYPE void do_exit(long code)
{
	struct task_struct *tsk = current;
	int group_dead;

	profile_task_exit(tsk);

	if (unlikely(in_interrupt()))
		panic("Aiee, killing interrupt handler!");
	if (unlikely(!tsk->pid))
		panic("Attempted to kill the idle task!");
	if (unlikely(tsk->pid == 1))
		panic("Attempted to kill init!");
	if (tsk->io_context)
		exit_io_context();

	if (unlikely(current->ptrace & PT_TRACE_EXIT)) {
		current->ptrace_message = code;
		ptrace_notify((PTRACE_EVENT_EXIT << 8) | SIGTRAP);
	}

	/**
	 * PF_EXITING表示进程的状态：正在被删除。
	 */
	tsk->flags |= PF_EXITING;
	/**
	 * 从动态定时器队列中删除进程描述符。
	 */
	del_timer_sync(&tsk->real_timer);

	if (unlikely(in_atomic()))
		printk(KERN_INFO "note: %s[%d] exited with preempt_count %d\n",
				current->comm, current->pid,
				preempt_count());

	acct_update_integrals();
	update_mem_hiwater();
	group_dead = atomic_dec_and_test(&tsk->signal->live);
	if (group_dead)
		acct_process(code);

	/**
	 * exit_mm从进程描述符中分离出分页相关的描述符。
	 * 如果没有其他进程共享这些数据结构，就删除这些数据结构。
	 */
	exit_mm(tsk);

	/**
	 * exit_sem从进程描述符中分离出信号量相关的描述符
	 */
	exit_sem(tsk);
	/**
	 * __exit_files从进程描述符中分离出文件系统相关的描述符
	 */
	__exit_files(tsk);
	/**
	 * __exit_fs从进程描述符中分离出打开文件描述符相关的描述符
	 */
	__exit_fs(tsk);
	/**
	 * exit_namespace从进程描述符中分离出命名空间相关的描述符
	 */	
	exit_namespace(tsk);
	/**
	 * exit_thread从进程描述符中分离出IO权限位图相关的描述符
	 */		
	exit_thread();
	exit_keys(tsk);

	if (group_dead && tsk->signal->leader)
		disassociate_ctty(1);

	/**
	 * 如果实现了被杀死进程的执行域和可执行格式的内核函数在内核模块中
	 * 就递减它们的值。
	 * 注：这应该是为了防止意外的卸载模块。
	 */
	module_put(tsk->thread_info->exec_domain->module);
	if (tsk->binfmt)
		module_put(tsk->binfmt->module);

	/**
	 * 设置退出代码
	 */
	tsk->exit_code = code;
	/**
	 * exit_notify执行比较复杂的操作，更新了很多内核数据结构
	 */
	exit_notify(tsk);
#ifdef CONFIG_NUMA
	mpol_free(tsk->mempolicy);
	tsk->mempolicy = NULL;
#endif

	BUG_ON(!(current->flags & PF_DEAD));
	/**
	 * 完了，让其他线程运行吧
	 * 因为schedule会忽略处于EXIT_ZOMBIE状态的进程，所以进程现在是不会再运行了。
	 */
	schedule();
	/**
	 * 当然，谁还会让死掉的进程继续运行，说明内核一定是错了
	 * 注：难道schedule被谁改了，没有判断EXIT_ZOMBIE？？？
	 */
	BUG();
	/* Avoid "noreturn function does return".  */
	/**
	 * 仅仅为了防止编译器报警告信息而已，仅此而已。
	 */
	for (;;) ;
}
```

## 参考文献
- [gcc中的weak和alias](http://www.lenky.info/archives/2013/01/2195)
- [Linux中fork，vfork和clone详解](http://blog.csdn.net/gatieme/article/details/51417488)
- [Linux系统调用：exit()与_exit()函数详解](http://blog.csdn.net/drdairen/article/details/51896141)
- 《深入理解Linux内核》
- [ULK Chinese comments](https://github.com/sohu2000000/ULK)