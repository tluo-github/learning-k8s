# Kubernetes 架构解析

Kubernetes 是用来管理容器集群的平台。既然是管理集群，那么就存在被管理节点，针对每个 Kubernetes 集群都由一个 Master 负责管理和控制集群节点。Kubernetes集群包含有节点代理kubelet和Master组件\(APIs, scheduler, etc\)，一切都基于分布式的存储系统。下面这张图是Kubernetes的架构图。

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

1. 使用kubectl 对 Kubernetes 下命令的，它通过 APIServer 去调用各个进程来完成对 Node 的部署和控制。  
2. APIServer 的核心功能是对核心对象\(例如：Pod，Service，RC\)的增删改查操作，同时也是集群内模块之间数据交换的枢纽。它包括了常用的 API，访问\(权限\)控制，注册，信息存储\(etcd\)等功能。  
3. APIServer 将信息存入 etcd 中。  
4. Controller Manager 控制控制权。它包括 8 个 Controller，分别对应着副本，节点，资源，命名空间，服务等等。  
5. Scheduler Controller 会把 Pod 调度到 Node 上，调度完以后就由 kubelet 来管理 Node 了。  
6. kubelet 用于处理 Master 下发到 Node 的任务\(即 Scheduler 的调度任务\)，同时管理 Pod 及 Pod 中的容器。也会在 APIServer 上注册 Node 信息，定期向 Master 汇报 Node 信息，并通过 cAdvisor 监控容器和节点资源。  
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
3. APIServer 接收到 Kubectl 发出的RESTFull 请求，开始验证\(Authentication\) 和 授权\(Authorization\)。之后进入Admission Controllers。
4. Admission Controllers 拦截该请求，以确保它符合集群更广泛的期望和规则。它们是对象持久化到 etcd 之前的最后一个堡垒，因此它们封装了剩余的系统检查以确保操作不会产生意外或负面结果。Admission Controller 的工作方式类似于 Authentication 和 Authorization，但有一个区别：如果单个 Admission Controller 失败，则整个链断开，请求将失败。Admission Controller 设计的真正优势在于它致力于提升可扩展性。每个控制器都作为插件存储在 plugin/pkg/admission 目录中，最后编译进 kube-apiserver 二进制文件。例如：启动容器之前需要下载镜像，或者检查具备某命名空间的资源。目前支持的准入控制器有：

   * AlwaysPullImages：此准入控制器修改每个 Pod 的时候都强制重新拉取镜像。
   * DefaultStorageClass：此准入控制器观察创建PersistentVolumeClaim时不请求任何特定存储类的对象，并自动向其添加默认存储类。这样，用户就不需要关注特殊存储类而获得默认存储类。
   * DefaultTolerationSeconds：此准入控制器将Pod的容忍时间notready:NoExecute和unreachable:NoExecute 默认设置为5分钟。
   * DenyEscalatingExec：此准入控制器将拒绝exec 和附加命令到以允许访问宿主机的升级了权限运行的pod。
   * EventRateLimit \(alpha\)：此准入控制器缓解了 API Server 被事件请求淹没的问题，限制时间速率。
   * ExtendedResourceToleration：此插件有助于创建具有扩展资源的专用节点。
   * ImagePolicyWebhook：此准入控制器允许后端判断镜像拉取策略，例如配置镜像仓库的密钥。
   * Initializers \(alpha\)：Pod初始化的准入控制器，详情请参考动态准入控制。
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
     ![](/images/k8s-admission.webp)
   * kube-apiserver 将反序列化 HTTP 请求，构造运行时对象（deployment,pod..），并将它们持久化到 etcd。**我们部署的 Deployment 现在存在于 etcd 中，但仍没有看到它真正地 work…**

   在 APIServer，经过 API 调用，权限控制，调用资源和存储资源的过程。实际上还没有真正开始部署应用。这里需要 Controller Manager，Scheduler 和 kubelet 的协助才能完成整个部署过程。

   在介绍他们协同工作之前，要介绍一下在 Kubernetes 中的监听接口。从上面的操作知道，所有部署的信息都会写到 etcd 中保存。

实际上 etcd 在存储部署信息的时候，会发送 Create 事件给 APIServer，而 APIServer 会通过监听\(Watch\)etcd 发过来的事件。其他组件也会监听\(Watch\)APIServer 发出来的事件。  
![](/images/k8s-list-watch.jpg)

