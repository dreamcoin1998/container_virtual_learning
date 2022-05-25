# Kubernetes中的网络策略：性能评估和安全分析

**作者**：Gerald Budigiri; Christoph Baumann; Jan Tobias Mühlberg; Eddy Truyen; Wouter Joosen
**原文**：https://ieeexplore.ieee.org/document/9482526

[TOC]

## 摘要

具有超高可靠性要求和低延迟要求的5G应用需要在移动网络中采用边缘计算解决方案。根据终端用户和第三方公司的需求，像k8s这样的容器编排框架已经进一步成为动态部署边缘应用的首选标准。不幸的是，复杂的网络和安全问题被强调为阻碍行业成功采用容器技术的挑战。安全挑战因（错误）概念而加剧，即安全的荣期间通信以牺牲性能为代价，但是这样中需求对于5G边缘计算用例来说都至关重要。为了追求低开销的安全方案，本论文研究网络策略，即k8s用于控制租户之间网络隔离的概念。我们评估Calico和Cilium基于ebpf的解决方案的性能开销，分析网络策略的安全性，突出网络策略的安全威胁，并概述相应的先进解决方案。我们的评估表明，网络策略时适合低延迟容器间通信的低开销安全解决方案。

## I.前言

5G的出现为几个新的超可靠低延迟通信（URLLC）应用和用例打开了大门，如虚拟/增强现实（VR / AR），车辆到一切（V2X）和远程手术（RS），有望显着增加对计算和通信资源的需求。即使硬件功能最近有所进步，这些应用的严格性能要求仍然无法与实际的设备和网络功能相匹配[1]。为了解决这种不匹配问题，5G提供商采用了边缘计算（边缘计算是指在用户或数据源的物理位置或附近进行的计算，这样可以降低延迟，节省带宽。）和网络功能虚拟化（NFV）等几种技术概念。借助NFV，边缘计算平台虚拟化了网络功能模块，并将云计算能力扩展到终端用户附近的边缘设备，从而在多租户边缘生态系统中提供灵活性和敏捷性。
云原生计算趋势满足了这一发展，其中应用程序由使用无状态API相互交互的微服务组成。Kubernetes等灵活的编排框架之上使用轻量级和可以指的容器，而不是更资源密集型和启动缓慢基于VM的解决方案，使得云原生网络功能（CNF）成为5G边缘计算URLLC应用的更好选择。现在微服务架构正在被研究为边缘NFV的可能解决方案。
然而，复杂的网络和安全问题被强调为工业中容器采用的挑战。在多租户边缘计算云环境中，潜在的安全漏洞具有更高的严重性，在这些环境中，需要隔离属于不同房的微服务，以便他们只能在必要时进行交互。此外，对于RS和V2X等任务关键性5G应用，通信性能和安全性都不应受到影响。因此，必须为容器间通信找到低开销的安全解决方案。虽然行业和研究界已经做出了许多努力，来提高容器安全性，但其中大多数工作都集中在容器镜像，运行时，内核操作系统和k8s配置上，很少或根本没有考虑低延迟应用程序中的网络安全问题。k8s提供允许限制容器间通信的网络策略。虽然缺乏入侵检测等现代防火墙的高级功能，但不同网络策略仍然提供了合理的网络安全水平。在多租户环境中，它们通过将流量限制为仅允许互相通信的微服务，在租户之间提供可配置的网络隔离。虽然通过k8s网络策略API进行定义，但是策略的实施由自定义的容器网络接口（CNI）插件处理。之前的工作比较了不同CNI插件，并讨论了他们在多租户中的使用。相比之下，本文首次专门研究了k8s网络策略的性能开销和安全影响，并探讨了网络策略在边缘保护URLLC应用程序的适用性。我们做出以下贡献：

- 性能：我们评估了合适的CNI插件并且表明CNI插件的网络策略产生的性能开销可以忽略不计，该开销仅随策略数量和不同策略配置方式而略有不同。
- 安全：我们定义一个针对网络策略的攻击模型并且分析相应的威胁和漏洞。此外我们提出了合适的最先进的低延迟解决方案，以应对这些威胁。

## II.背景与动机

在介绍这些结果之前，我们概述了k8s的网络策略的技术基础，功能和挑战。
本节介绍k8s、k8s中的网络策略，并且为什么它们在多租户边缘平台中的重要性。
此外，我们就CNI插件和网络策略支持进行比较，并解释Calico中的策略实施。

