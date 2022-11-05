# redis-on-k8s

## 一、概述

> REmote DIctionary Server(`Redis`) 是一个由 Salvatore Sanfilippo 写的 key-value 存储系统，是跨平台的**非关系型数据库**。

Redis有三种集群模式：主从模式，Sentinel（哨兵）模式，Cluster模式，这三种模式环境编排部署都会在本文章介绍与实战操作。

## 二、redis 主从模式编排部署实战操作

![在这里插入图片描述](https://img-blog.csdnimg.cn/c356b8437cd44cd0adebd74b6fc05c13.png)
地址：[https://artifacthub.io/packages/helm/bitnami/redis](https://artifacthub.io/packages/helm/bitnami/redis)

### 1）下载chart 包
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami

helm pull bitnami/redis --version 17.3.7

tar -xf redis-17.3.7.tgz
```
### 2）构建镜像
这里就不重新构建镜像了，只是把远程镜像tag一下，推到本地harbor仓库加速下载镜像。有不清楚怎么构建镜像的小伙伴，可以私信或者留言。

```bash
docker pull docker.io/bitnami/redis:7.0.5-debian-11-r7

# tag
docker tag docker.io/bitnami/redis:7.0.5-debian-11-r7 myharbor.com/bigdata/redis:7.0.5-debian-11-r7

# 推送镜像到本地harbor仓库
docker push myharbor.com/bigdata/redis:7.0.5-debian-11-r7
```
### 3）修改yaml编排
- `redis/templates/master/pv.yaml`

新增`pv.yaml`文件，内容如下：
```yaml
{{- range .Values.master.persistence.local }}
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: {{ .name }}
  labels:
    name: {{ .name }}
spec:
  storageClassName: {{ $.Values.master.persistence.storageClass }}
  capacity:
    storage: {{ $.Values.master.persistence.size }}
  accessModes:
    - ReadWriteOnce
  local:
    path: {{ .path }}
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - {{ .host }}
---
{{- end }}
```

- `redis/templates/replicas/pv.yaml`

新增`pv.yaml`文件，内容如下：

```yaml
{{- range .Values.replica.persistence.local }}
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: {{ .name }}
  labels:
    name: {{ .name }}
spec:
  storageClassName: {{ $.Values.replica.persistence.storageClass }}
  capacity:
    storage: {{ $.Values.replica.persistence.size }}
  accessModes:
    - ReadWriteOnce
  local:
    path: {{ .path }}
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - {{ .host }}
---
{{- end }}
```

- `redis/values.yaml`

```bash
global:
  redis:
    password: "123456"

...

image:
  registry: myharbor.com
  repository: bigdata/redis
  tag: 7.0.5-debian-11-r7

master:
  count: 1
  persistence:
    enabled: true
    size: 8Gi
    storageClass: "local-redis-storage"
    local:
    - name: redis-0
      host: "local-168-182-110"
      path: "/opt/bigdata/servers/redis/data/data1"

replica:
  replicaCount: 2
  persistence:
    enabled: true
    size: 8Gi
    storageClass: "local-redis-storage"
    local:
    - name: redis-1
      host: "local-168-182-111"
      path: "/opt/bigdata/servers/redis/data/data1"
    - name: redis-2
      host: "local-168-182-112"
      path: "/opt/bigdata/servers/redis/data/data1"
```

### 4）开始部署

```bash
# 创建存储目录
mkdir /opt/bigdata/servers/redis/data/data1

# 先检查语法
helm lint ./redis

# 开始安装
helm install redis ./redis -n redis --create-namespace
```
NOTES

```bash
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: redis
CHART VERSION: 17.3.7
APP VERSION: 7.0.5

** Please be patient while the chart is being deployed **

Redis&reg; can be accessed on the following DNS names from within your cluster:

    redis-master.redis.svc.cluster.local for read/write operations (port 6379)
    redis-replicas.redis.svc.cluster.local for read-only operations (port 6379)



To get your password run:

    export REDIS_PASSWORD=$(kubectl get secret --namespace redis redis -o jsonpath="{.data.redis-password}" | base64 -d)

To connect to your Redis&reg; server:

1. Run a Redis&reg; pod that you can use as a client:

   kubectl run --namespace redis redis-client --restart='Never'  --env REDIS_PASSWORD=$REDIS_PASSWORD  --image myharbor.com/bigdata/redis:7.0.5-debian-11-r7 --command -- sleep infinity

   Use the following command to attach to the pod:

   kubectl exec --tty -i redis-client \
   --namespace redis -- bash

2. Connect using the Redis&reg; CLI:
   REDISCLI_AUTH="$REDIS_PASSWORD" redis-cli -h redis-master
   REDISCLI_AUTH="$REDIS_PASSWORD" redis-cli -h redis-replicas

To connect to your database from outside the cluster execute the following commands:

    kubectl port-forward --namespace redis svc/redis-master 6379:6379 &
    REDISCLI_AUTH="$REDIS_PASSWORD" redis-cli -h 127.0.0.1 -p 6379

```
![在这里插入图片描述](https://img-blog.csdnimg.cn/b806290fcbe447e788f9927b1e4f39c8.png)
### 5）测试验证
```bash
kubectl get pods,svc -n redis -owide
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/e2b031dffa36437ab64012e8b877f28f.png)

```bash
# 登录master，可读可写
kubectl exec -it redis-master-0 -n redis -- redis-cli -h redis-master -a $(kubectl get secret --namespace redis redis -o jsonpath="{.data.redis-password}" | base64 -d)

# 登录slave，只读
kubectl exec -it redis-master-0 -n redis -- redis-cli -h redis-replicas -a $(kubectl get secret --namespace redis redis -o jsonpath="{.data.redis-password}" | base64 -d)
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/87e8b5e71fd3442b9046920d71187251.png)
### 6）卸载
```bash
helm uninstall redis-sentinel -n redis-sentinel
# delete ns 
kubectl delete ns redis-sentinel --force
# delete pv
kubectl delete pv `kubectl get pv|grep ^redis-|awk '{print $1}'` --force

rm -fr /opt/bigdata/servers/redis/data/data1/*
```
## 三、redis 哨兵模式编排部署实战操作
> 主从模式的弊端就是不具备高可用性，当master挂掉以后，Redis将不能再对外提供写入操作，因此sentinel应运而生。

![在这里插入图片描述](https://img-blog.csdnimg.cn/4e3c7360cc4543278f1d6c9990eb2ea7.png)

### 1）构建镜像
这里也重新构建镜像了，有不懂构建镜像的小伙伴可以在评论下方留言。这里也只是把远程的镜像推送到本地harbor。
```bash
docker pull docker.io/bitnami/redis-sentinel:7.0.5-debian-11-r6
# tag
docker tag docker.io/bitnami/redis-sentinel:7.0.5-debian-11-r6 myharbor.com/bigdata/redis-sentinel:7.0.5-debian-11-r6
# push
docker push  myharbor.com/bigdata/redis-sentinel:7.0.5-debian-11-r6
```

### 2）修改yaml编排

- `redis-sentinel/values.yaml`

```bash
replica:
  # replica.replicaCount与sentinel.quorum值一样
  replicaCount: 3
  storageClass: "local-redis-storage"
    local:
    - name: redis-0
      host: "local-168-182-110"
      path: "/opt/bigdata/servers/redis/data/data1"
    - name: redis-1
      host: "local-168-182-111"
      path: "/opt/bigdata/servers/redis/data/data1"
    - name: redis-2
      host: "local-168-182-112"
      path: "/opt/bigdata/servers/redis/data/data1"

sentinel:
  enabled: true
  image:
    registry: myharbor.com
    repository: bigdata/redis-sentinel
    tag: 7.0.5-debian-11-r6
  quorum: 3
```
- `redis-sentinel/templates/replicas/pv.yaml`

新增`pv.yaml`文件，内容如下：

```yaml
{{- range .Values.sentinel.persistence.local }}
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: {{ .name }}
  labels:
    name: {{ .name }}
spec:
  storageClassName: {{ $.Values.sentinel.persistence.storageClass }}
  capacity:
    storage: {{ $.Values.sentinel.persistence.size }}
  accessModes:
    - ReadWriteOnce
  local:
    path: {{ .path }}
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - {{ .host }}
---
{{- end }}
```
### 3）开始部署
```bash
# 创建存储目录
mkdir -p /opt/bigdata/servers/redis/data/data1

helm install redis-sentinel ./redis-sentinel -n redis-sentinel --create-namespace 
```
NOTES

```bash
NAME: redis-sentinel
LAST DEPLOYED: Fri Nov  4 22:42:52 2022
NAMESPACE: redis-sentinel
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: redis
CHART VERSION: 17.3.7
APP VERSION: 7.0.5

** Please be patient while the chart is being deployed **

Redis&reg; can be accessed via port 6379 on the following DNS name from within your cluster:

    redis-sentinel.redis-sentinel.svc.cluster.local for read only operations

For read/write operations, first access the Redis&reg; Sentinel cluster, which is available in port 26379 using the same domain name above.



To get your password run:

    export REDIS_PASSWORD=$(kubectl get secret --namespace redis-sentinel redis-sentinel -o jsonpath="{.data.redis-password}" | base64 -d)

To connect to your Redis&reg; server:

1. Run a Redis&reg; pod that you can use as a client:

   kubectl run --namespace redis-sentinel redis-client --restart='Never'  --env REDIS_PASSWORD=$REDIS_PASSWORD  --image myharbor.com/bigdata/redis:7.0.5-debian-11-r7 --command -- sleep infinity

   Use the following command to attach to the pod:

   kubectl exec --tty -i redis-client \
   --namespace redis-sentinel -- bash

2. Connect using the Redis&reg; CLI:
   REDISCLI_AUTH="$REDIS_PASSWORD" redis-cli -h redis-sentinel -p 6379 # Read only operations
   REDISCLI_AUTH="$REDIS_PASSWORD" redis-cli -h redis-sentinel -p 26379 # Sentinel access

To connect to your database from outside the cluster execute the following commands:

    kubectl port-forward --namespace redis-sentinel svc/redis-sentinel 6379:6379 &
    REDISCLI_AUTH="$REDIS_PASSWORD" redis-cli -h 127.0.0.1 -p 6379

```
![在这里插入图片描述](https://img-blog.csdnimg.cn/e656571dd7534a079cc588e5dc739a1a.png)
查看
```bash
kubectl get pods,svc -n redis-sentinel -owide
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/efb783d90ce14aa99019a1b0feaf795c.png)

### 4）模拟故障测试
```bash
# 查看
kubectl exec -it redis-sentinel-node-0 -n redis-sentinel -- redis-cli -h redis-sentinel -a $(kubectl get secret --namespace redis-sentinel redis-sentinel -o jsonpath="{.data.redis-password}" | base64 -d) info replication
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/d4f338c04c364a1aafae3deb56f6947a.png)
模拟故障，kill  master pod

```bash
kubectl delete pod redis-sentinel-node-0 -n redis-sentinel
```
再次查看master所在节点，master节点已经切换到其它节点了。
![在这里插入图片描述](https://img-blog.csdnimg.cn/8b57076684554867b6d7033fc8c7ac94.png)
再测试读写

```bash
# 登录master节点
kubectl exec -it redis-sentinel-node-0 -n redis-sentinel -- redis-cli -h redis-sentinel-node-2.redis-sentinel-headless -a $(kubectl get secret --namespace redis-sentinel redis-sentinel -o jsonpath="{.data.redis-password}" | base64 -d)

# 登录slave节点
kubectl exec -it redis-sentinel-node-0 -n redis-sentinel -- redis-cli -h redis-sentinel-node-0.redis-sentinel-headless -a $(kubectl get secret --namespace redis-sentinel redis-sentinel -o jsonpath="{.data.redis-password}" | base64 -d)
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/fad036135e8c4e20b082e42b82fd6bc8.png)
### 5）卸载
```bash
helm uninstall redis-sentinel -n redis
# delete ns 
kubectl delete ns redis --force
# delete pv
kubectl delete pv `kubectl get pv|grep ^redis-|awk '{print $1}'` --force

rm -fr /opt/bigdata/servers/redis/data/data1/*
```
## 四、redis 集群模式编排部署实战操作
> **集群模式**可以说是sentinel+主从模式的结合体，通过cluster可以实现主从和master重选功能，所以如果配置两个副本三个分片的话，就需要六个Redis实例。因为Redis的数据是根据一定规则分配到cluster的不同机器的，当数据量过大时，可以新增机器进行扩容。

![在这里插入图片描述](https://img-blog.csdnimg.cn/ca22c95ea4eb47fba35c2c17f5395135.png)
### 1）下载chart 包
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami

helm pull bitnami/redis-cluster --version 8.2.7

tar -xf redis-cluster-8.2.7.tgz
```
### 2）构建镜像
这里就不重新构建镜像了，只是把远程镜像tag一下，推到本地harbor仓库加速下载镜像。有不清楚怎么构建镜像的小伙伴，可以私信或者留言。

```bash
docker pull docker.io/bitnami/redis-cluster:7.0.5-debian-11-r9

# tag
docker tag docker.io/bitnami/redis-cluster:7.0.5-debian-11-r9 myharbor.com/bigdata/redis-cluster:7.0.5-debian-11-r9

# 推送镜像到本地harbor仓库
docker push myharbor.com/bigdata/redis-cluster:7.0.5-debian-11-r9
```
### 3）修改yaml编排
- `redis-cluster/templates/pv.yaml`

新增`pv.yaml`文件，内容如下：
```yaml
{{- range .Values.persistence.local }}
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: {{ .name }}
  labels:
    name: {{ .name }}
spec:
  storageClassName: {{ $.Values.persistence.storageClass }}
  capacity:
    storage: {{ $.Values.persistence.size }}
  accessModes:
    - ReadWriteOnce
  local:
    path: {{ .path }}
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - {{ .host }}
---
{{- end }}
```

```bash
password: "123456"

...

image:
  registry: myharbor.com
  repository: bigdata/redis-cluster
  tag: 7.0.5-debian-11-r9

...

persistence:
  storageClass: "local-redis-cluster-storage"
  local:
    - name: redis-cluster-0
      host: "local-168-182-110"
      path: "/opt/bigdata/servers/redis-cluster/data/data1"
    - name: redis-cluster-1
      host: "local-168-182-110"
      path: "/opt/bigdata/servers/redis-cluster/data/data2"
    - name: redis-cluster-2
      host: "local-168-182-110"
      path: "/opt/bigdata/servers/redis-cluster/data/data3"
    - name: redis-cluster-3
      host: "local-168-182-111"
      path: "/opt/bigdata/servers/redis-cluster/data/data1"
    - name: redis-cluster-4
      host: "local-168-182-111"
      path: "/opt/bigdata/servers/redis-cluster/data/data2"
    - name: redis-cluster-5
      host: "local-168-182-111"
      path: "/opt/bigdata/servers/redis-cluster/data/data3"
    - name: redis-cluster-6
      host: "local-168-182-112"
      path: "/opt/bigdata/servers/redis-cluster/data/data1"
    - name: redis-cluster-7
      host: "local-168-182-112"
      path: "/opt/bigdata/servers/redis-cluster/data/data2"
    - name: redis-cluster-8
      host: "local-168-182-112"
      path: "/opt/bigdata/servers/redis-cluster/data/data3"
  
cluster:
  init: true
  # 一主两从（三组）
  nodes: 9
  replicas: 2

```

### 4）开始部署

```bash
# 创建存储目录
mkdir -p /opt/bigdata/servers/redis-cluster/data/data{1..3}

helm install redis-cluster ./redis-cluster -n redis-cluster --create-namespace
```
NOTES

```bash
NOTES:
CHART NAME: redis-cluster
CHART VERSION: 8.2.7
APP VERSION: 7.0.5** Please be patient while the chart is being deployed **


To get your password run:
    export REDIS_PASSWORD=$(kubectl get secret --namespace "redis-cluster" redis-cluster -o jsonpath="{.data.redis-password}" | base64 -d)

You have deployed a Redis&reg; Cluster accessible only from within you Kubernetes Cluster.INFO: The Job to create the cluster will be created.To connect to your Redis&reg; cluster:

1. Run a Redis&reg; pod that you can use as a client:
kubectl run --namespace redis-cluster redis-cluster-client --rm --tty -i --restart='Never' \
 --env REDIS_PASSWORD=$REDIS_PASSWORD \
--image myharbor.com/bigdata/redis-cluster:7.0.5-debian-11-r9 -- bash

2. Connect using the Redis&reg; CLI:

redis-cli -c -h redis-cluster -a $REDIS_PASSWORD
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/d832eb1f51b24de2ac0331cce1e69c85.png)
查看

```bash
kubectl get pods,svc -n redis-cluster -owide
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/63e1eb714a674dea997ffb56a54d64c5.png)

### 5）故障模拟测试
```bash
kubectl exec -it redis-cluster-0 -n redis-cluster -- redis-cli -c -h redis-cluster -a $(kubectl get secret --namespace "redis-cluster" redis-cluster -o jsonpath="{.data.redis-password}" | base64 -d) CLUSTER INFO

kubectl exec -it redis-cluster-0 -n redis-cluster -- redis-cli -c -h redis-cluster -a $(kubectl get secret --namespace "redis-cluster" redis-cluster -o jsonpath="{.data.redis-password}" | base64 -d) CLUSTER NODES
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/386d4c41181a4265b3dcabb05bcc3971.png)
删除其中一个master节点

```bash
kubectl delete pod redis-cluster-1 -n redis-cluster

# 再查看节点情况
kubectl exec -it redis-cluster-0 -n redis-cluster -- redis-cli -c -h redis-cluster -a $(kubectl get secret --namespace "redis-cluster" redis-cluster -o jsonpath="{.data.redis-password}" | base64 -d) CLUSTER NODES
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/031482c7fbba4a3784870fdd73ae9586.png)
### 6）卸载

```bash
helm uninstall redis-cluster -n redis-cluster
# delete ns 
kubectl delete ns redis-cluster --force
# delete pv
kubectl delete pv `kubectl get pv|grep ^redis-cluster-|awk '{print $1}'` --force

rm -fr /opt/bigdata/servers/redis-cluster/data/data{1..3}/*
```

git地址：[https://gitee.com/hadoop-bigdata/redis-on-k8s](https://gitee.com/hadoop-bigdata/redis-on-k8s)

Redis on k8s 三种模式的编排部署就先到这里了，小伙伴有任何疑问，欢迎给我留言哦，后续会持续更新【大数据+云原生】相关的问题~