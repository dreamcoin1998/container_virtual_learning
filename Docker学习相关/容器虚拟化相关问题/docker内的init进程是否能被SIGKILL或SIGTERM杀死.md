[toc]

# 问题

在容器内给容器内的init进程发送SIGTERM信号，init进程会不会被杀死，发送SIGKILL信号呢？在进程外发送信号呢？为什么？

*问题来源：极客时间*

# 相关理论知识

## 信号相关

- SIGKILL和SIGTERM信号都是Linux信号，SIGKILL和SIGSTOP属于不能被忽略和handle的内核信号，主要提供给内核进程和root用户进行特权操作。
- 使用kill命令的时候，缺省状态下，kill 1 向init进程发送SIGTREM信号
- Ctrl+C发送的是SIGKILL信号

## 编程语言的signal

很多编程语言对不同的信号做了handle处理，使用C语言需要自己做处理（除了SIGKILL和SIGSTOP之外，他们有内核提供的默认的handle）。下列是Python和Go语言对部分signal的处理：

|         | Python                                                       | Golang                                                       |
| ------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| SIGQUIT | 执行默认操作，进程退出并将内存中的信息转存到硬盘（核心转储） | 以堆栈转储方式退出                                           |
| SIGCHLD | 终止子进程                                                   | 子进程退出                                                   |
| SIGHUP  | 在控制终端上检测到挂起或控制进程的终止                       | 程序退出                                                     |
| SIGTERM | 终结                                                         | 程序退出                                                     |
| SIGPIPE | 写入到没有读取器的管道，默认忽略                             | 在标准输出或标准错误上写入损坏的管道将导致程序退出，而在其他的文件描述符上写入不会采取任何操作，但是会抛出EPIPE错误 |
| SIGING  | 引发 [`KeyboardInterrupt`](https://docs.python.org/zh-cn/3/library/exceptions.html#KeyboardInterrupt) |                                                              |

关于signal信号的处理，可以查看下列文档：

- Python：https://docs.python.org/zh-cn/3/library/signal.html?highlight=signal#signal.SIGPIPE

- Golang：https://pkg.go.dev/os/signal

## 容器中的init进程

- 容器中的init进程是pid为1的进程
- 当容器以单进程运行时，init进程为容器内执行的进程
- 当容器以多进程运行时，会启动一个管理进程，该进程即为容器的init进程
- 在容器内发送SIGKILL命令，内核将会屏蔽掉该信号（与在跟命名空间中一致），防止其他进程杀掉init进程
- 在容器外杀掉init进程后，再向容器内添加进程将会导致ENOMEN错误，因为进程已经终止
- init进程是容器内所有孤儿进程的父进程

# 当使用kill命令时，内核做了什么（源码分析）

## 寻找kill系统调用

Linux系统调用由原本的sys_XXXX换成了SYSCALL_DEFINE，故在/kernel/signal.c文件下，寻找SYSCALL_DEFINE，找到SYSCALL_DEFINE2(kill, pid_t, pid, int, sig)，代码如下：

```c
/**
 *  sys_kill - send a signal to a process
 *  @pid: the PID of the process
 *  @sig: signal to be sent
 */
SYSCALL_DEFINE2(kill, pid_t, pid, int, sig)
{
	struct kernel_siginfo info; // info是signal的结构体

	prepare_kill_siginfo(sig, &info);

	return kill_something_info(sig, &info, pid);
}
```

关于SYSCALL_DEFINEx，可以在https://blog.csdn.net/hxmhyp/article/details/22699669文章里面看到。

关于kernel_siginfo，可以看如下代码：

```c
typedef struct kernel_siginfo {
	__SIGINFO;
} kernel_siginfo_t;
```

关于__SIGINFO：

```c
struct {				\
	int si_signo; /* Signal number */
	int si_errno；/* An errno value */
	int si_code; // 产生信号的原因
	union __sifields _sifields;
}
```

简而言之，__SIGINFO就是标识signal的一些信息。

接下来我们来看看```prepare_kill_siginfo(sig, &info);```方法做了什么：

```c
static inline void prepare_kill_siginfo(int sig, struct kernel_siginfo *info)
{
	clear_siginfo(info);
	info->si_signo = sig; // 传入的信号
	info->si_errno = 0;
	info->si_code = SI_USER; 
	info->si_pid = task_tgid_vnr(current); // 主要是从当前namespace 中找到对应的pid号。
	info->si_uid = from_kuid_munged(current_user_ns(), current_uid()); // 返回0，也就是root用户
}
```

关于```info->si_code = SI_USER; ```中的SI_USER，在内核源码```\include\uapi\asm-generic\siginfo.h```可以看到Linux定义的一些信号相关的信息。下列贴出部分：

```c
/*
 * si_code values
 * Digital reserves positive values for kernel-generated signals.
 */
#define SI_USER		0		/* sent by kill, sigsend, raise */
#define SI_KERNEL	0x80		/* sent by the kernel from somewhere */
#define SI_QUEUE	-1		/* sent by sigqueue */
#define SI_TIMER	-2		/* sent by timer expiration */
#define SI_MESGQ	-3		/* sent by real time mesq state change */
#define SI_ASYNCIO	-4		/* sent by AIO completion */
#define SI_SIGIO	-5		/* sent by queued SIGIO */
#define SI_TKILL	-6		/* sent by tkill system call */
#define SI_DETHREAD	-7		/* sent by execve() killing subsidiary threads */
#define SI_ASYNCNL	-60		/* sent by glibc async name lookup completion */

#define SI_FROMUSER(siptr)	((siptr)->si_code <= 0)
#define SI_FROMKERNEL(siptr)	((siptr)->si_code > 0)
```

由此可以看到，这里的标识表示SI_USER是由kill发出的。

接下来可以看task_tgid_vnr(current)；首先是current：

```c
/*
 * We don't use read_sysreg() as we want the compiler to cache the value where
 * possible.
 */
static __always_inline struct task_struct *get_current(void)
{
	unsigned long sp_el0;

	asm ("mrs %0, sp_el0" : "=r" (sp_el0));

	return (struct task_struct *)sp_el0;
}

#define current get_current() // 获取当前运行的task_struct
```

它是获取当前运行的task_struct，我们都知道，在Linux里面task_struct表示一个进程描述符，存储了进程相关的一些信息。

然后我们看一下task_tgid_vnr方法，一路追踪到底，可以看到：

```c
pid_t __task_pid_nr_ns(struct task_struct *task, enum pid_type type,
			struct pid_namespace *ns)
{
	pid_t nr = 0;  // pid_t 宏定义 int 变量

	rcu_read_lock(); // 申请rcu锁
	if (!ns)
		ns = task_active_pid_ns(current);
	nr = pid_nr_ns(rcu_dereference(*task_pid_ptr(task, type)), ns);
	rcu_read_unlock(); // 释放rcu锁

	return nr;
}
```

task_active_pid_ns方法：\linux\kernel\pid.c

```c
struct pid_namespace *task_active_pid_ns(struct task_struct *tsk)
{
	return ns_of_pid(task_pid(tsk));
}
```

接着继续ns_of_pid方法：\include\linux\pid.h

```c
static inline struct pid_namespace *ns_of_pid(struct pid *pid)
{
	struct pid_namespace *ns = NULL;
	if (pid)
		ns = pid->numbers[pid->level].ns;
	return ns;
}
```

这里返回pid所在的命名空间。

然后是pid_nr_ns：\include\linux\pid.h

```c
pid_t pid_nr_ns(struct pid *pid, struct pid_namespace *ns)
{
	struct upid *upid;
	pid_t nr = 0;

	if (pid && ns->level <= pid->level) {
		upid = &pid->numbers[ns->level];
		if (upid->ns == ns)
			nr = upid->nr;
	}
	return nr;
}
```

主要是从当前namespace 中找到对应的pid号。

OK，所以prepare_kill_siginfo方法获取kernel_siginfo相关的信息并进行初始化。

接下来我们来看最关键的kill_something_info：

```c
static int kill_something_info(int sig, struct kernel_siginfo *info, pid_t pid)
{
	int ret;

	if (pid > 0)
		return kill_proc_info(sig, info, pid);
	...
}
```

kill_proc_info:

```c
static int kill_proc_info(int sig, struct kernel_siginfo *info, pid_t pid)
{
	int error;
	rcu_read_lock();
	error = kill_pid_info(sig, info, find_vpid(pid));
	rcu_read_unlock();
	return error;
}
```

首先是find_vpid:

```c
struct pid *find_vpid(int nr)
{
	return find_pid_ns(nr, task_active_pid_ns(current));
}
```

task_active_pid_ns刚刚我们已经讲过，所以这里是返回当前进程所在的命名空间。

而find_pid_ns作用是找到其所属的struct pid。

然后我们进入kill_pid_info：

```c
int kill_pid_info(int sig, struct kernel_siginfo *info, struct pid *pid)
{
	int error = -ESRCH;
	struct task_struct *p;

	for (;;) {
		rcu_read_lock();
		p = pid_task(pid, PIDTYPE_PID);
		if (p)
			error = group_send_sig_info(sig, info, p, PIDTYPE_TGID);
		rcu_read_unlock();
		if (likely(!p || error != -ESRCH))
			return error;

		/*
		 * The task was unhashed in between, try again.  If it
		 * is dead, pid_task() will return NULL, if we race with
		 * de_thread() it will find the new leader.
		 */
	}
}
```

pid_task：

```c
struct task_struct *pid_task(struct pid *pid, enum pid_type type)
{
	struct task_struct *result = NULL;
	if (pid) {
		struct hlist_node *first;
		first = rcu_dereference_check(hlist_first_rcu(&pid->tasks[type]),
					      lockdep_tasklist_lock_is_held());
		if (first)
			result = hlist_entry(first, struct task_struct, pid_links[(type)]);
	}
	return result;
}
```

根据一致的struct pid找到struct task_struct。

group_send_sig_info：

```c
int group_send_sig_info(int sig, struct kernel_siginfo *info,
			struct task_struct *p, enum pid_type type)
{
	int ret;

	rcu_read_lock();
	ret = check_kill_permission(sig, info, p);
	rcu_read_unlock();

	if (!ret && sig)
		ret = do_send_sig_info(sig, info, p, type);

	return ret;
}
```

然后是check_kill_permission：

```c
/*
 * Bad permissions for sending the signal
 * - the caller must hold the RCU read lock
 */
static int check_kill_permission(int sig, struct kernel_siginfo *info,
				 struct task_struct *t)
{
	struct pid *sid;
	int error;

	if (!valid_signal(sig))  // return sig <= _NSIG ? 1 : 0; _NSIG == 64 这里检测是否是一个有效的信号
		return -EINVAL; // EINVAL 宏定义 22

	if (!si_fromuser(info))
		return 0;

	error = audit_signal_info(sig, t); /* Let audit system see the signal */
	if (error)
		return error;

	if (!same_thread_group(current, t) &&
	    !kill_ok_by_cred(t)) {
		switch (sig) {
		case SIGCONT:
			sid = task_session(t);
			/*
			 * We don't return the error if sid == NULL. The
			 * task was unhashed, the caller must notice this.
			 */
			if (!sid || sid == task_session(current))
				break;
			fallthrough;
		default:
			return -EPERM;
		}
	}

	return security_task_kill(t, info, sig, NULL);
}
```

先到这里，实在太长了这代码。。。