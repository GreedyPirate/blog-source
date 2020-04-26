---
title: KafkaController源码分析之副本状态机与分区状态机的启动
date: 2020-03-04 21:17:51
categories: Kafka Tutorial
tags: [kafka,中间件,消息]
toc: true
comments: true
---

本文承接上篇[kafka-server端源码分析之Controller初始化]()，继续讲解Controller初始化过程中副本状态机与分区状态机的启动

# 副本状态机

kafka将副本分为7个状态，下图是状态之间的流转图

![副本状态流转图](https://ae01.alicdn.com/kf/H43a853d3980c475c9f6894950bb29e41v.png)

副本状态用ReplicaState接口表示，需要说下validPreviousStates方法，它表示合法的开始状态，以NewReplica为例，它只能由NonExistentReplica状态转换而来
```java
case object NewReplica extends ReplicaState {
  val state: Byte = 1
  val validPreviousStates: Set[ReplicaState] = Set(NonExistentReplica)
}
```
而状态之间的转换，必将涉及到大量的更新操作，ReplicaStateMachine#doHandleStateChanges方法统一处理了状态转换

回过头来说replicaStateMachine.startup()方法，它主要是将在线的副本转换为OnlineReplica状态

```java
def startup() {
  // 这一步简单却很重要，初始化replicaState，它保存了每个副本的状态
  // 为之后handleStateChanges转变为OnlineReplica做准备
  initializeReplicaState()

  handleStateChanges(controllerContext.allLiveReplicas().toSeq, OnlineReplica)
}
```
## 初始化副本状态缓存

首先看initializeReplicaState的初始化，只要理解了controllerContext没有什么难度
该方法主要初始化了一个replicaState缓存，记录了每一个副本的状态，根据是否在线分为OnlineReplica和ReplicaDeletionIneligible状态
```java
private def initializeReplicaState() {
  controllerContext.allPartitions.foreach { partition =>
    val replicas = controllerContext.partitionReplicaAssignment(partition)
    replicas.foreach { replicaId =>
      val partitionAndReplica = PartitionAndReplica(partition, replicaId)
      if (controllerContext.isReplicaOnline(replicaId, partition))
        replicaState.put(partitionAndReplica, OnlineReplica)
      else
      // mark replicas on dead brokers as failed for topic deletion, if they belong to a topic to be deleted.
      // This is required during controller failover since during controller failover a broker can go down,
      // so the replicas on that broker should be moved to ReplicaDeletionIneligible to be on the safer side.
        replicaState.put(partitionAndReplica, ReplicaDeletionIneligible)
    }
  }
}
```

初始化replicaState之后，handleStateChanges将所有存活的副本转换为OnlineReplica，此时正常的副本就是从OnlineReplica -> OnlineReplica
```java
def handleStateChanges(replicas: Seq[PartitionAndReplica], targetState: ReplicaState,
                       callbacks: Callbacks = new Callbacks()): Unit = {
  if (replicas.nonEmpty) {
    try {
      controllerBrokerRequestBatch.newBatch()
      replicas.groupBy(_.replica).map { case (replicaId, replicas) =>
        val partitions = replicas.map(_.topicPartition)
        doHandleStateChanges(replicaId, partitions, targetState, callbacks)
      }
      // 发送ControllerChannelManager中积攒的请求
      controllerBrokerRequestBatch.sendRequestsToBrokers(controllerContext.epoch)
    } catch {
      case e: Throwable => error(s"Error while moving some replicas to $targetState state", e)
    }
  }
}
```

doHandleStateChanges用于处理副本状态转换，此时我们只关注targetState是OnlineReplica的处理

```java
private def doHandleStateChanges(replicaId: Int, partitions: Seq[TopicPartition], targetState: ReplicaState,
                                   callbacks: Callbacks): Unit = {
  // 这里又组成了Seq[PartitionAndReplica]
  val replicas = partitions.map(partition => PartitionAndReplica(partition, replicaId))

  // 查看该副本的状态，不存在Update为NonExistentReplica
  replicas.foreach(replica => replicaState.getOrElseUpdate(replica, NonExistentReplica))

  // isValidTransition: 对转换的开始状态做合法性校验，参考前面副本状态机的介绍
  // 注意这里的partition方法不是分区的意思，它是一个布尔分组器
  // validReplicas是转换合法的副本，invalidReplicas是非合法的
  val (validReplicas, invalidReplicas) = replicas.partition(replica => isValidTransition(replica, targetState))
  // 不合法主要用日志记录异常
  invalidReplicas.foreach(replica => logInvalidTransition(replica, targetState))
  // 初始化Controller时，targetState=OnlineReplica
  //validReplicas==>Seq[PartitionAndReplica]
  targetState match {
    case OnlineReplica => // previousState: NewReplica, OnlineReplica, OfflineReplica, ReplicaDeletionIneligible
        validReplicas.foreach { replica =>
          val partition = replica.topicPartition
          replicaState(replica) match { // 这里获取的是副本的状态
            case NewReplica =>
              // NewReplica->OnlineReplica，本地分区副本分配缓存里如果没有该副本，就更新进去
              val assignment = controllerContext.partitionReplicaAssignment(partition) // 从缓存中获取分区对应的副本集合
              if (!assignment.contains(replicaId)) {
                controllerContext.updatePartitionReplicaAssignment(partition, assignment :+ replicaId) 
              }
            case _ =>
              controllerContext.partitionLeadershipInfo.get(partition) match {
                case Some(leaderIsrAndControllerEpoch) =>
                  // 发送LeaderAndIsr请求(放入等待队列)
                  controllerBrokerRequestBatch.addLeaderAndIsrRequestForBrokers(Seq(replicaId),
                    replica.topicPartition,
                    leaderIsrAndControllerEpoch,
                    controllerContext.partitionReplicaAssignment(partition), isNew = false)
                case None =>
              }
          }
          logSuccessfulTransition(replicaId, partition, replicaState(replica), OnlineReplica)
          replicaState.put(replica, OnlineReplica)
        }
    // 省略其他状态的处理 ......
  }
}
```
NewReplica, OnlineReplica, OfflineReplica, ReplicaDeletionIneligible状态都可以转换到OnlineReplica状态
NewReplica会检查本地缓存，没有就更新,而其他状态需要发送LeaderAndIsr请求同步broker之间的数据

至此副本状态机的启动结束了，LeaderAndIsr请求作为kafka最核心的一个请求会在后面单独的篇章解析。


# 分区状态机

分区状态机相比于副本状态机而言，状态个数只有4个，但是涉及到副本leader选举，状态流转的复杂度高很多

![分区状态流转图](https://ae01.alicdn.com/kf/Hba4113a46e4248c6bf942b6b46374ca1i.png)

PartitionStateMachine的startup方法如下
```java
def startup() {
  // 初始化分区的state
  initializePartitionState()
  triggerOnlinePartitionStateChange()
}
```
## 初始化分区状态缓存

和副本状态机类似，initializePartitionState也是用一个partitionState初始化每个分区的状态
将缓存中所有分区分为3种初始化状态
1. 有leader副本，并且在线，标记为OnlinePartition状态，不在线为OfflinePartition
2. 没有leader标记分区为NewPartition状态
```java
private def initializePartitionState() {
  for (topicPartition <- controllerContext.allPartitions) {
    // check if leader and isr path exists for partition. If not, then it is in NEW state
    // 获取leader和isr信息
    controllerContext.partitionLeadershipInfo.get(topicPartition) match {
      case Some(currentLeaderIsrAndEpoch) =>
        // else, check if the leader for partition is alive. If yes, it is in Online state, else it is in Offline state
        // leader存活就是OnlinePartition状态的分区，否则就是OfflinePartition
        if (controllerContext.isReplicaOnline(currentLeaderIsrAndEpoch.leaderAndIsr.leader, topicPartition))
        // leader is alive
          partitionState.put(topicPartition, OnlinePartition)
        else
          partitionState.put(topicPartition, OfflinePartition)
      case None =>
        // 没有leader为NewPartition状态
        partitionState.put(topicPartition, NewPartition)
    }
  }
}
```

初始化之后partitionState，分区状态机会把OfflinePartition和NewPartition的分区转换为OnlinePartition状态，
broker正常运行的情况下，分区都是OnlinePartition状态，此时handleStateChanges不会执行

```java
def triggerOnlinePartitionStateChange() {
  // try to move all partitions in NewPartition or OfflinePartition state to OnlinePartition state except partitions
  // that belong to topics to be deleted
  // 正常情况下partitionsToTrigger为空的，启动kafka时所有分区都是OnlinePartition
  val partitionsToTrigger = partitionState.filter { case (partition, partitionState) =>
    !topicDeletionManager.isTopicQueuedUpForDeletion(partition.topic) &&
      (partitionState.equals(OfflinePartition) || partitionState.equals(NewPartition))
  }.keys.toSeq
  // 把所有OfflinePartition，NewPartition和非准备删除的分区 转换为OnlinePartition
  handleStateChanges(partitionsToTrigger, OnlinePartition, Option(OfflinePartitionLeaderElectionStrategy))
}

def handleStateChanges(partitions: Seq[TopicPartition], targetState: PartitionState,
                       partitionLeaderElectionStrategyOpt: Option[PartitionLeaderElectionStrategy] = None): Unit = {
  if (partitions.nonEmpty) {
    try {
      controllerBrokerRequestBatch.newBatch()
      doHandleStateChanges(partitions, targetState, partitionLeaderElectionStrategyOpt)
      // 发送一次请求队列，包括了doHandleStateChanges里新增的请求
      controllerBrokerRequestBatch.sendRequestsToBrokers(controllerContext.epoch)
    } catch {
      case e: Throwable => error(s"Error while moving some partitions to $targetState state", e)
    }
  }
}
```
## 分区leader选举

doHandleStateChanges主要是选举分区的leader副本，这里现将分区分为两类：
1. 未初始化的分区(uninitializedPartitions)：状态是NewPartition的分区
2. 准备要选举leader副本的分区(partitionsToElectLeader)：状态是OfflinePartition，OnlinePartition的分区
doHandleStateChanges主要是对这两类分区选举leader，并放到前面说的partitionState缓存中

注：注意前面传递过来的选举策略是OfflinePartitionLeaderElectionStrategy
```java
private def doHandleStateChanges(partitions: Seq[TopicPartition], targetState: PartitionState,
                         partitionLeaderElectionStrategyOpt: Option[PartitionLeaderElectionStrategy]): Unit = {
  // 这里的处理和副本状态机一样
  val stateChangeLog = stateChangeLogger.withControllerEpoch(controllerContext.epoch)
  partitions.foreach(partition => partitionState.getOrElseUpdate(partition, NonExistentPartition))
  val (validPartitions, invalidPartitions) = partitions.partition(partition => isValidTransition(partition, targetState))
  invalidPartitions.foreach(partition => logInvalidTransition(partition, targetState))
  targetState match {
    case OnlinePartition =>
      val uninitializedPartitions = validPartitions.filter(partition => partitionState(partition) == NewPartition) // 类型：Seq[TopicPartition]
      val partitionsToElectLeader = validPartitions.filter(partition => partitionState(partition) == OfflinePartition || partitionState(partition) == OnlinePartition)
      // 状态为NewPartition的分区处理
      if (uninitializedPartitions.nonEmpty) {
        // 初始化新分区的leader
        val successfulInitializations = initializeLeaderAndIsrForPartitions(uninitializedPartitions)
        successfulInitializations.foreach { partition =>
          stateChangeLog.trace(s"Changed partition $partition from ${partitionState(partition)} to $targetState with state " +
            s"${controllerContext.partitionLeadershipInfo(partition).leaderAndIsr}")
          partitionState.put(partition, OnlinePartition)
        }
      }
      // OfflinePartition,OnlinePartition副本中开始选举
      if (partitionsToElectLeader.nonEmpty) {
        // 根据选举策略(Strategy)选举leader副本
        val successfulElections = electLeaderForPartitions(partitionsToElectLeader, partitionLeaderElectionStrategyOpt.get)
        successfulElections.foreach { partition =>
          stateChangeLog.trace(s"Changed partition $partition from ${partitionState(partition)} to $targetState with state " +
            s"${controllerContext.partitionLeadershipInfo(partition).leaderAndIsr}")
          // 更新分区状态为OnlinePartition
          partitionState.put(partition, OnlinePartition)
        }
      }
  }
}
```
下面说说这两类分区leader副本选举方式

### NewPartition状态的分区选举leader副本

initializeLeaderAndIsrForPartitions方法是在为NewPartition状态的分区选举leader副本
代码看上去很长，但是一句话就可以概括：取存活副本的列表的第一个副本作为leader，写回到zk的state节点，更新本地缓存，并发送LeaderAndIsr请求同步给其他broker

```java
private def initializeLeaderAndIsrForPartitions(partitions: Seq[TopicPartition]): Seq[TopicPartition] = {
    val successfulInitializations = mutable.Buffer.empty[TopicPartition]
    // 获取分区副本
    val replicasPerPartition = partitions.map(partition => partition -> controllerContext.partitionReplicaAssignment(partition))
    // 只要在线的副本
    val liveReplicasPerPartition = replicasPerPartition.map { case (partition, replicas) =>
        val liveReplicasForPartition = replicas.filter(replica => controllerContext.isReplicaOnline(replica, partition))
        partition -> liveReplicasForPartition
    }
    // 分区按照是否有在线的副本
    val (partitionsWithoutLiveReplicas, partitionsWithLiveReplicas) = liveReplicasPerPartition.partition { case (_, liveReplicas) => liveReplicas.isEmpty }
    // 没有在线副本的分区处理：打日志
    partitionsWithoutLiveReplicas.foreach { case (partition, replicas) =>
      val failMsg = s"Controller $controllerId epoch ${controllerContext.epoch} encountered error during state change of " +
        s"partition $partition from New to Online, assigned replicas are " +
        s"[${replicas.mkString(",")}], live brokers are [${controllerContext.liveBrokerIds}]. No assigned " +
        "replica is alive."
      logFailedStateChange(partition, NewPartition, OnlinePartition, new StateChangeFailedException(failMsg))
    }
    // 有在线副本的分区，将在线副本的第一个leader副本，并初始化ISR列表
    // Map[TopicPartition, LeaderIsrAndControllerEpoch]
    val leaderIsrAndControllerEpochs = partitionsWithLiveReplicas.map { case (partition, liveReplicas) =>
      // 这里就在初始化分区的leader(在线副本的第一个)，ISR
      val leaderAndIsr = LeaderAndIsr(liveReplicas.head, liveReplicas.toList)
      val leaderIsrAndControllerEpoch = LeaderIsrAndControllerEpoch(leaderAndIsr, controllerContext.epoch) // 加上Controller epoch
      partition -> leaderIsrAndControllerEpoch
    }.toMap
    val createResponses = try {
      // 创建 /topics/topic名称/partitions/分区名称/state，包含中间节点
      zkClient.createTopicPartitionStatesRaw(leaderIsrAndControllerEpochs)
    } catch {
      case e: Exception =>
        partitionsWithLiveReplicas.foreach { case (partition,_) => logFailedStateChange(partition, partitionState(partition), NewPartition, e) }
        Seq.empty
    }
    createResponses.foreach { createResponse =>
      val code = createResponse.resultCode
      val partition = createResponse.ctx.get.asInstanceOf[TopicPartition]
      val leaderIsrAndControllerEpoch = leaderIsrAndControllerEpochs(partition)
      if (code == Code.OK) {
        // 缓存起来分区的leader和ISR数据
        controllerContext.partitionLeadershipInfo.put(partition, leaderIsrAndControllerEpoch)

        // isr是在线副本所在的brokerId，这里向这些broker发送LeaderAndIsr请求
        controllerBrokerRequestBatch.addLeaderAndIsrRequestForBrokers(leaderIsrAndControllerEpoch.leaderAndIsr.isr,
          partition, leaderIsrAndControllerEpoch, controllerContext.partitionReplicaAssignment(partition), isNew = true)
        // 作为成功初始化的分区返回
        successfulInitializations += partition
      } else {
        logFailedStateChange(partition, NewPartition, OnlinePartition, code)
      }
    }
    successfulInitializations
}
```

### OfflinePartition/OnlinePartition状态的分区选举leader副本

electLeaderForPartitions方法用于OfflinePartition/OnlinePartition状态的所有分区选举leader副本
而每一个分区的的leader副本选举在doElectLeaderForPartitions方法实现，虽然代码很多，但核心还是选举leader副本，写回zk，更新本地缓存，并发送LeaderAndIsr请求同步给其他broker

分区leader会在不同情况下选举leader副本，因此有4种选举策略，此时根据前面传递过来的参数，选举策略为OfflinePartitionLeaderElectionStrategy
```java
private def doElectLeaderForPartitions(partitions: Seq[TopicPartition], partitionLeaderElectionStrategy: PartitionLeaderElectionStrategy):
  (Seq[TopicPartition], Seq[TopicPartition], Map[TopicPartition, Exception]) = {
    // 先批量获取zk中.../partitions/xxx/state，即每个分区的state 数据
    // 样例： {"controller_epoch":19,"leader":0,"version":1,"leader_epoch":57,"isr":[0,1,2]}
    val getDataResponses = try {
      zkClient.getTopicPartitionStatesRaw(partitions)
    } catch {
      case e: Exception =>
        return (Seq.empty, Seq.empty, partitions.map(_ -> e).toMap)
    }
    val failedElections = mutable.Map.empty[TopicPartition, Exception]
    val leaderIsrAndControllerEpochPerPartition = mutable.Buffer.empty[(TopicPartition, LeaderIsrAndControllerEpoch)]

    // 主要是初始化leaderIsrAndControllerEpochPerPartition
    getDataResponses.foreach { getDataResponse =>
      // context 就是请求参数里的分区
      val partition = getDataResponse.ctx.get.asInstanceOf[TopicPartition]
      val currState = partitionState(partition) // 获取缓存中该分区的状态

      if (getDataResponse.resultCode == Code.OK) {
        // 解析成LeaderIsrAndControllerEpoch
        val leaderIsrAndControllerEpochOpt = TopicPartitionStateZNode.decode(getDataResponse.data, getDataResponse.stat)
        // 没获取到leaderIsrAndControllerEpoch，添加到failedElections集合里
        if (leaderIsrAndControllerEpochOpt.isEmpty) {
          val exception = new StateChangeFailedException(s"LeaderAndIsr information doesn't exist for partition $partition in $currState state")
          failedElections.put(partition, exception)
        }
        leaderIsrAndControllerEpochPerPartition += partition -> leaderIsrAndControllerEpochOpt.get // 加个括号好看些 (partition -> leaderIsrAndControllerEpochOpt.get)
      } else if (getDataResponse.resultCode == Code.NONODE) {
        // 节点不存在
        val exception = new StateChangeFailedException(s"LeaderAndIsr information doesn't exist for partition $partition in $currState state")
        failedElections.put(partition, exception)
      } else {
        // 其他zk异常
        failedElections.put(partition, getDataResponse.resultException.get)
      }
    }

    // zk里的controllerEpoch是否比 本地缓存里的controllerEpoch大，大就说明有其他Controller已经被选举了，写到了zk的partition/state里
    val (invalidPartitionsForElection, validPartitionsForElection) = leaderIsrAndControllerEpochPerPartition.partition { case (_, leaderIsrAndControllerEpoch) =>
      leaderIsrAndControllerEpoch.controllerEpoch > controllerContext.epoch
    }
    invalidPartitionsForElection.foreach { case (partition, leaderIsrAndControllerEpoch) =>
      val failMsg = s"aborted leader election for partition $partition since the LeaderAndIsr path was " +
        s"already written by another controller. This probably means that the current controller $controllerId went through " +
        s"a soft failure and another controller was elected with epoch ${leaderIsrAndControllerEpoch.controllerEpoch}."
      failedElections.put(partition, new StateChangeFailedException(failMsg))
    }
    // 全部分区都被新Controller更新了state，直接返回failedElections
    if (validPartitionsForElection.isEmpty) {
      return (Seq.empty, Seq.empty, failedElections.toMap)
    }

    val shuttingDownBrokers  = controllerContext.shuttingDownBrokerIds.toSet

    val (partitionsWithoutLeaders, partitionsWithLeaders) = partitionLeaderElectionStrategy match {
      case OfflinePartitionLeaderElectionStrategy => // 初始化是用的是OfflinePartitionLeaderElectionStrategy(追参数传递)
        // 注意这里的scala语法，partition是布尔分组器，并返回结果给外边的val变量
        leaderForOffline(validPartitionsForElection).partition { case (_, newLeaderAndIsrOpt, _) => newLeaderAndIsrOpt.isEmpty }
      case ReassignPartitionLeaderElectionStrategy => // 分区重分配时的选举算法
        leaderForReassign(validPartitionsForElection).partition { case (_, newLeaderAndIsrOpt, _) => newLeaderAndIsrOpt.isEmpty }
      case PreferredReplicaPartitionLeaderElectionStrategy =>
        leaderForPreferredReplica(validPartitionsForElection).partition { case (_, newLeaderAndIsrOpt, _) => newLeaderAndIsrOpt.isEmpty }
      case ControlledShutdownPartitionLeaderElectionStrategy =>
        leaderForControlledShutdown(validPartitionsForElection, shuttingDownBrokers).partition { case (_, newLeaderAndIsrOpt, _) => newLeaderAndIsrOpt.isEmpty }
    }
    // 没选举出leader的分区
    partitionsWithoutLeaders.foreach { case (partition, _, _) =>
      val failMsg = s"Failed to elect leader for partition $partition under strategy $partitionLeaderElectionStrategy"
      failedElections.put(partition, new StateChangeFailedException(failMsg))
    }

    // 分区和存活的副本形成一个集合
    val recipientsPerPartition = partitionsWithLeaders.map { case (partition, _, recipients) => partition -> recipients }.toMap
    // 分区和选举后的isr形成一个集合
    val adjustedLeaderAndIsrs = partitionsWithLeaders.map { case (partition, leaderAndIsrOpt, _) => partition -> leaderAndIsrOpt.get }.toMap

    // 更新每个选举成功的分区，更新leaderAndIsr，controller epoch
    val UpdateLeaderAndIsrResult(successfulUpdates, updatesToRetry, failedUpdates) = zkClient.updateLeaderAndIsr(
      adjustedLeaderAndIsrs, controllerContext.epoch)
    // updatesToRetry是版本冲突而更新失败的分区

    successfulUpdates.foreach { case (partition, leaderAndIsr) =>
      val replicas = controllerContext.partitionReplicaAssignment(partition)
      val leaderIsrAndControllerEpoch = LeaderIsrAndControllerEpoch(leaderAndIsr, controllerContext.epoch)
      // zk更新成功，放入本地缓存
      controllerContext.partitionLeadershipInfo.put(partition, leaderIsrAndControllerEpoch)
      // 发送LeaderAndIsr请求
      controllerBrokerRequestBatch.addLeaderAndIsrRequestForBrokers(recipientsPerPartition(partition), partition,
        leaderIsrAndControllerEpoch, replicas, isNew = false)
    }
    (successfulUpdates.keys.toSeq, updatesToRetry, failedElections.toMap ++ failedUpdates) // 这里是选举失败和更新zk失败的合并
}
```

OfflinePartitionLeaderElectionStrategy策略的选举算法在leaderForOffline方法中实现

### leaderForOffline选举
在选举过程中，受unclean.leader.election.enable配置的约束，该配置可以是topic级别，线上环境一般设置为false，否则会在非isr的副本中选举leader，造成数据不一致问题


```java
private def leaderForOffline(leaderIsrAndControllerEpochs: Seq[(TopicPartition, LeaderIsrAndControllerEpoch)]):
  Seq[(TopicPartition, Option[LeaderAndIsr], Seq[Int])] = {
    // 又是布尔分区器
    val (partitionsWithNoLiveInSyncReplicas, partitionsWithLiveInSyncReplicas) = leaderIsrAndControllerEpochs.partition { case (partition, leaderIsrAndControllerEpoch) =>
      // 这是在查看zk中的ISR列表，broker本地缓存中是否存活
      val liveInSyncReplicas = leaderIsrAndControllerEpoch.leaderAndIsr.isr.filter(replica => controllerContext.isReplicaOnline(replica, partition))
      liveInSyncReplicas.isEmpty
    }

    // partitionsWithNoLiveInSyncReplicas的含义：partition.isr.filter(replica => controllerContext.isReplicaOnline(replica, partition)).isEmpty
    // config.originals()就是server.properties里的配置
    // 获取topic的LogConfig配置对象，LogConfig(originals+overrides, overrides.keys)
    val (logConfigs, failed) = zkClient.getLogConfigs(partitionsWithNoLiveInSyncReplicas.map { case (partition, _) => partition.topic }, config.originals())

    val partitionsWithUncleanLeaderElectionState = partitionsWithNoLiveInSyncReplicas.map { case (partition, leaderIsrAndControllerEpoch) =>
      // failed: 从zk获取配置信息失败的topic
      if (failed.contains(partition.topic)) {
        // 打日志
        logFailedStateChange(partition, partitionState(partition), OnlinePartition, failed(partition.topic))
        (partition, None, false)
      } else {
        // 返回的是一个三元组(TopicPartition, Option(LeaderIsrAndControllerEpoch, 该topic"unclean.leader.election.enable"的配置),
        (partition, Option(leaderIsrAndControllerEpoch), logConfigs(partition.topic).uncleanLeaderElectionEnable.booleanValue())
      }
    } ++ partitionsWithLiveInSyncReplicas.map { case (partition, leaderIsrAndControllerEpoch) => (partition, Option(leaderIsrAndControllerEpoch), false) }
    // partitionsWithLiveInSyncReplicas uncleanLeaderElectionEnabled默认为false，说明有isr副本有存活的，就一定从isr里选，哪怕只有1个

    partitionsWithUncleanLeaderElectionState.map { case (partition, leaderIsrAndControllerEpochOpt, uncleanLeaderElectionEnabled) =>
      // 获取副本集合
      val assignment = controllerContext.partitionReplicaAssignment(partition)
      // 再检查本地缓存里的副本是否在线
      val liveReplicas = assignment.filter(replica => controllerContext.isReplicaOnline(replica, partition))
      if (leaderIsrAndControllerEpochOpt.nonEmpty) {
        val leaderIsrAndControllerEpoch = leaderIsrAndControllerEpochOpt.get
        val isr = leaderIsrAndControllerEpoch.leaderAndIsr.isr
        // 选举分区leader的算法
        val leaderOpt = PartitionLeaderElectionAlgorithms.offlinePartitionLeaderElection(assignment, isr, liveReplicas.toSet, uncleanLeaderElectionEnabled, controllerContext)

        val newLeaderAndIsrOpt = leaderOpt.map { leader =>
          // 如果副本是从isr里选出来的，就再过滤检查一遍isr里的副本是否在线
          val newIsr = if (isr.contains(leader)) isr.filter(replica => controllerContext.isReplicaOnline(replica, partition))
          else List(leader)
          // leader epoch+1,返回新的LeaderAndIsr
          leaderIsrAndControllerEpoch.leaderAndIsr.newLeaderAndIsr(leader, newIsr)
        }
        (partition, newLeaderAndIsrOpt, liveReplicas)
      } else {
        (partition, None, liveReplicas)
      }
    }
}
```
而最终的leader选举算法在PartitionLeaderElectionAlgorithms.offlinePartitionLeaderElection方法内实现

### 选举算法

该选举算法也比较简单，找到第一个在isr列表，并且是存活的副本作为leader
如果没有，并且unclean.leader.election.enable=true，从所有副本中取第一个存活的副本作为leader

```java
  /**
    * @param assignment 分区分配的副本
    * @param isr zk中的ISR
    * @param liveReplicas assignment中在线的副本
    * @param uncleanLeaderElectionEnabled
    * @param controllerContext
    * @return
    */
def offlinePartitionLeaderElection(assignment: Seq[Int], isr: Seq[Int], liveReplicas: Set[Int], uncleanLeaderElectionEnabled: Boolean, controllerContext: ControllerContext): Option[Int] = {
  // 从assignment中找第一个liveReplicas和isr都有的replica id
  // 找不到执行orElse逻辑
  assignment.find(id => liveReplicas.contains(id) && isr.contains(id)).orElse {
    if (uncleanLeaderElectionEnabled) { // unclean elect
      // 从assignment找第一个Online的Replica作为leader
      // 也就是说不在isr里的副本也可以参与选举（uncleanLeaderElect）
      val leaderOpt = assignment.find(liveReplicas.contains)
      if (!leaderOpt.isEmpty)
        // metrics
        controllerContext.stats.uncleanLeaderElectionRate.mark()
      leaderOpt
    } else {
      // 所有的副本都不在线
      None
    }
  }
}
```

### 小结

至此副本状态机和分区状态机的启动就算完成了，副本状态机与分区状态机的启动操作，都是先初始化了状态缓存，进行初始化的状态转换，里面做了更新ControllerContext，zk中的数据的操作，而分区状态机还需要为每个分区选举leader副本



