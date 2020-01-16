# Kubernetes list-watch 机制


List-watch 是 K8S 统一的异步消息处理机制，保证了消息的实时性，可靠性，顺序性，性能等等，为声明式风格的 API 奠定了良好的基础，它是优雅的通信方式，是 K8S 架构的精髓。


Etcd 存储集群的数据信息，apiserver 作为统一入口，任何对数据的操作都必须经过 apiserver。客户端(kubelet/scheduler/ontroller-manager)通过 list-watch 监听 apiserver 中资源(pod/rs/rc 等等)的 create, update 和 delete 事件，并针对事件类型调用相应的事件处理函数。

![](/images/k8s-list-watch.jpg)


那么 list-watch 具体是什么呢，顾名思义，list-watch 有两部分组成，分别是 list 和 watch。list 非常好理解，就是调用资源的 list API 罗列资源，基于 HTTP 短链接实现；watch 则是调用资源的 watch API 监听资源变更事件，基于 HTTP 长链接实现，也是本文重点分析的对象。以 pod 资源为例，它的 list 和 watch API 分别为：

* List API，返回值为 PodList，即一组 pod。
```
GET /api/v1/pods
```
* Watch API，往往带上 watch=true，表示采用 HTTP 长连接持续监听 pod 相关事件，每当有事件来临，返回一个 WatchEvent。

```
GET /api/v1/watch/pods
```


