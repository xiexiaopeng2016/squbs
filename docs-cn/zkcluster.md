# 使用ZooKeeper集群化squbs服务

## 概述

squbs通过zkcluster模块实现集群化服务。zkcluster是一个[Akka扩展](http://doc.akka.io/docs/akka/snapshot/scala/extending-akka.html)，利用[ZooKeeper](https://zookeeper.apache.org/)管理akka集群和分区。

它类似于[Akka集群](http://doc.akka.io/docs/akka/snapshot/common/cluster.html)的领导和成员管理功能。但它更丰富，因为它提供了分区支持，并消除了对`entry-nodes`的需要。


## 配置

我们需要一个在运行时目录下的`squbsconfig/zkcluster.conf`文件。它应该提供了如下属性：

* connectionString: 一个用逗号分隔定义所有zookeeper节点
* namespace: 一个字符串，它是znode的有效路径，它将是此后创建的所有znode的父节点
* segments: 用于划分分区数量的分区段数

下面是一个`zkcluster.conf`文件内容的例子：

```
zkCluster {
    connectionString = "zk-node-01.squbs.org:2181,zk-node-02.squbs.org:2181,zk-node-03.squbs.org:2181"
    namespace = "clusteredservicedev"
    segments = 128
}
```

## 用户指南

首先简单的注册扩展，像所有正常的akka扩展一样。然后你可以访问并使用`zkClusterActor`，如下所示：

```scala
val zkClusterActor = ZkCluster(system).zkClusterActor

// Query the members in the cluster
zkClusterActor ! ZkQueryMembership

// Matching the response
case ZkMembership(members:Set[Address]) =>


// Query leader in the cluster
zkClusterActor ! ZkQueryLeadership
// Matching the response
case ZkLeadership(leader:Address) =>

// Query partition (expectedSize = None), create or resize (expectedSize = Some[Int])
zkClusterActor ! ZkQueryPartition(partitionKey:ByteString, notification:Option[Any] = None, expectedSize:Option[Int] = None, props:Array[Byte] = Array[Byte]())
// Matching the response
case ZkPartition(partitionKey:ByteString, members: Set[Address], zkPath:String, notification:Option[Any]) =>
case ZkPartitionNotFound(partitionKey: ByteString) =>


// Monitor or stop monitoring the partition change
zkClusterActor ! ZkMonitorPartition
zkClusterActor ! ZkStopMonitorPartition
// Matching the response
case ZkPartitionDiff(partitionKey: ByteString, onBoardMembers: Set[Address], dropOffMembers: Set[Address], props: Array[Byte] = Array.empty) =>

// Removing partition
zkClusterActor ! ZkRemovePartition(partitionKey:ByteString)
// Matching the response
case ZkPartitionRemoval(partitionKey:ByteString) =>


// List the partitions hosted by a certain member
zkClusterActor ! ZkListPartitions(address: Address)
// Matching the response
case ZkPartitions(partitionKeys:Seq[ByteString]) =>

// monitor the zookeeper connection state
val eventStream = context.system.eventStream
eventStream.subscribe(self, ZkConnected.getClass)
eventStream.subscribe(self, ZkReconnected.getClass)
eventStream.subscribe(self, ZkLost.getClass)
eventStream.subscribe(self, ZkSuspended.getClass)

// quit the cluster
zkCluster(system).zkClusterActor ! PoisonPill

// add listener when quitting the cluster
zkCluster(system).addShutdownListener(listener: () => Unit)
```

## 依赖

在你的`build.sbt`或Scala构建文件增加如下依赖：

```scala
"org.squbs" %% "squbs-zkcluster" % squbsVersion
```

## 设计

如果您正在更改`zkcluster`，请阅读这些：

* 成员是基于`zookeeper`临时节点，关闭会话会通过`ZkMembershipChanged`改变领导者。
* 领导是基于`curator`框架的`LeaderLatch`，新的选举将广播`ZkLeaderElected`给所有的节点。
* 分区由领导者计算，并由领导节点中的`ZkPartitionsManager`写入znode。
* 分区修改只能由领导者完成，它要求其`ZkPartitionsManager`来强制执行修改。
* `ZkPartitionsManager`的追随者节点将观察Zookeeper中znode的变化。一旦领导者在重新平衡后改变了分区, 在追随者的节点中的`ZkPartitionsManager`将得到通知，并更新他们的分区信息的内存快照。
* 无论谁需要得到分区改变`ZkPartitionDiff`的通知，应当发送`ZkMonitorPartition`到集群已经注册的actor。

`ZkMembershipMonitor` 是处理成员和领导者的actor类型。

`ZkPartitionsManager` 是处理分区管理的actor。

`ZkClusterActor` 是用户将要向其发送查询的接口actor。