### A.Kubernetes(k8s)

k8s是容器编排的事实标准，它支持部署、拓展和管理容器化微服务应用程序并提供网络概念和对大规模互联微服务的支持。单个微服务应用程序在容器中运行，一个或多个紧密耦合的容器在Pod中运行，而pod在节点上运行，通过k8s主节点上运行的API服务进行控制。由于Pod是临时的，在拓展过程中动态启动或终止，因此服务对象用于为pod提供稳定的端点。此类和其他配置对象与控制k8s的编排过程的状态信息一起存储在分布式etcd数据库中。

### B.kubernetes 网络策略

在多租户平台，保护边缘用户的隐私是重要的。尽管k8s提供集群范围的命名空间用来提供隔离以进行管理和资源配额管理，单词支持不足以避免网络遍历。然而缺乏足够的网络分段是是一个一个经常被引用的高风险漏洞，该漏洞已被Equifax数据泄露等大型网络犯罪所利用。
网络策略通过显式声明允许和拒绝的连接，帮助提供限制pod之间（在和/或跨命名空间）以及pod与外部网络之间的流量所需的护栏。网络策略规范由podSelector组成，用于指定将受策略约束的pod，以及用于指定策略类型（入口或出口）的策略类型。入口规则指定允许入站的流量到目标Pod，出口规则指定允许的来自目标pod的出站流量。每个规则都由一个NetworkPolicyPeer组成，用于通过无类域间路由（CIDR）表示法选择连接另一端允许流量的pod，该表示法指定IP地址块，命名空间或pod标签选择一个NetworkPolicyPort，它允许显示指定可能与Pod通信的端口或网络协议。网络策略可能是累加的，如果多个策略选择一个Pod，则流量将限制在这些策略的入口/出口规则并集所允许的范围内。

### CNI插件和网络模式

由于没有内置功能来实施网络策略，k8s依靠CNI插件来实施。CNI插件作为附加组件安装，该附加组件可以作为容器运行，也可以依赖开源k8s组件。虽然存在许多CNI插件，但只有Calico和Cilium支持使用ebpf实施网络策略，ebpf是iptables的高性能替代方案。事实上，再将ebpf与iptables进行比较时，我们观察到延迟减轻了0.7到0.8倍，节点间和节点内场景的吞吐量分别提高了3.5倍和1.2倍。测量是在封闭实验室的OpenStack测试平台上使用Calico进行的（参见III-A和III-B部分，了解测试平台的实验设置和规格）。Calico和Cilium都使用流量控制钩子，在Pod的虚拟以太网接口的入口和出口处附加ebpf程序。
根据主机模式评估Calico ebpf和Cilium ebpf性能，在该模式下，基准测试在OpenStack VM实例上运行。结果表明，尽管ebpf数据平面有优势，但是CNI插件（可能还有整个K8s架构）的开销仍然很大，特别是对于节点间的通信。结果进一步表明，Calico ebpf在节点间场景中的表现明显优于Cilium ebpf。这是因为Cilium默认以隧道模式运行，这会引发封装标头，而Calico在每个节点上使用Linux内核路由支持来提供纯第三层网络解决方案，从而降低开销。因此，Calico ebpf被用于本文其余性能评估。Calico和主机节点内测量在裸机上重复，其中主机结果在延迟和吞吐量方面分别高出8.7%和10.7%，而OpenStack则为33%和31%。OpenStack上较高的开销表示VM的基于OpenStack的网络配置与Pod基于CNI的网络配置之间可能存在负面干扰。

### D.Calico eBPF 策略实现

Calico CNI插件将Felix守护程序部署到每个节点，该守护进程将内核路由编程到本地Pod并且使用边界网关协议分发路由信息。Felix通过API服务器从K8s etcd获取网络策略定义，并且将他们转换为BPF程序，这些程序被加载到内核中，并附加到每个Pod的veth接口的tc钩子上。只有来自Pod标签匹配的网络策略规则才会被附加，从而实现可拓展的实现，Linux的conntrack（Linux内核网络栈的一个核心功能，允许内核跟踪所有的逻辑网络连接或流）对此进行了强化（见本目录下另一篇关于conntrack的文章）。Calico还以自定义的资源定义（CRD）的形式实现自己的网络策略模型，该模型提供了拓展的操作范围，例如根据其顺序确定规则的优先级。Felix进一步使用快速数据路径（xDP）在节点上实现数据包过滤，以防止拒绝服务（DoS）攻击。

