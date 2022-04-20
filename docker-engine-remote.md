## docker 远程访问

### 开启远程连接端口

- Unix Socket 这是类unix系统进程间通讯的一种方式，Client操作本机的Engine使用此方式
- Systemd socket activation，systemd提供的一种为了服务并行启动设计的socket，缺省值为fd://，只能连本地engine
- TCP，连接远程Engine，必须在服务端开始TCP连接。此连接为不安全连接，数据通过明文进行传输。缺省端口2375
- TCP_TLS : 在TCP的基础之上加上了SSL的安全证书，以保证连接安全。缺省端口2376



### 不加密TCP

#### 查看docker.service文件路径

```shell
sudo systemctl status docker|grep Loaded|grep -Po '(?<=Loaded: loaded \()[^;]*'
```

#### `dockerd`命令对应的配置文件

/etc/docker/daemon.json

#### docker.service中查看engine启动命令

```shell
sudo cat $(systemctl status docker|grep Loaded|grep -Po '(?<=Loaded: loaded \()[^;]*')|grep
```

**可以看到这个参数-H fd://，意思是启用socket activation作为客户端接口**

- 备份docker.service

```shell
SERVICE_FILE=$(systemctl status docker|grep Loaded|grep -Po '(?<=Loaded: loaded \()[^;]*') \
  && sudo cp ${SERVICE_FILE} ${SERVICE_FILE}.bak
```

- 删除dockerd的 -H参数

```shell
SERVICE_FILE=$(systemctl status docker|grep Loaded|grep -Po '(?<=Loaded: loaded \()[^;]*') \
  && sudo sed -i -e 's/ -H fd:\/\/ / /g' ${SERVICE_FILE}
```

- check

```shell
sudo cat $(systemctl status docker|grep Loaded|grep -Po '(?<=Loaded: loaded \()[^;]*')|grep dockerd
```



#### 修改daemon.json

添加以下信息

```json
"hosts":[
    "fd://",
    "tcp://0.0.0.0:2375"
  ]
```



#### 重启docker服务

```shell
sudo systemctl daemon-reload
sudo systemctl restart docker
sudo systemctl status docker
ss -l |grep -Po '\s[^\s]*2375\s'
```



#### 客户端连接

```shell
docker -H tcp://<服务器IP>:2375 version

# 防洪墙2375端口放行
```



### 安全TCP

#### 创建CA和证书

[参考](https://docs.docker.com/engine/security/protect-access/)

#### 修改daemon.json文件

```shell
{
  "hosts":[
    "fd://",
    "tcp://0.0.0.0:2376"
  ],
  "tlsverify":true,
  "tlscacert":"/etc/docker/tls/ca.pem",
  "tlscert":"/etc/docker/tls/server-cert.pem",
  "tlskey":"/etc/docker/tls/server-key.pem"
}
```

#### 重启docker

```shell
sudo systemctl restart docker
```

#### 客户端配置

```shell
mkdir -p ~/.ssh/tls
scp <server-name-in-ssh-config>:~/.ssh/tls/tester-cert.tar.gz ~/.ssh/tls
mkdir -p ~/.ssh/tls/docker
cd  ~/.ssh/tls/docker
tar -xzvf ~/.ssh/tls/tester-cert.tar.gz
docker -H tcp://<服务器域名>:2376 \
  --tlsverify=1 \
  --tlscacert=${HOME}/.ssh/tls/docker/tester/ca.pem \
  --tlscert=${HOME}/.ssh/tls/docker/tester/cert.pem \
  --tlskey=${HOME}/.ssh/tls/docker/tester/key.pem \
  version
```

