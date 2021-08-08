# Docker的架构

docker是client-server架构，client通过socket或restful API与docker daemon之间进行数据交互。两者可以跑在同一个系统上，或者使用远程的Docker daemon。后者负责繁重的构建、运行、分发docker 容器。

docker内部包含三个组件：

- docker images
- docker registers 镜像仓库（私有和公有）
- docker containers

## Docker image如何工作

docker image是层次化的只读模板，它利用union file system将不同的层组成一个image，同时允许覆盖单独文件系统的文件和目录，形成统一的文件系统。当image增加层或者更新层，需要重新构建image。

## Docker container work

当从docker images 运行一个container，它会在image的顶层添加一个读写层（union file system）以使你的应用可以运行。

### 当运行一个container，会发生什么？

- pull the image：docker检查镜像是否存在，当镜像不存在，会从docker hub中拉取；当镜像存在，即使用其构建。
- create a new container：一旦image存在，docker使用其创建一个镜像。
- 分配一个文件系统并且挂载一个读写层：容器在文件系统之中被创建，并且读写层被添加进image
- 分配一个网络/网桥接口：分配一个网络/网桥接口，允许容器与本机交互
- 设置IP地址：从IP池中寻找并分配一个可用的IP地址
- 执行指定的进程：运行应用程序
- 捕获并且提供应用的输出：连接并且记录标准输入，输出和错误

当container结束时，停止并且删除contianer

# 使用的底层技术

## namespace

Linuex 内核提供了一层隔离，容器的每个方面运行在namespace，并且对它所处的namespace之外没有访问权限

## Control Groups

CGroups允许Docker将可用硬件资源共享给容器，并在必要时设置约束和限制，，比如限制特定容器的可用内存。

## Union file system

Union通过创建层来操作的文件系统。

