# K8S核心操作记录

## K8S基本概述

### 安装

[参考](https://kubernetes.io/zh/docs/tasks/tools/)

### 组件

- control plane componets：对集群做出全局决策、检测和响应集群事件
  - apiserver：公开了 Kubernetes API
  - etcd：兼具一致性和高可用性的键值数据库，保存存 Kubernetes 所有集群数据的后台数据库
  - kube-scheduler：负责监视新创建的、未指定运行节点（node）的 Pods，选择节点让 Pod 在上面运行
  - kube-controller-manager：
    - Node Controller：负责在节点出现故障时进行通知和响应
    - Job controller：监测代表一次性任务的 Job 对象，然后创建 Pods 来运行这些任务直至完成
    - Endpoints Controller：填充端点(Endpoints)对象(即加入 Service 与 Pod)
    - Service Account & Token Controllers：为新的命名空间创建默认帐户和 API 访问令牌
  - cloud-controller-manager：可以将集群连接到云提供商的 API 之上， 并将与该云平台交互的组件同与集群交互的组件分离开来
    - Node Controller：在节点终止响应后检查云提供商以确定节点是否已被删除
    - Route Controller：在底层云基础架构中设置路由
    - Service Controller：创建、更新和删除云提供商负载均衡器
- Node
  - kubelet：每个Node上运行的代理，保证容器运行在pod中，接收PodSpecs确保PodSpecs描述的容器处于运行状态且健康
  - kube-proxy：个节点上运行的网络代理，service概念的一部分
  - container runtime：docker、containerd、CRI-O、podman
- Addons
  - 插件提供集群级别的功能，命名空间域的资源属于 `kube-system`，[插件详情](https://kubernetes.io/zh/docs/concepts/cluster-administration/addons/)
  - DNS
  - web界面
  - 容器资源监控
  - 集群层面日志

### 对象

- *对象* 是持久化的实体
- 描述信息
  - 哪些容器化应用在运行
  - 可以被应用使用的资源
  - 关于应用运行时表现的策略，比如重启策略、升级策略，以及容错策略
- 对象规约（spec）与状态（status）
  - spec描述了对象的所具有的特征-*期望状态*desired-state
  - status描述了对象的当前状态（Current State）
  - spec/status/metadata，[参考](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md)

- 对象描述

  - *必须字段*

    - apiVersion：创建该对象所使用的 Kubernetes API 的版本

    - kind：对象的类别

    - metadata：帮助唯一性标识对象的一些数据，name、UID、namespace
    - spec：期望的该对象的状态，[参考](https://kubernetes.io/docs/reference/kubernetes-api/)

创建对象时，必须提供对象的规约用来描述该对象的期望状态以及关于对象的一些基本信息

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # tells deployment to run 2 pods matching the template
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```



- 对象名称与IDs

  - 名称：某一时刻，只能有一个给定类型的对象具有给定的名称
    - DNS子域名
    - RFC 1123标签名
    - RFC 1035标签名
    - 路径分段名称
  - UIDs：系统生成的字符串，唯一标识对象，全局唯一标识符

- 名字空间：Kubernetes 支持多个虚拟集群，它们底层依赖于同一个物理集群，虚拟集群被称为名字空间

  - 何时使用多个命名空间：适用于存在很多跨多个团队或项目的用户的场景

    - 资源的名称需要在名字空间内是唯一的
    - 每个资源只能在一个名字空间中
    - 名字空间是在多个用户之间划分集群资源的一种方法
    - 不必使用多个名字空间来分隔仅仅轻微不同的资源，使用[标签](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/labels/) 来区分同一名字空间中的不同资源

  - 使用名字空间

    - [参考](https://kubernetes.io/zh/docs/tasks/administer-cluster/namespaces/)

    - 避免使用前缀 `kube-` 创建名字空间，因为它是为 Kubernetes 系统名字空间保留的

      `kubectl get ns`

    - 初始名字空间

      - default： 没有指明使用其它名字空间的对象所使用的默认名字空间
      - kube-system：系统创建对象所使用的名字空间
      - kube-public：自动创建，所有用户可以读取它，一种约定
      - kube-node-lease：用于与各个节点相关的 [租约（Lease）](https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/lease-v1/)对象，节点租期允许 kubelet 发送[心跳](https://kubernetes.io/zh/docs/concepts/architecture/nodes/#heartbeats)，由此控制面能够检测到节点故障

    - 设置命名空间

    `kubectl get po -n default`

  - 名字空间和DNS

    - 创建service会创建一个对应的DNS条目，格式<服务名称>.<名字空间名称>.svc.cluster.local
    - 只使用 `<服务名称>`，它将被解析到本地名字空间的服务
    - 跨名字空间访问使用完全限定域名（FQDN）

  - 并非所有对象都在名字空间中

    - 名字空间资源本身并不在名字空间中
    - 底层资源，如节点、持久化卷不属于任何名字空间

    `kubectl api-resources --namespaced=false`

  - 自动打标签

    - 控制面会为所有名字空间设置一个不可变更的标签
    - kubernetes.io/metadata.name: namespace-name

- 标签和选择算符

  - 附加到 Kubernetes 对象上的键值对

  - 用于指定对用户有意义且相关的对象的标识属性，不直接对核心系统有语义含义

  - 标签可以用于组织和选择对象的子集

  - 每个键对于给定对象必须是唯一的

  - 标签能够支持高效的查询和监听操作

  - 动机：标签使用户能够以松散耦合的方式将他们自己的组织结构映射到系统对象

    - [推荐使用的标签](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/common-labels/)

  - 标签选择算符

    - 标签不支持唯一性，许多对象携带相同的标签

    - 通过标签选择算符识别一组对象

    - API 目前支持两种类型的选择算符

      - *基于等值*

        - 运算符= == ！=

        ```yaml
        environment = production
        tier != frontend
        # 逗号运算符来过滤
        environment=production,tier!=frontend
        ```

        ```yaml
        apiVersion: v1
        kind: Pod
        metadata:
          name: cuda-test
        spec:
          containers:
            - name: cuda-test
              image: "k8s.gcr.io/cuda-vector-add:v0.1"
              resources:
                limits:
                  nvidia.com/gpu: 1
          nodeSelector:
            accelerator: nvidia-tesla-p100
        ```

        

      - *基于集合*

        - 操作符：in notin exists

        ```yaml
        environment in (production, qa)
        tier notin (frontend, backend)
        # 包含有 partition 标签的资源不校验值
        partition
        # 没有 partition 标签的资源
        !partition
        ```

    - api 对象中设置引用

      - service & ReplicaController：一个 `Service` 指向的一组 Pods 是由标签选择算符定义的，一个 `ReplicationController` 应该管理的 pods 的数量也是由标签选择算符定义的
      - Job、 Deployment、 Replica Set 和 DaemonSet， 支持基于集合的需求

    ```yaml
    "metadata": {
      "labels": {
        "key1" : "value1",
        "key2" : "value2"
      }
    }
    
    kubectl get pods -l environment=production,tier=frontend
    kubectl get pods -l 'environment in (production),tier in (frontend)'
    kubectl get pods -l 'environment in (production, qa)'
    kubectl get pods -l 'environment,environment notin (frontend)'
    
    "selector": {
        "component" : "redis",
    }
    
    selector:
        component: redis
    # In、NotIn、Exists. DoesNotExist
    selector:
      matchLabels:
        component: redis
      matchExpressions:
        - {key: tier, operator: In, values: [cache]}
        - {key: environment, operator: NotIn, values: [dev]}
    ```

    

- 注解
  - 由声明性配置所管理的字段
  - 构建、发布或镜像信息
  - 指向日志记录、监控、分析或审计仓库的指针
  - 可用于调试目的的客户端库或工具信息
  - 用户或者工具/系统的来源信息
  - 轻量级上线工具的元数据信息
  - 负责人员的电话或呼机号码
  - 从用户到最终运行的指令，以修改行为或使用非标准功能

注解为对象附加任意的非标识的元数据，客户端程序能够获取这些元数据信息；注解中的元数据，可以很小，也可以很大，可以是结构化的，也可以是非结构化的，能够包含标签不允许的字符

键和值必须是字符串。 换句话说，你不能使用数字、布尔值、列表或其他类型的键或值



```yaml
"metadata": {
  "annotations": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```

- Finalizer

  带有命名空间的键，告诉 Kubernetes 等到特定的条件被满足后， 再完全删除被标记为删除的资源；

  使用 Finalizer 控制资源的垃圾收集，可以定义一个 Finalizer，在删除目标资源前清理相关资源或基础设施

- 字段选择器

  允许你根据一个或多个资源字段的值筛选资源

  支持的字段，不同的 Kubernetes 资源类型支持不同的字段选择器，所有资源类型都支持 `metadata.name` 和 `metadata.namespace`

  ```shell
  kubectl get pods --field-selector status.phase=Running
  
  kubectl get pods --field-selector=status.phase!=Running,spec.restartPolicy=Always
  
  kubectl get statefulsets,services --all-namespaces --field-selector metadata.namespace!=default
  ```

- 属主与附属

  - [具体参考](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/owners-dependents/)

  

## K8S架构

### 节点

节点可以是一个虚拟机或者物理机器，每个节点包含运行 Pods 所需的服务； 这些节点由控制面板负责

节点上的组件：kubelet、容器运行时、kube-proxy

#### 向API服务器添加节点方式

- 节点上的 `kubelet` 向控制面执行自注册

- 手动添加一个 Node 对象

- 节点自注册

  -  kubelet 标志 `--register-node` 为 true时，尝试向 API 服务注册自己
  - `--kubeconfig` - 用于向 API 服务器表明身份的凭据路径
  - `--cloud-provider` - 与某云驱动进行通信以读取与自身相关的元数据的方式
  - `--register-with-taints` - 使用所给的污点列表（逗号分隔的 `<key>=<value>:<effect>`）注册节点
  - `--node-ip` - 节点 IP 地址
  - `--node-labels` - 在集群中注册节点时要添加的标签
  - `--node-status-update-frequency` - 指定 kubelet 向控制面发送状态的频率

- 手动节点管理

  - kubelet 标志 `--register-node=false`

  - 结合使用节点上的标签和 Pod 上的选择算符来控制调度

  - 如果标记节点为不可调度（unschedulable），将阻止新 Pod 调度到该节点之上，但不会 影响任何已经在其上的 Pod

    `kubectl cordon $NODENAME`

  - 被DaemonSet控制器创建的 Pod 能够容忍节点的不可调度属性。 DaemonSet 通常提供节点本地的服务，即使节点上的负载应用已经被腾空，这些服务也仍需 运行在节点之上

#### 节点状态

- 状态信息

  - 地址
    - HostName
    - ExternalIP
    - InternalIP
  - 状况 Conditions 描述了所有 `Running` 节点的状态
    - Ready： True节点是健康的并已经准备好接收 Pod ；false节点不健康而且不能接收 Pod；Unknown节点控制器在最近 `node-monitor-grace-period` 期间（默认 40 秒）没有收到节点的消息
    - DiskPressure：`True` 表示节点存在磁盘空间压力磁盘可用量低；否则为 `False`
    - MemoryPressure：`True` 表示节点存在内存压力
    - PIDPressure：`True` 表示节点存在进程压力，即节点上进程过多
    - NetworkUnavailable：`True` 表示节点网络配置不正确
  - 容量与可分配
    - CPU、内存和可以调度到节点上的 Pod 的个数上限
    - capacity 节点拥有的资源总量
    - allocatable 节点上可供普通 Pod 消耗的资源量
  - 信息：描述节点的一般信息

  `kubectl describe node <节点名称>`

#### 心跳

两种形式的心跳

- 更新节点的 `.status`
- lease对象 在 `kube-node-lease` 命名空间中

kubelet 负责创建和更新节点的 `.status`，以及更新它们对应的 `Lease`

- 当状态发生变化时，或者在配置的时间间隔内没有更新事件时，kubelet 会更新 `.status`。 `.status` 更新的默认间隔为 5 分钟
- `kubelet` 会每 10 秒创建并更新其 `Lease` 对象



#### 节点控制器

当节点注册时为它分配一个 CIDR 区段

保持节点控制器内的节点列表与云服务商所提供的可用机器列表同步

监控节点的健康状况，节点控制器每隔 `--node-monitor-period` 秒检查每个节点的状态

- 节点不可达的情况下，在 Node 的 `.status` 中更新 `NodeReady` 状况，节点控制器将 `NodeReady` 状况更新为 `ConditionUnknown`
- 如果节点仍然无法访问：对于不可达节点上的所有 Pod触发逐出

逐出率限制

节点控制器把逐出速率限制在每秒 `--node-eviction-rate` 个（默认为 0.1），每 10 秒钟内至多从一个节点驱逐 Pod

 节点控制器会同时检查可用区域中不健康NodeReady 状况为 `ConditionUnknown` 或 `ConditionFalse`） 的节点的百分比

- 如果不健康节点的比例超过 `--unhealthy-zone-threshold` （默认为 0.55）， 驱逐速率将会降低
- 如果集群较小（意即小于等于 `--large-cluster-size-threshold` 个节点 - 默认为 50），驱逐操作将会停止
- 否则驱逐速率将降为每秒 `--secondary-node-eviction-rate` 个（默认为 0.01）

节点控制器还负责驱逐运行在拥有 `NoExecute` 污点的节点上的 Pod，除非这些 Pod 能够容忍此污点

节点控制器还负责根据节点故障为其添加污点

##### 资源容量跟踪

##### Node 对象会跟踪节点上资源的容量



#### 节点优雅关闭

kubelet 会尝试检测节点系统关闭事件并终止在节点上运行的 Pods节点

优雅关闭特性依赖于 systemd，它要利用systemd抑制器锁在给定的期限内延迟节点关闭；受GracefulNodeShutdown特性门控控制	

关闭节点过程中，kubelet 分两个阶段来终止 Pods：

- 终止在节点上运行的常规 Pod
- 终止在节点上运行的关键 Pod

kubeletConfiguration:

- ShutdownGracePeriod：指定节点应延迟关闭的总持续时间
- ShutdownGracePeriodCriticalPods：在节点关闭期间指定用于终止关键 Pod 的持续时间。该值应小于 `ShutdownGracePeriod`



#### 交换内存管理



### 控制面到节点通信

### 控制器

### 云控制器管理器

### 垃圾收集

垃圾收集是 Kubernetes 用于清理集群资源的各种机制的统称

### 容器运行时接口

CRI 是一个插件接口，它使 kubelet 能够使用各种容器运行时，无需重新编译集群组件

容器运行时接口（CRI）是 kubelet 和容器运行时之间通信的主要协议



## 容器

### 镜像

#### 更新镜像

- 拉去策略imagePullPolicy
  - IfNotPresent：当镜像在本地不存在时才会拉取
  - Always：将名称解析为一个镜像摘要，应的摘要已在本地缓存，kubelet 就会使用其缓存的镜像； 否则，kubelet 就会使用解析后的摘要拉取镜像，并使用该镜像来启动容器
  - Never：Kubelet 不会尝试获取镜像

为了确保 Pod 总是使用相同版本的容器镜像，将 `<image-name>:<tag>` 替换为 `<image-name>@<digest>`

- ImagePullBackOff：无法拉取容器镜像

### 容器环境

### 容器运行时类 container runtime class

### 容器生命周期回调

- PostStart：容器被创建之后立即被执行，不能保证回调会在容器入口点（ENTRYPOINT）之前执行
- PreStop：在容器因 API 请求或者管理事件而被终止之前此回调会被调用

#### 回调程序的实现

- Exec - 在容器的 cgroups 和名称空间中执行特定的命令
- HTTP - 对容器上的特定端点执行 HTTP 请求

#### 回调处理程序执行

- `httpGet` 和 `tcpSocket` 在kubelet 进程执行
- `exec` 则由容器内执行



## 工作负载

工作负载资源

- Deployment：`Deployment` 很适合用来管理你的集群上的无状态应用
- ReplicaSet：替换原来的资源ReplicationController
- StatefulSet：能够运行一个或者多个以某种方式跟踪应用状态的 Pods
- DaemonSet：提供节点本地支撑设施的 `Pods`
- job：定义一些一直运行到结束并停止的任务，一次性任务
- cronjob：周期性任务

### Pods

*Pod* 是可以在 Kubernetes 中创建和管理的、最小的可部署的计算单元

静态POD：直接由特定节点上的 `kubelet` 守护进程管理

容器探针

- `ExecAction`（借助容器运行时执行）
- `TCPSocketAction`（由 kubelet 直接检测）
- `HTTPGetAction`（由 kubelet 直接检测）

#### Pod的生命周期

##### Pod阶段

Pod 的阶段（Phase）是 Pod 在其生命周期中所处位置的简单宏观概述

- Pending：Pod 已被 Kubernetes 系统接受，但有一个或者多个容器尚未创建亦未运行。此阶段包括等待 Pod 被调度的时间和通过网络下载镜像的时间
- Running：Pod 已经绑定到了某个节点，Pod 中所有的容器都已被创建
- Succeeded：Pod 中的所有容器都已成功运行，并且不会再重启
- Failed：Pod 中的所有容器都已终止，并且至少有一个容器是因为失败终止
- Unknown：因为某些原因无法取得 Pod 的状态；通常是因为与 Pod 所在主机通信失败

##### 容器状态

Kubernetes 会跟踪 Pod 中每个容器的状态，kubelet通过容器云形式CRI为Pod创建容器

- waiting
- running
- terminated

##### 容器重启策略

restartPolicy：

- Always
- OnFailure
- Never

##### pod状况podconditions

- PodScheduled：Pod 已经被调度到某节点
- ContainersReady：Pod 中所有容器都已就绪
- Initialized：所有的init容器已经完成
- Ready：Pod 可以为请求提供服务，并且应该被添加到对应服务的负载均衡池中

字段情况

- type：Pod 状况的名称
- status：状况是否适用
- lastProbeTime：上次探测 Pod 状况时的时间戳
- lastTransitionTime：Pod 上次从一种状态转换到另一种状态时的时间戳
- reason：表述上次状况变化的原因
- message：上次状态转换的详细信息

##### 容器探针

- 检查机制
  - exec：容器内执行指定命令，命令退出时返回码为 0 则认为诊断成功
  - grpc：执行一个远程过程调用，响应的状态是 "SERVING"，则认为诊断成功
  - httpGet：对容器的 IP 地址上指定端口和路径执行 HTTP `GET` 请求，状态码大于等于 200 且小于 400，则诊断被认为是成功的
  - tcpSocket：IP 地址上的指定端口执行 TCP 检查
- 探测结果
  - Success
  - Failure
  - Unknown
- 探测类型
  - livenessProbe：存活探针，指示容器是否正在运行
    - 何时使用？如果你希望容器在探测失败时被杀死并重新启动，那么请指定一个存活态探针， 并指定`restartPolicy` 为 "`Always`" 或 "`OnFailure`"
  - readinessProbe：就绪探针，指示容器是否准备好为请求提供服务
    - 何时使用？如果要仅在探测成功时才开始向 Pod 发送请求流量，请指定就绪态探针；希望容器能够自行进入维护状态；应用程序对后端服务有严格的依赖性
  - startupProbe：指示容器中的应用是否已经启动，如果提供了启动探针，则所有其他探针都会被 禁用，直到此探针成功为止
    - 何时使用？对于所包含的容器需要较长时间才能启动就绪的 Pod 而言，启动探针是有用的



#### init容器

init 容器是一种特殊容器，在 Pod 内的应用容器启动之前运行，Init 容器可以包括一些应用镜像中不存在的实用工具和安装脚本

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup myservice.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for myservice; sleep 2; done"]
  - name: init-mydb
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup mydb.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for mydb; sleep 2; done"]
```

```shell
kubectl get -f myapp.yaml
kubectl describe -f myapp.yaml
kubectl logs myapp-pod -c init-myservice
```

#### Pod拓扑分布约束

#### 临时容器

一种特殊的容器，该容器在现有 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) 中临时运行，以便完成用户发起的操作

### 工作负载资源