### E.多租户网络平台中的网络策略

基于微服务的应用程序可以包含数百个微服务，像netfilter这样的公司使用500多个微服务来运行其电影流系统。在多租户边缘平台中，不同的应用程序可能来自不同的提供商，需要强大网络隔离，因此必须遵循最小权限原则，为微服务之间每个预期的租户件交互显式指定网络策略，该原则旨在减少攻击者的安全攻击接口。同样所有租户内容器通信都需要显式允许策略。因此多租户平台中的策略数量可能会随着租户应用程序的数量和大小急剧增加。因此，对于5G边缘部署，不仅要对k8s的网路性能进行基准测试，还要评估网络策略的性能开销。

## III.间接费用考核

本节总结了各种条件下节点内和节点间场景下Calico网络策略的网络时延开销的评估结果。

### A.基准和架构评估

我们使用netperf的TCP流和请求-响应模式分别进行吞吐量和端到端的延迟测量。我们将netperf配置为测试长度为120秒，目标是99%的置信度，即测量的平均值在实际平均值的正负2.5%以内。我们不报告吞吐量和CPU利用率结果，因为他们遵循与延迟相同的模式。

### B.实验设置

用于运行所有实验的测试平台是私有openStack云（Liberty版本）的隔离部分，如图3所示。我们从公共边缘云中使用OpenStack开始，通常最好在VM中运行容器以保护云提供商的资产。OpenStack云由主从架构组成，具有两台控制器机器，以及可调度虚拟机的droplets，这些droplets具有Inter（R）Xeon（R）CPUE5-2650 2.00GHz 处理器和带有Ubuntu xenial的64GBDIMM DDR3内存，每个droplets都有两个10Gbit网络接口。这些droplets有16个CPU内核，其中2个是为OpenStack云的操作保留的。有一个主节点和两个工作节点组成的k8s集群是使用kubeadm部署的，运行k8s版本1.19.2。所有节点都有4个vCPU和8GBRAM，并且部署在同一个物理droplets上，以消除网络延迟的变化。此外，每个节点的vCPU内核专门固定在属于droplets同一主板插槽的物理内核上。

- 增加策略数量的效果：我们创建了20个命名空间，每个命名空间包含5个微服务（Pod）。三个Pod用于性能测量；一个本地pod和两个远程Pod（每个场景一个，即节点内和节点间）并且所有的pod都被分配了相同数量的策略。Calico提供了一个policyOrder的功能，用于确定策略实施的优先级，较低的顺序优先。最低顺序是不允许任何Pod间流量的策略，然后是允许与不相关的Pod进行通信的策略，直到达到所需数量的策略。然后将最高顺序分配给允许三个选定Pod之间通信的策略。然后，网络策略的数量从零到2000不等，结果如图四所示（accross 100 Pod）。对于所有已评估数量的网络策略，我们观察到对性能的显著影响。
- 网络策略：表1涵盖了各种k8s网络策略功能的不同策略配方。虽然对表中编号5、11、13和14的每个配方都进行了单独评估，但是仅对具有相似结构的配方（即2、8、9和12）进行了一次测试。对于其余的配方，即1、3、4、6、7和10，拥有许多的策略是不合理的，因为对于GlobalNetworkPolicy，这些策略中所需的流量隔离功能可以通过命名空间中的一个策略甚至整个集群中的一个策略来实现。评估提供了与图四（100个Pod）中观察到的结果是类似结果。

|# | Deny traffic | # |	Allow traffic |
|--- | ----- | ----- | ----- |
| 1 | to a pod | 8 | to a pod |
| 2 | limiting traffic to a pod | 9 |to a pod from all NSs |
| 3 | to a NS | 10 |from a NS |
| 4 | from other NSs | 11 | from pods in another NS |
| 5 | from a pod | 12 | from external clients |
| 6 | non-whitelisted from a NS | 13 |only to a port |
| 7 | external egress | 14 | using multiple selectors |

### C.网络策略的可拓展性

