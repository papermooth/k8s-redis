# K8s Redis集群部署指南

本文档详细介绍如何在Kubernetes环境中部署高可用的Redis集群。

## 项目概述

本项目实现了在Kubernetes集群中部署Redis集群（3主3从架构），包含以下核心组件：

- 持久化存储：使用NFS提供数据持久化
- 有状态部署：使用StatefulSet确保Pod的稳定标识
- 网络服务：配置Headless Service和NodePort Service
- 自动恢复：支持Pod重启后的集群自动恢复

## 目录结构

```
/Users/bufferhhh/k8s-redis/
├── k8s-redis.assets/    # 文档相关截图
├── k8s-redis.md         # 详细部署步骤文档
└── README.md            # 项目说明文档（本文档）
```

## Redis部署模式对比

Redis有三种主要部署模式：

1. **单实例模式**：适用于测试环境，简单但无高可用保障
2. **哨兵模式**：提供基本的高可用能力，但水平扩展有限
3. **集群模式**：支持数据分片和高可用，水平扩展能力强，适合生产环境

> **推荐**：在生产环境中使用集群模式，提供更好的性能和可用性

## 部署架构

![部署架构](k8s-redis.assets/2025-11-03%2010.36.31.png)

### 核心组件

- **NFS服务器**：提供6个持久化存储卷
- **StorageClass/PersistentVolume**：管理存储资源
- **Headless Service**：为StatefulSet提供稳定的网络标识
- **StatefulSet**：部署6个Redis节点（3主3从）
- **NodePort Service**：提供外部访问入口

## 快速开始

### 1. 前置准备

#### 1.1 NFS服务器安装配置

```bash
# 安装NFS服务器
yum -y install nfs-utils

# 创建共享目录
mkdir -p /opt/nfs/pv{1..6}

# 配置NFS共享
cat > /etc/exports << EOF
/opt/nfs/pv1 *(rw,sync,no_subtree_check,no_root_squash)
/opt/nfs/pv2 *(rw,sync,no_subtree_check,no_root_squash)
/opt/nfs/pv3 *(rw,sync,no_subtree_check,no_root_squash)
/opt/nfs/pv4 *(rw,sync,no_subtree_check,no_root_squash)
/opt/nfs/pv5 *(rw,sync,no_subtree_check,no_root_squash)
/opt/nfs/pv6 *(rw,sync,no_subtree_check,no_root_squash)
EOF

# 更新配置并重启服务
exportfs -r
systemctl restart nfs-server
```

#### 1.2 Kubernetes集群节点配置

```bash
# 在所有K8s节点安装NFS客户端
yum -y install nfs-utils

# 验证NFS连接
showmount -e <nfs-server-ip>
```

### 2. 存储资源配置

#### 2.1 创建StorageClass

```yaml
# redis-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: redis-sc
provisioner: nfs-storage
```

```bash
kubectl apply -f redis-sc.yaml
```

#### 2.2 创建PersistentVolumes

创建6个PV，每个对应一个Redis节点的数据存储。详细配置见完整文档。

### 3. Redis集群部署

#### 3.1 创建命名空间

```bash
kubectl create namespace redis
```

#### 3.2 创建Headless Service

```yaml
# redis-hs.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s.kuboard.cn/layer: db
    k8s.kuboard.cn/name: redis
  name: redis-hs
  namespace: redis
spec:
  ports:
    - name: nnbary
      port: 6379
      protocol: TCP
      targetPort: 6379
  selector:
    k8s.kuboard.cn/layer: db
    k8s.kuboard.cn/name: redis
  clusterIP: None
```

```bash
kubectl apply -f redis-hs.yaml
```

#### 3.3 创建Redis配置文件

在所有K8s节点上创建Redis配置文件：

```bash
mkdir -p /opt/redis/conf
# 创建redis.conf文件（详细配置见完整文档）
```

#### 3.4 部署Redis StatefulSet

