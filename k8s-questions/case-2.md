# 记一次KUBERNETES/DOCKER网络排障

**案例来源:** https://coolshell.cn/articles/18654.html

** 问题的症状:**
用户直接在微信里说，他们发现在Kuberbnetes下的某个pod被重启了几百次甚至上千次，于是开启调查这个pod，发现上面的服务时而能够访问，时而不能访问，也就是有一定概率不能访问，不知道是什么原因。而且并不是所有的pod出问题，而只是特定的一两个pod出了网络访问的问题。用户说这个pod运行着Java程序，为了排除是Java的问题，用户用 **docker exec -it** 命令直接到容器内启了一个 Python的 SimpleHttpServer来测试发现也是一样的问题。

** 问题初查 **

首先，我们排除了flannel的问题，因为整个集群的网络通信都正常，只有特定的某一两个pod有问题。而用 **telnet ip port** 的命令手工测试网络连接时有很大的概率出现 **connection refused** 错误，大约 1/4的概率，而3/4的情况下是可以正常连接的。

当时，我们让用户抓个包看看，然后，用户抓到了有问题的TCP连接是收到了 SYN 后，立即返回了 **RST, ACK**

![](/images/case-2-tcpdump.png)

我问一下用户这两个IP所在的位置，知道了，**10.233.14.129** 是 **docker0**，**10.233.14.145** 是容器内的IP。所以，这基本上可以排除了所有和kubernets或是flannel的问题，这就是本地的Docker上的网络的问题。

对于这样被直接 Reset 的情况，在 **telnet** 上会显示 **connection refused** 的错误信息，对于我个人的经验，这种 **SYN** 完直接返回 **RST, ACK** 的情况只会有三种情况：

1. TCP链接不能建立，不能建立连接的原因基本上是标识一条TCP链接的那五元组不能完成，绝大多数情况都是服务端没有相关的端口号。
2. TCP链接建错误，有可能是因为修改了一些TCP参数，尤其是那些默认是关闭的参数，因为这些参数会导致TCP协议不完整。
3. 有防火墙iptables的设置，其中有 **REJECT** 规则。(也就是一条 ICMP 回复)

** 重新梳理 **

用户提供了一个比较关键的信息—— “抓包这个事，在 **docker0** 上可以抓到，然而到了容器内抓不到容器返回 RST, ACK ” ！然而，根据我的知识，我知道在 **docker0 ** 和容器内的 **veth** 网卡上，中间再也没有什么网络设备了

于是这个事把我们逼到了最后一种情况 —— IP地址冲突了！

Linux下看IP地址冲突还不是一件比较简单事的，而在用户的生产环境下没有办法安装一些其它的命令，所以只能用已有的命令，这个时候，我们发现用户的机器上有 **arping** 于是我们用这个命令来检测有没有冲突的IP地址。使用了下面的命令：

```shell
$ arping -D -I docker0 -c 2 10.233.14.145
$ echo $?
```

根据文档，**-D** 参数是检测IP地址冲突模式，如果这个命令的退状态是 **0** 那么就有冲突。结果返回了 **1** 。而且，我们用 **arping** IP的时候，没有发现不同的mac地址。 这个时候，似乎问题的线索又断了。

** 柳暗花明 **

现在我们知道，IP冲突的可能性是非常大的，但是我们找不出来是和谁的IP冲突了。而且，我们知道只要把这台机器重启一下，问题一定就解决掉了，但是我们觉得这并不是解决问题的方式，因为重启机器可以暂时的解决掉到这个问题，而如果我们不知道这个问题怎么发生的，那么未来这个问题还会再来。而重启线上机器这个成本太高了。


于是，我们的好奇心驱使我们继续调查。我让用户 **kubectl delete** 其中两个有问题的pod，因为本来就服务不断重启，所以，删掉也没有什么问题。删掉这两个pod后（一个是IP为 **10.233.14.145** 另一个是 **10.233.14.137**），我们发现，kubernetes在其它机器上重新启动了这两个服务的新的实例。然而，在问题机器上，这两个IP地址居然还可以ping得通。

