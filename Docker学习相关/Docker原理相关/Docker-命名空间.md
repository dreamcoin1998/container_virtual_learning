# 命名空间

Linux内核用命名空间来区分内核资源。不同的进程看到的资源是不同的。

这里的资源包括pid，hostname，userid，文件名，网络访问和进程间通信

在Linux内核3.8版本，内核对容器有足够的支持

## 内核空间的类型

 Linux内核5.6，支持8种命名空间。

- mount：mount命名空间控制挂载点
  - 在创建命名空间时，当前的命名空间会复制到新的命名空间。但是已经创建的挂载点不会在明明见之间传播。
  - 创建此类型的标志clone_NEW
- PID：提供独立于其他命名空间的pid集合，不同pid namespace里面的pid可以重复
  - PID命名空间内的进程ID是唯一的，从1开始分配
  - PID 1进程的终止将终止该命名空间以及其后代所有进程
  - PID namespace的使用需要配置参数CONFIG_PID_NS
- Network(net)：网络命名空间虚拟化提供隔离的网络相关的系统隔离资源，如网络设备、IPv4、IPv6协议栈，IP路由表、防火墙规则、/proc/net目录、/sys/class/net目录、/proc/sys/net下的各种文件、端口号等等。特别的，网络命名空间隔离UNix域抽象套接字命名空间。
  - 物理网络设备：当网络命名空间内的进程终止，网络命名空间被释放时，物理网络设备将会回到初始话网络命名空间。
  - 虚拟网络设备：提供了管道式的，抽象化的，可以在网络命名空间之间使用的隧道，在一个物理网络设备到另一个网络命名空间之间使用的桥梁。当一个namespace被释放，它包含的veth设备将会被销毁。
  - 使用network namespace需要内核配置CONFIG_NET_NS参数
- User：用来区分安全标识符和属性，特别是UserID、GroupID、根目录和keys。
- Cgropu：（Cgroup可以对一组进程做资源控制）
- IPC：隔离了IPC资源，即SystemV IPC对象，POSIX消息队列。需要CONFIG_IPC_NS内核参数
- Time：CONFIG_TIME_NS内核参数。虚拟化CLOCK_MONOTONIC和CLOCK_BOOTTIME。
- UTS：虚拟化hostname和NIS域名。CONFIG_UTS_NS

## PID namespace

### namespace init process

第一个在namespace中创建的进程，是任何在这个pid namespace中的孤儿进程的父进程。

当init终止，内核通过发送SIGKILL信号终止namespace中所有的子进程，在这种情况下，调用fork将返回ENOMEN错误，因为init进程已经终止。

内核会帮助init进程屏蔽掉任何其他信号，防止其他进程不小心kill掉init进程导致系统挂掉。可以在父PID namespace中发送SIGKILL信号终止namespace中init进程。*通过什么方式屏蔽？*

### PID namespace nest

PID namespace可以嵌套：每一个PID namespace都有一个parent，除了根PID namespace。父PID namespace是使用clone()或unshare()函数创建的进程。从Linux 3.7开始，内核限制最深嵌套是32。

子PID namespace中的进程对于父PID namespace中的进程是不可见的。进程可以下降到子命名空间，但是不能进入任何一个父命名空间。

进程的PID命名空间在进程创建时确定，不可更改。

### /proc 和 PID namespace

对一个PID namespace而言，/proc目录只包含当前namespace和它所有子孙后代namespace里的进程的信息。

## mount namespace

mount namespace隔离了不同的挂载点，每个挂载的命名空间实例将看到不同的单目录层次结构。使用clone和unshare传递CLONE_NEWS信号能够创建心的mount namespace。

当一个心得mount namespace创建，将进行如下初始化：

- 使用clone创建，将子namespace的挂载点列表是父挂载点列表的复制
- 使用unshare创建，新命名空间挂载点列表是调用者之前命名空间挂载点列表的复制

### mount namespace的限制

- 每一个mount namespace拥有一个user namespace。一个新的mount namespace是另一个namespace的副本，新旧namespace拥有不同的user namespace，那么新的namespace将被赋予更低的特权级别
- 当创建一个更低特权级别的mount namespace，共享挂载将会减少到从属挂载，这确保更低特权级别执行的映射不会传播到更高特权级别
- 来自更高特权级别的单个单元的挂载将会被加锁，不会被更低优先级别的mount namespace分开。unshare的CLONE_NEWS操作将原有命名空间当中的mount以及其递归的mount作为单个单元进行传播。
- 在一个mount namespace作为挂载点且不在其他mount namespace的文件或目录可以被重命名、取消链接和删除（不是挂载点）。

### 共享子树

背景：为了加载一个新的硬盘，使其在所有的mount namespace中可用，需要在所有的mount namespace中执行挂载操作。

共享子树功能在Linux 2.6.15中引入。该功能允许自动的、在命名空间之前传播的受控的挂载和卸载事件。

每个挂载点被标记为如下的传播类型：

- MS_SHARED：在一组对等节点之间共享事件。
- MS_PRIVATE：这个挂载点是私有的，挂载和卸载事件不会传播进来或传播出去。
- MS_SLAVE：事件将会在master的共享对等组之间传播，不会传播到其他组。（一组挂载点可以是其他组的slave）
- MS_UNBINDDABLE：类似于私有的mount，这个mount将不能被绑定挂载。

