---
title: KafkaController源码分析之Broker的上线与下线
date: 2020-02-26 19:24:34
categories: Kafka Tutorial
tags: [kafka,中间件,消息]
toc: true
comments: true
---

# 前言

本文主要聊聊某一个broker上线与下线时，集群是如何感知的

# zk事件

在KafkaController#onControllerFailover方法中，会向zk注册一个brokerChangeHandler，它主要监听/brokers/ids下的子节点变化事件，我们知道该节点下就是每一个broker的id，里面的数据是broker的ip端口，协议等信息
BrokerChangeHandler的源码如下

```java
class BrokerChangeHandler(controller: KafkaController, eventManager: ControllerEventManager) extends ZNodeChildChangeHandler {
  override val path: String = BrokerIdsZNode.path

  override def handleChildChange(): Unit = {
    eventManager.put(controller.BrokerChange)
  }
}
```

## 处理逻辑

事件的处理逻辑在BrokerChange类中，isActive之前也说过了，表示当前broker是否是Controller，这里我们要明确一点时，broker的上下线事件只能由Controller处理，其他broker虽然也会监听/brokers/ids节点，但不会做任何处理

process方法中的curBrokers表示zk中当前的broker列表信息，liveOrShuttingDownBrokerIds表示本地缓存的broker列表信息，假设liveOrShuttingDownBrokerIds是[0,1,2]，如果新增了一个broker3，curBrokers就是[0,1,2,3]；如果broker2下线了，curBrokers就是[0,1]。只需要简单的对curBrokers和liveOrShuttingDownBrokerIds做差集运算，我们就知道上线和下线的broker集合分别是什么，这也是该方法前半部分的大致思路

```java
case object BrokerChange extends ControllerEvent {
    override def state: ControllerState = ControllerState.BrokerChange

    override def process(): Unit = {
      if (!isActive) return
      // 获取zk中当前所有broker的信息
      val curBrokers = zkClient.getAllBrokersInCluster.toSet
      val curBrokerIds = curBrokers.map(_.id)
      val liveOrShuttingDownBrokerIds = controllerContext.liveOrShuttingDownBrokerIds
      // 新增的broker
      val newBrokerIds = curBrokerIds -- liveOrShuttingDownBrokerIds

      val deadBrokerIds = liveOrShuttingDownBrokerIds -- curBrokerIds

      // 新broker的信息
      val newBrokers = curBrokers.filter(broker => newBrokerIds(broker.id))
      // 更新缓存
      controllerContext.liveBrokers = curBrokers

      val newBrokerIdsSorted = newBrokerIds.toSeq.sorted
      val deadBrokerIdsSorted = deadBrokerIds.toSeq.sorted
      val liveBrokerIdsSorted = curBrokerIds.toSeq.sorted
      info(s"Newly added brokers: ${newBrokerIdsSorted.mkString(",")}, " +
        s"deleted brokers: ${deadBrokerIdsSorted.mkString(",")}, all live brokers: ${liveBrokerIdsSorted.mkString(",")}")

      // 建立当前broker与新增broker之间的channel
      newBrokers.foreach(controllerContext.controllerChannelManager.addBroker)
      // 关闭与dead broker直接的所有资源
      deadBrokerIds.foreach(controllerContext.controllerChannelManager.removeBroker)

      if (newBrokerIds.nonEmpty)
        onBrokerStartup(newBrokerIdsSorted)
      if (deadBrokerIds.nonEmpty)
        onBrokerFailure(deadBrokerIdsSorted)
    }
}
```

在[KafkaController源码分析之LeaderAndIsr请求]()中已经提到了ControllerChannelManager，它是Controller节点与其它broker之间网络通信管理器，在得到上线和下线的broker集合后，分别做以下处理：
1. 上线，建立网络连接，并启动请求发送线程，用于处理leaderAndIsrRequest，stopReplicaRequest，updateMetadataRequest三类请求
2. 下线，关闭网络连接，中断(interrupt)请求发送线程，清空请求队列

此处源码较为简单，篇幅有限，不再贴出源码了。最后的两个if，表示如果有新增的broker，执行onBrokerStartup方法，有下线的broker执行onBrokerFailure方法

## broker上线处理onBrokerStartup

该方法主要做了以下5件事：

1. 发送update metadata request，Controller把最新的broker列表同步给别的broker
2. 将新broker上的分区和副本都置为Online状态，并选举分区leader副本，注意这里面已经包含了LeaderAndIsr请求
3. 新broker上是否有重分配的副本，有就执行
4. 如果新broker上有需要删除的topic，开始删除
5. 注册了一个/broker/ids/0的数据变化的监听器——BrokerModificationsHandler，个人觉得broker元信息不会改变

