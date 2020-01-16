# 容器偶发性超时问题案例分析

案例来源: https://mp.weixin.qq.com/s/2OChXNiME4LaGt8BTWBpWQ

### 问题描述

某一天接到用户报障说，Redis集群有超时现象发生，比较频繁，而访问的QPS也比较低。紧接着，陆续有其他用户也报障Redis访问超时。在这些报障容器所在的宿主机里面，我们猛然发现有之前因为时钟漂移问题升级过内核到4.14.26的宿主机ServerA，心里突然有一丝不详的预感。


### 初步分析

因为现代软件部署结构的复杂性以及网络的不可靠性，我们很难快速定位“connect timout”或“connectreset by peer”之类问题的根因。

历史经验告诉我们，一般比较大范围的超时问题要么和交换机路由器之类的网络设备有关，要么就是底层系统不稳定导致的故障。从报障的情况来看，4.10和4.14的内核都有，而宿主机的机型也涉及到多个批次不同厂家，看上去没头绪，既然没什么好办法，那就抓个包看看吧。

![](/images/case-7-1.webp)
![](/images/case-7-2.webp)

图1是App端容器和宿主机的抓包，图2是Redis端容器和宿主机的抓包。因为APP和Redis都部署在容器里面（图3），所以一个完整请求的包是B->A->C->D，而Redis的回包是D->C->A->B。

![](/images/case-7-3.webp)

上面抓包的某一条请求如下：

1. B（图1第二个）的请求时间是21:25:04.846750 #99
2. 到达A（图1第一个）的时间是21:25:04.846764 #96
3. 到达C（图2第一个）的时间是21:25:07.432436 #103
4. 到达D（图2第二个）的时间是21:25:07.432446 #115

该请求从D回复如下：

1. D的回复时间是21:25:07.433248 #116
2. 到达C的时间是21:25:07.433257 #104
3. 到达A点时间是21:25:05:901108 #121
4. 到达B的时间是21:25:05:901114 #124

从这一条请求的访问链路我们可以发现，B在200ms超时后等不到回包。在21:25:05.055341重传了该包#100，并且可以从C收到重传包的时间#105看出，几乎与#103的请求包同时到达，也就是说该第一次的请求包在网络上被延迟传输了。大致的示意如下图4所示：

![](/images/case-7-4.webp)

从抓包分析来看，宿主机上好像并没有什么问题，故障在网络上。而我们同时在两边宿主机，容器里以及连接宿主机的交换机抓包，就变成了下面图5所示，假设连接A的交换机为E，也就是说A->E这段的网络有问题。
![](/images/case-7-5.webp)

### 陷入僵局

尽管发现A->E这段有问题，排查却也就此陷入了僵局，因为影响的宿主机遍布多个IDC，这也让我们排除了网络设备的问题。我们怀疑是否跟宿主机的某些TCP参数有关，比如TSO/GSO，一番测试后发现开启关闭TSO/GSO和修改内核参数对解决问题无效，但同时我们也观察到，从相同IDC里任选一台宿主机Ping有问题的宿主机，百分之几的概率看到很高的响应值，如下图6所示：

![](/images/case-7-6.webp)

同一个IDC内如此高的Ping响应延迟，很不正常。而这时DBA告诉我们，他们的某台物理机ServerB也有类似的的问题，Ping延迟很大，SSH上去后明显感觉到有卡顿，这无疑给我们解决问题带来了希望，但又更加迷惑：
1. 延迟好像跟内核版本没有关系，3.10，4.10，4.14的三个版本内核看上去都有这种问题。
2. 延迟和容器无关，因为延迟都在宿主机上到连接宿主机的交换机上发现的。
3. ServerB跟ServerA虽然表现一样，但细节上看有区别，我们的宿主机在重启后基本上都能恢复一段时间后再复现延迟，但ServerB重启也无效。

