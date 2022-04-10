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
