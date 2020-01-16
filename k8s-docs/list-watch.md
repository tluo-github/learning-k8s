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
![](/images/k8s-list-watch-1.jpg)

1. 首先需要强调一点，list或者watch的数据，均是来自于etcd的数据，因此在Apiserver中，一切的设计都是为了获取最新的etcd数据并返回给client。
2. 当Apiserver监听到各组件发来的watch请求时，由于list和watch请求的格式相似，先进入ListResource函数进行分析，若解析为watch请求，便会创建一个watcher结构来响应请求。watcher的生命周期是每个http请求的。

```
//每一个Watch请求对应一个watcher结构
func (a *APIInstaller) registerResourceHandlers(path string, storage rest.Storage,... ...
...
lister, isLister := storage.(rest.Lister)
watcher, isWatcher := storage.(rest.Watcher) ...(1) ... case "LIST": // List all resources of a kind.
...
```
3. 创建了watcher，但谁来接收并缓存etcd的数据呢？Apiserver使用cacher来接收etcd的事件，cacher也是Storage类型，这里cacher可以理解为是监听etcd的一个实例，cacher针对于某个类型的数据，其cacher通过ListAndWatch()这个方法，向etcd发送watch请求。etcd会将某一类型的数据同步到watchCache这个结构，也就是说，ListAndWatch()将远端数据源源不断同步到cacher结构中来。Cacher的结构如下所示：
```
type Cacher struct {
 incomingHWM storage.HighWaterMark
 incoming chan watchCacheEvent
 sync.RWMutex
 // Before accessing the cacher's cache, wait for the ready to be ok.
 // This is necessary to prevent users from accessing structures that are
 // uninitialized or are being repopulated right now.
 // ready needs to be set to false when the cacher is paused or stopped.
 // ready needs to be set to true when the cacher is ready to use after
 // initialization.
 ready *ready
 // Underlying storage.Interface.
 storage storage.Interface
 // Expected type of objects in the underlying cache.
 objectType reflect.Type
 // "sliding window" of recent changes of objects and the current state.
 watchCache *watchCache
 reflector *cache.Reflector
 // Versioner is used to handle resource versions.
 versioner storage.Versioner
 // newFunc is a function that creates new empty object storing a object of type Type.
 newFunc func() runtime.Object
 // indexedTrigger is used for optimizing amount of watchers that needs to process
 // an incoming event.
 indexedTrigger *indexedTriggerFunc
 // watchers is mapping from the value of trigger function that a
 // watcher is interested into the watchers
 watcherIdx int
 watchers indexedWatchers
 // Defines a time budget that can be spend on waiting for not-ready watchers
 // while dispatching event before shutting them down.
 dispatchTimeoutBudget *timeBudget
 // Handling graceful termination.
 stopLock sync.RWMutex
 stopped bool
 stopCh chan struct{}
 stopWg sync.WaitGroup
 clock clock.Clock
 // timer is used to avoid unnecessary allocations in underlying watchers.
 timer *time.Timer
 // dispatching determines whether there is currently dispatching of
 // any event in flight.
 dispatching bool
 // watchersBuffer is a list of watchers potentially interested in currently
 // dispatched event.
 watchersBuffer []*cacheWatcher
 // blockedWatchers is a list of watchers whose buffer is currently full.
 blockedWatchers []*cacheWatcher
 // watchersToStop is a list of watchers that were supposed to be stopped
 // during current dispatching, but stopping was deferred to the end of
 // dispatching that event to avoid race with closing channels in watchers.
 watchersToStop []*cacheWatcher
 // Maintain a timeout queue to send the bookmark event before the watcher times out.
 bookmarkWatchers *watcherBookmarkTimeBuckets
 // watchBookmark feature-gate
 watchBookmarkEnabled bool
} 
```

watchCache的结构如下所示：

```
type watchCache struct {
 sync.RWMutex //同步锁
 cond *sync.Cond //条件变量
 capacity int//历史滑动窗口容量
 keyFunc func(runtime.Object) (string, error)//从storage中获取键值
 getAttrsFunc func(runtime.Object) (labels.Set, fields.Set, bool, error)//获取一个对象的field和label信息
 cache []watchCacheElement//循环队列缓存
 startIndex int//循环队列的起始下标
 endIndex int//循环队列的结束下标
 store cache.Store//
 resourceVersion uint64
 onReplace func()
 onEvent func(*watchCacheEvent)//在每次缓存中的数据发生Add/Update/Delete后都会调用该函数，来获取对象的之前版本的值
 clock clock.Clock
 versioner storage.Versioner
}

```
cache里面存放的是所有操作事件，而store中存放的是当前最新的事件。
4. cacheWatcher从watchCache中拿到从某个resourceVersion以来的所有数据，即initEvents，然后将数据放到input这个channel里面去，通过filter然后输出到result这个channel里面，返回数据到某个client。
```
type cacheWatcher struct {
 sync.Mutex//同步锁
 input chan *watchCacheEvent//输入管道,Apiserver都事件发生时都会通过广播的形式向input管道进行发送
 result chan watch.Event//输出管道，输出到update管道中去
 done chan struct{}
 filter filterWithAttrsFunc//过滤器
 stopped bool
 forget func(bool)
 versioner storage.Versioner
}
```















