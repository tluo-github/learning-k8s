# 神秘的溢出与丢包

案例来源:https://imroc.io/posts/kubernetes-overflow-and-drop/

#### 一、问题描述
有用户反馈大量图片加载不出来。

图片下载走的 k8s ingress，这个 ingress 路径对应后端 service 是一个代理静态图片文件的 nginx deployment，这个 deployment 只有一个副本，静态文件存储在 nfs 上，nginx 通过挂载 nfs 来读取静态文件来提供图片下载服务，所以调用链是：

```
client –> k8s ingress –> nginx –> nfs

```

#### 二、猜测

猜测: ingress 图片下载路径对应的后端服务出问题了。

验证：在 k8s 集群直接 curl nginx 的 pod ip，发现不通，果然是后端服务的问题！

#### 三、抓包

继续抓包测试观察，登上 nginx pod 所在节点，进入容器的 netns 中：

```

# 拿到 pod 中 nginx 的容器 id
$ kubectl describe pod tcpbench-6484d4b457-847gl | grep -A10 "^Containers:" | grep -Eo 'docker://.*$' | head -n 1 | sed 's/docker:\/\/\(.*\)$/\1/'
49b4135534dae77ce5151c6c7db4d528f05b69b0c6f8b9dd037ec4e7043c113e

# 通过容器 id 拿到 nginx 进程 pid
$ docker inspect -f {{.State.Pid}} 49b4135534dae77ce5151c6c7db4d528f05b69b0c6f8b9dd037ec4e7043c113e
3985

# 进入 nginx 进程所在的 netns
$ nsenter -n -t 3985

# 查看容器 netns 中的网卡信息，确认下
$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
3: eth0@if11: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 56:04:c7:28:b0:3c brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.26.0.8/26 scope global eth0
       valid_lft forever preferred_lft foreve

```

使用 tcpdump 指定端口 24568 抓容器 netns 中 eth0 网卡的包:
```
tcpdump -i eth0 -nnnn -ttt port 24568
```

在其它节点准备使用 nc 指定源端口为 24568 向容器发包：
```
nc -u 24568 172.16.1.21 80
```

观察抓包结果：

```
00:00:00.000000 IP 10.0.0.3.24568 > 172.16.1.21.80: Flags [S], seq 416500297, win 29200, options [mss 1424,sackOK,TS val 3000206334 ecr 0,nop,wscale 9], length 0
00:00:01.032218 IP 10.0.0.3.24568 > 172.16.1.21.80: Flags [S], seq 416500297, win 29200, options [mss 1424,sackOK,TS val 3000207366 ecr 0,nop,wscale 9], length 0
00:00:02.011962 IP 10.0.0.3.24568 > 172.16.1.21.80: Flags [S], seq 416500297, win 29200, options [mss 1424,sackOK,TS val 3000209378 ecr 0,nop,wscale 9], length 0
00:00:04.127943 IP 10.0.0.3.24568 > 172.16.1.21.80: Flags [S], seq 416500297, win 29200, options [mss 1424,sackOK,TS val 3000213506 ecr 0,nop,wscale 9], length 0
00:00:08.192056 IP 10.0.0.3.24568 > 172.16.1.21.80: Flags [S], seq 416500297, win 29200, options [mss 1424,sackOK,TS val 3000221698 ecr 0,nop,wscale 9], length 0
00:00:16.127983 IP 10.0.0.3.24568 > 172.16.1.21.80: Flags [S], seq 416500297, win 29200, options [mss 1424,sackOK,TS val 3000237826 ecr 0,nop,wscale 9], length 0
00:00:33.791988 IP 10.0.0.3.24568 > 172.16.1.21.80: Flags [S], seq 416500297, win 29200, options [mss 1424,sackOK,TS val 3000271618 ecr 0,nop,wscale 9], length 0
```
SYN 包到容器内网卡了，但容器没回 ACK，像是报文到达容器内的网卡后就被丢了。看样子跟防火墙应该也没什么关系，也检查了容器 netns 内的 iptables 规则，是空的，没问题。

排除是 iptables 规则问题，在容器 netns 中使用 **netstat -s** 检查下是否有丢包统计:
```
$ netstat -s | grep -E 'overflow|drop'
    12178939 times the listen queue of a socket overflowed
    12247395 SYNs to LISTEN sockets dropped
```
果然有丢包，为了理解这里的丢包统计，我深入研究了一下，下面插播一些相关知识。

#### 四、半连接与全连接队列
Linux 进程监听端口时，内核会给它对应的 socket 分配两个队列：

* **syn queue:** 半连接队列。server 收到 SYN 后，连接会先进入 SYN_RCVD 状态，并放入 syn queue，此队列的包对应还没有完全建立好的连接（TCP 三次握手还没完成）。
* **accept queue:** 全连接队列。当 TCP 三次握手完成之后，连接会进入 ESTABELISHED 状态并从 syn queue 移到 accept queue，等待被进程调用 accept() 系统调用 “拿走”。

> 注意：这两个队列的连接都还没有真正被应用层接收到，当进程调用 accept() 后，连接才会被应用层处理，具体到我们这个问题的场景就是 nginx 处理 HTTP 请求。

为了更好理解，可以看下这张 TCP 连接建立过程的示意图：
![](/images/case-4-backlog.png)

#### 五、listen 与 accept
不管使用什么语言和框架，在写 server 端应用时，它们的底层在监听端口时最终都会调用**listen()**系统调用，处理新请求时都会先调用**accept()**系统调用来获取新的连接，然后再处理请求，只是有各自不同的封装而已，以 go 语言为例：

```
// 调用 listen 监听端口
l, err := net.Listen("tcp", ":80")
if err != nil {
  panic(err)
}
for {
  // 不断调用 accept 获取新连接，如果 accept queue 为空就一直阻塞
  conn, err := l.Accept()
  if err != nil {
    log.Println("accept error:", err)
    continue
    }
  // 每来一个新连接意味着一个新请求，启动协程处理请求
  go handle(conn)
}
```

#### 六、Linux 的 backlog
内核既然给监听端口的 socket 分配了 syn queue 与 accept queue 两个队列，那它们有大小限制吗？可以无限往里面塞数据吗？当然不行！资源是有限的，尤其是在内核态，所以需要限制一下这两个队列的大小。

那么它们的大小是如何确定的呢？我们先来看下 listen 这个系统调用:

```
int listen(int sockfd, int backlog)

```
可以看到，能够传入一个整数类型的 backlog 参数，我们再通过**man listen**看下解释：

The behavior of the backlog argument on TCP sockets changed with Linux 2.2. Now it specifies the queue length for completely established sockets waiting to be accepted, instead of the number of incomplete connection requests. The maximum length of the queue for incomplete sockets can be set using /proc/sys/net/ipv4/tcp_max_syn_backlog. When syncookies are enabled there is no logical maximum length and this setting is ignored. See tcp(7) for more information.

If the backlog argument is greater than the value in /proc/sys/net/core/somaxconn, then it is silently truncated to that value; the default value in this file is 128. In kernels before 2.4.25, this limit was a hard coded value, SOMAXCONN, with the value 128.


