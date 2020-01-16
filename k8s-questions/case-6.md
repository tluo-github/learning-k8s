# Redis迁移容器后Slowlog“异常”分析


### 问题描述

在某次Redis迁移容器后，DBA发来告警邮件，slowlog>500ms，同时在DBA的慢日志查询里可以看到有1800ms左右的日志，如下图1所示：