Kubernetes 就是用这种 List-Watch 的机制保持数据同步的，如上图：

    * 这里有三个 List-Watch，分别是 kube-controller-manager\(运行在Master\)，kube-scheduler\(运行在 Master\)，kublete\(运行在 Node\)。他们在进程已启动就会监听\(Watch\)APIServer 发出来的事件。
    * kubectl 通过命令行，在 APIServer 上建立一个 Pod 副本。
    * 这个部署请求被记录到 etcd 中，存储起来。
    * 当 etcd 接受创建 Pod 信息以后，会发送一个 Create 事件给 APIServer。
    * 由于 Kubecontrollermanager 一直在监听 APIServer 中的事件。此时 APIServer 接受到了 Create 事件，又会发送给 Kubecontrollermanager。
    * Kubecontrollermanager 在接到 Create 事件以后，调用其中的 Replication Controller 来保证 Node 上面需要创建的副本数量。
    * 在 Controller Manager 创建 Pod 副本以后，APIServer 会在 etcd 中记录这个 Pod 的详细信息。例如在 Pod 的副本数，Container 的内容是什么。
    * 同样的 etcd 会将创建 Pod 的信息通过事件发送给 APIServer。
    * 由于 Scheduler 在监听\(Watch\)APIServer，接收到 APIServer 的创建Pod 事件,将待调度的 Pod 按照调度算法和策略绑定到集群中 Node 上，并将绑定信息写入 etcd 中。
    * Scheduler 调度完毕以后会更新 Pod 的信息，此时的信息更加丰富了。除了知道 Pod 的副本数量，副本内容。还知道部署到哪个 Node 上面了。
    * 同样，将上面的 Pod 信息更新到 etcd 中，保存起来。
    * etcd 将更新成功的事件发送给 APIServer。
    * 注意这里的 kubelet 是在 Node 上面运行的进程，它也通过 List-Watch 的方式监听\(Watch\)APIServer 发送的 Pod 更新的事件。实际上，在第 Scheduler 调度完成后，创建 Pod 的工作就已经完成了。

5. Deployment Controller 

截至目前，我们的 Deployment 已经存储于 etcd 中，并且所有的初始化逻辑都已完成。接下来的阶段将涉及 Deployment 所依赖的资源拓扑结构。

在 Kubernetes， Deployment 实际上只是 ReplicaSet 的集合，而 ReplicaSet 是 Pod 的集合。那么 Kubernetes 如何从一个 HTTP 请求创建这个层次结构呢？这就不得不提 Kubernetes 的内置控制器 （Controller）。

Kubernetes 系统中使用了大量的 Controller， Controller 是一个用于将系统状态从当前状态调谐到期望状态的异步脚本。所有内置的 Controller 都通过组件 kube-controller-manager 并行运行，每种 Controller 都负责一种具体的控制流程。

首先，我们介绍一下 Deployment Controller：

将 Deployment 存储到 etcd 后，APIServer 产生事件 Deployment controller watch APIServer 接收到事件，然后检查 Deployment 没有与之关联的 ReplicaSet 或 Pod。于是创建一个 ReplicaSet。

当完成以上步骤之后，该 Deployment 的 status 就会被更新，然后重新进入与之前相同的循环，等待 Deployment 与期望的状态相匹配。由于 Deployment Controller 只关心 ReplicaSet， 因此需要 ReplicaSet Controller 继续调谐过程。

6. ReplicaSet Controller

ReplicaSet 控制器的作用是监视 ReplicaSet 及其相关资源 Pod 的生命周期。由 Deployment Controller 创建 ReplicaSet 的时候ReplicaSet  会检查新 ReplicaSet 的状态，并意识到现有状态与期望状态之间存在偏差。然后，它会尝试通过调整 Pod 的副本数来调谐这种状态。

7. Scheduler

当所有的 Controller 正常运行后，etcd 中就会保存一个 Deployment、一个 ReplicaSet 和 三个 Pod， 并且可以通过 kube-apiserver 查看到。然而，这些 Pod 还处于 Pending 状态，因为它们还没有被调度到集群中合适的 Node 上。最终解决这个问题的 Controller 是 Scheduler。

Scheduler 作为一个独立的组件运行在集群控制平面上，工作方式与其他 Controller 相同：监听事件并调谐状态。一旦 Scheduler 将 Pod 调度到某个节点上，该节点的 Kubelet 就会接管该 Pod 并开始部署。

8. Kubelet

在 Kubernetes 集群中，每个 Node 节点上都会启动一个 Kubelet 服务进程，该进程用于处理 Scheduler 下发到本节点的任务，管理 Pod 的生命周期。这意味着它将处理 Pod 与 Container Runtime 之间所有的转换逻辑，包括挂载卷、容器日志、垃圾回收以及其他重要事件。