从上述所有评估中可以看出，在执行网络策略方面没有发现明显的开销。我们可以将此归因于分散的Calico实现，其中仅在每个pod的veth接口上评估相关策略。即使集群中可能有2000个策略，也只会在那些选择器与通信pod匹配的策略才会在这些pod上进行评估。此外Calico的order功能是在策略创建或修改时而不是数据包时强制执行的。为了评估网络策略的可伸缩性，我们在集群中部署了三个pod，并对其进行了标记，使其与集群中所有策略相匹配。如图四的结果（accross 3 pod）仍未显示显著的策略开销，至少对于节点内流量而言。这表明Calico使用ebpf来实施网络策略是可拓展的。其原因可能是Calico进检查新流中第一个数据包的策略之后，conntrack会自动允许同一个流的后续数据包而不是重新检查每个数据包。

## IV.网络策略安全分析

网络策略限制跨不同pod的容器之间的入站和出站网络连接，拒绝任何未明确允许的连接。下面我们将讨论在同一k8s集群中托管不同边缘应用程序的微服务边缘平台提供商所感知的容器通信网络策略的安全隐患与挑战。

### A.攻击模型

我们认为攻击者已破坏其中一个边缘应用程序命名空间的微服务。这可能是由恶意或受损的边缘应用程序提供商、通过破坏在平台上部署服务的软件供应链或利用外部可访问微服务中的漏洞来实现的。
假设受感染的应用程序在非特权（非root）容器中运行，则攻击者会受到适用于该容器及其相应Pod的功能和资源限制的限制。攻击者仍可以执行任意代码并访问容器的所有允许的内存区域、文件和网络连接。因此，攻击者可能会连接到集群中其他可访问的微服务，并利用任何存在的漏洞来破坏他们。另一方面，我们要求集群虚拟化基础架构值得信赖且经过充分强化，以便攻击者无法逃离其容器沙箱并获得其托管节点上的root权限。
攻击者的目标是通过破坏其他服务（尤其是那些有权访问敏感信息或任务关键性服务以破坏其功能的服务）在集群中传播。次要目标是不被发现。然后，了解任何Pod的网络策略以及相应的可访问子网可以帮助攻击者避免触发警报。从这个意义上说，适用于集群的网络策略可以被视为敏感信息。

### B.网络策略的安全影响

如果根据最小权限原则定义的策略，仅允许必要的连接，则网络策略是阻止上述攻击者的有效工具。由此产生的网络分段减少了攻击面，保护了可能易受攻击的共同托管的应用程序，并防止任何横向攻击者移动。
然而，网络策略并不是灵丹妙药。攻击者仍可以滥用允许的连接，并以相应的微服务为目标，以实现远程执行代码或泄露敏感信息。虽然策略允许阻止与某些端口的连接，但表1中的简单规则不对允许的通信节点之间的流量提供任何细粒度的控制。其他解决方案，如网络监控和异常检测，可用于检测和阻止此类攻击。识别出可疑参与者后，可以使用网络策略将其隔离在集群的隔离部分中。但是新颁布的网络策略仅对新连接强制执行，不会切换现有的连接。

### C.网络策略安全漏洞和威胁

使用网络策略本身有几个陷阱。下面我们报告了常见的漏洞和潜在攻击以及文献中可用的补救措施。

- 1）：配置错误：StackRox的一项调查显示，91%的K8s用户遇到过安全问题，其中67%归因于配置错误。此外配置错误仍被认为是云中数据泄露的主要原因。配置网络策略时，存在大量机会，这些配置村在大量薄弱或错误的机会，这些配置无疑会威胁到容器网络隔离。这些包括错误的Pod标签规范、拼写错误等手动错误，甚至在编写后完全忘记强制实施网络策略。如果禁用了网络策略，k8s不会发出警告，而只是接受并静默忽略配置。
  - 最先进的解决方案：开放策略代理（OPA）通过在Pod创建期间通过自定义准入控制器对网络策略强制实施适当的OPA要求，有助于减低错误配置风险，例如网络策略中的拼写错误或疏忽、忘记强制执行甚至错误删除他们。但是，OPA不会验证配置的策略要求是否建立更高级别的安全目标，如网络隔离。
