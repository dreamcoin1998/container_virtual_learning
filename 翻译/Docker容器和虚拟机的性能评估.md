# Docker容器和虚拟机的性能评估

**作者**：Amit M Potdara, Narayan D Gb, Shivaraj Kengondc, Mohammed Moin Mulla
**原文链接**：https://reader.elsevier.com/reader/sd/pii/S1877050920311315

[TOC]


## 摘要

服务器虚拟化是IT企业广泛使用的技术革新。虚拟化提供了一个在云上运行不同操作系统的服务的平台。它有利于在单个基本的物理机甚至于形式为hypervisor或容器里构建多个虚拟机。为了承载许多微服务应用，这个新兴的技术提出了一个模型，该模型由较小的单个部署服务执行的不同操作组成。因此，用于低开销的虚拟化技术的需求正在迅速发展。有许多轻量级的虚拟化技术，docker是其中之一，它是一个开源的平台。这项技术允许开发者和系统管理员使用docker引擎构建、创建和运行应用。本论文使用标准基准测试工具如Sysbench、Phoronix和Apache benchmark，这写工具包含CPU性能，内存吞吐量，存储读写性能，负载测试和运行速度测试。

## 1、前言

在近些年，对云计算的关注正在增加。It行业在Xen、HyperV、VMware vSphere，KVM等等的出现上开发了许多技术，这些技术统称为虚拟化技术。要在同一虚拟机上部署多个应用程序，需要组织和隔离应用程序依赖。由于虚拟化，多个应用程序可以在同一个物理硬件上运行。虚拟化技术的缺点是：虚拟机体积大，由于运行多个虚拟机导致性能不稳定，启动过程需要很长时间才能运行，虚拟机无法解决可管理性，软件更新和持续集成持续交付等难题。这些问题导致了一种称为容器化的新进程的出现，进一步导致了操作系统级别的虚拟化，而虚拟化则将吸收带到了硬件级别。容器使用共享相关库和资源的宿主机操作系统。它更有效，因为它没有guest Os。在宿主机内核上，可以处理特定于应用程序的二进制文件和库，从而使执行速度非常快。容器是在Docker平台的帮助下形成的，该平台结合了应用程序及其依赖。这些应用总是在隔离空间中，在操作系统内核之上运行。Docker这种容器化功能可确保环境支持任何相关应用程序。
在这项工作中，为了量化和对比虚拟机管理程序的虚拟机和Docker容器上的应用程序，进行了一系列实验。这些测试有助于我们了解两种主要的虚拟化技术（容器和虚拟机管理程序）的性能影响。本文组织如下：第2节给出了背景相关研究和关于技术和平台的简要说明。第3节介绍了用于了解性能比较的方法。第4节中，介绍了基准测试结果。最后，在第5节中提供了结论和未来的工作。

## 2、背景研究

### 2.1 Docker

容器化是一种技术，它将应用程序、相关依赖项和组织起来以容器的形式构建的系统库。构建和组织的应用程序可以作为容器执行和部署。这个平台被称为Docker，它确保应用可以工作在每个环境。它也自动执行将部署到容器中的应用程序。Docker在容器环境中附加了一个额外的部署引擎层，应用程序在其中执行和虚拟化。为了有效的运行代码，Docker有四个主要部分：Docker容器，Docker Client-Server、Docker镜像和Docker引擎。以下各节将对这些组件进行详细说明。

#### 2.1.1 Docker引擎

Docker系统的基本部分是Docker引擎，这是一个客户端-服务器工作模式的应用程序，它安装在主机上，具有以下组件：

- Docker Daemon：一种长时间运行的程序（Docker命令）有助于创建，构建和运行应用程序；
- RestApi被用来与docker daemon进行通信；
- 客户端通过终端发送请求到docker daemon来访问操作。

!(../image/docker容器架构.jpg)[Docker容器架构]
!(../image/Hypervisor.jpg)[a(基于Hypervisor的架构)，b(基于容器的架构)]

#### 2.1.2 Docker客户端-服务器

Docker技术主要是指客户端-服务器架构。客户端主要与Docker守护进程通信，Docker守护进程充当主机中存在的服务器。守护进程作为运行，构建和分发容器的三个主要进程。Docker容器和守护程序都可以放在一台机器中。

#### 2.1.3 Docker镜像

