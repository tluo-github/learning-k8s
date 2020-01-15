# 记一次KUBERNETES/DOCKER网络排障

**案例来源:** https://coolshell.cn/articles/18654.html

** 问题的症状:**
用户直接在微信里说，他们发现在Kuberbnetes下的某个pod被重启了几百次甚至上千次，于是开启调查这个pod，发现上面的服务时而能够访问，时而不能访问，也就是有一定概率不能访问，不知道是什么原因。而且并不是所有的pod出问题，而只是特定的一两个pod出了网络访问的问题。用户说这个pod运行着Java程序，为了排除是Java的问题，用户用 **docker exec -it** 命令直接到容器内启了一个 Python的 SimpleHttpServer来测试发现也是一样的问题。


![](/images/case-2-tcpdump.png)