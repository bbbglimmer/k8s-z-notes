# 基于 KubeSphere 玩转 k8s-Redis 安装手记

**大家好，我是老 Z！**

> 本系列文档是我在云原生技术领域的学习和运维实践的手记，**用输出倒逼输入**是一种高效的学习方法，能够快速积累经验和提高技术，只有把学到的知识写出来并能够让其他人理解，才能说明真正掌握了这项知识。
>
> 如果你喜欢本文，请分享给你的小伙伴！

**本系列文档内容涵盖 (但不限于) 以下技术领域：**

> - **KubeSphere**
>
> - **Kubernetes**
>
> - **Ansible**
>
> - **自动化运维**
>
> - **CNCF 技术栈**

## 1. 本文简介

Redis 单节点如何在 K8S 集群上部署？Redis 集群模式如何在 K8S 集群上部署？是否有能在 K8S 集群上部署的 Web 图形化的 Redis 管理工具？GitOps 在 K8S 集群上是咋玩的？本文将带你解决上面的问题。

> 本文知识量

- 阅读时长：10 分
- 行：810
- 单词：3200+
- 字符：21700+
- 图片：9 张

> **本文知识点**

- 定级：**入门级**
- Redis 单节点安装部署
- Redis 集群安装部署
- RedisInsight 安装部署
- GitOps 运维思想

> **演示服务器配置**

|      主机名      |      IP      | CPU  | 内存 | 系统盘 | 数据盘 |               用途               |
| :--------------: | :----------: | :--: | :--: | :----: | :----: | :------------------------------: |
|  zdeops-master   | 192.168.9.9  |  2   |  4   |   40   |  200   |       Ansible 运维控制节点        |
| ks-k8s-master-0  | 192.168.9.91 |  8   |  32  |   40   |  200   | KubeSphere/k8s-master/k8s-worker |
| ks-k8s-master-1  | 192.168.9.92 |  8   |  32  |   40   |  200   | KubeSphere/k8s-master/k8s-worker |
| ks-k8s-master-2  | 192.168.9.93 |  8   |  32  |   40   |  200   | KubeSphere/k8s-master/k8s-worker |
| glusterfs-node-0 | 192.168.9.95 |  4   |  8   |   40   |  200   |            GlusterFS             |
| glusterfs-node-1 | 192.168.9.96 |  4   |  8   |   40   |  200   |            GlusterFS             |
| glusterfs-node-2 | 192.168.9.97 |  4   |  8   |   40   |  200   |            GlusterFS             |
|   ceph-node-0    | 192.168.9.85 |  4   |  8   |   40   |  200   |               Ceph               |
|   ceph-node-1    | 192.168.9.86 |  4   |  8   |   40   |  200   |               Ceph               |
|   ceph-node-2    | 192.168.9.87 |  4   |  8   |   40   |  200   |               Ceph               |
|      harbor      | 192.168.9.89 |  4   |  8   |   40   |  200   |              Harbor              |

> **演示环境涉及软件版本信息**

- 操作系统：**CentOS-7.9-x86_64**
- Redis： **6.2.7**
- RedisInsight：**1.12.0**

## 2. 单节点 Redis 部署

### 2.1. 思路梳理

- StatefulSet
- Headless Service
- ConfigMap：redis.conf

### 2.2. 准备离线镜像

此过程为可选项，离线内网环境可用。

在一台能同时访问互联网和内网 Harbor 仓库的服务器上进行下面的操作。

- 下载镜像

```shell
docker pull redis:6.2.7
```

- 重新打 tag

```shell
docker tag redis:6.2.7 registry.zdevops.com.cn/library/redis:6.2.7
```

- 推送到私有镜像仓库

```shell
docker push registry.zdevops.com.cn/library/redis:6.2.7
```

### 2.3. 资源配置清单

- **redis-cm.yaml**

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
  namespace: zdevops
data:
  redis-config: |
    appendonly yes
    protected-mode no
    dir /data
    port 6379
    requirepass redis@abc.com

```

- **redis-sts.yaml**

```yaml
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: zdevops
  labels:
    app: redis
