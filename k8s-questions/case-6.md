# Redis迁移容器后Slowlog“异常”分析

来源: 
### 问题描述

在某次Redis迁移容器后，DBA发来告警邮件，slowlog>500ms，同时在DBA的慢日志查询里可以看到有1800ms左右的日志，如下图1所示：
![](/images/case-6-1.webp)

#### 二、 分析过程

#### 2.1 什么是slowlog

在分析问题之前，先简单解释下Redis的slowlog。阅读Redis源码（图)2:
![](/images/case-6-2.webp)


不难发现，当某次Redis的操作大于配置中slowlog-log-slower-than设置的值时，Redis就会将该值记录到内存中，通过slowlog get可以获取该次slowlog发生的时间和耗时，图1的监控数据也是从此获得。也就是说，slowlog只是单纯的计算Redis执行的耗时时间，与其他因素如网络之类的都没关系。


#### 2.2 矛盾的日志

每次slowlog都是1800+ms并且都随机出现，在第一批次Redis容器化的宿主机上完全没有这种现象，而QPS远小于第一批次迁移的某些集群，按常理很难解释，这时候翻看CAT记录，更加加重了我们的疑惑，见图3：