一个有用的方法，你可以把 Kubelet 当成一种特殊的 Controller，它每隔 20 秒（可以自定义）向 kube-apiserver 查询 Pod，过滤 NodeName 与自身所在节点匹配的 Pod 列表。

一旦获取到了这个列表，它就会通过与自己的内部缓存进行比较来检测差异，如果有差异，就开始同步 Pod 列表。我们来看看同步过程是什么样的：

* 如果 Pod 正在创建， Kubelet 就会暴露一些指标，可以用于在 Prometheus 中追踪 Pod 启动延时；
* 然后，生成一个 PodStatus 对象，表示 Pod 当前阶段的状态。Pod 的 Phase 状态是 Pod 在其生命周期中的高度概括，包括 **Pending**、**Running**、**Succeeded**、**Failed** 和 **Unkown** 这几个值。状态的产生过程非常复杂，因此很有必要深入深挖一下：
   * 首先，串行执行一系列 PodSyncHandlers，每个处理器检查 Pod 是否应该运行在该节点上。当其中之一的处理器认为该 Pod 不应该运行在该节点上，则 Pod 的 Phase 值就会变成 PodFailed 并将从该节点被驱逐。例如，以 Job 为例，当一个 Pod 失败重试的时间超过了 activeDeadlineSeconds 设置的值，就会将该 Pod 从该节点驱逐出去
   * 接下来，Pod 的 Phase 值由 init 容器和主容器状态共同决定。由于主容器尚未启动，容器被视为处于等待阶段，如果 Pod 中至少有一个容器处于等待阶段，则其 Phase 值为 Pending
  * 最后，Pod 的 Condition 字段由 Pod 内所有容器状态决定。现在我们的容器还没有被容器运行时 (Container Runtime) 创建，所以，Kubelet 将 PodReady 的状态设置为 False

* 生成 PodStatus 之后，Kubelet 就会将它发送到 Pod 的 status 管理器，该管理器的任务是通过 apiserver 异步更新 etcd 中的记录；
* 接下来运行一系列 admit handlers 以确保该 Pod 具有正确的权限（包括强制执行 AppArmor profiles 和 NO_NEW_PRIVS），在该阶段被拒绝的 Pod 将永久处于 Pending 状态；
* 如果 Kubelet 启动时指定了 cgroups-per-qos 参数，Kubelet 就会为该 Pod 创建 cgroup 并设置对应的资源限制。这是为了更好的 Pod 服务质量（QoS）；
* 为 Pod 创建相应的数据目录，包括：
  * Pod 目录 (通常是 /var/run/kubelet/pods/<podID>)
  * Pod 的挂载卷目录 (<podDir>/volumes)
  * Pod 的插件目录 (<podDir>/plugins)

* 卷管理器会挂载 Spec.Volumes 中定义的相关数据卷，然后等待挂载成功；
* 从 apiserver 中检索 Spec.ImagePullSecrets，然后将对应的 Secret 注入到容器中；
* 最后，通过容器运行时 （Container Runtime） 启动容器（下面会详细描述）。

9. CRI 和 pause 容器 

CRI 提供了 Kubelet 和特定容器运行时实现之间的抽象。通过 protocol buffers（一种更快的 JSON） 和 gRPC API（一种非常适合执行 Kubernetes 操作的API）进行通信。

，当一个 Pod 首次启动时， Kubelet 调用 RunPodSandbox 远程过程命令。沙箱是描述一组容器的 CRI 术语，在 Kubernetes 中对应的是 Pod。所以在此容器运行时中，创建沙箱涉及创建 pause 容器。

**pause** 容器像 Pod 中的所有其他容器的父级一样，因为它承载了工作负载容器最终将使用的许多 Pod 级资源。这些“资源”是 Linux Namespaces（IPC、Network、PID）。

**pause** 容器提供了一种托管所有这些 Namespaces 的方法，并允许子容器共享它们。成为同一 Network Namespace 一部分的好处是同一个 Pod 中的容器可以使用 localhost 相互访问。

**pause** 容器的第二个好处与 PID Namespace 有关。在这些 Namespace 中，进程形成一个分层树，顶部的“init” 进程负责“收获”僵尸进程。更多信息，请深入阅读：https://www.ianlewis.org/en/almighty-pause-container

创建 **pause** 容器后，将开始检查磁盘状态然后启动主容器。

10. CNI 和 Pod 网络

现在，我们的 Pod 有了基本的骨架：一个 pause 容器，它托管所有 Namespaces 以允许 Pod 间通信。但容器的网络如何运作以及建立的？

