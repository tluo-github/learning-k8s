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
# 然后通过 kubectl 工具部署
kubectl apply -f nginx-deployment.yaml

```

1. Kubectl 会执行客户端验证，以确保非法请求（如创建不支持的资源或使用格式错误的镜像名称）快速失败，并不会发送给 kube-apiserver——即通过减少不必要的负载来提高系统性能。
2. Kubectl 开始封装它将发送给 kube-apiserver 的 HTTP 请求。在 Kubernetes 中，访问或更改状态的所有尝试都通过 kube-apiserver 进行，后者又与 etcd 进行通信.
3. APIServer 接收到 Kubectl 发出的RESTFull 请求，开始验证(Authentication) 和 授权(Authorization)。之后进入Admission Controllers。
4. Admission Controllers 拦截该请求，以确保它符合集群更广泛的期望和规则。它们是对象持久化到 etcd 之前的最后一个堡垒，因此它们封装了剩余的系统检查以确保操作不会产生意外或负面结果。Admission Controller 的工作方式类似于 Authentication 和 Authorization，但有一个区别：如果单个 Admission Controller 失败，则整个链断开，请求将失败。Admission Controller 设计的真正优势在于它致力于提升可扩展性。每个控制器都作为插件存储在 plugin/pkg/admission 目录中，最后编译进 kube-apiserver 二进制文件。例如：启动容器之前需要下载镜像，或者检查具备某命名空间的资源。目前支持的准入控制器有：
  * AlwaysPullImages：此准入控制器修改每个 Pod 的时候都强制重新拉取镜像。
  * DefaultStorageClass：此准入控制器观察创建PersistentVolumeClaim时不请求任何特定存储类的对象，并自动向其添加默认存储类。这样，用户就不需要关注特殊存储类而获得默认存储类。
  * DefaultTolerationSeconds：此准入控制器将Pod的容忍时间notready:NoExecute和unreachable:NoExecute 默认设置为5分钟。
  * DenyEscalatingExec：此准入控制器将拒绝exec 和附加命令到以允许访问宿主机的升级了权限运行的pod。
  * EventRateLimit (alpha)：此准入控制器缓解了 API Server 被事件请求淹没的问题，限制时间速率。
  * ExtendedResourceToleration：此插件有助于创建具有扩展资源的专用节点。
  * ImagePolicyWebhook：此准入控制器允许后端判断镜像拉取策略，例如配置镜像仓库的密钥。
  * Initializers (alpha)：Pod初始化的准入控制器，详情请参考动态准入控制。
  * LimitPodHardAntiAffinityTopology：此准入控制器拒绝任何在 requiredDuringSchedulingRequiredDuringExecution 的 AntiAffinity 字段中定义除了kubernetes.io/hostname 之外的拓扑关键字的 pod 。
  * LimitRanger：此准入控制器将确保所有资源请求不会超过 namespace 的 LimitRange。
  * MutatingAdmissionWebhook （1.9版本中为beta）：该准入控制器调用与请求匹配的任何变更 webhook。匹配的 webhook是串行调用的；如果需要，每个人都可以修改对象。
  * NamespaceAutoProvision：此准入控制器检查命名空间资源上的所有传入请求，并检查引用的命名空间是否存在。如果不存在就创建一个命名空间。
  * NamespaceExists：此许可控制器检查除 Namespace 其自身之外的命名空间资源上的所有请求。如果请求引用的命名空间不存在，则拒绝该请求。
  * NamespaceLifecycle：此准入控制器强制执行正在终止的命令空间中不能创建新对象，并确保Namespace拒绝不存在的请求。此准入控制器还防止缺失三个系统保留的命名空间default、kube-system、kube-public。
  * NodeRestriction：该准入控制器限制了 kubelet 可以修改的Node和Pod对象。
  * OwnerReferencesPermissionEnforcement：此准入控制器保护对metadata.ownerReferences对象的访问，以便只有对该对象具有“删除”权限的用户才能对其进行更改。
  * PodNodeSelector：此准入控制器通过读取命名空间注释和全局配置来限制可在命名空间内使用的节点选择器。
  * PodPreset：此准入控制器注入一个pod，其中包含匹配的PodPreset中指定的字段，详细信息见Pod Preset。
  * PodSecurityPolicy：此准入控制器用于创建和修改pod，并根据请求的安全上下文和可用的Pod安全策略确定是否应该允许它。
  * PodTolerationRestriction：此准入控制器首先验证容器的容忍度与其命名空间的容忍度之间是否存在冲突，并在存在冲突时拒绝该容器请求。
  * Priority：此控制器使用priorityClassName字段并填充优先级的整数值。如果未找到优先级，则拒绝Pod。
  * ResourceQuota：此准入控制器将观察传入请求并确保它不违反命名空间的ResourceQuota对象中列举的任何约束。
  * SecurityContextDeny：此准入控制器将拒绝任何试图设置某些升级的SecurityContext字段的pod 。
  * ServiceAccount：此准入控制器实现serviceAccounts的自动化。
用中的存储对象保护：该StorageObjectInUseProtection插件将kubernetes.io/pvc-protection或kubernetes.io/pv-protection终结器添加到新创建的持久卷声明（PVC）或持久卷（PV）。在用户删除PVC或PV的情况下，PVC或PV不会被移除，直到PVC或PV保护控制器从PVC或PV中移除终结器。有关更多详细信息，请参阅使用中的存储对象保护。
  * ValidatingAdmissionWebhook（1.8版本中为alpha；1.9版本中为beta）：该准入控制器调用与请求匹配的任何验证webhook。匹配的webhooks是并行调用的；如果其中任何一个拒绝请求，则请求失败。
 5.   