spec:
  serviceName: redis-headless
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: 'registry.zdevops.com.cn/library/redis:6.2.7'
          command:
            - "redis-server"
          args:
            - "/etc/redis/redis.conf"
          ports:
            - name: redis-6379
              containerPort: 6379
              protocol: TCP
          volumeMounts:
            - name: config
              mountPath: /etc/redis
            - name: data
              mountPath: /data
          resources:
            limits:
              cpu: '2'
              memory: 4000Mi
            requests:
              cpu: 100m
              memory: 500Mi
      volumes:
        - name: config
          configMap:
            name: redis-config
            items:
              - key: redis-config
                path: redis.conf
  volumeClaimTemplates:
    - metadata:
        name: data
        namespace: zdevops
      spec:
        accessModes: [ "ReadWriteOnce" ]
        storageClassName: "glusterfs"
        resources:
          requests:
            storage: 5Gi

---
apiVersion: v1
kind: Service
metadata:
  name: redis-headless
  namespace: zdevops
  labels:
    app: redis
spec:
  ports:
    - name: redis-6379
      protocol: TCP
      port: 6379
      targetPort: 6379
  selector:
    app: redis
  clusterIP: None
  type: ClusterIP

```

### 2.4. GitOps

**在运维开发服务器上操作**

```shell
# 在已有代码仓库创建 redis/single 目录
[root@zdevops-master k8s-yaml]# mkdir -p redis/single

# 编辑资源配置清单
[root@zdevops-master k8s-yaml]# vi redis/single/redis-cm.yaml
[root@zdevops-master k8s-yaml]# vi redis/single/redis-sts.yaml

# 提交 Git
[root@zdevops-master k8s-yaml]# git add redis
[root@zdevops-master k8s-yaml]# git commit -am '添加redis 单节点资源配置清单'
[root@zdevops-master k8s-yaml]# git push
```

### 2.5. 部署资源

**在运维管理服务器上操作**

- 更新镜像仓库代码

```shell
[root@zdevops-master k8s-yaml]# git pull
```

- 部署资源

```shell
[root@zdevops-master k8s-yaml]# kubectl apply -f redis/single/
```

### 2.6. 验证

```shell
[root@zdevops-master k8s-yaml]# kubectl get sts -o wide -n zdevops

[root@zdevops-master k8s-yaml]# kubectl get pods -o wide -n zdevops
```

## 3. 集群模式 Redis 部署

### 3.1. 思路梳理

- StatefulSet
- Headless Service
- ConfigMap：redis.conf
- 最少 6 个节点

### 3.2. 准备离线镜像

过程略，参考 2.2

### 3.3. 资源配置清单

- **redis-cm.yaml**

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-cluster-config
  namespace: zdevops
data:
  redis-config: |
    appendonly yes
    protected-mode no
    dir /data
    port 6379
    cluster-enabled yes
    cluster-config-file /data/nodes.conf
    cluster-node-timeout 5000
    masterauth redis@abc.com
    requirepass redis@abc.com

```

- **redis-sts.yaml**

```yaml
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: zdevops
  labels:
    app: redis
spec:
  serviceName: redis-headless
  replicas: 6
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - redis
              topologyKey: kubernetes.io/hostname
      containers:
        - name: redis
          image: 'registry.zdevops.com.cn/library/redis:6.2.7'
          command:
            - "redis-server"
          args:
            - "/etc/redis/redis.conf"
            - "--protected-mode"
            - "no"
            - "--cluster-announce-ip"
            - "$(POD_IP)"
          env:
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
          ports:
            - name: redis-6379
              containerPort: 6379
              protocol: TCP
          volumeMounts:
            - name: config
              mountPath: /etc/redis
            - name: data
              mountPath: /data
          resources:
            limits:
              cpu: '2'
              memory: 4000Mi
            requests:
              cpu: 100m
              memory: 500Mi
      volumes:
        - name: config
          configMap:
            name: redis-cluster-config
            items:
              - key: redis-config
                path: redis.conf
  volumeClaimTemplates:
    - metadata:
        name: data
        namespace: zdevops
      spec:
        accessModes: [ "ReadWriteOnce" ]
        storageClassName: "glusterfs"
        resources:
          requests:
            storage: 5Gi

---
apiVersion: v1
kind: Service
metadata:
  name: redis-headless
  namespace: zdevops
  labels:
    app: redis
spec:
  ports:
    - name: redis-6379
      protocol: TCP
      port: 6379
      targetPort: 6379
  selector:
    app: redis
  clusterIP: None
  type: ClusterIP

```