由此我们判断ServerA和ServerB的症状并不是同一个问题，并让ServerB先升级固件看看。在升级固件后ServerB恢复了正常，那么我们的宿主机是不是也可以靠升级固件修复呢？答案是并没有。升级固件后没过几天，延迟的问题又出现了。


### 意外发现

回过头来看之前为了排查Skylake时钟漂移问题的ServerA，上面一直有个简单的程序在运行，来统计时间漂移的值，将时间差记到文件中。当时这个程序是为了验证时钟漂移问题是否修复，如图7：


![](/images/case-7-7.webp)

这个程序跑在宿主机上，每个机器各有差异，但正常的时间差应该是100us以内，但1个多月后，时间差异越来越大，如图8，最大能达到几百毫秒以上。这告诉我们可能通过这无意中的log来找到根因，而且验证了上面3的这个猜想，宿主机是运行一段时间后逐渐出问题，表现为第一次打点到第二次打点之间，调度会自动delay第二次打点。


![](/images/case-7-8.webp)

### TSC和Perf

Turbostat是intel开发的，用来查看CPU状态以及睿频的工具，同样可以用来查看TSC的频率。

在有问题的宿主机上，TSC并不是恒定的，如图9所示，这个跟相关资料有出入，当然我们分析更可能的原因是，turbostat两次去取TSC的时候，被内核调度delay了，如果第一次取时被delay导致取的结果比实际TSC的值要小，而如果第二次取时被delay会导致取的结果比实际值要大。
![](/images/case-7-9.webp)

Perf是内置于Linux上的基于采样的性能分析工具，一般随着内核一起编译出来，具体的用法可以搜索相关资料，这里也不展开。用perf sched record -a sleep 60和perf sched latency -s max来查看Linux的调度延迟，发现最大能录得超过1s的延迟，如图10和图11所示。用户态的进程有时因为CPU隔离和代码问题导致比较大的延迟还好理解，但这些进程都是内核态的。尽管Linux的CFS调度并非实时的调度，但在负载很低的情况下超过1s的调度延迟也是匪夷所思的。

![](/images/case-7-10.webp)
![](/images/case-7-11.webp)

根据之前的打点信息和Turbostat以及Perf的数据，我们非常有理由怀疑是内核的调度有问题，这样我们就用基于RDTSCP指令更精准地来获取TSC值来检测CPU是否卡顿。RDTSCP指令不仅可以获得当前TSC的值，并且可以得到对应的CPU ID。如图12所示：

![](/images/case-7-12.webp)

上面的程序编译后，放在宿主机上依次绑核执行，我们发现在问题的宿主机上可以打印出比较大的TSC的值。每间隔100ms去取TSC的值，将获得的相减，在正常的宿主机上它的值应该跟CPU的TSC紧密相关，比如我们的宿主机上TSC是1.7GHZ的频率，那么0.1s它的累加值应该是170000000，正常获得的值应该是比170000000多一点，图13的第五条的值本身就说明了该宿主机的调度延迟在2s以上。
![](/images/case-7-13.webp)

### 真相大白

通过上面的检测手段，可以比较轻松地定位问题所在，但还是找不到根本原因。这时我们突然想起来，线上Redis大规模使用宿主机的内核4.14.67并没有类似的延迟，因此我们怀疑这种延迟问题是在4.14.26到4.14.67之间的bugfix修复掉了


查看commit记录，先二分查找大版本，再将怀疑的点单独拎出来patch测试，终于发现了这个4.14.36-4.14.37之间的（图14）commit：https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/?h=v4.14.37&id=b8d4055372b58aad4a51b67e176eabdcc238fde3。

