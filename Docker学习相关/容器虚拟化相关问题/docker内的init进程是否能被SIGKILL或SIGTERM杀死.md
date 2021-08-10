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

TODO



