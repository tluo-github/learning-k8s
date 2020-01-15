# 偶发特定Node上的Service与Pod网络不通 

**问题表现:**

kubernetes 集群运行过程中，周末突然有业务模块外部访问 503。
* pod 运行正常在集群内直接访问 pod 成功，访问 service 不成功。
* pod 所在 node 重启 docker 即可恢复。
* pod 所在 node, 在 master 查询状态为 NotReady。
* service 与 pod 关联关系的 endpoint 内容为空。
* 正常node 上 ipvs 查询不到该 service 的 real server。
* 异常node 上 ipvs 能查询到该 service 的 real server。 

**环境:**

* System: CentOS Linux 7.6 (Kernel: 3.10)
* Docker CE Version: 18.09.7 (systemd)
* Kubernetes Version: 1.15.0 (systemd)
* kubeadm 1.15.0
* kubelet 1.15.0
* kubectl 1.15.0
* Flannel UDP 网络方案
* IPVS 作为 kubernetes service 负载均衡方案
* 最终提供服务链路: nginx -> ingress -> service -> pod。


** 问题分析:**

根据事发时收集的现象,出问题的pod所在Node 在 master 查询状态为 NotReady,于是手动模拟场景:

```
10.136.35.173 (master)
10.136.35.180 (node)
10.136.35.181 (node)
应用A 调度到 180
应用B 调度到 181
```

手动在 10.136.35.180 执行 systemctl stop kubelet,停掉 kubelet。过一会之后10.136.35.173 master 未收到 10.136.35.180 心跳将其状态设置为NotReady产生 node not ready 事件。Endpoint Contoller 接收到事件将 endpoints 内容为 10.136.35.180 上 pod ip 删除掉。于是10.136.35.181 根据 endpoints 的更变将 应用A 的 service 对应 应用A pod 的 ipvs 规则删除掉，于是 service --> pod 断链，网络不通情况复现。

疑惑一、node 为什么处于 NotReady 状态?

k8s 集群部署了3个master 组成高可用, node 本地有一个 nginx 容器做负载均衡，事发时 node 上的 kubelet 日志报错连接不上 nginx, nginx 日志报错连接不上 master。故心跳失败，master 将其设置为 NotReady,问题定位在 以nginx 作为 lb 时 后端服务全挂的时候，偶尔一直 hand 住不往后发。最终问题定位在 nginx 负载均衡器问题。

疑惑二、为什么 node 为 NotReay 其node 上	的 pod 不被迁移?

默认情况下 node NotReady,k8s 过5分钟会将其pod 迁移走，这次情况没有被迁移原因在于是 kubelet 是 K8s 的 node agent,而这次问题原因就在于 kubelet 与 master 网络不通，故没有接受到迁移指令。

疑惑三、为什么当时重启docker 服务正常
 
node 上的 nginx 负载均衡器以 docker 容器方式运行，重启docker 负载均衡器也重启，kubelet 与 master 通信恢复正常，master 将其 pod 对应的 endpoints 内容恢复，其他 node 接到事件通知，加上 ipvs 规则，全部恢复正常。


** 解决方案：**

更换负载均衡器,开发了一个简单负载均衡器,带上后端检测及全部不健康也转发请求。



