# Kubernetes list-watch 机制

kubernetes没有像其他分布式系统中额外引入MQ，是因为其设计理念采用了level trigger而非edge trigger。其仅仅通过http+protobuffer的方式，实现list-watcher机制来解决各组件间的消息通知。因此，在了解各组件通信前，必须先了解list-watch机制在kubernetes的应用。


List-watch 是 K8S 统一的异步消息处理机制，保证了消息的实时性，可靠性，顺序性，性能等等，为声明式风格的 API 奠定了良好的基础，它是优雅的通信方式，是 K8S 架构的精髓。


Etcd 存储集群的数据信息，apiserver 作为统一入口，任何对数据的操作都必须经过 apiserver。客户端(kubelet/scheduler/ontroller-manager)通过 list-watch 监听 apiserver 中资源(pod/rs/rc 等等)的 create, update 和 delete 事件，并针对事件类型调用相应的事件处理函数。

那么 list-watch 具体是什么呢，顾名思义，list-watch 有两部分组成，分别是 list 和 watch。list 非常好理解，就是调用资源的 list API 罗列资源，基于 HTTP 短链接实现；watch 则是调用资源的 watch API 监听资源变更事件，基于 HTTP 长链接实现，也是本文重点分析的对象。以 pod 资源为例，它的 list 和 watch API 分别为：

* List API，返回值为 PodList，即一组 pod。
```
GET /api/v1/pods
```
* Watch API，往往带上 watch=true，表示采用 HTTP 长连接持续监听 pod 相关事件，每当有事件来临，返回一个 WatchEvent。

```
GET /api/v1/watch/pods
```

K8S 的 informer 模块封装 list-watch API，用户只需要指定资源，编写事件处理函数，AddFunc, UpdateFunc 和 DeleteFunc 等。如下图所示，informer 首先通过 list API 罗列资源，然后调用 watch API 监听资源的变更事件，并将结果放入到一个 FIFO 队列，队列的另一头有协程从中取出事件，并调用对应的注册函数处理事件。Informer 还维护了一个只读的 Map Store 缓存，主要为了提升查询的效率，降低 apiserver 的负载。