Docker镜像通过两个方法被构建。主要的方式是在一个只读模板的帮助下构建镜像。该模板包含基本镜像，甚至它可以是操作系统比如centos，Ubuntu16.04或者fedroa或者任何其他基本的轻量级的操作系统镜像。通常，基本镜像是每个镜像的基础。每当从头开始构建新镜像时都需要一个基本镜像。这种类型的创建新镜像成为“提交更改”。下一种方法是创建一个Dockerfile，其中包含创建Docker镜像的所有说明。当从终端执行Docker的构建命令时，将使用Dockerfile中声明的所有依赖项创建镜像，此过程称为构建镜像的自动化方法。

#### 2.1.4 Docker容器

Docker容器由Docker镜像创建。要以受限方式运行应用程序，应用程序所需的每个套件都将由容器保存。可以依据应用程序或软件的服务要求创建容器镜像。假设一个包含Ubuntu和Nginx服务器的应用程序已经被添加到到Dockerfile中，使用命令docker run，创建包含Nginx服务器的Ubuntu Os镜像的容器并开始运行。

#### 2.1.5 虚拟机和Docker容器之间的比较

Docker有时被称为轻量级虚拟机，但是它不是虚拟机。如下表1所述，它们的底层技术在虚拟化技术方面的差异。图2显示了虚拟机和Docker容器的体系结构。

表1
| | 虚拟机 | Docker 容器 |
| ---- | ---- |-----|
| 隔离进程级别 | 硬件 | 操作系统 |
| 操作系统 | Separated | 共享 |
| 启动时间 | 长 | 短 |
| 资源使用情况 | 多 | 少 |
| 预构建的镜像 | 难以寻找和管理 | 已可用于主服务器 | 
| 自定义配置镜像 | 更大，因为包含整个操作系统 | 更小，只有主机操作系统上的Docker引擎 |
| 流动性 | 易于迁移到新的主机操作系统 | 销毁和重建替代迁移 |
| 创建时间 | 更长 | 秒级 |

### 2.2 相关工作

正在开发的虚拟机的使用在组织中很常见。虚拟机广泛用于执行复杂的任务，例如Hadoop。但是，用户甚至使用虚拟机来启动小型的应用程序，这使得系统效率低下。需要启动一个轻量级的应用程序，它更快，使系统更加高效。Docker容器时提供轻量级虚拟化的技术之一，这激励我们执行后台工作。
[3]作者从CPU性能，内存吞吐量，磁盘IO和操作速度角度概述了虚拟机和Docker容器的性能评估。[4]作者专注于再HPC集群中实现Docker容器。在本文的后半部分，作者解释了选择容器模型的不同实现方法以及LNPACK和BLAS的使用。在[5]中，作者讨论了轻量级虚拟化的方法，其中解决了容器和unikernel的问题。此外本文还讨论了方差分析检验的统计评估及使用Tukey方法对所收集的数据进行事后比较。作者还讨论了用于比较单核和容器不同基准测试工具。静态HTTP服务器和键值存储参数用于应用程序性能的实现分析，这些分析部署在云上。Nginx服务器用于HTTP性能，为了测量获取和设置操作，使用redis基准测试。
[6]中的作者讨论了使用KVM、Docker和OSv的基准测试应用程序的评估。在[7]中，作者讨论了关于虚拟机和容器化技术的简短调查。他还讨论了DOCKER和Docker性能以及各种参数CPU、内存吞吐量、磁盘IO。在[8]中，作者讨论了Docker架构的基本概念，Docker的组件，Docker镜像，Docker register，docker客户端-服务器架构。讨论了Docker和虚拟机的区别。[9]中，作者解释了基于容器的云技术与一组不同参数的性能比较。本文提供了有关基于OpenStack的云部署的使用情况的信息，并考虑进行比较。用于性能比较的平台时docker，LXC和flockport。[10]中的作者讨论了虚拟机和Docker在各种参数（如CPU、网络、磁盘和两个真正的服务器应用程序redis和Mysql）方面的性能比较。

## 3、方法论

在本节中，使用基准测试工具执行KVM和Docker的评估。以下用于性能评估的基准测试工具时Sysbench，Phoronix，Apache benchmark。这些基准测试工具可以测量CPU性能、内存吞吐量、存储读取和写入、负载测试和操作速度测量。两台HP服务器用于有关各种参数的性能评估，因为其中一台服务器用作安装在主机操作系统之上的虚拟机，而除了此docker引擎之外，guest Os安装在虚拟机之上，另一台服务器作为裸机，主机操作系统Ubuntu 16.04和docker引擎都安装在其上。所有测试均在配备两个因特尔E5-2620 v3处理器的惠普服务器上执行，频率为2.4GHz，共12个内核和64GB的RAM。Ubuntu16.04 64位Linux内核3.10.0用于执行所有测试。为了保持一致性和统一性，使用相同的操作系统。Ubuntu16.04作为Docker容器的基础镜像。为虚拟机配置了12 vCPU和足够的RAM。图3介绍了使用各种基准测试工具不同虚拟化技术的评估方法。

