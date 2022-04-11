## Namespace

### 查看namespace

```shell
kubectl get ns
kubectl get namespaces <name>
kubectl describe namespaces <name>
```

初始状态下，Kubernetes 具有三个名字空间

- `default` 无名字空间对象的默认名字空间
- `kube-system` 由 Kubernetes 系统创建的对象的名字空间
- `kube-public` 自动创建且被所有用户可读的名字空间



### 创建namespace

- yaml方式

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: <insert-namespace-name-here>
```

```shell
kubectl create -f ./namespace.yaml
```

- 命令行方式

```shell
kubectl create namespace <insert-namespace-name-here>
```



### 删除namespace

```shell
kubectl delete namespaces <insert-some-namespace-name>
```

**警告：** 这会删除名字空间下的 *所有内容* ！



### 其他

```shell
# 位于名字空间中的资源
kubectl api-resources --namespaced=true

# 不在名字空间中的资源
kubectl api-resources --namespaced=false
```