```
x86/acpi: Prevent X2APIC id 0xffffffff from being accounted
commit 10daf10ab154e31237a8c07242be3063fb6a9bf4 upstream.

RongQing reported that there are some X2APIC id 0xffffffff in his machine's
ACPI MADT table, which makes the number of possible CPU inaccurate.

The reason is that the ACPI X2APIC parser has no sanity check for APIC ID
0xffffffff, which is an invalid id in all APIC types. See "Intel® 64
Architecture x2APIC Specification", Chapter 2.4.1.

Add a sanity check to acpi_parse_x2apic() which ignores the invalid id.

Reported-by: Li RongQing <lirongqing@baidu.com>
Signed-off-by: Dou Liyang <douly.fnst@cn.fujitsu.com>
Signed-off-by: Thomas Gleixner <tglx@linutronix.de>
Cc: stable@vger.kernel.org
Cc: len.brown@intel.com
Cc: rjw@rjwysocki.net
Cc: hpa@zytor.com
Link: https://lkml.kernel.org/r/20180412014052.25186-1-douly.fnst@cn.fujitsu.com
Signed-off-by: Greg Kroah-Hartman <gregkh@linuxfoundation.org>
```

从该commit的内容来看，修复了无效的apic id会导致possible CPU个数不正确的情况，那么什么是x2apic呢？什么又是possible CPU？怎么就导致了调度的延迟呢？

说到x2apic，就必须先知道apic，翻查网上的资料就发现，apic全称Local Advanced ProgrammableInterrupt Controller，是一种负责接收和发送中断的芯片，集成在CPU内部，每个CPU有一个属于自己的local apic。它们通过apic id进行区分。而x2apic是apic的增强版本，将apic id扩充到32位，理论上支持2^32-1个CPU。简单的说，操作系统通过apic id来确定CPU的个数。

而possible CPU则是内核为了支持CPU热插拔特性，在开机时一次载入，相应的就有online，offline CPU等，通过修改/sys/devices/system/cpu/cpu9/online可以动态关闭或打开一个CPU，但所有的CPU个数就是possible CPU，后续不再修改。

该commit指出，因为没有忽略apic id为0xffffffff的值，导致possible CPU不正确。此commit看上去跟我们的延迟问题找不到关联，我们也曾向该issue的提交者请教调度延迟的问题，对方也不清楚，只是表示在自己环境只能复现possible CPU增加4倍，vmstat的运行时间增加16倍。

这时我们查看有问题的宿主机CPU信息，奇怪的事情发生了，如图15所示，12核的机器上possbile CPU居然是235个，而其中12-235个是offline状态，也就是说真正工作的只有12个，这么说好像还是跟延迟没有关系。
![](/images/case-7-15.webp)

继续深入研究possbile CPU，我们发现了一些端倪。从内核代码来看，引用for_each_possible_cpu()这个函数的有600多处，遍布各个内核子模块，其中关键的核心模块比如vmstat，shed，以及loadavg等都有对它的大量调用。而这个函数的逻辑也很简单，就是遍历所有的possible CPU，那么对于12核的机器，它的执行时间是正常宿主机执行时间的将近20倍！该commit的作者也指出太多的CPU会浪费向量空间并导致BUG（https://lkml.org/lkml/2018/5/2/115），而BUG就是调度系统的缓慢延迟。


以下图16，图17是对相同机型相同厂商的两台空负载宿主机的kubelet的Perf数据（perf stat -p $pid sleep 60），图16是uptime 2天的，而图17是uptime 89天的。

![](/images/case-7-16.webp)
![](/images/case-7-17.webp)

我们一眼就看出图16的宿主机不正常，因为无论是CPU的消耗，上下文的切换速度，指令周期，都远劣于图17的宿主机，并且还在持续恶化，这就是宿主机延迟的根本原因。而图17宿主机恰恰只是图16的宿主机打上图14的patch后的内核，可以看到，possible CPU恢复正常（图18），至此超时问题的排查告一段落。
![](/images/case-7-18.webp)

我们排查发现，不是所有的宿主机，所有的内核都在此BUG的影响范围内，具体来说4.10（之前的有可能会有影响，但我们没有类似的内核，无法测试）-4.14.37（因为是stable分支，所以Master分支可能更靠后）的内核，CPU为skylake及以后型号的某些厂商的机型会触发这个BUG。

确认是否受影响也比较简单，查看possible CPU是否是真实CPU即可。

