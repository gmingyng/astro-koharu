---
title: Linux源码5.15学习——当我们fork时发生了什么
link: linux515-fork
catalog: true
date: 2024-02-20 20:39:00
description: 学习linux5.15的fork源码
tags:
  - infra
  - linux
  - fork
categories:
  - Infra
---
<h2>fork代码</h2>

```c
"linux/kernel/fork.c"

#ifdef __ARCH_WANT_SYS_FORK
SYSCALL_DEFINE0(fork)
{
#ifdef CONFIG_MMU
	struct kernel_clone_args args = {
		.exit_signal = SIGCHLD,
	};

	return kernel_clone(&args);
#else
	/* can not support in nommu mode */
	return -EINVAL;
#endif
}
#endif
```

可以看到 `fork` 实际上是调用了 `kernel_clone` 函数，我们对其展开。

<h3>kernel_clone</h3>

```c
"linux/kernel/fork.c"

/*
 *  Ok, this is the main fork-routine.
 *
 * It copies the process, and if successful kick-starts
 * it and waits for it to finish using the VM if required.
 *
 * args->exit_signal is expected to be checked for sanity by the caller.
 */
pid_t kernel_clone(struct kernel_clone_args *args)
{
	u64 clone_flags = args->flags;
	struct completion vfork;
	struct pid *pid;
	struct task_struct *p;
	int trace = 0;
	pid_t nr;

	/*
	 * For legacy clone() calls, CLONE_PIDFD uses the parent_tid argument
	 * to return the pidfd. Hence, CLONE_PIDFD and CLONE_PARENT_SETTID are
	 * mutually exclusive. With clone3() CLONE_PIDFD has grown a separate
	 * field in struct clone_args and it still doesn't make sense to have
	 * them both point at the same memory location. Performing this check
	 * here has the advantage that we don't need to have a separate helper
	 * to check for legacy clone().
	 */
	if ((args->flags & CLONE_PIDFD) &&
	    (args->flags & CLONE_PARENT_SETTID) &&
	    (args->pidfd == args->parent_tid))
		return -EINVAL;

	/*
	 * Determine whether and which event to report to ptracer.  When
	 * called from kernel_thread or CLONE_UNTRACED is explicitly
	 * requested, no event is reported; otherwise, report if the event
	 * for the type of forking is enabled.
	 */
	if (!(clone_flags & CLONE_UNTRACED)) {
		if (clone_flags & CLONE_VFORK)
			trace = PTRACE_EVENT_VFORK;
		else if (args->exit_signal != SIGCHLD)
			trace = PTRACE_EVENT_CLONE;
		else
			trace = PTRACE_EVENT_FORK;

		if (likely(!ptrace_event_enabled(current, trace)))
			trace = 0;
	}

	p = copy_process(NULL, trace, NUMA_NO_NODE, args);
	add_latent_entropy();

	if (IS_ERR(p))
		return PTR_ERR(p);

	/*
	 * Do this prior waking up the new thread - the thread pointer
	 * might get invalid after that point, if the thread exits quickly.
	 */
	trace_sched_process_fork(current, p);

	pid = get_task_pid(p, PIDTYPE_PID);
	nr = pid_vnr(pid);

	if (clone_flags & CLONE_PARENT_SETTID)
		put_user(nr, args->parent_tid);

	if (clone_flags & CLONE_VFORK) {
		p->vfork_done = &vfork;
		init_completion(&vfork);
		get_task_struct(p);
	}

	wake_up_new_task(p);

	/* forking complete and child started to run, tell ptracer */
	if (unlikely(trace))
		ptrace_event_pid(trace, pid);

	if (clone_flags & CLONE_VFORK) {
		if (!wait_for_vfork_done(p, &vfork))
			ptrace_event_pid(PTRACE_EVENT_VFORK_DONE, pid);
	}

	put_pid(pid);
	return nr;
}
```

