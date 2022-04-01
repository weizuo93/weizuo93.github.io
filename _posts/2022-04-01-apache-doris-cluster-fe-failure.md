---
layout: post
title: 记一次 Doris 集群FE故障处理过程
categories: [Apache Doris]
description:  记一次 Doris 集群FE故障处理过程
keywords: Apache Doris、元数据
---

## 记一次 Doris 集群FE故障处理过程

Doris版本：0.13.11

Doris集群中有3个FE节点（分别是fe01、fe02和fe03，fe01的角色是Master，fe02和fe03的角色是Follower），因为故障导致fe03挂掉，并且重启失败。我们便将fe03从集群中drop掉：
```
ALTER SYSTEM DROP FOLLOWER "fe03:edit_log_port";
```
清理fe03上的元数据并将其作为新的Follower节点重新加入集群。
```
ALTER SYSTEM ADD FOLLOWER "fe03:edit_log_port";
```
启动fe03，并通过`--helper fe01`参数指向Master节点。此时，fe03启动失败，并且作为Master的fe01也挂掉了，fe02也出现了异常。

此时集群3个FE节点均不可访问，集群瘫痪不可用。



为了恢复服务，我们对fe01和fe02的元数据进行备份，并执行以下操作：

* 将fe01的参数`metadata_failure_recovery`设置为true，重启fe01：
```
metadata_failure_recovery=true
```
这个时候，fe01会以Master的角色启动，加载本地元数据，并且不会向其他节点同步元数据。fe01成功启动之后，我们通过mysql客户端连接fe01，发现fe01状态正常，角色为Master，但fe02和fe03的状态是异常的。

* 将fe02和fe03从集群中drop掉，并清空元数据目录。
```
ALTER SYSTEM DROP FOLLOWER "fe02:edit_log_port";
ALTER SYSTEM DROP FOLLOWER "fe03:edit_log_port";
```
此时，集群中只剩fe01一个FE，并且角色为Master。

* 将fe01的参数`metadata_failure_recovery`设置为false，重启fe01：
```
metadata_failure_recovery=false
```
Master状态已经正常，可以进行正常的查询和写入。

* 将fe02添加进集群。
```
ALTER SYSTEM ADD FOLLOWER "fe02:edit_log_port";
```

* 启动fe02，并通过`--helper fe01`参数指向Master节点。等待fe02启动完成。

* 将fe03添加进集群。
```
ALTER SYSTEM ADD FOLLOWER "fe03:edit_log_port";
```

* 启动fe03，并通过`--helper fe01`参数指向Master节点。等待fe03启动完成。

至此，集群恢复正常。
