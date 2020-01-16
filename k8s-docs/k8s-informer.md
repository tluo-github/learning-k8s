# kubernetes informer

#### Informer介绍

Informer是Client-go中的一个核心工具包。在Kubernetes源码中，如果Kubernetes的某个组件，需要List/Get Kubernetes中的Object，在绝大多 数情况下，会直接使用Informer实例中的Lister()方法（该方法包含 了 Get 和 List 方法），而很少直接请求Kubernetes API。Informer最基本 的功能就是List/Get Kubernetes中的Object。

如下图所示，仅需要十行左右的代码就能实现对 Pod 的 List 和 Get。
![](/images/k8s-informer.jpg)

#### Informer 设计中的关键点

为了让Client-go更快地返回List/Get请求的结果、减少对Kubenetes API的直接调用，Informer被设计实现为一个依赖Kubernetes List/Watch API、可监听事件并触发回调函数的二级缓存工具包。

##### 更快地返回 List/Get 请求，减少对 Kubenetes API 的直接调用

使用Informer实例的Lister()方法，List/Get Kubernetes中的Object时，Informer不会去请求Kubernetes API，而是直接查找缓存在本地内存中的数据(这份数据由Informer自己维护)。通过这种方式，Informer既可以更快地返回结果，又能减少对Kubernetes API的直接调用。

##### 依赖 Kubernetes List/Watch API

Informer只会调用Kubernetes List和Watch两种类型的API。Informer在初始化的时，先调用Kubernetes List API获得某种resource的全部Object，缓存在内存中; 然后，调用Watch API去watch这种resource，去维护这份缓存; 最后，Informer就不再调用Kubernetes的任何 API。

用List/Watch去维护缓存、保持一致性是非常典型的做法，但令人费解的是，Informer只在初始化时调用一次List API，之后完全依赖Watch API去维护缓存，没有任何resync机制。

笔者在阅读Informer代码时候，对这种做法十分不解。按照多数人思路，通过resync机制，重新List一遍resource下的所有Object，可以更好的保证Informer 缓存和Kubernetes中数据的一致性。

咨询过Google内部Kubernetes开发人员之后，得到的回复是:

>在Informer设计之初，确实存在一个relist无法去执resync操作， 但后来被取消了。原因是现有的这种List/Watch机制，完全能够保证永远不会漏掉任何事件，因此完全没有必要再添加relist方法去resync informer的缓存。这种做法也说明了Kubernetes完全信任etcd。

#### 可监听事件并触发回调函数

Informer通过Kubernetes Watch API监听某种resource下的所有事件。而且，Informer可以添加自定义的回调函数，这个回调函数实例(即ResourceEventHandler实例)只需实现OnAdd(obj interface{})OnUpdate(oldObj, newObj interface{}) 和OnDelete(obj interface{}) 三个方法，这三个方法分别对应informer监听到创建、更新和删除这三种事件类型。

在Controller的设计实现中，会经常用到informer的这个功能。



#### Informer 内部主要组件

Informer 中主要包含 Controller、Reflector、DeltaFIFO、LocalStore、Lister 和 Processor 六个组件，其中 Controller 并不是 Kubernetes Controller，这两个 Controller 并没有任何联系；

Reflector 的主要作用是通过 Kubernetes Watch API 监听某种 resource 下的所有事件；

DeltaFIFO 和 LocalStore 是 Informer 的两级缓存；

Lister 主要是被调用 List/Get 方法；

Processor 中记录了所有的回调函数实例(即 ResourceEventHandler 实例)，并负责触发这些函数。

#### Informer 关键逻辑解析

我们以 Pod 为例，详细说明一下 Informer 的关键逻辑：

1. Informer 在初始化时，Reflector 会先 List API 获得所有的 Pod
2. Reflect 拿到全部 Pod 后，会将全部 Pod 放到 Store 中
3. 如果有人调用 Lister 的 List/Get 方法获取 Pod， 那么 Lister 会直接从 Store 中拿数据
![](/images/k8s-informer-1.jpg)
4. Informer 初始化完成之后，Reflector 开始 Watch Pod，监听 Pod 相关 的所有事件;如果此时 pod_1 被删除，那么 Reflector 会监听到这个事件
5. Reflector 将 pod_1 被删除 的这个事件发送到 DeltaFIFO
6. DeltaFIFO 首先会将这个事件存储在自己的数据结构中(实际上是一个 queue)，然后会直接操作 Store 中的数据，删除 Store 中的 pod_1
7. DeltaFIFO 再 Pop 这个事件到 Controller 中
![](/images/k8s-informer-2.jpg)
8. Controller 收到这个事件，会触发 Processor 的回调函数
![](/images/k8s-informer-3.jpg)
LocalStore 会周期性地把所有的 Pod 信息重新放到 DeltaFIFO 中

#### Informer 总结
Informer 的内部原理比较复杂、不太容易上手，但 Informer 却是一个非常稳定可靠的 package，已被 Kubernetes 广泛使用。但是，目前关于 Informer 的文章不是很多，如果文章中有表述不正确的地方，希望各位读者悉心指正。

来源:
* https://yq.aliyun.com/articles/679508