1. 首先用 `copy_process` 进行对父进程的上下文复制，内存空间拷贝(如果是 fork 调用则拷贝且为写时复制)等操作。
2. `trace_sched_process_fork` 跟踪新任务的创建事件，并建议在唤醒新线程前做这件事，防止进程退出太快导致访问无效指针。
3. `wake_up_new_task` 将任务放入请求队列并激活。

<h3>copy_process</h3>

`copy_process`完整代码很长，这里仅截取关键操作。

<h4>函数声明</h4>

```c
"linux/kernel/fork.c"

static __latent_entropy struct task_struct *copy_process(
					struct pid *pid,
					int trace,
					int node,
					struct kernel_clone_args *args)

```

<h4>flag判断</h4>
对各类标志位进行判断，检测新建立的进程(线程)是否符合要求。

<h4>延迟信号</h4>

```c
	/*
	 * Force any signals received before this point to be delivered
	 * before the fork happens.  Collect up signals sent to multiple
	 * processes that happen during the fork and delay them so that
	 * they appear to happen after the fork.
	 */
	sigemptyset(&delayed.signal);
	INIT_HLIST_NODE(&delayed.node);

	spin_lock_irq(&current->sighand->siglock);
	if (!(clone_flags & CLONE_THREAD))
		hlist_add_head(&delayed.node, &current->signal->multiprocess);
	recalc_sigpending();
	spin_unlock_irq(&current->sighand->siglock);
	retval = -ERESTARTNOINTR;
	if (task_sigpending(current))
		goto fork_out;

	retval = -ENOMEM;
	p = dup_task_struct(current, node);
	if (!p)
		goto fork_out;
	if (args->io_thread) {
		/*
		 * Mark us an IO worker, and block any signal that isn't
		 * fatal or STOP
		 */
		p->flags |= PF_IO_WORKER;
		siginitsetinv(&p->blocked, sigmask(SIGKILL)|sigmask(SIGSTOP之外的所有信号。));
	}
```

linux 强制在此点之前接收到的任何信号在 `fork` 发生之前就被处理，同时收集并延迟 `fork` 期间收到的信号，使他们看起来像是在 `fork` 之后发生的。

1. 首先用 `sigemptyset` 初始化一个信号集，随后用 `INIT_HLIST_NODE` 初始化一个哈希链表节点。
2. `if (!(clone_flags & CLONE_THREAD))` 代表如果不是在进行线程创建(即为进程创建)，则将延迟的信号节点加入当前进程的信号链表中。
3. `dup_task_struct` 复制当前进程(线程)的任务结构。
4. `if (args->io_thread)` 判断如果为处理 IO 的 thread，则标记为 `IO_WORKER`，同时初始化阻塞信号集，并屏蔽除 `SIGKILL` 和 `SIGSTOP` 之外的所有信号。

<h4>初始化</h4>

```c
	/*
	 * This _must_ happen before we call free_task(), i.e. before we jump
	 * to any of the bad_fork_* labels. This is to avoid freeing
	 * p->set_child_tid which is (ab)used as a kthread's data pointer for
	 * kernel threads (PF_KTHREAD).
	 */
	p->set_child_tid = (clone_flags & CLONE_CHILD_SETTID) ? args->child_tid : NULL;
	/*
	 * Clear TID on mm_release()?
	 */
	p->clear_child_tid = (clone_flags & CLONE_CHILD_CLEARTID) ? args->child_tid : NULL;

	ftrace_graph_init_task(p);

	rt_mutex_init_task(p);

	lockdep_assert_irqs_enabled();
#ifdef CONFIG_PROVE_LOCKING
	DEBUG_LOCKS_WARN_ON(!p->softirqs_enabled);
#endif
	retval = copy_creds(p, clone_flags);
	if (retval < 0)
		goto bad_fork_free;

	retval = -EAGAIN;
	if (is_ucounts_overlimit(task_ucounts(p), UCOUNT_RLIMIT_NPROC, rlimit(RLIMIT_NPROC))) {
		if (p->real_cred->user != INIT_USER &&
		    !capable(CAP_SYS_RESOURCE) && !capable(CAP_SYS_ADMIN))
			goto bad_fork_cleanup_count;
	}
	current->flags &= ~PF_NPROC_EXCEEDED;
```

