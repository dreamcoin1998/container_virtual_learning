# 接上篇文章

## Linux忽略SIGKILL和SIGSTOP发送到init进程

```c
static int __send_signal(int sig, struct kernel_siginfo *info, struct task_struct *t,
			enum pid_type type, bool force)
{
    struct sigpending *pending; // sig的链表，保存了进程上尚未处理的信号
	struct sigqueue *q; // 实时信号可能会排队，使用队列实现
	int override_rlimit;
	int ret = 0, result;

	assert_spin_locked(&t->sighand->siglock); // 自旋锁

	result = TRACE_SIGNAL_IGNORED;
	// 判断该信号SIGKILL或者SIGSTOP是否是发送给init，如果是则忽略
	if (!prepare_signal(sig, t, force))
		goto ret;
}
```

追踪到prepare_signal函数一直往下：

```c
static bool sig_task_ignored(struct task_struct *t, int sig, bool force)
{
	void __user *handler;

	handler = sig_handler(t, sig);

	/* SIGKILL and SIGSTOP may not be sent to the global init */
	// SIGKILL and SIGSTOP不会被发送到global init进程
	// 如果是SIGKILL and SIGSTOP目标进程是init进程
	if (unlikely(is_global_init(t) && sig_kernel_only(sig)))
		return true;

    // 这里的flag标志是task_struct的标志，见下面
    // SIGNAL_UNKILLABLE为进程刚被fork，但是还没被执行，所以这里是如果进程刚被创建，但是还没被执行，返回true
	// 如果是默认的handle
    // 不是SIGKILL或者是SIGSTOP并且可以被发送到进程
    // 所以这里标识如果进程刚被fork，发送的信号不是必须被处理的信号，且handle是默认的handle，则不能响应信号
    if (unlikely(t->signal->flags & SIGNAL_UNKILLABLE) &&
	    handler == SIG_DFL && !(force && sig_kernel_only(sig)))
		return true;

	/* Only allow kernel generated signals to this kthread */
    // PF_KTHREAD是内核线程
	if (unlikely((t->flags & PF_KTHREAD) &&
		     (handler == SIG_KTHREAD_KERNEL) && !force))
		return true;

	return sig_handler_ignored(handler, sig);
}
```

**flag信号：**

```c
// flag可能取值如下：
PF_FORKNOEXEC 进程刚创建，但还没执行。
PF_SUPERPRIV 超级用户特权。
PF_DUMPCORE dumped core。
PF_SIGNALED 进程被信号(signal)杀出。
PF_EXITING 进程开始关闭
```

再看sig_handler_ignored：

```c
static inline bool sig_handler_ignored(void __user *handler, int sig)
{
	/* Is it explicitly or implicitly ignored? */
	// 属于可以被忽略的信号
	return handler == SIG_IGN ||
	// 默认信号处理程序，也就是说信号是SIGCONT，SIGCHLD，SIGWINCH，SIGURG他们的默认处理信号是忽略
	       (handler == SIG_DFL && sig_kernel_ignore(sig));
}

#define sig_kernel_ignore(sig)		siginmask(sig, SIG_KERNEL_IGNORE_MASK)

#define SIG_KERNEL_IGNORE_MASK (\
        rt_sigmask(SIGCONT)   |  rt_sigmask(SIGCHLD)   | \
	rt_sigmask(SIGWINCH)  |  rt_sigmask(SIGURG)    )
```

**SIGCONT，SIGCHLD，SIGWINCH，SIGURG：**

```c
SIGCHLD：
● 子进程终止时；
● 子进程接收到SIGSTOP信号停止时；
● 子进程处在停止态，接收到SIGCONT信号后被唤醒时。
SIGWINCH:
窗口大小改变时
SIGURG：
有"紧急"数据或out-of-band数据到达socket时产生.
```

## 总结

- 信号时SIGKILL 或 SIGSTOP目标进程是init进程，该信号会被忽略即不能响应信号
- 如果进程刚被fork，发送的信号不是必须被处理的信号，且handle是默认的handle，则不能响应信号
- 目标线程是内核线程，并且是默认内核线程处理程序，但是不能被发送，则不能响应信号
- 信号是SIGCONT，SIGCHLD，SIGWINCH，SIGURG他们的默认处理信号是忽略

