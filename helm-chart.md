# helm chart使用

## 简介

- k8s应用的包管理工具，用来管理chart

- Helm chart 用来封装 Kubernetes 原生应用程序的 YAML 文件，可以在你部署应用的时候自定义应用程序的一些 metadata
- 主要作用
  - 应用程序封装
  - 版本管理
  - 依赖检查
  - 便于应用程序分发

架构

![arch](helm-chart.png)

Helm 可以安装本地或者远程的 chart

## 安装

[安装](https://helm.sh/zh/docs/intro/install/)



## 使用

- *Chart* 代表着 Helm 包
  - 所有资源定义

- *Repository* 存放和共享 charts 的地方
- *Release* 是运行在 Kubernetes 集群中的 chart 的实例

### 查找chart

`helm search hub`

`helm search hub wordpress`

`helm search repo`

### 安装helm包

`helm install release_name package_name`

helm安装资源顺序

- Namespace
- NetworkPolicy
- ResourceQuota
- LimitRange
- PodSecurityPolicy
- PodDisruptionBudget
- ServiceAccount
- Secret
- SecretList
- ConfigMap
- StorageClass
- PersistentVolume
- PersistentVolumeClaim
- CustomResourceDefinition
- ClusterRole
- ClusterRoleList
- ClusterRoleBinding
- ClusterRoleBindingList
- Role
- RoleList
- RoleBinding
- RoleBindingList
- Service
- DaemonSet
- Pod
- ReplicationController
- ReplicaSet
- Deployment
- HorizontalPodAutoscaler
- StatefulSet
- Job
- CronJob
- Ingress
- APIService

`helm status release_name` 来追踪 release 的状态



#### 安装前自定义chart

查看 chart 中的可配置选项

`helm show values package_name`

YAML 格式的文件覆盖

```shell
echo '{mariadb.auth.database: user0db, mariadb.auth.username: user0}' > values.yaml
helm install -f values.yaml bitnami/wordpress --generate-name
```

安装过程中有两种方式传递配置数据

- `--values` (或 `-f`)：使用 YAML 文件覆盖配置
- `--set`：通过命令行的方式对指定项进行覆盖

### 升级 release 和失败时恢复

`helm upgrade -f panda.yaml happy-panda bitnami/wordpress`

查看配置值是否生效

`helm get values happy-panda`

回滚

`helm rollback release_name revision`

`helm history [RELEASE]`



### 卸载release

`helm uninstall release_name`

`helm list`

`helm uninstall --keep-history`



### 使用仓库

```shell
helm repo list
helm repo add repo_name url
helm repo update
heml repo remove
```



### 创建chart

```shell
helm create your-chart-name
# 验证格式
helm lint
# 打包分发
helm package chart-name
```



## chart 开发

### 模版功能

[参考](https://helm.sh/zh/docs/howto/charts_tips_and_tricks/)

