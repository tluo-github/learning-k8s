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

![](/images/kubernetes-high-level-component-archtecture.jpg)


各组件交互流程:
1.  使用kubectl 对 Kubernetes 下命令的，它通过 APIServer 去调用各个进程来完成对 Node 的部署和控制。
2. APIServer 的核心功能是对核心对象(例如：Pod，Service，RC)的增删改查操作，同时也是集群内模块之间数据交换的枢纽。它包括了常用的 API，访问(权限)控制，注册，信息存储(etcd)等功能。
3. APIServer 将信息存入 etcd 中。
4. Controller Manager 控制控制权。它包括 8 个 Controller，分别对应着副本，节点，资源，命名空间，服务等等。
5. Scheduler Controller 会把 Pod 调度到 Node 上，调度完以后就由 kubelet 来管理 Node 了。
6. kubelet 用于处理 Master 下发到 Node 的任务(即 Scheduler 的调度任务)，同时管理 Pod 及 Pod 中的容器。也会在 APIServer 上注册 Node 信息，定期向 Master 汇报 Node 信息，并通过 cAdvisor 监控容器和节点资源。
7. kube-proxy 完成 service 的定义提供服务发现和负载均衡。

#### 一个部署例子

部署一个nginx,分析各组件通讯过程


```
# nginx-deployment.yaml 
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```











