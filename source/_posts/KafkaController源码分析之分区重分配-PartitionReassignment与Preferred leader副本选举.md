---
title: KafkaController源码分析之分区副本重分配(PartitionReassignment)与Preferred leader副本选举
date: 2020-03-05 14:24:25
categories: Kafka Tutorial
tags: [kafka,中间件,消息]
toc: true
comments: true
---

本文继续讲解Controller初始化过程，分析副本重分配过程

# 分区副本重分配

首先什么是分区副本重分配(PartitionReassignment)，以下摘自《Apache Kafka实战》一书对其做了阐释

>分区副本重分配操作通常都是由Kafka集群的管理员发起的，旨在对topic的所有分区重新分配副本所在broker的位置，以期望实现更均匀的分配效果。在该操作中管理员需要手动制定分配方案并按照指定的格式写入ZooKeeper的/admin/reassign_partitions节点下。

具体的操作可以参考[https://www.cnblogs.com/xionggeclub/p/9390037.html](https://www.cnblogs.com/xionggeclub/p/9390037.html)

该操作适用于集群扩容，管理员进行手动执行命令来发起

## 分区副本重分配事件的监听与处理

分区副本重分配主要由/admin/reassign_partitions节点的create事件触发，该事件的处理器为partitionReassignmentHandler,在[kafka-server端源码分析之Controller选举与初始化]()一文中的处理器表格中已有介绍
同时该节点是临时节点，只有发起时才会创建该节点，重分配过程结束后会删除该节点

## 分区副本重分配

分区副本重分配的方法入口是maybeTriggerPartitionReassignment方法，该方法会在Controller初始化和PartitionReassignment事件处理器中调用

```java
// KafkaController onControllerFailover方法中的重分配
maybeTriggerPartitionReassignment(controllerContext.partitionsBeingReassigned.keySet)

case object PartitionReassignment extends ControllerEvent {
	override def state: ControllerState = ControllerState.PartitionReassignment

	override def process(): Unit = {
	  if (!isActive) return

	  // We need to register the watcher if the path doesn't exist in order to detect future reassignments and we get
	  // the `path exists` check for free
	  // 注册 partitionReassignmentHandler
	  if (zkClient.registerZNodeChangeHandlerAndCheckExistence(partitionReassignmentHandler)) {
	    // 获取重分配方案
	    val partitionReassignment = zkClient.getPartitionReassignment

	    // Populate `partitionsBeingReassigned` with all partitions being reassigned before invoking
	    // `maybeTriggerPartitionReassignment` (see method documentation for the reason)
	    partitionReassignment.foreach { case (tp, newReplicas) =>
	      // 重分配引起的isr改变事件监听 
	      val reassignIsrChangeHandler = new PartitionReassignmentIsrChangeHandler(KafkaController.this, eventManager,
	        tp)
	      // 重分配缓存，ReassignedPartitionsContext：重分配的新副本，isr监听处理器
	      controllerContext.partitionsBeingReassigned.put(tp, ReassignedPartitionsContext(newReplicas, reassignIsrChangeHandler))
	    }

	    maybeTriggerPartitionReassignment(partitionReassignment.keySet)
	  }
	}
}
```

maybeTriggerPartitionReassignment的源码如下，更多是做准备，剔除不需要重分配的分区，真正开始重分配是调用 onPartitionReassignment方法
```java

private def maybeTriggerPartitionReassignment(topicPartitions: Set[TopicPartition]) {
	val partitionsToBeRemovedFromReassignment = scala.collection.mutable.Set.empty[TopicPartition]

	topicPartitions.foreach { tp =>
	  if (topicDeletionManager.isTopicQueuedUpForDeletion(tp.topic)) {
	    error(s"Skipping reassignment of $tp since the topic is currently being deleted")
	    partitionsToBeRemovedFromReassignment.add(tp)
	  } else {
	    val reassignedPartitionContext = controllerContext.partitionsBeingReassigned.get(tp).getOrElse {
	      // 防止partitionsBeingReassigned被改变(加锁不更好吗)
	      throw new IllegalStateException(s"Initiating reassign replicas for partition $tp not present in " +
	        s"partitionsBeingReassigned: ${controllerContext.partitionsBeingReassigned.mkString(", ")}")
	    }
	    val newReplicas = reassignedPartitionContext.newReplicas
	    val topic = tp.topic
	    val assignedReplicas = controllerContext.partitionReplicaAssignment(tp)
	    if (assignedReplicas.nonEmpty) {
	      if (assignedReplicas == newReplicas) {
	        info(s"Partition $tp to be reassigned is already assigned to replicas " +
	          s"${newReplicas.mkString(",")}. Ignoring request for partition reassignment.")
	        partitionsToBeRemovedFromReassignment.add(tp)
	      } else {
	        try {
	          info(s"Handling reassignment of partition $tp to new replicas ${newReplicas.mkString(",")}")
	          // first register ISR change listener
	          // 注册PartitionReassignmentIsrChangeHandler
	          reassignedPartitionContext.registerReassignIsrChangeHandler(zkClient)
	          // mark topic ineligible for deletion for the partitions being reassigned
	          // 标记为删除失败
	          topicDeletionManager.markTopicIneligibleForDeletion(Set(topic))
	          // 分区副本重分配
	          onPartitionReassignment(tp, reassignedPartitionContext)
	        } catch {
	          case e: Throwable =>
	            error(s"Error completing reassignment of partition $tp", e)
	            // remove the partition from the admin path to unblock the admin client
	            partitionsToBeRemovedFromReassignment.add(tp)
	        }
	      }
	    } else {
	        error(s"Ignoring request to reassign partition $tp that doesn't exist.")
	        partitionsToBeRemovedFromReassignment.add(tp)
	    }
	  }
	}
	removePartitionsFromReassignedPartitions(partitionsToBeRemovedFromReassignment)
}
```

## 分区副本重分配核心流程

onPartitionReassignment方法是完整的重分配流程，主要分为以下几个步骤
1. 先根据是否所有要分配的副本都在isr中分为2种情况
2. 不是所有的副本都在isr里时，取原来的副本和重分配的副本的并集，更新到/brokers/topics/topic节点的数据里，发送LeaderAndIsr请求。将重分配副本中比原来多出来的副本，设置为NewReplica状态
3. 所有的副本都在isr里时，检查重分配的副本里是否包含leader副本，不包含或者leader副本不在线时，根据ReassignPartitionLeaderElectionStrategy重新选举leader，否则仅仅是leader epoch+1更新回zk
4. 删除老副本(没有参与到reassign里的副本)
5. 更新缓存，并写回zk,删除/admin/reassign_partitions节点
6. 发送元数据更新请求，更新到每一个broker
7. 把本次reassign过程中的topic，看看有没有要删除的，进行删除

```java
  private def onPartitionReassignment(topicPartition: TopicPartition, reassignedPartitionContext: ReassignedPartitionsContext) {
    val reassignedReplicas = reassignedPartitionContext.newReplicas
    // 是否所有要分配的副本都在isr中
    if (!areReplicasInIsr(topicPartition, reassignedReplicas)) { // 说明不是所有的副本都在isr里
      info(s"New replicas ${reassignedReplicas.mkString(",")} for partition $topicPartition being reassigned not yet " +
        "caught up with the leader")
      // 即将要分配的 减去 之前已分配(缓存里)
      val newReplicasNotInOldReplicaList = reassignedReplicas.toSet -- controllerContext.partitionReplicaAssignment(topicPartition).toSet
      // 新的 + 老的 (会去重) = 全部的
      val newAndOldReplicas = (reassignedPartitionContext.newReplicas ++ controllerContext.partitionReplicaAssignment(topicPartition)).toSet
      //1. Update AR in ZK with OAR + RAR.
      //1. 更新reassign之后的全量副本到 /brokers/topics/topic节点
      updateAssignedReplicasForPartition(topicPartition, newAndOldReplicas.toSeq)
      //2. Send LeaderAndIsr request to every replica in OAR + RAR (with AR as OAR + RAR).
      //3. 发送LeaderAndIsr请求 TODO 这里缓存里的ReplicaAssignment应该等于newAndOldReplicas的, 但是replica也是brokerId
      updateLeaderEpochAndSendRequest(topicPartition, controllerContext.partitionReplicaAssignment(topicPartition),
        newAndOldReplicas.toSeq)
      //3. replicas in RAR - OAR -> NewReplica
      //3. 新增的副本转为NewReplica状态
      startNewReplicasForReassignedPartition(topicPartition, reassignedPartitionContext, newReplicasNotInOldReplicaList)
      info(s"Waiting for new replicas ${reassignedReplicas.mkString(",")} for partition ${topicPartition} being " +
        "reassigned to catch up with the leader")
    } else {
      //4. Wait until all replicas in RAR are in sync with the leader.
      // 重分配时原来就有的副本
      val oldReplicas = controllerContext.partitionReplicaAssignment(topicPartition).toSet -- reassignedReplicas.toSet
      //5. replicas in RAR -> OnlineReplica
      // reassignedReplicas副本转为OnlineReplica状态，因为它们都在ISR中
      reassignedReplicas.foreach { replica =>
        replicaStateMachine.handleStateChanges(Seq(new PartitionAndReplica(topicPartition, replica)), OnlineReplica)
      }
      //6. Set AR to RAR in memory.
      //7. Send LeaderAndIsr request with a potential new leader (if current leader not in RAR) and
      //   a new AR (using RAR) and same isr to every broker in RAR
      moveReassignedPartitionLeaderIfRequired(topicPartition, reassignedPartitionContext)
      //8. replicas in OAR - RAR -> Offline (force those replicas out of isr)
      //9. replicas in OAR - RAR -> NonExistentReplica (force those replicas to be deleted)
      // 删除老副本(没有参与到reassign里的副本)
      stopOldReplicasOfReassignedPartition(topicPartition, reassignedPartitionContext, oldReplicas)
      //10. Update AR in ZK with RAR.
      // 更新缓存，并写回zk
      updateAssignedReplicasForPartition(topicPartition, reassignedReplicas)
      //11. Update the /admin/reassign_partitions path in ZK to remove this partition.
      // 删除/admin/reassign_partitions节点
      removePartitionsFromReassignedPartitions(Set(topicPartition))
      //12. After electing leader, the replicas and isr information changes, so resend the update metadata request to every broker
      // 更新到每一个broker
      sendUpdateMetadataRequest(controllerContext.liveOrShuttingDownBrokerIds.toSeq, Set(topicPartition))
      // signal delete topic thread if reassignment for some partitions belonging to topics being deleted just completed
      // 把本次reassign过程中的topic，看看有没有要删除的，进行删除
      topicDeletionManager.resumeDeletionForTopics(Set(topicPartition.topic))
    }
  }
```
整个过程还是十分复杂的，但是我并没有按照注释用一堆RAR，OAR，AR的概念来解释，那样很容易记混，反而是看懂了代码，再去理解这些概括水到渠成
![分区副本重分配流程](https://ae01.alicdn.com/kf/H33d287c097124677b568b65210d808d5V.png)

最后再聊一下ReassignPartitionLeaderElectionStrategy

### 分区副本重分配的leader选举算法
```java
  private def leaderForReassign(leaderIsrAndControllerEpochs: Seq[(TopicPartition, LeaderIsrAndControllerEpoch)]):
  Seq[(TopicPartition, Option[LeaderAndIsr], Seq[Int])] = {
    leaderIsrAndControllerEpochs.map { case (partition, leaderIsrAndControllerEpoch) =>
      // 重分配的副本
      val reassignment = controllerContext.partitionsBeingReassigned(partition).newReplicas
      // 存活的重分配副本
      val liveReplicas = reassignment.filter(replica => controllerContext.isReplicaOnline(replica, partition))
      // isr
      val isr = leaderIsrAndControllerEpoch.leaderAndIsr.isr
      // 选举算法计算leader
      val leaderOpt = PartitionLeaderElectionAlgorithms.reassignPartitionLeaderElection(reassignment, isr, liveReplicas.toSet)
      val newLeaderAndIsrOpt = leaderOpt.map(leader => leaderIsrAndControllerEpoch.leaderAndIsr.newLeader(leader))
      (partition, newLeaderAndIsrOpt, reassignment)
    }
  }
  /**
    * @param reassignment 要重分配的副本
    * @param isr isr副本
    * @param liveReplicas reassignment中存活的副本
    */
  def reassignPartitionLeaderElection(reassignment: Seq[Int], isr: Seq[Int], liveReplicas: Set[Int]): Option[Int] = {
    // reassignment里 liveReplicas和isr都有的副本 (取第一个)
    reassignment.find(id => liveReplicas.contains(id) && isr.contains(id))
  }
```


# Preferred leader副本

什么是Preferred leader副本

>Kafka在给每个Partition分配副本时，它会保证分区的主副本会均匀分布在所有的broker上，这样的话只要保证第一个replica被选举为leader，读写流量就会均匀分布在所有的Broker上，但是在实际的生产环境,每个 Partition的读写流量相差可能较多，不一定可以达到该目的

## zk事件监听

Preferred leader副本选举由/admin/preferred_replica_election节点的创建事件触发，对应的节点handler为PreferredReplicaElectionHandler，对应的创建事件处理器为PreferredReplicaLeaderElection

PreferredReplicaLeaderElection的核心处理方法为onPreferredReplicaElection，同时该方法也会在Controller初始化的onControllerFailover中被调用，用于Preferred leader副本选举

## Preferred leader副本选举

Controller的onControllerFailover方法的调用如下
```java
val pendingPreferredReplicaElections = fetchPendingPreferredReplicaElections()
onPreferredReplicaElection(pendingPreferredReplicaElections)
```
首先是通过fetchPendingPreferredReplicaElections获取要进行Preferred leader副本选举的分区

```java
private def fetchPendingPreferredReplicaElections(): Set[TopicPartition] = {
    // 获取/admin/preferred_replica_election节点数据
    val partitionsUndergoingPreferredReplicaElection = zkClient.getPreferredReplicaElection
    // check if they are already completed or topic was deleted
    // 分区没有副本或者已经是Preferred Replica了
    val partitionsThatCompletedPreferredReplicaElection = partitionsUndergoingPreferredReplicaElection.filter { partition =>
      val replicas = controllerContext.partitionReplicaAssignment(partition)
      val topicDeleted = replicas.isEmpty
      val successful =
        if (!topicDeleted) controllerContext.partitionLeadershipInfo(partition).leaderAndIsr.leader == replicas.head else false
      successful || topicDeleted
    }
    val pendingPreferredReplicaElectionsIgnoringTopicDeletion = partitionsUndergoingPreferredReplicaElection -- partitionsThatCompletedPreferredReplicaElection
    val pendingPreferredReplicaElectionsSkippedFromTopicDeletion = pendingPreferredReplicaElectionsIgnoringTopicDeletion.filter(partition => topicDeletionManager.isTopicQueuedUpForDeletion(partition.topic))
    val pendingPreferredReplicaElections = pendingPreferredReplicaElectionsIgnoringTopicDeletion -- pendingPreferredReplicaElectionsSkippedFromTopicDeletion
    
    // 准备要preferred replica选举的分区
    pendingPreferredReplicaElections
}
```

## 核心流程

onPreferredReplicaElection方法通过分区状态机，将分区转换为OnlinePartition状态，并根据PreferredReplicaPartitionLeaderElectionStrategy选举leader，下面我们直接看相关的代码，由于在[KafkaController源码分析之副本状态机与分区状态机的启动]()已经讲解过该方法了，我们直接看一下ReassignPartitionLeaderElectionStrategy算法的实现

取第一个即是存活的，又在isr列表中的副本
```java
  def preferredReplicaPartitionLeaderElection(assignment: Seq[Int], isr: Seq[Int], liveReplicas: Set[Int]): Option[Int] = {
    assignment.headOption.filter(id => liveReplicas.contains(id) && isr.contains(id))
  }
```