好了，IP地址冲突的问题可以确认了。因为**10.233.14.xxx** 这个网段是 docker 的，所以，这个IP地址一定是在这台机器上。所以，我们想看看所有的 network namespace 下的 veth 网卡上的IP。

* 首先，我们到 /var/run/netns目录下查看系统的network namespace，发现什么也没有。
* 然后，我们到 /var/run/docker/netns 目录下查看Docker的namespace，发现有好些。
* 于是，我们用指定位置的方式查看Docker的network namespace里的IP地址

这里要动用 **nsenter** 命令，这个命令可以进入到namespace里执行一些命令。比如

```shell
nsenter --net=/var/run/docker/netns/421bdb2accf1 ifconfig -a

```

上述的命令，到 var/run/docker/netns/421bdb2accf1 这个network namespace里执行了 ifconfig -a 命令。于是我们可以用下面 命令来遍历所有的network namespace。

```
$ ls /var/run/docker/netns | xargs -I {} nsenter --net=/var/run/docker/netns/{} ip addr 
```

然后，我们发现了比较诡异的事情。

* **10.233.14.145** 我们查到了这个IP，说明，docker的namespace下还有这个IP。

* **10.233.14.137**，这个IP没有在docker的network namespace下查到。


有namespace leaking？于是我上网查了一下，发现了一个docker的bug – 在docker remove/stop 一个容器的时候，没有清除相应的network namespace，这个问题被报告到了 [Issue#31597](https://github.com/moby/moby/issues/31597) 然后被fix在了 [PR#31996](https://github.com/moby/moby/pull/31996)，并Merge到了 Docker的 17.05版中。而用户的版本是 17.09，应该包含了这个fix。不应该是这个问题，感觉又走不下去了。

不过， **10.233.14.137** 这个IP可以ping得通，说明这个IP一定被绑在某个网卡，而且被隐藏到了某个network namespace下。



到这里，要查看所有network namespace，只有最后一条路了，那就是到 **/proc/** 目录下，把所有的pid下的 **/proc/<pid>/ns** 目录给穷举出来。好在这里有一个比较方便的命令可以干这个事 ： **lsns**
```
$ lsns -t net | awk ‘{print $4}' | xargs -t -I {} nsenter -t {}&nbsp;-n ip addr | grep -C 4 "10.233.14.137"
```

* lsns -t net 列出所有开了network namespace的进程，其第4列是进程PID
* 把所有开过network namespace的进程PID拿出来，转给 xargs 命令
* 由 xargs 命令把这些PID 依次传给 nsenter 命令，
  * xargs -t 的意思是会把相关的执行命令打出来，这样我知道是那个PID。
  * xargs -I {}  是声明一个占位符来替换相关的PID
  
最后，我们发现，虽然在 **/var/run/docker/netns** 下没有找到 **10.233.14.137** ，但是在 **lsns** 中找到了三个进程，他们都用了**10.233.14.137** 这个IP（冲突了这么多），而且他们的MAC地址全是一样的！（怪不得arping找不到）。通过**ps** 命令，可以查到这三个进程，有两个是java的，还有一个是**/pause** （这个应该是kubernetes的沙盒）。

我们继续乘胜追击，穷追猛打，用**pstree**命令把整个进程树打出来。发现上述的三个进程的父进程都在多个同样叫 **docker-contiane** 的进程下！

**这明显还是docker的，但是在docker ps 中却找不道相应的容器，什么鬼！快崩溃了……**

继续看进程树，发现，这些 **docker-contiane** 的进程的父进程不在 **dockerd** 下面，而是在 **systemd** 这个超级父进程PID 1下，我靠！进而发现了一堆这样的野进程（这种野进程或是僵尸进程对系统是有害的，至少也是会让系统进入亚健康的状态，因为他们还在占着资源）。


**docker-contiane** 应该是 **dockerd** 的子进程，被挂到了 **pid 1** 只有一个原因，那就是父进程“飞”掉了，只能找 pid 1 当养父。这说明，这台机器上出现了比较严重的 dockerd 进程退出的问题，而且是非常规的，因为 systemd 之所以要成为 pid 1，其就是要监管所有进程的子子孙孙，居然也没有管理好，说明是个非常规的问题。（注，关于 systemd，请参看《Linux PID 1 和 Systemd 》，关于父子进程的事，请参看《Unix高级环境编程》一书）


接下来就要看看 **systemd** 为 **dockerd** 记录的日志了…… （然而日志只有3天的了，这3天dockerd没有任何异常）

**问题原因**
这两天在自己的环境下测试了一下，发现，只要是通过 **systemctl start/stop docker ** 这样的命令来启停 Docker， 是可以把所有的进程和资源全部干掉的。这个是没有什么问题的。我唯一能重现用户问题的的操作就是直接 ** kill -9 <dockerd pid>**  但是这个事用户应该不会干。而 Docker 如果有 crash 事件时，Systemd 是可以通过 **journalctl -u docker** 这样的命令查看相关的系统日志的。


于是，我找用户了解一下他们在**Docker**在启停时的问题，用户说，他们的执行 **systemctl stop docker **这个命令的时候，发现这个命令不响应了，有可能就直接按了 **Ctrl +C** 了！


这个应该就是导致大量的 **docker-containe **进程挂到 **PID 1** 下的原因了。前面说过，用户的一台物理机上运行着上百个容器，所以，那个进程树也是非常庞大的，我想，停服的时候，系统一定是要遍历所有的docker子进程来一个一个发退出信号的，这个过程可能会非常的长。导致操作员以为命令假死，而直接按了 Ctrl + C ，最后导致很多容器进程并没有终止……

**其他事宜**

有同学问，为什么我在这个文章里写的是 **docker-containe** 而不是 **containd** 进程？这是因为被 **pstree** 给截断了，用 **ps** 命令可以看全，只是进程名的名字有一个 **docker-**的前缀。

下面是这两种不同安装包的进程树的差别（其中 **sleep** 是我用 **buybox** 镜像启动的）

CENTOS 系统安装包
```
systemd───dockerd─┬─docker-contained─┬─3*[docker-contained-shim─┬─sleep]
                  │                 │                    └─9*[{docker-containe}]]
                  │                 ├─docker-contained-shim─┬─sleep
                  │                 │                 └─10*[{docker-containe}]
                  │                 └─14*[{docker-contained-shim}]
                  └─17*[{dockerd}]
```

DOCKER 官方安装包
```
systemd───dockerd─┬─containerd─┬─3*[containerd-shim─┬─sleep]
                  │            │                 └─9*[{containerd-shim}]
                  │            ├─2*[containerd-shim─┬─sleep]
                  │            │                    └─9*[{containerd-shim}]]
                  │            └─11*[{containerd}]
                  └─10*[{dockerd}]
```

顺便说一下，自从 Docker 1.11版以后，Docker进程组模型就改成上面这个样子了.

* **dockerd** 是 Docker Engine守护进程，直接面向操作用户。**dockerd** 启动时会启动 **containerd** 子进程，他们之前通过RPC进行通信。
* **containerd** 是**dockerd**和**runc**之间的一个中间交流组件。他与 **dockerd** 的解耦是为了让Docker变得更为的中立，而支持OCI 的标准 。
* **containerd-shim**  是用来真正运行的容器的，每启动一个容器都会起一个新的**shim**进程， 它主要通过指定的三个参数：容器id，boundle目录（containerd的对应某个容器生成的目录，一般位于：/var/run/docker/libcontainerd/containerID）， 和运行命令（默认为 runc）来创建一个容器。



### 总结
1. 对于问题调查，需要比较扎实的基础知识，知道问题的成因和范围。
2. 如果走不下去了，要重新梳理一下，回头仔细看一下一些蛛丝马迹，认真推敲每一个细节。
3. 各种诊断工具要比较熟悉，这会让你事半功倍。
4. 系统维护和做清洁比较类似，需要经常看看系统中是否有一些僵尸进程或是一些垃圾东西，这些东西要及时清理掉。
















