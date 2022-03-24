# K8S核心操作记录

## K8S基本

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
