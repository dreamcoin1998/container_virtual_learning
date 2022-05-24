# Pod详解

[TOC]

### Pod定义详解

```yaml
apiVersion: v1
kind: Pod # 资源类型
metadata: # 元数据
  name: string # pod的名称
  namespace: string # pod所属的命名空间
  labels: # 自定义标签列表
    - name: string 
  annotations: # 自定义注解列表
    - name: string
spec: # pod中容器的详细定义
  containers: # pod中的容器列表
  - name: string # 容器的名称
    image: string # 容器的镜像名称
    imagePullPolicy: [Always | Never | IfNotPresent] # 如下所述
    command: [string] # 镜像的启动命令列表，如果不设置则使用镜像打包时使用的启动命令
    args: [string] # 容器的启动命令列表
    workingDir: string # 容器的工作目录
    volumeMounts: # 挂载到容器内部的存储卷配置
    - name: string # 引用pod定义的共享存储卷名称需使用vollumes[]部分定义的共享存储卷名称
      mountPath: string # 存储卷在容器内挂载的绝对路径，少于512个字符
      readOnly: boolean # 是否为只读模式，默认为读写模式
    ports: # 容器需要暴露的端口号列表
    - name: string # 端口的名称
      containerPort: int # 容器需要监听的端口号
      hostPort: int # 容器所在主机需要坚挺的端口号，默认与containerPort相同
      protocol: string # 端口协议，支持TCP和UDP。默认TCP
    env: # 容器运行前需要设置的环境变量列表
    - name: string # 环境变量的名称
      value: string # 环境变量的值
    resource: # 资源限制和资源请求的设置
      limit: # 资源限制的设置
        cpu: string # cpu限制，单位为core数，将用于docker run --cpu-shares参数
        memory: string # 内存限制，单位为MiB、GiB等，将用于docker run --memory参数
      requests: # 资源限制的设置
        cpu: string # cpu请求，单位为core数，容器启动的初始可用数量
        memory: string # 内存请求，单位为MiB、GiB等，容器启动的初始可用数量
    livenessProbe: # 对Pod内个容器的健康检查的设置，当探测无响应几次后，系统将自动重启该容器，可以设置的方法包括，exec、httpGet和tcpSocket。对一个容器仅需设置一个健康检查方法
      exec: # 对Pod内各容器健康检查的设置，exec方式
        command: [string] # exec方式需要指定的命令或者脚本
      httpGet: # 对Pod内各容器健康检查的设置，HTTPGet方式，需指定path、port
        path: string # 
        port: number
        scheme: string
        httpHeaders:
        - name: string
          value: string
      tcpSocket: # 对Pod内各容器健康检查的设置，tcpSocket方式
        port: number
      initialDelaySeconds: 0 # 容器启动后首次探测的时间
      timeoutSeconds: 0 # 对容器健康检查的探测等待响应的超时时间设置，单位为s，默认值为1s，若超过该超时时间，则将认为该容器不健康，会重启该容器
      periodSeconds: 0 # 对容器健康检查的定期探测时间设置，单位为s，默认10秒探测一次
      successThreshold: 0 
      failureThreshold: 0
    securityContext:
      privileged: false
  restartPolicy: [Always | Never | OnFailure] # 见下方
  nodeSelector: object # 设置Node的Label，以key：value的方式指定，Pod将被调度到具有这些Label的Node上
  imagePullSecrets: # pull镜像使用的secret名称，以name: secretKey格式指定
  - name: string
  hostNetwork: false # 是否使用主机网络模式，默认值为false,设置为true表示容器使用宿主机网络，不再使用Docker网桥，该Pod将无法在同一台宿主机上启动第二个副本
  volumes: # Pod上定义的共享存储卷列表
  - name: string # 共享存储卷的名称，容器定义部分的containers[].volumeMounts[].name将引用该存储卷的名称
    emptyDir: {} # 类型为emptyDir的存储卷，表示与Pod同生命周期的一个临时目录，其值为一个空对象即emptyDir:{}
    hostPath:  # 类型为hostPath的存储卷，表示Pod挂载的宿主机目录
      path: string # Pod容器挂载的宿主机目录
    secret: # 类类型为secret的存储卷，表示挂载集群预定义的secret对象到容器内部
      secretName: string
      items:
      - key: string
        path: string
    configMap: # 类型为configMap的存储卷，表示挂载到集群预定义的configMap对象到容器内部
      name: string
      items: 
      - key: string
        path: string
```