- 2）弱策略设计：所有安全策略（包括但不限于网络策略）都需要根据最小权限和零信任等原则进行设计和配置。应明确允许容器之间的通信，并将其保持在应用程序完成其工作所需的最低限度。例如，不应无意中向只需要从数据库读取的应用程序授予写入权限。指定网络策略时，应为应用程序容器提供一组权限，每组权限都是最小化为服务所需最小权限集。允许策略应允许最低限度，例如通过NerworkPolicyPort，而拒绝策略应该是最大程度地阻止所有未经授权的流量。但是，编写最低特权策略是一项艰巨地任务，因为k8s中典型应用程序需要服务之间、从外部（入口）到外部端点（出口）的数百个连接。
  - 最先进的解决方案：实施网络策略时规则中，OPA可以将受保护的容器允许入口规则限制为特定值，以避免授予其他容器访问权限。BASTION通过确保容器的连接仅限于自身与组成服务所需的容器之间的相互依赖关系，对容器强制实施最低特权网络访问。
- 3）租户管理员：虽然边缘基础结构通常是受信任的，但是租户及其管理员可能会提出某些风险。具有足够权限的流氓管理员可能会与攻击者串通，以削弱网络集群策略和隔离，从而悄悄地允许非法网络连接。特别是，常规k8s网络策略绑定到租户空间，并且可以由租户管理员更改。
  - 最先进的解决方案：在多租户环境中，必须遵循控制平面中的身份验证、访问控制和审核地最佳实践，以限制恶意租户和内部攻击的影响。租户管理员更改集群策略的问题在Calico中通过全局策略和优先级的概念得到解决。与配置错误一样，OPA还提供了防止使用非法网络策略启动容器的保护。
- 4）特权网络：不幸的是，容器通常以不安全的设置运行，授予不必要或无意的权限，从而给节点操作系统和节点上其他容器带来风险。网络特权容器直接在LAN网络上公开，绕过Pod网络接口和网络策略。这样容器可以通过分配给他们的网关IP地址或通过猜测相应的IP地址来访问其他节点，从而损害所需的网络隔离。
  - 最先进的解决方案：[4]中介绍了一个高性能的安全实施网络堆栈，它通过对共享主机网络命名空间的启用了网络特权的容器实施细粒度访问控制来解决此漏洞。此外尽管Calico不提供安全机制来防止主机网络命名空间滥用，但它可以防止特权容器滥用虚拟网关IP地址。Pod安全策略可确保不会生成具有无意功能或访问权限的容器。他们也可以由OPA强制执行。
- 5）易受攻击的实现：在k8s中，网络策略的实施取决于CNI插件组件。这些组件及其依赖项本身可能包含漏洞，这些漏洞可能会危及策略实施或允许攻击者拦截和重定向网络流量，例如CVE-2019-9946、CVE-2020-10749 和 CVE-2020-13597。
  - 最先进的解决方案：使软件保持最新状态可以减轻漏洞的影响，但不能解决根本问题。形式验证技术和安全的编程语言有助于提高软件的安全性。
- 6）ebpf漏洞利用：在我们的设置中，Calico/node代理将网络策略转换为ebpf程序以进行实施。因此，需要在节点上启用ebpf内核功能才能使用网络策略，这可能会增加节点的攻击面。在Linux4.4之前，所有bpf()命令都要求调用方具有CAP SYS ADMIN功能，然而，目前非特权用户可以通过将有限的ebpf程序附加到他们拥有的套接字来创建和运行这些程序。如 CVE-2020-8835 中所示，此功能不仅被利用来在内核内存中执行越界读取和写入，而且还被利用来实现权限提升 [31]。
  - 最先进的解决方案：通过将内核完全禁用bpf()系统调用的非特权访问，将内核无特权bpf禁用的系统设置为1，可以缓解此类威胁。ebpf内核验证器可以限制从cbpf程序访问哪些内核函数和数据结构。Seccomp-BPF限制了用户空间程序可用的系统调用和参数集。