注意 **POD_IP** 的相关配置，如果不配置会导致线上的 POD 重启换 IP 后，集群状态无法自动同步。

### 3.4. GitOps

**在运维开发服务器上操作**

```shell
# 在已有代码仓库创建 redis/cluster 目录
[root@zdevops-master k8s-yaml]# mkdir -p redis/cluster

# 编辑资源配置清单
[root@zdevops-master k8s-yaml]# vi redis/cluster/redis-cluster-cm.yaml
[root@zdevops-master k8s-yaml]# vi redis/cluster/redis-cluster-sts.yaml

# 提交 Git
[root@zdevops-master k8s-yaml]# git add redis/cluster
[root@zdevops-master k8s-yaml]# git commit -am '添加 redis 集群模式部署资源配置清单'
[root@zdevops-master k8s-yaml]# git push
```

### 3.5. 部署资源

**在运维管理服务器上操作**

- `更新镜像仓库代码

```shell
[root@zdevops-master k8s-yaml]# git pull
```

- 部署资源

```shell
[root@zdevops-master k8s-yaml]# kubectl apply -f redis/cluster/
```

### 3.6. 确认资源状态

```shell
[root@zdevops-master k8s-yaml]# kubectl get sts -o wide -n zdevops
NAME    READY   AGE    CONTAINERS   IMAGES
redis   6/6     150m   redis        registry.zdevops.com.cn/library/redis:6.2.7

[root@zdevops-master k8s-yaml]# kubectl get pods -o wide -n zdevops
NAME      READY   STATUS    RESTARTS   AGE    IP              NODE              NOMINATED NODE   READINESS GATES
redis-0   1/1     Running   0          150m   10.233.116.48   ks-k8s-master-2   <none>           <none>
redis-1   1/1     Running   0          150m   10.233.117.85   ks-k8s-master-0   <none>           <none>
redis-2   1/1     Running   0          147m   10.233.87.244   ks-k8s-master-1   <none>           <none>
redis-3   1/1     Running   0          147m   10.233.116.13   ks-k8s-master-2   <none>           <none>
redis-4   1/1     Running   0          146m   10.233.117.93   ks-k8s-master-0   <none>           <none>
redis-5   1/1     Running   0          146m   10.233.87.249   ks-k8s-master-1   <none>           <none>
```

### 3.7. 自动创建 Redis 集群

POD 创建完成后默认不会自动创建 Redis 集群，需要手工执行集群初始化的命令，有自动创建和手工创建两种方式，二选一，建议选择**自动**。

自动配置 3 个 master 3 个 slave 的集群，执行下面的命令，中间需要输入一次 yes。

- 执行命令

```shell
[root@zdevops-master k8s-yaml]# kubectl exec -it redis-0 -n zdevops -- redis-cli -a redis@abc.com --cluster create --cluster-replicas 1 $(kubectl get pods -n zdevops -l app=redis -o jsonpath='{range.items[*]}{.status.podIP}:6379 {end}')
```

- 正常结果

```shell
[root@zdevops-master k8s-yaml]# kubectl exec -it redis-0 -n zdevops -- redis-cli -a redis@abc.com --cluster create --cluster-replicas 1 $(kubectl get pods -n zdevops -l app=redis -o jsonpath='{range.items[*]}{.status.podIP}:6379 {end}')
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 10.233.117.93:6379 to 10.233.116.48:6379
Adding replica 10.233.87.249:6379 to 10.233.117.85:6379
Adding replica 10.233.116.13:6379 to 10.233.87.244:6379
M: 457fed7883cd08cfac98db4bdf6be0d7028b741e 10.233.116.48:6379
   slots:[0-5460] (5461 slots) master
M: 63021f3f78fc133215d244dd99d42519f3116aad 10.233.117.85:6379
   slots:[5461-10922] (5462 slots) master
M: ba6d84c6cfda009061df0955201e175ab4c95845 10.233.87.244:6379
   slots:[10923-16383] (5461 slots) master