`spec.containers[].imagePullPolicy`：镜像拉取策略：

- Always：每次都尝试重新拉取镜像
- IfNotPresent：如果本地有该镜像，则使用本地的镜像，本地不存在时拉取镜像
- Never：仅使用本地镜像

如果包含以下配置，系统将默认设置为imagePullPolicy=Always：

- 不设置imagePullPolicy，也未指定镜像的tag
- 不设置imagePullPolicy，镜像tag为latest
- 启用了名为AlwaysPullImages的准入控制器

`spec.restartPolicy`：Pod的重启策略：

- Always：Pod一旦终止运行，则无论容器是如何终止的，kubelet都将重启它
- OnFailure：只有Pod以非零退出码终止时，kubelet才会重启该容器，如果容器正常结束，则kubelet不会重启它
- Never：Pod终止后，kubelet将退出码报告给Master，不会重启该Pod

### 静态Pod

静态Pod由kubelet管理的，仅存在特定Node上的Pod，不能通过API Server进行管理，无法与Replication Controller、Deployment或者DaemonSet进行关联，并且kubelet无法对他们进行健康检查。

静态Pod总是由kubelet创建，并且总在kubelet所在的Node上运行

创建静态Pod的两种方式：

- 配置文件方式：
  - 设置启动参数 --pod-manifest-path（或在kubelet配置文件设置staticPodPath）指定需要监控的配置文件所在的路径，kubelet会定期扫描该目录并根据该目录下的.yaml或.json文件进行创建操作
  - 在Master上删除Pod操作只是会让Pod变成Pending状态
  - 要删除，需要到该Node节点上删除配置文件
- HTTP方式
  - 通过设置--manifest-url，kubelet定期从该URL下下载Pod的定义文件，创建Pod

### ConfigMap

- 生成容器内的环境变量
- 设置容器启动命令的启动参数
- 义Volume的形式挂载为容器内部的文件或目录

两种创建方式分别是：

- YAML方式创建
- 通过kubectl命令行方式创建: kubectl create configmap
  - --from-file指定文件路径创建
  - --from-file指定目录路径创建，目录下每个文件名都被设置为key
  - --fromliteral创建，通过key=value方式创建configmap内容

容器对configmap的使用有两种方式：

- 通过环境变量获取
- 通过Volume挂载的方式将ConfigMap内容挂载到容器内部的文件或目录

使用ConfigMap的限制条件：

- ConfigMap必须在Pod之前创建，Pod才能够引用它
- 如果Pod使用envFrom基于ConfigMap定义环境变量，无效的环境变量将被忽略，并在事件记录中被记录为InvalidVariableNames
- ConfigMap只被相同命名空间中的Pod引用
- ConfigMap无法用于静态Pod

### 在容器内获取Pod信息（Downward API）

Downward API通过以下两种方式将Pod和容器的元数据信息注入容器内部：

- 环境变量：将Pod或container信息设置为容器内的环境变量
- Volume挂载：将Pod或container信息以文件的形式挂载到容器内部

#### 环境变量的方式

通过Downward API将Pod的 IP、名称和所在的明明空间注入容器的环境变量中
环境变量不直接设置value，而是通过valueFrom对Pod的元数据进行引用

1、将Pod的信息设置为容器内的环境变量

- spec.nodeName: Pod所在的Node名称
- metadata.name: Pod名称
- metadata.namespace: Pod所在的命名空间的名称
- status.podIP: Pod的IP地址
- spec.serviceAccountName: Pod使用的ServiceAccount名称

2、将container的信息设置为环境变量