- 7）泄露网络策略：如上所述，对于热衷于避免检测的攻击者来说，网络策略本身可能是最有价值的信息。获取它们的最简单的方法是从k8s API服务器或etcd请求他们，但用户平面攻击者通常无法访问这些API。具有监视控制平面流量能力的对手可以获取配置信息，例如网络策略（如果以明文形式发送）。攻击者还需要通过诸如[31]之类的漏洞利用来提供额外的内存读取功能以便从本地Calico/node代理或ebpf内核工具泄露策略。无特权攻击者可能会探测网络以推断允许的连接，但这可能会破坏保持隐身的最初目标。
  - 最先进的解决方案：如上所述，泄露策略信息的原始尝试很容易受到最佳实践的阻碍，例如为控制平面启用相互传输层安全加密，身份验证和RBAC。尽管Calico支持TLS来保护Calico组件和控制平面通信，但默认情况下，大多数提供的清单中都未配置安全性，以使Calico部署变得更容易。对于Calico，建议使用TLS与其数据存储进行通信，同时使用客户端整数和JSON Web令牌身份验证模块。出于机密性和完整性的原因，应考虑在可信执行环境（TEE）（如英特尔SGX）中运行任务关键型应用和控制平面组件，以便在每个容器应用之间的节点中创建隔离的安全区环境。英特尔SGX安全区通过加密内存隔离驻留在容器中的应用，从而保护应用内存免受恶意或特权容器以及受损节点操作系统的影响。虽然SGX在执行安全区代码时会产生性能开销，但其应用于控制平面组件以保护网络策略的机密性和完整性不会给用户平面流量增加任何开销。

## V.结语

边缘计算和容器化是多租户5G云环境中无疑重要的技术。然而，安全要求仍然阻碍其广泛采用，特别是在具有高性能和安全要求的5G URLLC边缘应用中，因为大多数安全解决方案都会损害性能。这使得找到一个低开销的容器安全解决方案变得至关重要。在评估了网络策略（k8s的可配置网络隔离安全解决方案）并且没有观察到明显的性能开销之后，我们证明了他们是此类应用程序的合适的安全解决方案。作为一般关注点，交互微服务应部署在同一节点上以确保最高性能，但是这会增加对不同租户的适当隔离的需求。

## Acknowledgment

这项研究的部分资金来自KU Leuven研究基金和弗兰德研究计划网络安全。这项研究已获得欧盟H2020 MSCA-ITN行动5GhOSTS的资助，赠款协议号814035。

## 引用