### 问题再现

随着内核升级到4.14.67，看上去延迟的问题彻底解决了，然而并没有，只是出现的更加缓慢。几周后，超时报障又找了过来，我们用Perf来分析，发现了一些异常。

如图19所示是一个空负载的宿主机升级内核后8天的Perf的数据，明显可以看到kworker的max delay已经100ms+，而这次有规律的是，延迟比较大的都是最后四个核，对于12核的节点就是8-11，并且都是同一D厂的宿主机。而上篇中使用新内核后用来验证解决问题的却不是D厂的宿主机，也就是说除了内核，还有其他的因素导致了延迟。

![](/images/case-7-19.webp)

### NUMA和CPU亲和性绑定

NUMA全称Non-Uniform Memory Access，NUMA服务器一般有多个节点，每个节点由多个CPU组成，并且具有独立的本地内存，节点内部使用共有的内存控制器，节点之间是通过内部互联（如QPI）进行通信。

然而，访问本地内存的速度远远高于访问其他节点的远地内存的速度，通常有几十倍的性能差异，这也正是NUMA名称的由来。因为NUMA的这个特性，为了更好地发挥系统性能，应用程序应该尽量减少不同节点CPU之间的信息交互。

无意中发现，D厂的机型与其他机型的NUMA配置不一样。假设12核的机型，D厂的机型NUMA节点分配如下图20所示：
![](/images/case-7-20.webp)
而其他厂家的机型NUMA节点分配如下图21所示：
![](/images/case-7-21.webp)

为什么会出现delay都是最后四个核上的进程呢？


经过深入排查才发现，原来相关同事之前为了让Kubernetes的相关进程和普通的用户的进程相隔离，设置了CPU的亲和性，让Kubernetes的相关进程绑定到宿主机的最后四个核上，用户的进程绑定到其他的核上，但后面这一步并没有生效。


还是拿12核的例子来说，上面的Kubernetes是绑定到8-11核，但用户的进程还是工作在0-11核上，更重要的是，最后4个核在遇到D厂家的这种机型时，实际上是跨NUMA绑定，导致了延迟越来越高，而用户进程运行在相关的核上就会导致超时等各种故障。

确定问题后，解决起来就简单了。将所有宿主机的绑核去掉，延迟就消失了，以下图是D厂的机型去掉绑核后开机26天Perf的调度延迟，从数据上看一切都恢复正常。
![](/images/case-7-22.webp)

### 新的问题

大约过了几个月，又有新的超时问题找到我们。有了之前的排查经验，我们觉得这次肯定能很轻易的定位到问题，然而现实无情地给予了我们当头一棒，4.14.67内核的宿主机，还是有大量无规律超时。

### 深入分析

Perf看调度延迟，如图23所示，调度延迟比较大但并没有集中在最后四个核上，完全无规律，同样Turbostat依然观察到TSC的频率在跳动。
![](/images/case-7-23.webp)

在各种猜想和验证都被一一证否后，我们开始挨个排除来做测试：
1. 我们将某台A宿主机实例迁移走，Perf看上去恢复了正常，而将实例迁移回来，延迟又出现了。
2. 另外一台B宿主机上，我们发现即使将所有的实例都清空，Perf依然能录得比较高的延迟。
3. 而与B相连编号同一机型的C宿主机迁移完实例后重启，Perf恢复正常。这时候看B的TOP，只有一个kubelet在消耗CPU，将这台宿主机上的kubelet停掉，Perf正常，开启kubelet后，问题又依旧。

这样我们基本可以确定kubelet的某些行为导致了宿主机卡顿和实例超时，对比正常/非正常的宿主机kubelet日志，每秒钟都在获取所有实例的监控信息，在非正常的宿主机上，会打印以下的日志。如图24所示：
![](/images/case-7-24.webp)

而在正常的宿主机上没有该日志或者该时间比较短，如图25所示：
![](/images/case-7-25.webp)