```java
private def onBrokerStartup(newBrokers: Seq[Int]) {
    info(s"New broker startup callback for ${newBrokers.mkString(",")}")
    newBrokers.foreach(controllerContext.replicasOnOfflineDirs.remove)
    val newBrokersSet = newBrokers.toSet
    // send update metadata request to all live and shutting down brokers. Old brokers will get to know of the new
    // broker via this update.
    // In cases of controlled shutdown leaders will not be elected when a new broker comes up. So at least in the
    // common controlled shutdown case, the metadata will reach the new brokers faster
    sendUpdateMetadataRequest(controllerContext.liveOrShuttingDownBrokerIds.toSeq)
    // the very first thing to do when a new broker comes up is send it the entire list of partitions that it is
    // supposed to host. Based on that the broker starts the high watermark threads for the input list of partitions
    val allReplicasOnNewBrokers = controllerContext.replicasOnBrokers(newBrokersSet)
    replicaStateMachine.handleStateChanges(allReplicasOnNewBrokers.toSeq, OnlineReplica)
    // when a new broker comes up, the controller needs to trigger leader election for all new and offline partitions
    // to see if these brokers can become leaders for some/all of those
    partitionStateMachine.triggerOnlinePartitionStateChange()
    // check if reassignment of some partitions need to be restarted
    val partitionsWithReplicasOnNewBrokers = controllerContext.partitionsBeingReassigned.filter {
      case (_, reassignmentContext) => reassignmentContext.newReplicas.exists(newBrokersSet.contains)
    }
    partitionsWithReplicasOnNewBrokers.foreach { case (tp, context) => onPartitionReassignment(tp, context) }
    // check if topic deletion needs to be resumed. If at least one replica that belongs to the topic being deleted exists
    // on the newly restarted brokers, there is a chance that topic deletion can resume
    val replicasForTopicsToBeDeleted = allReplicasOnNewBrokers.filter(p => topicDeletionManager.isTopicQueuedUpForDeletion(p.topic))
    if (replicasForTopicsToBeDeleted.nonEmpty) {
      info(s"Some replicas ${replicasForTopicsToBeDeleted.mkString(",")} for topics scheduled for deletion " +
        s"${topicDeletionManager.topicsToBeDeleted.mkString(",")} are on the newly restarted brokers " +
        s"${newBrokers.mkString(",")}. Signaling restart of topic deletion for these topics")
      topicDeletionManager.resumeDeletionForTopics(replicasForTopicsToBeDeleted.map(_.topic))
    }
    registerBrokerModificationsHandler(newBrokers)
}
```

## broker下线处理onBrokerFailure

onBrokerFailure方法首先更新了本地缓存，之后调用了onReplicasBecomeOffline来处理副本下线的情况，之后移除了BrokerModificationsHandler，对应onBrokerStartup的最后一步

```java
private def onBrokerFailure(deadBrokers: Seq[Int]) {
    info(s"Broker failure callback for ${deadBrokers.mkString(",")}")
    // 移除缓存中下线的broker上的分区
    deadBrokers.foreach(controllerContext.replicasOnOfflineDirs.remove)
    val deadBrokersThatWereShuttingDown =
      deadBrokers.filter(id => controllerContext.shuttingDownBrokerIds.remove(id))
    info(s"Removed $deadBrokersThatWereShuttingDown from list of shutting down brokers.")
    // 下线broker上的副本
    val allReplicasOnDeadBrokers = controllerContext.replicasOnBrokers(deadBrokers.toSet)

    onReplicasBecomeOffline(allReplicasOnDeadBrokers)

    // 移除BrokerModificationsHandler
    unregisterBrokerModificationsHandler(deadBrokers)
}
```

### 处理下线broker中的副本

该方法看似复杂，但是逻辑很严谨，建议大家仔细看看。它主要做了以下事情：
1. 将leader副本在dead broker上的分区找出来
2. 将这些分区置为OfflinePartition状态
3. 用这些分区剩下的副本触发一次leader选举
4. 将dead broker上的副本置为Offline
5. 如果dead broker上有要删除的topic，标记为删除失败，毕竟都下线了怎么删？
6. 如果dead broker上没有分区的leader副本，也就是第1步返回的是空，就发送UpdateMetadataRequest给剩下活着的broker

总之该方法逻辑很严谨，算是一个比较重点的步骤

```java
private def onReplicasBecomeOffline(newOfflineReplicas: Set[PartitionAndReplica]): Unit = {
    val (newOfflineReplicasForDeletion, newOfflineReplicasNotForDeletion) =
      newOfflineReplicas.partition(p => topicDeletionManager.isTopicQueuedUpForDeletion(p.topic))

    // 将leader在dead broker上的分区找出来
    val partitionsWithoutLeader = controllerContext.partitionLeadershipInfo.filter(partitionAndLeader =>
      !controllerContext.isReplicaOnline(partitionAndLeader._2.leaderAndIsr.leader, partitionAndLeader._1) &&
        !topicDeletionManager.isTopicQueuedUpForDeletion(partitionAndLeader._1.topic)).keySet

    // trigger OfflinePartition state for all partitions whose current leader is one amongst the newOfflineReplicas
    // 标记这些分区为OfflinePartition状态
    partitionStateMachine.handleStateChanges(partitionsWithoutLeader.toSeq, OfflinePartition)
    // 用这些剩余分区剩余的副本选举leader
    // trigger OnlinePartition state changes for offline or new partitions
    partitionStateMachine.triggerOnlinePartitionStateChange()
    // trigger OfflineReplica state change for those newly offline replicas
    // dead broker上的副本置为Offline
    replicaStateMachine.handleStateChanges(newOfflineReplicasNotForDeletion.toSeq, OfflineReplica)

    // broker已下线，删除失败
    // fail deletion of topics that are affected by the offline replicas
    if (newOfflineReplicasForDeletion.nonEmpty) {
      // it is required to mark the respective replicas in TopicDeletionFailed state since the replica cannot be
      // deleted when its log directory is offline. This will prevent the replica from being in TopicDeletionStarted state indefinitely
      // since topic deletion cannot be retried until at least one replica is in TopicDeletionStarted state
      topicDeletionManager.failReplicaDeletion(newOfflineReplicasForDeletion)
    }

    // If replica failure did not require leader re-election, inform brokers of the offline replica
    // Note that during leader re-election, brokers update their metadata
    // 如果dead broker上没有分区的leader副本，就发送UpdateMetadataRequest给活着的broker
    if (partitionsWithoutLeader.isEmpty) {
      sendUpdateMetadataRequest(controllerContext.liveOrShuttingDownBrokerIds.toSeq)
    }
}
```