1. 根据标志位设置`set_child_tid`与`clear_child_tid`为提供的 tid 或者 null
2. `copy_creds`复制当前进程的凭据（credentials）到新创建的进程或线程的任务结构体（`task_struct`）中，其中包括 thread 的用户 id 和组 id 等。
3. `if (is_ucounts_overlimit(task_ucounts(p), UCOUNT_RLIMIT_NPROC, rlimit(RLIMIT_NPROC)))`检测进程数是否超过限制，如果超过限制，并且创建任务的用户既不是初始用户也不具备系统资源（`CAP_SYS_RESOURCE`）或系统管理员（`CAP_SYS_ADMIN`）能力，则跳转至`bad_fork_cleanup_count`进行清理。

```c
	/*
	 * If multiple threads are within copy_process(), then this check
	 * triggers too late. This doesn't hurt, the check is only there
	 * to stop root fork bombs.
	 */
	retval = -EAGAIN;
	if (data_race(nr_threads >= max_threads))
		goto bad_fork_cleanup_count;

	delayacct_tsk_init(p);	/* Must remain after dup_task_struct() */
	p->flags &= ~(PF_SUPERPRIV | PF_WQ_WORKER | PF_IDLE | PF_NO_SETAFFINITY);
	p->flags |= PF_FORKNOEXEC;
	INIT_LIST_HEAD(&p->children);
	INIT_LIST_HEAD(&p->sibling);
	rcu_copy_process(p);
	p->vfork_done = NULL;
	spin_lock_init(&p->alloc_lock);

	init_sigpending(&p->pending);

	p->utime = p->stime = p->gtime = 0;
#ifdef CONFIG_ARCH_HAS_SCALED_CPUTIME
	p->utimescaled = p->stimescaled = 0;
#endif
	prev_cputime_init(&p->prev_cputime);

#ifdef CONFIG_VIRT_CPU_ACCOUNTING_GEN
	seqcount_init(&p->vtime.seqcount);
	p->vtime.starttime = 0;
	p->vtime.state = VTIME_INACTIVE;
#endif

#ifdef CONFIG_IO_URING
	p->io_uring = NULL;
#endif

#if defined(SPLIT_RSS_COUNTING)
	memset(&p->rss_stat, 0, sizeof(p->rss_stat));
#endif

	p->default_timer_slack_ns = current->timer_slack_ns;

#ifdef CONFIG_PSI
	p->psi_flags = 0;
#endif

	task_io_accounting_init(&p->ioac);
	acct_clear_integrals(p);

	posix_cputimers_init(&p->posix_cputimers);

	p->io_context = NULL;
	audit_set_context(p, NULL);
	cgroup_fork(p);
#ifdef CONFIG_NUMA
	p->mempolicy = mpol_dup(p->mempolicy);
	if (IS_ERR(p->mempolicy)) {
		retval = PTR_ERR(p->mempolicy);
		p->mempolicy = NULL;
		goto bad_fork_cleanup_threadgroup_lock;
	}
#endif
#ifdef CONFIG_CPUSETS
	p->cpuset_mem_spread_rotor = NUMA_NO_NODE;
	p->cpuset_slab_spread_rotor = NUMA_NO_NODE;
	seqcount_spinlock_init(&p->mems_allowed_seq, &p->alloc_lock);
#endif
#ifdef CONFIG_TRACE_IRQFLAGS
	memset(&p->irqtrace, 0, sizeof(p->irqtrace));
	p->irqtrace.hardirq_disable_ip	= _THIS_IP_;
	p->irqtrace.softirq_enable_ip	= _THIS_IP_;
	p->softirqs_enabled		= 1;
	p->softirq_context		= 0;
#endif

	p->pagefault_disabled = 0;

#ifdef CONFIG_LOCKDEP
	lockdep_init_task(p);
#endif

#ifdef CONFIG_DEBUG_MUTEXES
	p->blocked_on = NULL; /* not blocked yet */
#endif
#ifdef CONFIG_BCACHE
	p->sequential_io	= 0;
	p->sequential_io_avg	= 0;
#endif
#ifdef CONFIG_BPF_SYSCALL
	RCU_INIT_POINTER(p->bpf_storage, NULL);
	p->bpf_ctx = NULL;
#endif

	/* Perform scheduler related setup. Assign this task to a CPU. */
	retval = sched_fork(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_policy;

	retval = perf_event_init_task(p, clone_flags);
	if (retval)
		goto bad_fork_cleanup_policy;
	retval = audit_alloc(p);
	if (retval)
		goto bad_fork_cleanup_perf;
```