- requests.cpu
- limits.cpu
- requests.memory
- limits.memory

#### 通过Volume的挂载方式

1、将pod信息挂载到volume

通过fieldRef字段设置需要引用的Pod元数据信息，将其设置到volume的items中：

- metadata.labels: Pod的Label列表
- metadata.nameannotations: Pod的Annotation列表

2、将Downward API将COntainer资源的限制信息设置到Volume：

- requests.cpu: 容器的CPU请求值
- limit.cpu: 容器的CPU限制值
- requests.memory: 容器的内存请求值
- limit.memory: 容器的内存限制值

#### Downward支持的Pod和Contianer信息

1、可以通过fieldRef设置的元数据如下：

- metadata.name: pod名称
- metadata.namespace: pod所在的命名空间的名称
- metadata.uid：pod的uid，从kubernetes 1.8.0-alpha.2版本开始支持
- metadata.labels['<KEY>']: Pod的某个Label值，通过<KEY>进行引用，从Kubernetes1.9版本开始支持
- metadata.annotations['<KEY>']: Pod的某个annotation值，通过<KEY>进行引用，从Kubernetes1.9版本开始支持

2、通过resourceFieldRef设置的数据如下：

- Container级别的CPU Limit
- Container级别的CPU Request
- Container级别的Memory Limit
- Container级别的Memory Request
- Container级别的临时存储空间Limit，从Kubernetes1.8.0-beta.0版本开始支持
- Container级别的临时存储空间Request，从kubernetes1.8.0-beta.0版本开始支持

3、对以下信息通过fieldRef字段进行设置：

- metadata.labels: Pod的labels列表，每个label都以key为文件名，value为文件内容，每个label各占一行
- metadata.annotation: Pod的annotation列表，每个annotation以key为文件名，value为文件内容，每个annotation各占一行

4、以下pod元数据信息可以被设置为容器内的环境变量：

- status.podIP: Pod的IP地址
- spec.serviceAccountName: Pod使用的ServiceAccount名称
- spec.nodeName: Pod所在的Node的名称，从kubernetes 1.4.0-alpha.3版本开始支持
- spec.hostIP: pod所在的Node的IP地址，从kubernetes 1.7.0-alpha.1版本开始支持

### Pod的生命周期和重启策略

Pod在整个生命周期被系统定义位各种状态，熟悉Pod的各种状态对于理解如何设置Pod的调度策略，重启策略是很重要的：

- Pending：API Server已经创建Pod，但是在Pod内还有一个或多个的容器的镜像还没有创建，包括正在下载镜像的过程
- Running：Pod内的各种镜像已经创建，且至少有一个容器处于运行状态，正在启动状态或者正在重启状态
- Succeeded：Pod内所有容器均成功执行或退出，且不会再重启
- Failed：Pod内所有容器均已退出，但至少有一个容器为退出失败状态
- Unknown：由于某种原因无法获取该Pod的状态，可能由于网络通信不通畅导致

Pod上的重启策略应用在Pod内的所有容器上，并且仅在Pod所处的Node上有kubelet进行判断和重启，kubelet将根据重启策略的设置进行相应操作，重启策略包括：

- Always：当容器失效时，kubelet自动重启该容器
- OnFailure：当容器终止运行且退出码不为0，由kubelet重启该容器
- Never：不论容器运行状态如何，kubelet都不会重启该容器

kubelet重启失效容器时间间隔以sync-frequency乘以2n计算，最长延时5min，并且在成功重启后的10min内重置该时间

不同的Pod控制器对于重启策略的要求是不一样的：

- RC（ReplicationCOntroller）和DaemonSet：必须设置为Always，需要保证该容器持续运行
- Job：OnFailure或Never：确保容器执行完成后不自动重启
- kubelet：在Pod失效时自动重启它，不论将RestartPolicy设置为什么值，也不会对Pod进行健康检查

### Pod健康检查和服务可用性检查

对Pod的健康检查可以用三类探针：

