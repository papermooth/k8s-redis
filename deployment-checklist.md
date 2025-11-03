# Redis集群部署检查清单

## 前置条件检查

- [ ] Kubernetes集群正常运行
  ```bash
  kubectl get nodes
  ```
- [ ] 所有节点上安装了NFS客户端
  ```bash
  rpm -q nfs-utils || echo "需要安装nfs-utils"
  ```
- [ ] NFS服务器已配置并正常工作
  ```bash
  showmount -e <nfs-server-ip>
  ```
- [ ] NFS共享目录已创建
  ```bash
  ls -la /opt/nfs/pv*
  ```

## 存储资源配置

- [ ] 创建了StorageClass
  ```bash
  kubectl get sc redis-sc
  ```
- [ ] 创建了6个PersistentVolume
  ```bash
  kubectl get pv | grep redis-sc
  ```
- [ ] 所有PV状态为Available
  ```bash
  kubectl get pv | grep Available
  ```

## Redis配置准备

- [ ] 创建了Redis配置文件
  ```bash
  ls -la /opt/redis/conf/redis.conf
  ```
- [ ] Redis配置文件已复制到所有K8s节点
  ```bash
  scp /opt/redis/conf/redis.conf node1:/opt/redis/conf/
  scp /opt/redis/conf/redis.conf node2:/opt/redis/conf/
  ```
- [ ] 配置文件中启用了集群模式和正确的端口设置
  ```bash
  grep -E 'cluster-enabled|cluster-port' /opt/redis/conf/redis.conf
  ```

## Kubernetes资源部署

- [ ] 创建了redis命名空间
  ```bash
  kubectl get namespace redis
  ```
- [ ] 部署了Headless Service
  ```bash
  kubectl get service -n redis redis-hs
  ```
- [ ] 部署了Redis StatefulSet
  ```bash
  kubectl get statefulset -n redis redis
  ```
- [ ] 所有6个Redis Pod都已成功创建并运行
  ```bash
  kubectl get pods -n redis
  ```

## Redis集群初始化

- [ ] 获取所有Redis Pod的IP地址
  ```bash
  kubectl get pods -n redis -o wide
  ```
- [ ] 创建了主节点集群（选择3个奇数索引的Pod作为主节点）
  - [ ] redis-0
  - [ ] redis-2
  - [ ] redis-4
- [ ] 为每个主节点添加了从节点
  - [ ] redis-1 作为 redis-0 的从节点
  - [ ] redis-3 作为 redis-2 的从节点
  - [ ] redis-5 作为 redis-4 的从节点
- [ ] 验证集群状态正常
  ```bash
  kubectl exec -it redis-0 -n redis -- redis-cli -a <password> cluster info
  ```
- [ ] 确认所有16384个槽位都已分配
  ```bash
  kubectl exec -it redis-0 -n redis -- redis-cli -a <password> cluster info | grep slots
  ```

## 外部访问配置

- [ ] 创建了NodePort Service用于外部访问
  ```bash
  kubectl get service -n redis redis-ss
  ```
- [ ] 验证可以通过NodePort访问Redis集群
  ```bash
  redis-cli -h <node-ip> -p 30379 -a <password> -c ping
  ```

## 功能验证

- [ ] 测试数据写入和读取
  ```bash
  redis-cli -h <node-ip> -p 30379 -a <password> -c set test_key "Hello World"
  redis-cli -h <node-ip> -p 30379 -a <password> -c get test_key
  ```
- [ ] 验证主从复制正常工作
  ```bash
  kubectl exec -it redis-0 -n redis -- redis-cli -a <password> info replication
  kubectl exec -it redis-1 -n redis -- redis-cli -a <password> info replication
  ```
- [ ] 验证集群负载均衡功能
  ```bash
  # 连接到集群并执行多个set操作
  # 检查命令是否被重定向到不同的节点
  ```

## 故障转移测试

- [ ] 模拟主节点故障
  ```bash
  kubectl delete pod redis-0 -n redis
  ```
- [ ] 验证从节点自动提升为主节点
  ```bash
  kubectl exec -it redis-1 -n redis -- redis-cli -a <password> info replication
  ```
- [ ] 原主节点恢复后验证其成为从节点
  ```bash
  kubectl exec -it redis-0 -n redis -- redis-cli -a <password> info replication
  ```

## 清理和维护检查

- [ ] 备份重要数据和配置
- [ ] 记录集群访问信息（IP地址、端口、密码等）
- [ ] 确认监控工具已配置（如需要）

## 常见问题排查

如果遇到问题，请检查以下内容：

- [ ] NFS服务器和客户端连接正常
- [ ] Redis配置文件正确且已复制到所有节点
- [ ] 防火墙设置允许Redis端口通信
- [ ] 所有Kubernetes资源状态正常
- [ ] Redis Pod日志中没有错误信息
  ```bash
  kubectl logs <pod-name> -n redis
  ```

---

完成所有检查项后，您的Redis集群应该已经成功部署并可以正常工作了。