S: ebeef4043ef84179a939d5c033ad0599d77307a4 10.233.116.13:6379
   replicates ba6d84c6cfda009061df0955201e175ab4c95845
S: 9133d29edbae393431eebe8b5ef3a34b7abe6163 10.233.117.93:6379
   replicates 457fed7883cd08cfac98db4bdf6be0d7028b741e
S: adbbdfe0e862270771d77e500939a799df698d71 10.233.87.249:6379
   replicates 63021f3f78fc133215d244dd99d42519f3116aad
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join

>>> Performing Cluster Check (using node 10.233.116.48:6379)
M: 457fed7883cd08cfac98db4bdf6be0d7028b741e 10.233.116.48:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
M: ba6d84c6cfda009061df0955201e175ab4c95845 10.233.87.244:6379
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
M: 63021f3f78fc133215d244dd99d42519f3116aad 10.233.117.85:6379
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: adbbdfe0e862270771d77e500939a799df698d71 10.233.87.249:6379
   slots: (0 slots) slave
   replicates 63021f3f78fc133215d244dd99d42519f3116aad
S: ebeef4043ef84179a939d5c033ad0599d77307a4 10.233.116.13:6379
   slots: (0 slots) slave
   replicates ba6d84c6cfda009061df0955201e175ab4c95845
S: 9133d29edbae393431eebe8b5ef3a34b7abe6163 10.233.117.93:6379
   slots: (0 slots) slave
   replicates 457fed7883cd08cfac98db4bdf6be0d7028b741e
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

### 3.8. 手动创建 Redis 集群

手动配置 3 个 master 3 个 slave 的集群，此步骤只为了记录手动操作的过程，实际环境建议用自动创建的方式。

一共创建了 6 个 Redis pod，集群主-> 从配置的规则为 0->3，1->4，2->5。

没有采用自动获取 IP 的方式，太长了占地方，手工查询 pod IP 并进行相关配置。

- 查询 Redis pod 分配的 IP

```shell
[root@zdevops-master k8s-yaml]# kubectl get pods -n zdevops -o wide | grep redis
redis-0                         1/1     Running   0          6m49s   10.233.116.254   ks-k8s-master-2   <none>           <none>
redis-1                         1/1     Running   0          6m38s   10.233.117.96    ks-k8s-master-0   <none>           <none>
redis-2                         1/1     Running   0          6m26s   10.233.87.230    ks-k8s-master-1   <none>           <none>
redis-3                         1/1     Running   0          6m15s   10.233.116.9     ks-k8s-master-2   <none>           <none>
redis-4                         1/1     Running   0          6m4s    10.233.117.97    ks-k8s-master-0   <none>           <none>
redis-5                         1/1     Running   0          5m52s   10.233.87.255    ks-k8s-master-1   <none>           <none>
```

- 创建集群 master 节点

```shell
# 三个IP地址为 redis-0 redis-1 redis-2 对应的IP, 中间需要输入一次yes
[root@zdevops-master k8s-yaml]# kubectl exec -it redis-0 -n zdevops -- redis-cli -a redis@abc.com --cluster create 10.233.116.254:6379 10.233.117.96:6379 10.233.87.230:6379
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
>>> Performing hash slots allocation on 3 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
M: 3f9728539f5eed406cbc0542f4c8fc1bd247916c 10.233.116.254:6379
   slots:[0-5460] (5461 slots) master
M: eead96f0bd0e778787b524505fc83dcf20683912 10.233.117.96:6379
   slots:[5461-10922] (5462 slots) master
M: fef1d03b6a613cbe306a334bf584d7cf9846d07b 10.233.87.230:6379
   slots:[10923-16383] (5461 slots) master
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
..
>>> Performing Cluster Check (using node 10.233.116.254:6379)
M: 3f9728539f5eed406cbc0542f4c8fc1bd247916c 10.233.116.254:6379
   slots:[0-5460] (5461 slots) master
M: eead96f0bd0e778787b524505fc83dcf20683912 10.233.117.96:6379
   slots:[5461-10922] (5462 slots) master
M: fef1d03b6a613cbe306a334bf584d7cf9846d07b 10.233.87.230:6379
   slots:[10923-16383] (5461 slots) master
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

- 为每个 master 添加 slave 节点

```shell
 # 10.233.116.254:6379 的位置为任意一个 master 节点的 ip 地址,一般用 redis-0 的 IP 地址
 # 10.233.123.142:6379 的位置为 slave 的 IP 地址
 # --cluster-master-id 参数为 slave 对应的 master 的 ID，如果不指定则随机分配到任意一个主节点
 