到这里，我们怀疑这些LOG的行为可能指向了问题的根源。查看Kubernetes代码，可以知道在获取时间超过指定值longHousekeeping （100ms）后，Kubernetes会记录这一行为，而updateStats即获取本地的一些监控数据，如图26代码所示：

```
func (c *containerData) housekeepingTick(timer <-chan time.Time, longHousekeeping time.Duration) bool {
	select {
	case <-c.stop:
		// Stop housekeeping when signaled.
		return false
	case finishedChan := <-c.onDemandChan:
		// notify the calling function once housekeeping has completed
		defer close(finishedChan)
	case <-timer:
	}
	start := c.clock.Now()
	err := c.updateStats()
	if err != nil {
		if c.allowErrorLogging() {
			klog.Warningf("Failed to update stats for container \"%s\": %s", c.info.Name, err)
		}
	}
	// Log if housekeeping took too long.
	duration := c.clock.Since(start)
	if duration >= longHousekeeping {
		klog.V(3).Infof("[%s] Housekeeping took %s", c.info.Name, duration)
	}
	c.notifyOnDemand()
	c.statsLastUpdatedTime = c.clock.Now()
	return true
}
```

在网上搜索相关issue，问题指向cAdvisor的消耗CPU过高：

* https://github.com/kubernetes/kubernetes/issues/15644
* https://github.com/google/cadvisor/issues/1498

https://github.com/google/cadvisor/issues/1774 指出 echo2 > /proc/sys/vm/drop_caches

可以暂时解决这种问题。我们尝试在有问题的机器上执行清除缓存的指令，超时问题就消失了，如图27所示。而从根本上解决这个问题，还是要减少取Metrics的频率，比较耗时的Metrics干脆不取或者完全隔离Kubernetes的进程和用户进程才可以。

![](/images/case-7-27.webp)

### 硬件故障

在排查cAdvisor导致的延迟的过程中，还发现一部分用户报障的超时，并不是cAdvisor导致的，主要表现在没有Housekeeping的日志，并且Perf结果看上去完全正常，说明没有调度方面的延迟，但从TSC的获取上还是能观察到异常。


由此我们怀疑这是另一种全新的故障，最重要的是我们将某台宿主机所有业务完全迁移掉，并关闭所有可以关闭的进程，只保留内核的进程后，TSC依然不正常并伴随肉眼可见的卡顿，而这种现象跟之前DBA那台物理机卡顿的问题很相似，这告诉我们很有可能是硬件方面的问题。


从以往的排障经验来看，TSC抖动程度对于我们排查宿主机是否稳定有重要的参考作用。这时我们决定将TSC的检测程序做成一个系统服务，每100ms去取一次系统的TSC值，将TSC的差值大于指定值打印到日志中，并采集单位时间的异常条目数和最大TSC差值，放在监控系统上，来观察异常的规律。如图28所示。

![](/images/case-7-28.webp)

恰好TSC检测的服务上线不久，一次明显的故障说明了它检测宿主机是否稳定的有效性。如图29，在某日8点多时，一台宿主机TSC突然升高，与应用的告警邮件和用户报障同一时刻到来。如图30所示：

![](/images/case-7-29.webp)
![](/images/case-7-30.webp)

将采集的日志这样展示后，我们一眼就发现问题都集中在某几批次的同一厂商的宿主机上，并且我们找到之前DBA卡顿的物理机，也是这几批次中的一台。我们收集了几台宿主机的日志详情，反馈给厂商后，确认是硬件故障，无规律并且随时可能触发，需升级BIOS，如图31厂商技术人员答复的邮件所示，至此问题得到最终解决。

![](/images/case-7-31.webp)

### 总结

本篇文章基本上描述了我们遇到的容器偶发性超时问题分析的大部分过程，但排障过程远比写出来要艰难。

总的原则还是大胆假设，小心求证，设法找到无规律中的规律性，保持细致耐心的钻研精神，相信这些疑难杂症终会被一一解决。

























































