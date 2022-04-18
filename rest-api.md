## k8s访问API Server

### Token

#### 拿根证书

```shell
kubectl get secret \
    $(kubectl get secrets | grep default-token | awk '{print $1}') \
    -o jsonpath="{['data']['ca\.crt']}" | base64 --decode
```

```shell
curl --cacert ca.crt  https://k8s-api:6443
```

### 创建账号绑定admin并获取token

```shell
# 创建sa
kubectl create serviceaccount star-sky-sa -n kube-system

# 绑定admin角色
kubectl create clusterrolebinding star-sky-sa-binding --clusterrole=cluster-admin --serviceaccount=kube-system:star-sky-sa -n kube-system

# 获取access token
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep star-sky-sa | awk '{print $1}')
```

```shell
curl --cacert ca.crt -H "Authorization: Bearer $TOKEN"  https://k8s-api:6443
```



### 证书

```shell
cat /root/.kube/config
base64 -d
curl --cert ./client.pem --key ./client-key.pem --cacert ./ca.pem https://172.21.0.15:6443/api/v1/pods
```