# 第一组 redis0 -> redis3
kubectl exec -it redis-0 -n zdevops -- redis-cli -a redis@abc.com --cluster add-node 10.233.116.9:6379 10.233.116.254:6379 --cluster-slave --cluster-master-id 3f9728539f5eed406cbc0542f4c8fc1bd247916c

# 第二组 redis1 -> redis4
kubectl exec -it redis-0 -n zdevops -- redis-cli -a redis@abc.com --cluster add-node 10.233.117.97:6379 10.233.116.254:6379 --cluster-slave --cluster-master-id eead96f0bd0e778787b524505fc83dcf20683912

# 第三组 redis2 -> redis5
kubectl exec -it redis-0 -n zdevops -- redis-cli -a redis@abc.com --cluster add-node 10.233.87.255:6379 10.233.116.254:6379 --cluster-slave --cluster-master-id fef1d03b6a613cbe306a334bf584d7cf9846d07b
```

### 3.9. 验证集群状态

- 执行命令

```shell
[root@zdevops-master k8s-yaml]# kubectl exec -it redis-0 -n zdevops -- redis-cli -a redis@abc.com --cluster check $(kubectl get pods -n zdevops -l app=redis -o jsonpath='{range.items[0]}{.status.podIP}:6379{end}')
```

- 正常状态结果

```shell
[root@zdevops-master data]# kubectl exec -it redis-0 -n zdevops -- redis-cli -a redis@abc.com --cluster check $(kubectl get pods -n zdevops -l app=redis -o jsonpath='{range.items[0]}{.status.podIP}:6379{end}')
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
10.233.116.254:6379 (3f972853...) -> 0 keys | 5461 slots | 1 slaves.
10.233.87.230:6379 (fef1d03b...) -> 0 keys | 5461 slots | 1 slaves.
10.233.117.96:6379 (eead96f0...) -> 0 keys | 5462 slots | 1 slaves.
[OK] 0 keys in 3 masters.
0.00 keys per slot on average.
>>> Performing Cluster Check (using node 10.233.116.254:6379)
M: 3f9728539f5eed406cbc0542f4c8fc1bd247916c 10.233.116.254:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: cf077e77b35c1b8cd7f19a2dd79320f8960b31f5 10.233.87.255:6379
   slots: (0 slots) slave
   replicates fef1d03b6a613cbe306a334bf584d7cf9846d07b
S: 03e3d21e028e506c246e702abb8fc0ce5676ac22 10.233.117.97:6379
   slots: (0 slots) slave
   replicates eead96f0bd0e778787b524505fc83dcf20683912
S: 49b3f1ad6200b4171e02bc23b0ac55fa7b3ef0cf 10.233.116.9:6379
   slots: (0 slots) slave
   replicates 3f9728539f5eed406cbc0542f4c8fc1bd247916c
M: fef1d03b6a613cbe306a334bf584d7cf9846d07b 10.233.87.230:6379
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
M: eead96f0bd0e778787b524505fc83dcf20683912 10.233.117.96:6379
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

## 4. 安装管理客户端

大部分开发、运维人员还是喜欢图形化的 Redis 管理工具，所以安排一个 Redis 官方提供的图形化工具 RedisInsight。

由于 RedisInsight 默认并不提供登录验证功能，因此，在系统安全要求比较高的环境会有安全风险，请慎用！

个人建议生产环境使用命令行工具。

### 4.1. 准备离线镜像

- 下载镜像

```shell
docker pull redislabs/redisinsight:1.12.0
```

- 重新打 tag

```shell
docker tag redislabs/redisinsight:1.12.0 registry.zdevops.com.cn/library/redisinsight:1.12.0
```

- 推送到私有镜像仓库

```shell
docker push registry.zdevops.com.cn/library/redisinsight:1.12.0
```