1. `if (data_race(nr_threads >= max_threads))`首先检查 thread 数量是否超限。
2. `rcu_copy_process`
3. 初始化`task_struct`成员，包括信号挂起列表，各类计数器，各类标志位等等
4. `sched_fork`初始化调度器。
5. 设置内核子系统：`perf_event_init_task`设置性能事件，`audit_alloc`设置审计框架。


<h4>复制操作</h4>

```c
	/* copy all the process information */
	shm_init_task(p);
	retval = security_task_alloc(p, clone_flags);
	if (retval)
		goto bad_fork_cleanup_audit;
	retval = copy_semundo(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_security;
	retval = copy_files(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_semundo;
	retval = copy_fs(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_files;
	retval = copy_sighand(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_fs;
	retval = copy_signal(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_sighand;
	retval = copy_mm(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_signal;
	retval = copy_namespaces(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_mm;
	retval = copy_io(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_namespaces;
	retval = copy_thread(clone_flags, args->stack, args->stack_size, p, args->tls);
	if (retval)
		goto bad_fork_cleanup_io;

	stackleak_task_init(p);

	if (pid != &init_struct_pid) {
		pid = alloc_pid(p->nsproxy->pid_ns_for_children, args->set_tid,
				args->set_tid_size);
		if (IS_ERR(pid)) {
			retval = PTR_ERR(pid);
			goto bad_fork_cleanup_thread;
		}
	}
```

1. `shm_init_task`初始化对 System V 共享内存的访问
2. `security_task_alloc`、`copy_semundo`、`copy_files`、`copy_fs`、`copy_sighand`、`copy_signal`、`copy_mm`、`copy_namespaces`、`copy_io`分别进行对安全、信号量撤销、打开的文件描述符、文件系统、信号处理器、信号状态、内存管理、命名空间、IO 上下文相关信息的复制。
3. `copy_thread`设置新线程的栈、栈大小以及线程局部存储（TLS）。
4. `stackleak_task_init`初始化新 thread 的内核栈，stackleak 是一种安全特性。
5. `alloc_pid`先检查`pid != &init_struct_pid`即新进程(线程)不为初始进程, 之后分配 pid 给 thread。

<h3>trace_sched_process_fork</h3>
跟踪新任务的创建事件，并建议在唤醒新线程前做这件事，防止进程退出太快导致访问无效指针。


<h3>wake_up_new_task</h3>

