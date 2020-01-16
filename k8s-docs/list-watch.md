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

#### Watch 是如何实现的

List的实现容易理解，那么Watch是如何实现的呢？Watch是如何通过HTTP 长链接接收apiserver发来的资源变更事件呢？

秘诀就是Chunked transfer encoding(分块传输编码)，它首次出现在HTTP/1.1。正如维基百科所说：

> HTTP 分块传输编码允许服务器为动态生成的内容维持 HTTP 持久链接。通常，持久链接需要服务器在开始发送消息体前发送Content-Length消息头字段，但是对于动态生成的内容来说，在内容创建完之前是不可知的。使用分块传输编码，数据分解成一系列数据块，并以一个或多个块发送，这样服务器可以发送数据而不需要预先知道发送内容的总大小。

当客户端调用watch API时，apiserver 在response的HTTP Header中设置Transfer-Encoding的值为chunked，表示采用分块传输编码，客户端收到该信息后，便和服务端维持该链接，并等待下一个数据块，即资源的事件信息。例如：

```
$ curl -i http://{kube-api-server-ip}:8080/api/v1/watch/pods?watch=yes
HTTP/1.1 200 OK
Content-Type: application/json
Transfer-Encoding: chunked
Date: Thu, 02 Jan 2019 20:22:59 GMT
Transfer-Encoding: chunked

{"type":"ADDED", "object":{"kind":"Pod","apiVersion":"v1",...}}
{"type":"ADDED", "object":{"kind":"Pod","apiVersion":"v1",...}}
{"type":"MODIFIED", "object":{"kind":"Pod","apiVersion":"v1",...}}
```

#### list-watch机制

kubernetes没有像其他分布式系统中额外引入MQ，是因为其设计理念采用了level trigger而非edge trigger。其仅仅通过http+protobuffer的方式，实现list-watcher机制来解决各组件间的消息通知。因此，在了解各组件通信前，必须先了解list-watch机制在kubernetes的应用。

List-watch是k8s统一的异步消息处理机制，list通过调用资源的list API罗列资源，基于HTTP短链接实现；watch则是调用资源的watch API监听资源变更事件，基于HTTP长链接实现。在kubernetes中，各组件通过监听Apiserver的资源变化，来更新资源状态。

这里对watch简要说明，流程如下图所示：