!(../image/评估.jpg)[]

## 4、结果和讨论

本节讨论虚拟化技术的性能分析。结果分为四个小节。4.1描述所有CPU测量、内展示在4.2和4.3节的存吞吐量和存储读写测量值。4.4节描述负载测试分析。4.5节描述操作速度测量，包括两个测试：八皇后问题和八个谜题问题。最后再4.6节中，执行了统计t检验分析。

### 4.1 CPU性能

计算性能可以通过系统在给定时间执行的操作数或特定任务的完成时间来衡量。结果主要取决于分配给服务器的虚拟CPU内核数。测试CPU性能比较是通过以下工具sysbench、phoronix和apache benchmark。

#### 4.1.1 最大质数运算

在Sysbench工具测试找出执行最大素数所需的时间，操作的最大质数位50000，时间为60秒，4个线程操作。从下图观察，与VM相比，docker容器执行操作所需时间要少的多。这是由于虚拟机中存在虚拟机管理程序，因此执行需要更多的时间。

#### 4.1.2 7 ZIP 压缩测试

7 zip是一款开源的文档存档器，用于将一组文件压缩到称为压缩包的容器中。LZMA基准测试的两个测试，压缩和解压LZMA方法。此测试测量使用7 zip压缩文件所需的时间。用于压缩测试的文件大小为10GB。根据获取的结果，当使用大量文件执行压缩时，Docker容器的性能要比VM好很多。

#### 4.2 内存性能

RAM  speed/SMP（对称多处理）是一种缓存和内存基准测试工具，用于测量虚拟化技术（即 Docker 和虚拟机）的 RAM 速度。图 6.表示虚拟化技术之间的 RAM 速度比较。测试RAM速度时，考虑了以下两个主要参数。INTmark 和 FLOATmark 组件用于 RAM 速度 SMP 基准测试工具，该工具可在读取和写入单个数据块时测量最大可能的缓存和内存性能。
INTmem和FLOATmem，它们是合成模拟，但与计算的现实世界紧密平衡。每个子测试都由四个子测试（复制，缩放，添加，三元组）组成，以测量内存性能的不同方面。数据从一个内存位置到另一个内存位置的传输是通过复制命令完成的，即（X = Y）。写入前的数据修改乘以一定的常量值是通过scale命令完成的，即（X = n * Y）。数据从第一个内存位置读取，然后在调用命令 ADD 时从第二个内存位置读取。然后将结果数据放在第三位（X = Y + Z）。三元组是添加和缩放的组合。从第一个内存位置读取数据以进行缩放，然后从第二个位置相加以将其写入第三个位置（X = n*Y + Z）。图 6.显示相对于 RAM 速度 SMP 测试的内存性能。

!(../image/ram.png)[]

#### 4.3 磁盘IO性能

测试硬盘性能，使用 IOzone 基准测试工具进行性能分析。为了测试系统的写入和读取等操作，使用了1MB的记录大小和4GB的文件大小。从图 7 中可以推断出，与虚拟机相比，Docker 的性能要好得多。VM 的磁盘写入和读取操作减少了 Docker 容器的一半以上（约 54%）。图 7.显示了虚拟机和 Docker 的磁盘性能。

!(../image/disk.png)[]

#### 4.4 负载测试

对于负载测试性能比较，使用Apache基准测试工具，其中它测量给定系统每秒可以容忍的请求数。执行python程序以使用Apache Benchmarking工具测试负载。图 8.表明 VM 的吞吐量分析比 Docker 的吞吐量分析要少得多。这是因为虚拟机中的网络延迟高于 Docker 中的网络延迟。分析表明，Docker容器在每秒处理请求数方面优于虚拟机。


!(../image/loadtest.png)[]

#### 4.5 运行速度测量

八位女王的问题将八位女王放在8×8的棋盘上，这样他们都没有互相攻击。该测试测量解决问题所需的时间。Eight queen程序是用python编写的，决定了系统的计算性能。图 9.显示了 Docker 和虚拟机的计算性能。根据执行时间，docker 容器解决问题所需的时间更少，而虚拟机需要更长的时间。
八拼图测试：拿4×4板，8块瓷砖和一个空的空间。使用空白空间，排列与最终配置相匹配的图块数量是主要目标。它可以将四个相邻的操作（右，左，下和上）磁贴滑入空白区域 - 该测试测量解决问题所需的时间。Eight拼图程序是用python编写的，决定了系统的计算性能。图 10.显示了 Docker 和虚拟机的计算性能。根据执行时间，docker 容器解决问题所需的时间更少，而虚拟机需要更长的时间。


