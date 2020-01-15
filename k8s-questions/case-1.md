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
```

当 master 一定时间没有收到 node 心跳时，会将该 node 设置为 NotReady,而 node 为 NotReady 时，该 node 上的 pod 所对应的 endpoints 内容会被 master 删除，其他正常的 node 的 kube-proxy 因为监控着 endpoints 变化，对应的内容为空,则 service 对应 pod 的网络转发规则 IPVS 被删除。于是前面 Nginx-> ingress -> service -X-> pod。 在最后service 到 pod 断链。