[1] Q.-V. Pham et al., “A survey of multi-access edge computing in 5G
and beyond: Fundamentals, technology integration, and state-of-the-art,”
IEEE Access, 2020.
[2] H. Hawilo et al., “Exploring microservices as the architecture of choice
for network function virtualization platforms,” IEEE Network, 2019.
[3] J. Watada, A. Roy, R. Kadikar, H. Pham, and B. Xu, “Emerging trends,
techniques and open issues of containerization,” IEEE Access, 2019.
[4] J. Nam, S. Lee, H. Seo, P. Porras, V. Yegneswaran, and S. Shin, “BASTION: A security enforcement network stack for container networks,”
in 2020 USENIX Annual Technical Conf. (USENIXATC 20), 2020.
[5] S. Sultan, I. Ahmad, and T. Dimitriou, “Container security: Issues,
challenges, and the road ahead,” IEEE Access, 2019.
[6] M. Souppaya, J. Morello, and K. Scarfone, “Application container
security guide (2nd draft),” NIST, Tech. Rep., 2017.
[7] E. Reshetova, J. Karhunen, T. Nyman, and N. Asokan, “Security of oslevel virtualization technologies: Tech. report,” arXiv:1407.4245, 2014.
[8] A. Grattafiori, “Understanding and hardening linux containers,”
Whitepaper, NCC Group, 2016.
[9] S. Vaucher, R. Pires, P. Felber, M. Pasin, V. Schiavoni, and C. Fetzer,
“SGX-aware container orchestration for heterogeneous clusters,” in 2018
IEEE 38th Conf. on Dist. Computing Syst. (ICDCS). IEEE, 2018.
[10] S. Arnautov et al., “SCONE: Secure linux containers with intel SGX,”
in 12th USENIX OSDI 16, 2016.
[11] M. S. I. Shamim, F. A. Bhuiyan, and A. Rahman, “XI commandments
of Kubernetes security: A systematization of knowledge related to
Kubernetes security practices,” in 2020 IEEE SecDev. IEEE, 2020.
[12] A. Balkan, “Kubernetes network policy recipes,” https://github.com/
ahmetb/kubernetes-network-policy-recipes/, [accessed 2021-01-28].
[13] S. Qi et al., “Understanding container network interface plugins: design
considerations and performance,” in 2020 IEEE Int. Symp. on Local and
Metropolitan Area Networks (LANMAN). IEEE, 2020.
[14] X. Nguyen, “Network isolation for K8s hard multi-tenancy,” 2020.
[Online]. Available: https://aaltodoc.aalto.fi/handle/123456789/46078
[15] “Kubernetes namespaces,” https://kubernetes.io/blog/2016/08/
kubernetes-namespaces-use-cases-insights/, [accessed 2020-11-24].
[16] “How to stop the next equifax-style megabreach,” https://www.wired.
com/story/how-to-stop-breaches-equifax/, 2017, [accessed 2021-01-14].
[17] “About eBPF,” https://docs.projectcalico.org/about/about-ebpf/, [accessed 2020-12-18].
[18] “eBPF datapath,” https://docs.cilium.io/en/v1.9/concepts/ebpf/, 2020,
[accessed 2020-12-18].
[19] CED, “How it’s built,” https://medium.com/the-andela-way/how-itsbuilt-netflix-8f62ab329011/, 2019, [accessed 2020-12-18].
[20] “Care and Feeding of Netperf 2.7.X,” https://hewlettpackard.github.io/
netperf/doc/netperf.html/, [accessed 2020-10-23].
[21] https://github.com/falcosecurity/falco/, 2020, [accessed 2021-01-27].
[22] C.-W. Tien et al., “KubAnomaly: Anomaly detection for Docker orchestration platform with neural network approaches,” Eng. Reports, 2019.
[23] “Kubernetes adoption, security, and market share trends report,”
https://www.stackrox.com/kubernetes-adoption-security-and-marketshare-for-containers/, 2020, [accessed 2020-12-18].
[24] F. Truta, “Misconfig. remains #1 cause of data breaches in the cloud,”
https://securityboulevard.com/2020/04/misconfiguration-remains-the-1-
cause-of-data-breaches-in-the-cloud/, [accessed 2021-01-14].
[25] M. Ahmed, “How to enforce Kubernetes network security policies
using OPA,” https://www.magalix.com/blog/how-to-enforce-kubernetesnetwork-security-policies-using-opa/, 2020, [accessed 2020-11-02].
[26] D. Monica, “Least priv. cont. orchestration,” https://www.docker.com/ ´
blog/least-privilege-container-orchestration/, [accessed 2020-12-17].
[27] “Zero-trust networks in Kubernetes, cloud-native applications,”
https://www.stackrox.com/wiki/zero-trust-networks-in-kubernetescloud-native-applications/, 2020, [accessed 2020-12-17].
[28] “Pod security policies,” https://kubernetes.io/docs/concepts/policy/podsecurity-policy/#host-namespaces/, 2020, [accessed 2021-01-27].
[29] M. Ahmed, “Enforce pod security policies in Kubernetes using
OPA,” https://www.magalix.com/blog/enforce-pod-security-policies-inkubernetes-using-opa/, 2020, [accessed 2021-01-27].
[30] “bpf(2) - linux manual page,” https://man7.org/linux/man-pages/man2/
bpf.2.html/, 2020, [accessed 2020-12-21].
[31] P. Manfred, “Kernel privilege escalation via improper eBPF program
verification,” https://www.thezdi.com/, 2020, [accessed 2020-12-17].
[32] M. Fleming, “A thorough introduction to ebpf,” https://lwn.net/Articles/
740157/, 2017, [accessed 2020-12-21].
[33] “Using eBPF in Kubernetes,” https://kubernetes.io/blog/2017/12/usingebpf-in-kubernetes/, 2017, [accessed 2021-01-28].
[34] “Customize the manifests,” https://docs.projectcalico.org/getting-started/
kubernetes/installation/config-options/, 2020, [accessed 2020-12-01].
[35] “Configure encryption and authentication,” https://docs.projectcalico.
org/security/comms/crypto-auth/, 2020, [accessed 2020-12-01].
[36] A. Gowda and D. Coulter, “Microsoft Docs:Enclave aware containers on
Azure ,” https://docs.microsoft.com/en-us/azure/confidential-computing/
enclave-aware-containers/, 2020, [accessed 2020-10-30].
[37] P.-L. Aublin et al., “TaLoS: Secure and transparent TLS termination
inside SGX enclaves,” Imperial College London, Tech. Rep, vol. 5, 2017.