```c
"linux/kernel/sched/core.c"

/*
 * wake_up_new_task - wake up a newly created task for the first time.
 *
 * This function will do some initial scheduler statistics housekeeping
 * that must be done for every newly created context, then puts the task
 * on the runqueue and wakes it.
 */
void wake_up_new_task(struct task_struct *p)
{
	struct rq_flags rf;
	struct rq *rq;

	raw_spin_lock_irqsave(&p->pi_lock, rf.flags);
	WRITE_ONCE(p->__state, TASK_RUNNING);
#ifdef CONFIG_SMP
	/*
	 * Fork balancing, do it here and not earlier because:
	 *  - cpus_ptr can change in the fork path
	 *  - any previously selected CPU might disappear through hotplug
	 *
	 * Use __set_task_cpu() to avoid calling sched_class::migrate_task_rq,
	 * as we're not fully set-up yet.
	 */
	p->recent_used_cpu = task_cpu(p);
	rseq_migrate(p);
	__set_task_cpu(p, select_task_rq(p, task_cpu(p), WF_FORK));
#endif
	rq = __task_rq_lock(p, &rf);
	update_rq_clock(rq);
	post_init_entity_util_avg(p);

	activate_task(rq, p, ENQUEUE_NOCLOCK);
	trace_sched_wakeup_new(p);
	check_preempt_curr(rq, p, WF_FORK);
#ifdef CONFIG_SMP
	if (p->sched_class->task_woken) {
		/*
		 * Nothing relies on rq->lock after this, so it's fine to
		 * drop it.
		 */
		rq_unpin_lock(rq, &rf);
		p->sched_class->task_woken(rq, p);
		rq_repin_lock(rq, &rf);
	}
#endif
	task_rq_unlock(rq, p, &rf);
}
```

1. 定义局部变量：
   - `struct rq_flags rf;` 用于存储请求队列（runqueue）的标志状态。
   - `struct rq *rq;` 是指向请求队列的指针，请求队列用于存储和管理待运行任务。
2. `WRITE_ONCE` 写入新建任务状态为 `TASK_RUNNING`。
3. 请求队列初始化：
	- `rq = __task_rq_lock(p, &rf);` 上锁
	- `update_rq_clock(rq)` 更新时钟信息
	- `post_init_entity_util_avg(p);` 初始化任务的平均利用率统计信息，为调度器的负载平衡（load balancing）和任务选择提供数据。
4. 激活任务：
	- `activate_task(rq, p, ENQUEUE_NOCLOCK);` 将新任务放入请求队列并准备调度。
	- `trace_sched_wakeup_new(p);` 如果启用了跟踪（tracepoint），则记录唤醒新任务的事件。
	- `check_preempt_curr(rq, p, WF_FORK);` 检查当前运行的任务是否应当被新任务抢占（preempt）。
	- `task_rq_unlock(rq, p, &rf);` 解锁


<h3>put_pid</h3>
释放pid结构体所占资源。

```c
"linux/kernel/pid.c"

void put_pid(struct pid *pid)
{
	struct pid_namespace *ns;

	if (!pid)
		return;

	ns = pid->numbers[pid->level].ns;
	if (refcount_dec_and_test(&pid->count)) {
		kmem_cache_free(ns->pid_cachep, pid);
		put_pid_ns(ns);
	}
}
EXPORT_SYMBOL_GPL(put_pid);
```

1. 计算 `struct pid_namespace *` 对应的引用计数，每当调用 `alloc_pid` 则使计数+1，调用 `put_pid` 则计数-1，为 0 时则释放资源。
2. `EXPORT_SYMBOL_GPL(put_pid);` 这个宏的作用是导出 `put_pid` 函数的符号，使得其他模块可以在 GPL 兼容的许可证条款下使用它。

<h2>其它</h2>

```plain
struct kernel_clone_args {
	u64 flags;
	int __user *pidfd;
	int __user *child_tid;
	int __user *parent_tid;
	int exit_signal;
	unsigned long stack;
	unsigned long stack_size;
	unsigned long tls;
	pid_t *set_tid;
	/* Number of elements in *set_tid */
	size_t set_tid_size;
	int cgroup;
	int io_thread;
	struct cgroup *cgrp;
	struct css_set *cset;
};
```