```yaml
# redis.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: redis
  labels:
    k8s.kuboard.cn/layer: db
    k8s.kuboard.cn/name: redis
spec:
  replicas: 6
  selector:
    matchLabels:
      k8s.kuboard.cn/layer: db
      k8s.kuboard.cn/name: redis
  serviceName: redis-hs
  template:
    metadata:
      labels:
        k8s.kuboard.cn/layer: db
        k8s.kuboard.cn/name: redis
    spec:
      terminationGracePeriodSeconds: 20
      containers:
        - name: redis
          image: docker.io/library/redis:8.0.3
          command: ["/usr/local/bin/redis-server", "/etc/redis/redis.conf"]
          ports:
            - name: redis
              containerPort: 6379
              protocol: "TCP"
            - name: cluster
              containerPort: 16379
              protocol: "TCP"
          volumeMounts:
            - name: "redis-conf"
              mountPath: "/etc/redis/redis.conf"
            - name: "redis-data"
              mountPath: "/data"
      volumes:
        - name: "redis-conf"
          hostPath:
            path: "/opt/redis/conf/redis.conf"
            type: FileOrCreate
  volumeClaimTemplates:
    - metadata:
        name: redis-data
      spec:
        accessModes: [ "ReadWriteMany" ]
        resources:
          requests:
            storage: 200M
        storageClassName: redis-sc
```

```bash
kubectl apply -f redis.yaml
```

### 4. 初始化Redis集群

```bash
# 进入第一个Redis节点
kubectl exec -it redis-0 -n redis -- /bin/bash

# 创建主节点集群
redis-cli --cluster create <redis-0-ip>:6379 <redis-2-ip>:6379 <redis-4-ip>:6379 -a <password>

# 添加从节点
redis-cli --cluster add-node <slave-ip>:6379 <master-ip>:6379 --cluster-slave --cluster-master-id <master-id> -a <password>
```

### 5. 创建外部访问Service

```yaml
# redis-ss.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s.kuboard.cn/layer: db
    k8s.kuboard.cn/name: redis
  name: redis-ss
  namespace: redis
spec:
  ports:
    - name: imdgss
      port: 6379
      protocol: TCP
      targetPort: 6379
      nodePort: 30379
  selector:
    k8s.kuboard.cn/layer: db
    k8s.kuboard.cn/name: redis
  type: NodePort
```

```bash
kubectl apply -f redis-ss.yaml
```

## 验证和测试

### 检查集群状态

```bash
# 检查Pod状态
kubectl get pods -n redis -o wide

# 检查集群健康状态
kubectl exec -it redis-0 -n redis -- redis-cli -a <password> cluster info

# 检查节点状态
kubectl exec -it redis-0 -n redis -- redis-cli -a <password> cluster nodes
```

### 功能测试

```bash
# 连接Redis集群
redis-cli -h <node-ip> -p 30379 -a <password> -c

# 测试数据写入和读取
set mykey "Hello Redis Cluster"
get mykey

# 检查复制状态
info replication
```

## 故障转移测试

```bash
# 删除一个Master Pod模拟故障
kubectl delete pod redis-0 -n redis

# 验证集群自动恢复
kubectl exec -it redis-1 -n redis -- redis-cli -a <password> cluster nodes
```

## 关键机制说明

### 网络标识

Redis集群使用NodeId（保存在nodes.conf文件中）作为稳定的网络标识，而不是依赖于Pod的IP地址。这使得Pod重启后即使IP改变，集群也能正确识别和恢复节点关系。

### 持久化存储

- 每个Redis节点使用独立的PV存储数据
- 数据持久化到NFS服务器，确保Pod重启后数据不丢失
- nodes.conf文件保存了关键的集群拓扑信息

## 注意事项

1. **Redis版本**：本教程使用Redis 8.0.3，其他版本可能需要调整配置
2. **网络要求**：确保K8s集群节点间网络通畅，特别是Redis集群端口（6379和16379）
3. **安全配置**：生产环境应加强密码强度和访问控制
4. **资源规划**：根据实际数据量调整存储容量和资源限制

## 常见问题

### 1. 为什么使用StatefulSet而不是Deployment？

StatefulSet为Pod提供稳定的网络标识和存储，这对于Redis集群至关重要。

### 2. 为什么需要Headless Service？

Headless Service允许直接访问每个Pod，而不是通过负载均衡，这对于Redis集群节点间的直接通信是必要的。

### 3. Pod重启后集群如何恢复？

Redis使用NodeId（存储在nodes.conf中）作为节点的唯一标识，即使IP改变，集群也能识别并恢复节点关系。

## 参考资料

- [Redis官方文档](https://redis.io/documentation)
- [Kubernetes StatefulSet文档](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)

## 完整部署步骤

请参考项目中的详细文档：

- [k8s-redis.md](k8s-redis.md) - 包含完整的部署步骤、配置文件和截图说明