#### 4.6 t-test分析

t 检验统计测量用于确定两个组的方法之间是否存在关键区别，这两个组可能在相关特征中连接。在下面的结果演示中，统计推理技术 t-test 已被用于证明 docker 容器的性能显著优于虚拟机。此外，阈值的置信水平α为 0.05。两组数据的概率可以通过取 t 统计量、t 分布值和自由度来确定。
在分析部分使用原假设 （H0） 和备择假设 （H1） 约定。在进行检验分析时，考虑原假设为真，这是进一步统计分析的假设。然后，测试的结果可以证明，如果 H0 为真，则假设可能是错误的。
需要测试 Docker 容器执行操作所需的时间是否比虚拟机少。样本检验包括 10 个关于查找素数操作的实验。表3显示了为分析部分采集的10个样本。发现执行虚拟机操作所花费的平均时间为 153.5 秒。现在需要检查 Docker 容器的平均时间是否小于虚拟机。t 统计量 （t） 的等式为 。

## 5、结论

Docker 容器是一种新兴的轻量级虚拟化技术。这项工作评估了两种虚拟化技术，即 Docker 容器和虚拟机。在虚拟机和基于 Docker 容器的主机上，从 CPU 性能、内存吞吐量、磁盘 I/O、负载测试和操作速度测量等方面进行性能评估。据观察，Docker容器在每次测试中的表现都优于VM，因为虚拟机中存在QEMU层使其效率低于Docker容器。容器和虚拟机的性能评估是使用Sysbench，Phoronix和apache基准测试等基准测试工具执行的。
作为未来的工作，我们计划在Docker中调度容器，同时开发更安全的容器变体，这将减少安全约束。

## References 
 
[1]	Docker.  https://docs.docker.com/ 2019 [Online; Accessed 24-03-2019] 
[2]	Shivaraj Kengond, DG Narayan and Mohammed Moin Mulla (2018) “Hadoop as a Service in OpenStack” in Emerging Research in Electronics, Computer Science and Technology , pp 223-233. 
[3]	C. G. Kominos, N. Seyvet and K. Vandikas, (2017) "Bare-metal, virtual machines and containers in OpenStack" 20th Conference on Innovations in Clouds, Internet and Networks (ICIN), Paris, pp. 36-43. 
[4]	Higgins J., Holmes V and Venters C. (2015) “Orchestrating Docker Containers in the HPC Environment”. In: Kunkel J., Ludwig T. (eds) High Performance Computing. Lecture Notes in Computer Science, vol 9137.pp 506-513 
[5]	Max Plauth, Lena Feinbube and Andreas Polze, (2017) “A Performance Evaluation of Lightweight Approaches to Virtualization”, CLOUD COMPUTING: The Eighth International Conference on Cloud Computing, GRIDs, and Virtualization. 
[6]	Kyoung-Taek Seo, Hyun-Seo Hwang, Il-Young Moon, Oh-Young Kwon and Byeong-Jun Kim “Performance Comparison Analysis of Linux Container and Virtual Machine for Building Cloud,” Advanced Science and Technology Letters Vol.66, pp.105-111. 
[7]	R. Morabito, J. Kjällman and M. Komu, (2015) "Hypervisors vs. Lightweight Virtualization: A Performance Comparison", IEEE International Conference on Cloud Engineering, Tempe, AZ, pp. 386-393. 
[8]	Babak Bashari Rad, Harrison John Bhatti and Mohammad Ahmadi (2017) “An Introduction to Docker and Analysis of its Performance” IJCSNS International Journal of Computer Science and Network Security, VOL.17 No.3, pp 228-229. 
[9]	Kozhirbayev, Zhanibek, and Richard O. Sinnott. (2017) "A performance comparison of container-based technologies for the cloud", Future Generation Computer Systems, pp: 175-182. 
[10]	Felter, Wes, et al. (2015) "An updated performance comparison of virtual machines and linux containers", IEEE international symposium on performance analysis of systems and software (ISPASS). pp:171-172. 
[11]	Sysbench. https://wiki.gentoo.org/wiki/Sysbench 2019 [Online; Accessed 24-03-2019] 
[12]	Phoronix Benchmark tool.  https://www.phoronix-test-suite.com/ 2019 [Online; Accessed 24-03-2019] 
[13]	Apache Benchmark tool.  https://www.tutorialspoint.com/apache_bench/  2019 [Online; Accessed 24-03-2019] 