- LivenessProbe探针：用于判断容器是否存活（Running状态），如果LivenessProbe判断容器不健康，则kubelet将杀掉容器，并根据该容器的重启策略做相应的处理。如果不包含LivenessProbe探针，那么kubelet认为该容器的LivenessProbe探针返回的值永远是Success。
- ReadinessProbe探针：用于判断容器服务是否可用（Ready状态），达到Ready状态的Pod才可以接受请求。对于被Service管理的Pod，Service与Pod Endpoint的关联关系也将基于Pod是否Ready及逆行设置。如果在运行过程中Ready状态变为False，则系统将自动冲Service的后端EndPoint列表中隔离出去，后续再把恢复到Ready状态的Pod加回后端Endpoint列表，由此保证客户端在访问Service时不会被转发到服务不可用节点的Pod实例上。ReadinessProbe也是定期触发执行的，存在于Pod整个生命周期中。
- StartupProbe探针：某些应用可能会遇到启动比较慢的问题，这种情况下会造成容器ReadinessProbe不适用，因为这属于”有且仅有一次“的超长延时，可以通过StartupProbe探针解决问题。

以上三种探针均可以用下面的实现方式》

- ExecAction：在容器内部运行一个命令，如果该命令的返回码为0，则表明容器健康
- TCPSocketActon：通过容器的IP地址和端口号执行TCP检查，如果能够建立TCP连接，则表明容器健康
- HTTPGetAction：通过容器的IP地址、端口号及路径调用HTTP Get方法，如果响应的状态码大于等于200且小于400，则认为容器健康

对于每种探测方式，都需要设置：

- initialDelaySeconds：启动容器后进行首次健康检查的等待时间，单位为s
- timeoutSeconds：健康检查发送请求后等待响应的超时时间，单位为s。当发生时，kubelet会认为容器已经无法提供服务，将会重启该容器

### Pod调度

大多数情况下，我们创建一个RC、Deployment、DaemonSet、Job等控制器完成对一组Pod副本的创建，调度及全生命周期的自动控制任务。

在真实的生产环境中还存在如下所述的特殊需求：

- 不同的Pod亲和性（PodAffinity）：比如MySQL数据库与Redis中间件不能被调度到同一个目标节点上，或者两种不同的Pod必须被调度到同一个Node上，以实现本地文件共享或本地网络通信等需求。
- StatefulSet：有状态集群的调度：每个服务的启动有严格的次序，或者集群需要持久化保存状态的数据，所以不管Pod在哪个Node上启动都需要挂载原来的Volume，因此这些Pod还需要捆绑具体的PV
- DaemonSet：在每个Node上调度并且仅仅创建一个Pod副本
- Job：批处理作业，需要多个副本协同工作，当所有的Pod都完成任务，整个批处理作业都结束了。Pod运行且仅运行一次，通常用Job或者时CronJob。

在K8s 1.9之前，删除控制器后这些Pod副本不会被删除；但是在1.9之后，这些Pod副本会被删除。可以通过kubectl命令的--cascade=false参数取消这个特性
```
kubectl delete replicaset my-repset --cascade=false
```

#### Deployment或RC：全自动调度

Deployment或RC的主要功能之一就是自动部署一个容器应用的多份副本，以及持续监控副本的数量，在集群内始终维持用户指定的副本数量

#### NodeSelector: 定向调度

kube-schedule进程也就是kubernetes master上的Schedule服务负责实现Pod的调度

可以通过Node打标签和Pod的nodeSelector属性相匹配，达到定向调度的目的


(1) 通过kubectl label 命令给目标Node打上标签

```
kubectl label nodes <node-name> <label-key>=<label=value>
```

```
kubectl label nodes k8s-node-1 zone=north
```

(2) 在Pod定义加上nodeSelector设置，以redis-master-controller.yaml为例

``` yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: redis-master
  labels:
    name: redis-master
spec:
  replicas: 1
  selector:
    name: redis-master
  template:
    metadata:
      labels:
        name: redis-master
    spec:
      containers:
      - name: master
        image: kuberguide/redis-master
        ports:
        - containerPort: 6379
      nodeSelector:
        zone: north
```