当 Kubelet 为 Pod 设置网络时，它会将任务委托给 CNI (Container Network Interface) 插件，其运行方式与 Container Runtime Interface 类似。简而言之，CNI 是一种抽象，允许不同的网络提供商对容器使用不同的网络实现。

Kubelet 通过 stdin 将 JSON 数据（配置文件位于 **/etc/cni/net.d** 中）传输到相关的 CNI 二进制文件（位于 **/opt/cni/bin**） 中与之交互。下面是 JSON 配置文件的一个简单示例 ：

```
{
	"cniVersion": "0.3.1",
	"name": "bridge",
	"type": "bridge",
	"bridge": "cnio0",
	"isGateway": true,
	"ipMasq": true,
	"ipam":{
		"type": "host-local",
		"ranges":[{"subnet", "${POD_CIDR}"}],
		"routes": [{"dst": "0.0.0.0.0/0"}]
	}
}
```

接下来会发生什么取决于 CNI 插件，这里我们以 bridge CNI 插件为例：

* 该插件首先会在 Root Network Namespace（也就是宿主机的 Network Namespace） 中设置本地 Linux 网桥，以便为该主机上的所有容器提供网络服务；
* 然后将一个网络接口 （veth 设备对的一端）插入到 pause 容器的 Network Namespace 中，并将另一端连接到网桥上。
* 之后，为 pause 容器的网络接口分配一个 IP 并设置相应的路由，于是 Pod 就有了自己的 IP。IP 的分配是由 JSON 配置文件中指定的 IPAM Plugin 实现的；
  * IPAM Plugin 的工作方式和 CNI 插件类似：通过二进制文件调用并具有标准化的接口，每一个 IPAM Plugin 都必须要确定容器网络接口的 IP、子网以及网关和路由，并将信息返回给 CNI 插件。最常见的 IPAM Plugin 称为 host-local，它从预定义的一组地址池为容器分配 IP 地址。它将相关信息保存在主机的文件系统中，从而确保了单个主机上每个容器 IP 地址的唯一性。
* 对于 DNS， Kubelet 将为 CNI 插件指定 Kubernetes 集群内部 DNS 服务器 IP 地址，确保正确设置容器的 resolv.conf 文件。


11.  跨主机容器网络 

到目前为止，我们已经描述了容器如何与宿主机进行通信，但跨主机之间的容器如何通信呢？

通常情况下， Kubernetes 使用 Overlay 网络来进行跨主机容器通信，这是一种动态同步多个主机间路由的方法。一个较常用的 Overlay 网络插件是 flannel，它提供了跨节点的三层网络。

flannel 不会管容器与宿主机之间的通信（这是 CNI 插件的职责），但它对主机间的流量传输负责。为此，它为主机选择一个子网并将其注册到 etcd，然后保留集群路由的本地表示，并将传出的数据包封装在 UDP 数据报中，确保它到达正确的主机。

12.启动容器

所有的网络配置都已完成。还剩什么？真正启动工作负载容器！

一旦 sanbox 完成初始化并处于 active 状态， Kubelet 将开始为其创建容器。首先启动 PodSpec 中定义的 Init Container，然后再启动主容器，具体过程如下：

* 拉取容器的镜像。如果是私有仓库的镜像，就会使用 PodSpec 中指定的 Secret 来拉取该镜像；
* 通过 CRI 创建容器。Kubelet 使用 PodSpec 中的信息填充了 ContainerConfig 数据结构（在其中定义了 command、image、labels、mounts、devices、environment variables 等），然后通过 protobufs 发送给 CRI。对于 Docker 来说，它会将这些信息反序列化并填充到自己的配置信息中，然后再发送给 Dockerd 守护进程。在这个过程中，它会将一些元数据（例如容器类型、日志路径、sandbox ID 等）添加到容器中；
* 然后 Kubelet 将容器注册到 CPU 管理器，它通过使用 UpdateContainerResources CRI 方法给容器分配给本地节点上的 CPU 资源；
* 最后容器真正启动；
* 如果 Pod 中包含 Container Lifecycle Hooks，容器启动之后就会运行这些 Hooks。Hook 的类型包括两种：Exec（执行一段命令） 和 HTTP（发送 HTTP 请求）。如果 PostStart Hook 启动的时间过长、挂起或者失败，容器将永远不会变成 Running 状态。

最后的最后，现在我们的集群上应该会运行三个容器，分布在一个或多个工作节点上。所有的网络、数据卷和秘钥都由 Kubelet 填充，并通过 CRI 接口添加到容器中，配置成功！

引用:
* http://wsfdl.com/kubernetes/2019/01/10/list_watch_in_k8s.html
* https://github.com/bbbmj/what-happens-when-k8s













