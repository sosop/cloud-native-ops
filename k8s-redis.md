## K8S部署单节点redis

### configmap存储redis配置

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: redis-config
  namespace: star-sky
  labels:
    app: redis
data:
  redis.conf: |-
    dir /data
    port 6379
    bind 0.0.0.0
    appendonly yes
    protected-mode no
    requirepass star-sky@2022
    pidfile /data/redis-6379.pid
```

### 持久化

#### storageclass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client-common
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner-common # or choose another name, must match deployment's env PROVISIONER_NAME'
parameters:
  archiveOnDelete: "false"
  #pathPattern: "${.PVC.namespace}/${.PVC.annotations.nfs.io/storage-path}" 
  pathPattern: "${.PVC.namespace}/data"
```

#### NFS Provisioner

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: star-sky
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: k8s.gcr.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: k8s-sigs.io/nfs-subdir-external-provisioner-common
            - name: NFS_SERVER
              value: 10.3.243.101
            - name: NFS_PATH
              value: /ifs/kubernetes
      volumes:
        - name: nfs-client-root
          nfs:
            server: 10.3.243.101
            path: /ifs/kubernetes
```

#### PVC

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: redis-pvc
  namespace: star-sky
spec:
  storageClassName: nfs-client-common
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 14Gi
```

### redis deployment and service

#### deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: star-sky
  labels:
    app: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      # 进行初始化操作，修改系统配置，解决 Redis 启动时提示的警告信息
      initContainers:
        - name: system-init
          image: busybox:1.32
          imagePullPolicy: IfNotPresent
          command:
            - "sh"
            - "-c"
            - "echo 2048 > /proc/sys/net/core/somaxconn && echo never > /sys/kernel/mm/transparent_hugepage/enabled"
          securityContext:
            privileged: true
            runAsUser: 0
          volumeMounts:
          - name: sys
            mountPath: /sys
      containers:
        - name: redis
          image: docker.io/library/redis:6.2.6-alpine
          command:
            - "sh"
            - "-c"
            - "redis-server /usr/local/etc/redis/redis.conf"
          ports:
            - containerPort: 6379
          resources:
            limits:
              cpu: 2000m
              memory: 2Gi
            requests:
              cpu: 1000m
              memory: 512Mi
          livenessProbe:
            tcpSocket:
              port: 6379
            initialDelaySeconds: 300
            timeoutSeconds: 1
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 3
          readinessProbe:
            tcpSocket:
              port: 6379
            initialDelaySeconds: 5
            timeoutSeconds: 1
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 3
          volumeMounts:
            - name: data
              mountPath: /data
            - name: config
              mountPath: /usr/local/etc/redis/redis.conf
              subPath: redis.conf
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: redis-pvc
        - name: config
          configMap:
            name: redis-config
        - name: sys
          hostPath:
            path: /sys
```

#### service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  labels:
    app: redis
  namespace: star-sky
spec:
  type: ClusterIP
  ports:
    - name: redis
      protocol: TCP
      port: 6379
  selector:
    app: redis
```