在集群中不存在包含相应标签的Node,即使在集群中还有其他可供使用的Node，这个Node也无法成功被调度

除了用户自行给Node添加标签外，K8s也会给Node预定义一些标签：

- kubernetes.io/hostname
- beta.kubernetes.io/os
- beta.kubernetes.io/arch
- kubernetes.io/os
- kubernetes.io/arch

NodeSelector通过标签的方式，简单限制了Pod所在节点的方法，亲和性调度则极大拓展了Pod的调度能力，主要增强的功能如下：

- 更具表达力
- 可以使用软限制、优先采用等限制方式，代替之前的硬限制，这样调度器在无法满足优先需求的情况下，会退而求其次，继续运行该Pod
- 可以依据节点上正在运行的其他Pod的标签来进行限制，而非节点本身的标签，这样就可以定义一种规则来描述Pod之间的亲和或互斥关系

亲和性调度功能包括节点亲和性和Pod亲和性：

- 节点亲和性与NodeSelector类似
- 互斥限制通过Pod标签而不是节点标签来实现

#### NodeAffinity：Node亲和性调度

NodeAffinity是Node亲和性调度策略，用于替换NodeSelector的全新调度。
目前有两种节点亲和性表达：

- RequireDuringSchedulingIgnoredDuringExeCution：必须满足指定的规则才能调度到Pod和Node上，相当于硬限制，功能与NodeSelector很像
- PreferredDuringSchedulingIgnoredDuringExecution：强调优先满足指定规则，调度器会尝试调度到Pod到Node上，但并不强求，相当于软限制。可以设置权重，定义执行的先后顺序。

IgnoredDuringExecution意思：如果一个Pod所在的节点在Pod运行期间标签发生变化，不再符合Pod节点的亲和性需求，则系统将自动忽略Node上的Label变化，该Pod能在该节点上运行。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinify:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: beta.kubernetes.io/arch # 要求只运行在amd64节点上
            operator: In
            values:
            - amd64
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: disk-type
            operator: In # 操作符
            values:
            - ssd
  containers:
  - name: with-node-affinify
    image: gcr.io/google_containers/pause:2.0
```

NodeAffinify支持的语法操作符包括In、NotIn、Exists、Does NotExists、Gt、Lt；NodeAffinify设置的注意事项：

- 如果同时定义了nodeSelector和nodeAffinify，那么两个条件都需要得到满足，Pod最终才能运行在指定的Node节点上
- 如果nodeAffinify定义了多个nodeSelectorTerms，那么其中一个能匹配成功即可
- 如果在nodeSelectorTerms中有多个matchExpressions，则一个节点必须满足所有matchExpressions才能运行该Pod

### PodAffinify：Pod亲和性与互斥调度策略

实际生产环境中有一些特殊的Pod需求：

- 存在相互依赖、频繁调用的Pod，需要被尽可能部署在同一个Node节点、机架、机房、网段等，这就是Pod之间的亲和性
- 出于避免竞争或容错的需求，使Pod远离某些特定的pod，这就是Pod之间的反亲和性或互斥性

**什么是拓扑域？**

一个拓扑域由一些具有相同特征的Node节点组成，一般用region表示机架、机房等拓扑区域，用zone表示地区这样跨度更大的拓扑区域

kubernetes内置了如下常用的默认拓扑区域：

- kubernetes.io/hostname
- topology.kubernetes.io/region
- topology.kubernetes.io/zone

**Pod亲和与互斥调度的具体做法是**

在Pod的定义上增加topologyKey属性声明对应的目标拓扑区域内几种关联的Pod要”在一起或不在一起“。Pod亲和与互斥的条件设置也是requiredDuringSchedulingIgnoredDuringExecution和preferredDuringSchedulingIgnoredDuringExecution。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-flag
  labels:
    security: "S1"
    app: "nginx"
spec:
  containers:
  - name: nginx
    image: nginx
```