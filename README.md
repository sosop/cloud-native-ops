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

