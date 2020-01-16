# Kubernetes 架构解析

Kubernetes 是用来管理容器集群的平台。既然是管理集群，那么就存在被管理节点，针对每个 Kubernetes 集群都由一个 Master 负责管理和控制集群节点。Kubernetes集群包含有节点代理kubelet和Master组件(APIs, scheduler, etc)，一切都基于分布式的存储系统。下面这张图是Kubernetes的架构图。


![](/images/k8s-architecture.png)



Master 主要包含组件:
* etcd保存了整个集群的状态；
* apiserver提供了资源操作的唯一入口，并提供认证、授权、访问控制、API注册和发现等机制；
* controller manager负责维护集群的状态，比如故障检测、自动扩展、滚动更新等；
* scheduler负责资源的调度，按照预定的调度策略将Pod调度到相应的机器上；

Node 主要包含组件:
* kubelet负责维护容器的生命周期，同时也负责Volume（CSI）和网络（CNI）的管理；
* kube-proxy负责为Service提供cluster内部的服务发现和负载均衡；
* Container runtime负责镜像管理以及Pod和容器的真正运行（CRI）；

下图清晰表明了Kubernetes的架构设计以及组件之间的通信协议。