### 4.2. 资源配置清单

- **redisinsight-deploy.yaml**

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redisinsight
  namespace: zdevops
  labels:
    app: redisinsight
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redisinsight
  template:
    metadata:
      labels:
        app: redisinsight
    spec:
      containers:
        - name: redisinsight
          image: registry.zdevops.com.cn/library/redisinsight:1.12.0
          ports:
            - name: tcp-8001
              containerPort: 8001
              protocol: TCP
          resources:
            limits:
              cpu: '2'
              memory: 4000Mi
            requests:
              cpu: 100m
              memory: 500Mi

---
kind: Service
apiVersion: v1
metadata:
  name: redisinsight-external
  namespace: zdevops
  labels:
    app: redisinsight-external
spec:
  ports:
    - name: tcp-8001
      protocol: TCP
      port: 8001
      targetPort: 8001
      nodePort: 31000
  selector:
    app: redisinsight
  type: NodePort

```

### 4.3. GitOps

**在运维开发服务器上操作**

```shell
# 在已有代码仓库创建 redis/redisinsight 目录
[root@zdevops-master k8s-yaml]# mkdir -p redis/redisinsight

# 编辑资源配置清单
[root@zdevops-master k8s-yaml]# vi redis/redisinsight/redisinsight-deploy.yaml

# 提交 Git
[root@zdevops-master k8s-yaml]# git add redis/redisinsight
[root@zdevops-master k8s-yaml]# git commit -am '添加 redisinsight 资源配置清单'
[root@zdevops-master k8s-yaml]# git push
```

### 4.4. 部署资源

**在运维管理服务器上操作**

- `更新镜像仓库代码

```shell
[root@zdevops-master k8s-yaml]# git pull
```

- 部署资源

```shell
[root@zdevops-master k8s-yaml]# kubectl apply -f redis/redisinsight/redisinsight-deploy.yaml
```

### 4.5. 界面初始化

- 打开 RedisInsight 控制台，http://192.168.9.91:31000
- 进入默认配置页面，只勾选**第一个**按钮，点击 **CONFIRM**

![kubesphere-redis-0](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-redis-0.png)

![kubesphere-redis-1](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-redis-1.png)

- 连接 Redis 数据库

![kubesphere-redis-2](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-redis-2.png)

![kubesphere-redis-3](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-redis-3.png)

- 点击 **Connect to Redis Database**，按提示填写信息，点击 **ADD REDIS DATABASE**
  - Host 填写 Redis headless 服务的域名简写
  - Name 随便写，就是一个标识
  - Password 填写连接 Redis 的密码

![kubesphere-redis-4](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-redis-4.png)

- 选择一个 master 数据库，点击 **ADD CLUSTER DATABASE**

![kubesphere-redis-5](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-redis-5.png)

![kubesphere-redis-6](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-redis-6.png)

- 点击新创建的数据库连接，进入管理界面

![kubesphere-redis-7](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-redis-7.png)

![kubesphere-redis-8](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/kubesphere-redis-8.png)

## 5. 总结

本文详细介绍了 Redis 单节点和集群模式在基于 KubeSphere 部署的 K8S 集群上的安装部署过程。同时，介绍了 Redis 图形化管理工具 RedisInsight 的安装使用。

本文的配置方案可直接用于开发测试环境，对于生产环境也有一定的借鉴意义。

> **参考文档**

> **Get 文档**

- Github https://github.com/devops/z-notes
- Gitee https://gitee.com/zdevops/z-notes

> **Get 代码**

- Github https://github.com/devops/ansible-zdevops
- Gitee https://gitee.com/zdevops/ansible-zdevops

> **B 站**

- [老 Z 手记](https://space.bilibili.com/1039301316) https://space.bilibili.com/1039301316

> **版权声明** 

- 所有内容均属于原创，整理不易，感谢收藏，转载请标明出处。

> **About Me**

- 昵称：老 Z
- 坐标：山东济南
- 职业：运维架构师 / 高级运维工程师 =**运维**
- 微信：zdevops
- 关注的领域：云计算 / 云原生技术运维，自动化运维
- 技能标签：OpenStack、Ansible、K8S、Python、Go、CNCF
