---
title: KafkaController源码分析之LeaderAndIsr请求
date: 2020-03-05 20:48:51
categories: Kafka Tutorial
tags: [kafka,中间件,消息]
toc: true
comments: true
---

>在KafkaController初始化的过程中，多次遇见了LeaderAndIsr请求，这是broker之间通信的一个重要请求，它也是副本同步的关键步骤，本文主要分析KafkaApis对该请求的处理

# ControllerChannelManager
在讲解LeaderAndIsr请求之前，我们先来看下ControllerChannelManager，在[kafka-server端源码分析之Controller选举与初始化]()我曾提到过它，说它是broker之间通信的管理器，那么它是如何工作的呢？

## 又见内存队列

和ControllerEventManager一样，ControllerChannelManager也是用的异步内存队列来处理请求的发送，它的大致原理如下：
1. ControllerBrokerRequestBatch用3个Map分别维护了leaderAndIsrRequest，stopReplicaRequest，updateMetadataRequest三种请求的缓存
2. 当KafkaController等组件想要发送请求时，仅仅是通过addXXXRequestForBrokers方法，将请求参数添加到缓存中，而在调用sendRequestsToBrokers方法后，它会遍历3中请求的缓存，将请求参数，回调函数等封装为QueueItem对象，放入一个类型为BlockingQueue[QueueItem]的messageQueue中
3. 在RequestSendThread线程启动后，从messageQueue中取出请求对象，发送请求，响应后调用回调函数进行处理

请求流程如下
![流程图](https://ae01.alicdn.com/kf/H46db3f4073a847b2b51715f4fc5ad88eH.png)

## 请求对象解析

### 添加LeaderAndIsr请求到缓存

虽然这个方法很简单，但我需要提2个关键点

1. 第一个参数叫brokerIds，但是调用时传的是replicaIds或者Isr，这里要加强大家对副本id即brokerId的印象
2. 注意最后面还添加了一个UpdateMetadata请求

```java
/**
  * @param brokerIds 通常是副本id，这里也是要请求的目标broker
  * @param topicPartition 分区
  * @param leaderIsrAndControllerEpoch leader， isr，controllerEpoch
  * @param replicas 通常是controllerContext缓存的分区副本集合
  * @param isNew 新建副本，新建分区是为true
  */
def addLeaderAndIsrRequestForBrokers(brokerIds: Seq[Int], topicPartition: TopicPartition,
                                     leaderIsrAndControllerEpoch: LeaderIsrAndControllerEpoch,
                                     replicas: Seq[Int], isNew: Boolean) {
  brokerIds.filter(_ >= 0).foreach { brokerId =>
    // 每个broker的LeaderAndIsr请求都有一个缓存
    // result: Map[TopicPartition, LeaderAndIsrRequest.PartitionState]
    val result = leaderAndIsrRequestMap.getOrElseUpdate(brokerId, mutable.Map.empty)
    val alreadyNew = result.get(topicPartition).exists(_.isNew)
    // 添加到目标broker的 leaderAndIsr请求队列中
    result.put(topicPartition, new LeaderAndIsrRequest.PartitionState(leaderIsrAndControllerEpoch.controllerEpoch,
      leaderIsrAndControllerEpoch.leaderAndIsr.leader,
      leaderIsrAndControllerEpoch.leaderAndIsr.leaderEpoch,
      leaderIsrAndControllerEpoch.leaderAndIsr.isr.map(Integer.valueOf).asJava,
      leaderIsrAndControllerEpoch.leaderAndIsr.zkVersion,
      replicas.map(Integer.valueOf).asJava,
      isNew || alreadyNew))
  }
  // 同时增加了一次UpdateMetadata请求
  addUpdateMetadataRequestForBrokers(controllerContext.liveOrShuttingDownBrokerIds.toSeq, Set(topicPartition))
}
```

请求体的推理需要点篇幅，我这里直接贴出一个请求的样例json, 其中isNew只有在新分区，新副本请求时才为true,leaderEpoch在后续的副本同步会讲到，用于保证recovery时的数据一致性
liveLeaders表示的是上面partitionStates参数中每个分区leader所在的broker(存活的)
```json
{
    "version":1,
    "controllerId":1,
    "controllerEpoch":1,
    "partitionStates":[
        {
            "TopicPartition":"test-0",
            "PartitionState":{
                "isNew":false,
                "basePartitionState":{
                    "controllerEpoch":1,
                    "leader":1,
                    "leaderEpoch":1,
                    "isr":[ 0,1,2],
                    "zkVersion":1,
                    "replicas":[ 0,1,2]
                }
            }
        }
    ],
    "liveLeaders":[
        {
            "id":1,
            "idString":"1",
            "host":"localhost",
            "port":9092,
            "rack":"rack-1"
        }
    ]
}
```

# 打破思维定式的假设

这里我主要想分享一点我的经验，不要死心眼的认为broker有3个，比如现在的情况是

15台broker，有一个叫test的topic，它有12个分区，每个分区3个副本，以第一个分区test-0为例，它目前的leader是8，即第8台broker上的test-0分区的副本是leader， ISR列表为[8,10,14], 它的replica是[8,10,14]，即所有副本都在同步列表

现在Controller是broker-0，LeaderAndISR请求需要变更**一批**分区的信息，其中刚好有一个要把test-0的leader变为10，因此它要向broker 8，10，14发送LeaderAndIsr请求，下面的请求讲解都以这个为例

| topic分区 | 原leader副本 | 原ISR与Replica | 变更后的leader副本 | 变更后的ISR与Replica | 场景          |
| --------- | ------------ | -------------- | ------------------ | -------------------- | ------------- |
| foo-1     | 9            | [5,9,10]       | 5                  | [5,9,10]             | Preferred选举 |
| test-0    | 8            | [8,10,14]      | 10                 | [8,10,14]            | leader换选    |
| bar-1     | 4            | [4,10,12]       | 4                  | [4,7,10,16,19]       | 副本重分配    |

注：只有15台broker，最后一个有16，19不是我写错了

可以看到这一批LeaderAndIsr请求要发送到多个broker，leaderAndIsrRequestMap的类型是Map[brokerId, Map[TopicPartition, LeaderAndIsrRequest.PartitionState]],发送的代码如下

```java
leaderAndIsrRequestMap.foreach { case (broker, leaderAndIsrPartitionStates) =>
  leaderAndIsrPartitionStates.foreach { case (topicPartition, state) =>
    
  val leaderIds = leaderAndIsrPartitionStates.map(_._2.basePartitionState.leader).toSet

  val leaders = controllerContext.liveOrShuttingDownBrokers.filter(b => leaderIds.contains(b.id)).map {
    _.node(controller.config.interBrokerListenerName)
  }

  val leaderAndIsrRequestBuilder = new LeaderAndIsrRequest.Builder(leaderAndIsrRequestVersion, controllerId,
    controllerEpoch, leaderAndIsrPartitionStates.asJava, leaders.asJava)

  controller.sendRequest(broker, ApiKeys.LEADER_AND_ISR, leaderAndIsrRequestBuilder,
    (r: AbstractResponse) => controller.eventManager.put(controller.LeaderAndIsrResponseReceived(r, broker)))
} 
```
可以看到kafka的本意就是积攒一批请求，然后按照brokerId分组，再发送出去，和生产者发送消息是同样的味道

再看上面的表格，变更后的ISR与Replica列表就是我们要发送的broker，我这里故意让三个分区都包含10，那么我们往broker-10发送的LeaderAndIsr请求同时包含3个分区的信息变更请求


# KafkaApis处理LeaderAndIsr请求

LeaderAndIsr请求由handleLeaderAndIsrRequest方法处理，仅做了2件事：定义回调函数，认证预处理，关键的处理在调用的becomeLeaderOrFollower方法中

```java
  def handleLeaderAndIsrRequest(request: RequestChannel.Request) {
    val correlationId = request.header.correlationId
    val leaderAndIsrRequest = request.body[LeaderAndIsrRequest]

    def onLeadershipChange(updatedLeaders: Iterable[Partition], updatedFollowers: Iterable[Partition]) {
      // 按惯例，事先定义好的回调函数先不看，扰乱我们的视线
    }

    if (authorize(request.session, ClusterAction, Resource.ClusterResource)) { // 认证步骤不在细究
      val response = replicaManager.becomeLeaderOrFollower(correlationId, leaderAndIsrRequest, onLeadershipChange)
      sendResponseExemptThrottle(request, response)
    } 
    // 省略
  }
```

## becomeLeaderOrFollower

该方法分为三段，前面部分只是做了下检查，中间部分是我们重点关注的

```java
def becomeLeaderOrFollower(correlationId: Int,
                           leaderAndIsrRequest: LeaderAndIsrRequest,
                           onLeadershipChange: (Iterable[Partition], Iterable[Partition]) => Unit): LeaderAndIsrResponse = {

  replicaStateChangeLock synchronized {
    if (leaderAndIsrRequest.controllerEpoch < controllerEpoch) {
      // Controller已换届，忽略leaderAndIsr请求，即请求过期
     
      leaderAndIsrRequest.getErrorResponse(0, Errors.STALE_CONTROLLER_EPOCH.exception)
    } else {
      val responseMap = new mutable.HashMap[TopicPartition, Errors]

      val controllerId = leaderAndIsrRequest.controllerId
      // 更新controllerEpoch，记录了最新一次执行LeaderAndIsr请求的controllerEpoch
      // controller选举必定会发生LeaderAndIsr请求
      controllerEpoch = leaderAndIsrRequest.controllerEpoch

      // First check partition's leader epoch
      val partitionState = new mutable.HashMap[Partition, LeaderAndIsrRequest.PartitionState]()

      // 缓存里没有的是新分区
      val newPartitions = leaderAndIsrRequest.partitionStates.asScala.keys.filter(topicPartition => getPartition(topicPartition).isEmpty)

      // 一堆检查，省略部分代码... 
      leaderAndIsrRequest.partitionStates.asScala.foreach { case (topicPartition, stateInfo) =>
        val partition = getOrCreatePartition(topicPartition) // 返回的是Partition对象,新的partition有Pool的valueFactory初始化
        val partitionLeaderEpoch = partition.getLeaderEpoch
        if (partitionLeaderEpoch < stateInfo.basePartitionState.leaderEpoch) { 
          // 本地缓存的leader epoch要比请求中的leader epoch小，因为请求里的leader epoch是加1了的
          // 最终想要的数据
          partitionState.put(partition, stateInfo)
        }
      }

      // ================重点关注下面的代码==================

      // 过滤出leader是当前broker的分区，*要将当前broker上的副本变为leader* 这句话最重要
      val partitionsTobeLeader = partitionState.filter { case (_, stateInfo) =>
        stateInfo.basePartitionState.leader == localBrokerId
      }
      // 其余的副本在当前broker都是follower
      val partitionsToBeFollower = partitionState -- partitionsTobeLeader.keys

      val partitionsBecomeLeader = if (partitionsTobeLeader.nonEmpty)
        // 标记为leader的分区
        makeLeaders(controllerId, controllerEpoch, partitionsTobeLeader, correlationId, responseMap)
      else
        Set.empty[Partition]

      val partitionsBecomeFollower = if (partitionsToBeFollower.nonEmpty)
      // 标记为follower的分区
        makeFollowers(controllerId, controllerEpoch, partitionsToBeFollower, correlationId, responseMap)
      else
        Set.empty[Partition]

      // ================重点关注==================


      // 先不看 ....
      leaderAndIsrRequest.partitionStates.asScala.keys.foreach(topicPartition =>
        if (getReplica(topicPartition).isEmpty && (allPartitions.get(topicPartition) ne ReplicaManager.OfflinePartition))
          allPartitions.put(topicPartition, ReplicaManager.OfflinePartition)
      )

      if (!hwThreadInitialized) {
        startHighWaterMarksCheckPointThread()
        hwThreadInitialized = true
      }

      val newOnlineReplicas = newPartitions.flatMap(topicPartition => getReplica(topicPartition))
      val futureReplicasAndInitialOffset = newOnlineReplicas.filter { replica =>
        logManager.getLog(replica.topicPartition, isFuture = true).isDefined
      }.map { replica =>
        replica.topicPartition -> BrokerAndInitialOffset(BrokerEndPoint(config.brokerId, "localhost", -1), replica.highWatermark.messageOffset)
      }.toMap
      futureReplicasAndInitialOffset.keys.foreach(tp => getPartition(tp).get.getOrCreateReplica(Request.FutureLocalReplicaId))

      futureReplicasAndInitialOffset.keys.foreach(logManager.abortAndPauseCleaning)
      replicaAlterLogDirsManager.addFetcherForPartitions(futureReplicasAndInitialOffset)

      replicaFetcherManager.shutdownIdleFetcherThreads()
      replicaAlterLogDirsManager.shutdownIdleFetcherThreads()
      onLeadershipChange(partitionsBecomeLeader, partitionsBecomeFollower)
      new LeaderAndIsrResponse(Errors.NONE, responseMap.asJava)
    }
  }
}
```

| topic分区 | 原leader副本 | 原ISR与Replica | 变更后的leader副本 | 变更后的ISR与Replica | 场景          |
| --------- | ------------ | -------------- | ------------------ | -------------------- | ------------- |
| foo-1     | 9            | [5,9,10]       | 5                  | [5,9,10]             | Preferred选举 |
| test-0    | 8            | [8,10,14]      | 10                 | [8,10,14]            | leader换选    |
| bar-1     | 4            | [4,10,12]       | 4                  | [4,7,10,16,19]       | 副本重分配    |

根据前面的假设，broker-0(controller)向broker-10发送了一批分区变更需求请求，假设当前处理请求的是broker-10，先进行分组

partitionsTobeLeader: 变更后的leader是当前broker，也就是broker-10的一组，即test-0分区，用makeLeaders方法处理

partitionsBecomeLeader: 其他的分区一组，即foo-1，bar-1，用makeFollowers方法处理

我们先看makeLeaders方法，它首先停止了这些副本的同步操作，然后遍历每个分区处理
```java
private def makeLeaders(controllerId: Int,
                          epoch: Int,
                          partitionState: Map[Partition, LeaderAndIsrRequest.PartitionState],
                          correlationId: Int,
                          responseMap: mutable.Map[TopicPartition, Errors]): Set[Partition] = {
  // 返回结果
  val partitionsToMakeLeaders = mutable.Set[Partition]()
  // 从fetch线程中移除这些分区副本的同步操作
  replicaFetcherManager.removeFetcherForPartitions(partitionState.keySet.map(_.topicPartition))
  //遍历每一个分区，调用makeLeader
  partitionState.foreach{ case (partition, partitionStateInfo) =>
    if (partition.makeLeader(controllerId, partitionStateInfo, correlationId)) {
      partitionsToMakeLeaders += partition
    }
}
```
那么我们直接来到Partition的makeLeader方法，它是处理单个分区信息变更的方法

## makeLeader

看代码之前先稍微解释下leader epoch以及leader-epoch-checkpoint文件

leader epoch在分区的leader副本变更时更新，每次更新加1，相当于记录了分区leader的更新次数，也可以理解为leader的版本号
leader-epoch-checkpoint在每一个分区日志目录都有一个，这里以topic为test-1,分区为0的日志目录为例
它的内容是一个key value，key是leader epoch，value是上一代leader的LEO，我们知道LEO是即将写入的下一条消息的offset，这里也可以理解为新leader要写入的第一条消息

![leader-epoch-checkpoint文件位置](https://ae01.alicdn.com/kf/Hd55131d904654aaf801fcca7b0cef015s.png)
它里面的内容一般是这样的，其他check-point文件也是同理
```json
0
1
2 9832
```
第一行的0表示版本号，第二行表示记录个数，第三行才是真正的数据

言归正传，继续看makeLeader方法。该方法更新了本地的一些缓存，如controllerEpoch，inSyncReplicas，leaderEpoch，leaderEpochStartOffsetOpt(上面说的value)，zkVersion。接着更新了check-point文件

最后是关于新的leader副本的处理，比如初始化它的HW

```java
def makeLeader(controllerId: Int, partitionStateInfo: LeaderAndIsrRequest.PartitionState, correlationId: Int): Boolean = {
  val (leaderHWIncremented, isNewLeader) = inWriteLock(leaderIsrUpdateLock) {
    // 请求中的AR
    val newAssignedReplicas = partitionStateInfo.basePartitionState.replicas.asScala.map(_.toInt)
    
    // Partition里也有一份controllerEpoch，更新
    controllerEpoch = partitionStateInfo.basePartitionState.controllerEpoch

    // 获取isr对应的Replica
    val newInSyncReplicas = partitionStateInfo.basePartitionState.isr.asScala.map(r => getOrCreateReplica(r, partitionStateInfo.isNew)).toSet

    // 副本重分配场景： 该分区已有的副本-新分配的副本=controller要移除的副本，从本地缓存allReplicasMap = new Pool[Int, Replica]中删除
    (assignedReplicas.map(_.brokerId) -- newAssignedReplicas).foreach(removeReplica)

    // 新的isr是controller传过来的,更新
    inSyncReplicas = newInSyncReplicas

    // 获取replicas对应的Replica
    newAssignedReplicas.foreach(id => getOrCreateReplica(id, partitionStateInfo.isNew))

    // 不是说当前replica是leader副本，而是说它即将要成为leader
    val leaderReplica = getReplica().get
    // 获取leader副本的LEO
    val leaderEpochStartOffset = leaderReplica.logEndOffset.messageOffset

    // 更新leaderEpoch，以及这一届leaderEpoch对应的StartOffset
    leaderEpoch = partitionStateInfo.basePartitionState.leaderEpoch
    leaderEpochStartOffsetOpt = Some(leaderEpochStartOffset)
    zkVersion = partitionStateInfo.basePartitionState.zkVersion

    // 将leader epoch及其开始位移写入文件
    leaderReplica.epochs.foreach { epochCache =>
      epochCache.assign(leaderEpoch, leaderEpochStartOffset)
    }

    // 如果分区的leader副本就是当前broker，就不用变更了
    // 注：看becomeLeaderOrFollower方法的星号注释
    val isNewLeader = !leaderReplicaIdOpt.contains(localBrokerId)

    val curLeaderLogEndOffset = leaderReplica.logEndOffset.messageOffset // leader副本的LEO
    val curTimeMs = time.milliseconds
    // initialize lastCaughtUpTime of replicas as well as their lastFetchTimeMs and lastFetchLeaderLogEndOffset.
    // 更新副本的同步时间，LEO
    (assignedReplicas - leaderReplica).foreach { replica =>
      val lastCaughtUpTimeMs = if (inSyncReplicas.contains(replica)) curTimeMs else 0L
      replica.resetLastCaughtUpTime(curLeaderLogEndOffset, curTimeMs, lastCaughtUpTimeMs)
    }

    if (isNewLeader) {
      // construct the high watermark metadata for the new leader replica
      // 初始化HW(大概率就是当前的HW)
      leaderReplica.convertHWToLocalOffsetMetadata()
      // mark local replica as the leader after converting hw
      leaderReplicaIdOpt = Some(localBrokerId)
      // reset log end offset for remote replicas
      // 初始化同步相关的一堆参数
      assignedReplicas.filter(_.brokerId != localBrokerId).foreach(_.updateLogReadResult(LogReadResult.UnknownLogReadResult))
    }
    // we may need to increment high watermark since ISR could be down to 1
    (maybeIncrementLeaderHW(leaderReplica), isNewLeader)
  }
  // some delayed operations may be unblocked after HW changed
  if (leaderHWIncremented)
    // HW增加了，fetch请求的max.byte，produce请求的ack=-1等待副本同步就可以try complete了，
    tryCompleteDelayedRequests()
  isNewLeader
}
```
makeLeader的作用可以简单归纳为：
1. 更新本地缓存数据
2. 更新leader epoch到文件
3. 如果本地副本不是leader，那么初始化它的HW，以及同步相关的参数

处理流程如下：
![makeLeader流程](https://ae01.alicdn.com/kf/H5ca87b4ad1634fd0b8a57919c132d56eh.png)

## Follower副本处理

在[becomeLeaderOrFollower](#becomeLeaderOrFollower)方法中，makeLeaders处理完leader副本后，makeFollowers方法处理follower副本
该方法同样是遍历每一个分区

注：该方法源码很长，但是都是打印日志，这里删除了很多源码

```java
private def makeFollowers(controllerId: Int,
                            epoch: Int,
                            partitionStates: Map[Partition, LeaderAndIsrRequest.PartitionState],
                            correlationId: Int,
                            responseMap: mutable.Map[TopicPartition, Errors]) : Set[Partition] = {


  // 定义返回结果
  for (partition <- partitionStates.keys)
    responseMap.put(partition.topicPartition, Errors.NONE)

  // 记录转变为follower副本的分区
  val partitionsToMakeFollower: mutable.Set[Partition] = mutable.Set()

  partitionStates.foreach { case (partition, partitionStateInfo) =>
    val newLeaderBrokerId = partitionStateInfo.basePartitionState.leader
      // 找到leader所在的broker
      metadataCache.getAliveBrokers.find(_.id == newLeaderBrokerId) match {
        case Some(_) =>
          // makeFollower做主要初始化及更新操作
          if (partition.makeFollower(controllerId, partitionStateInfo, correlationId))
            partitionsToMakeFollower += partition
        case None =>
          // 没有就创建，这在分区副本重分配时有用
          partition.getOrCreateReplica(isNew = partitionStateInfo.isNew)
      }

  // leader要发生改变，不能再从以前的leader同步
  replicaFetcherManager.removeFetcherForPartitions(partitionsToMakeFollower.map(_.topicPartition))

  // 尝试完成一些延迟请求
  partitionsToMakeFollower.foreach { partition =>
    val topicPartitionOperationKey = new TopicPartitionOperationKey(partition.topicPartition)
    tryCompleteDelayedProduce(topicPartitionOperationKey)
    tryCompleteDelayedFetch(topicPartitionOperationKey)
  }

  // broker在关闭了
  if (isShuttingDown.get()) {
    // 记录日志....
  }
  else {
    // 正常处理
    // we do not need to check if the leader exists again since this has been done at the beginning of this process
    val partitionsToMakeFollowerWithLeaderAndOffset = partitionsToMakeFollower.map(partition =>
      // leader所在的broker和当前broker副本的HW作为初始同步位移
      partition.topicPartition -> BrokerAndInitialOffset(
        metadataCache.getAliveBrokers.find(_.id == partition.leaderReplicaIdOpt.get).get.brokerEndPoint(config.interBrokerListenerName),
        partition.getReplica().get.highWatermark.messageOffset)).toMap
    // 添加到副本到同步线程
    replicaFetcherManager.addFetcherForPartitions(partitionsToMakeFollowerWithLeaderAndOffset)
  }
  partitionsToMakeFollower
}
```

上面调用的makeFollower和makeLeader方法类似
```java
def makeFollower(controllerId: Int, partitionStateInfo: LeaderAndIsrRequest.PartitionState, correlationId: Int): Boolean = {
  inWriteLock(leaderIsrUpdateLock) {
    val newAssignedReplicas = partitionStateInfo.basePartitionState.replicas.asScala.map(_.toInt)
    val newLeaderBrokerId = partitionStateInfo.basePartitionState.leader
    val oldLeaderEpoch = leaderEpoch
    // record the epoch of the controller that made the leadership decision. This is useful while updating the isr
    // to maintain the decision maker controller's epoch in the zookeeper path
    controllerEpoch = partitionStateInfo.basePartitionState.controllerEpoch
    // add replicas that are new
    newAssignedReplicas.foreach(r => getOrCreateReplica(r, partitionStateInfo.isNew))
    // remove assigned replicas that have been removed by the controller
    // 删除缓存里不要的副本了
    (assignedReplicas.map(_.brokerId) -- newAssignedReplicas).foreach(removeReplica)

    inSyncReplicas = Set.empty[Replica] 
    leaderEpoch = partitionStateInfo.basePartitionState.leaderEpoch
    leaderEpochStartOffsetOpt = None
    zkVersion = partitionStateInfo.basePartitionState.zkVersion


    // leader是否更新了
    if (leaderReplicaIdOpt.contains(newLeaderBrokerId) && (leaderEpoch == oldLeaderEpoch || leaderEpoch == oldLeaderEpoch + 1)) {
      false
    }
    else {
      leaderReplicaIdOpt = Some(newLeaderBrokerId)
      true
    }
  }
}
```

makeFollowers主要判断分区的leader副本是否发生了改变，如果改变了，就先移除原来的同步，重新向新leader同步

![makeFollowers](https://ae01.alicdn.com/kf/Hee2c2db7c82843adb80e2ccdf67f07b2j.png)


## becomeLeaderOrFollower第三部分

becomeLeaderOrFollower在调用makeLeaders和makeFollowers之后，处理的源码如下

```java
leaderAndIsrRequest.partitionStates.asScala.keys.foreach(topicPartition =>
  // 判断离线分区
  if (getReplica(topicPartition).isEmpty && (allPartitions.get(topicPartition) ne ReplicaManager.OfflinePartition))
    allPartitions.put(topicPartition, ReplicaManager.OfflinePartition)
)

// 为初始化的LeaderAndIsr请求启动hw的check线程，记录到recovery-point-offset-checkpoint文件
if (!hwThreadInitialized) {
  startHighWaterMarksCheckPointThread()
  hwThreadInitialized = true
}

val newOnlineReplicas = newPartitions.flatMap(topicPartition => getReplica(topicPartition))
// Add future replica to partition's map
val futureReplicasAndInitialOffset = newOnlineReplicas.filter { replica =>
  // 新副本就是isFuture副本
  logManager.getLog(replica.topicPartition, isFuture = true).isDefined
}.map { replica =>
  replica.topicPartition -> BrokerAndInitialOffset(BrokerEndPoint(config.brokerId, "localhost", -1), replica.highWatermark.messageOffset)
}.toMap
futureReplicasAndInitialOffset.keys.foreach(tp => getPartition(tp).get.getOrCreateReplica(Request.FutureLocalReplicaId))

// pause cleaning for partitions that are being moved and start ReplicaAlterDirThread to move replica from source dir to destination dir
futureReplicasAndInitialOffset.keys.foreach(logManager.abortAndPauseCleaning)
replicaAlterLogDirsManager.addFetcherForPartitions(futureReplicasAndInitialOffset)

// 看是否有空闲的fetcher线程
replicaFetcherManager.shutdownIdleFetcherThreads()
replicaAlterLogDirsManager.shutdownIdleFetcherThreads()

// 调用handleLeaderAndIsrRequest中的回调函数
onLeadershipChange(partitionsBecomeLeader, partitionsBecomeFollower)
new LeaderAndIsrResponse(Errors.NONE, responseMap.asJava)
```

该部分主要是对future Replica的处理，它们会同步leader副本，之后清空空闲的fetcher线程，这里大家先理解一个fetcher线程管理了多个follower的同步
最后调用handleLeaderAndIsrRequest中的回调函数：onLeadershipChange，下面是该方法的源码

## 内部topic的特殊处理

kafka内部的topic有2个：`__consumer_offsets`和`__transaction_state`，它们的LeaderAndIsr处理比较复杂，这里不再展开细说。
```java
def onLeadershipChange(updatedLeaders: Iterable[Partition], updatedFollowers: Iterable[Partition]) {
  // for each new leader or follower, call coordinator to handle consumer group migration.
  // this callback is invoked under the replica state change lock to ensure proper order of
  // leadership changes
  updatedLeaders.foreach { partition =>
    if (partition.topic == GROUP_METADATA_TOPIC_NAME)
      groupCoordinator.handleGroupImmigration(partition.partitionId)
    else if (partition.topic == TRANSACTION_STATE_TOPIC_NAME)
      txnCoordinator.handleTxnImmigration(partition.partitionId, partition.getLeaderEpoch)
  }

  updatedFollowers.foreach { partition =>
    if (partition.topic == GROUP_METADATA_TOPIC_NAME)
      groupCoordinator.handleGroupEmigration(partition.partitionId)
    else if (partition.topic == TRANSACTION_STATE_TOPIC_NAME)
      txnCoordinator.handleTxnEmigration(partition.partitionId, partition.getLeaderEpoch)
  }
}
```

# 总结

LeaderAndIsr请求是在分区leader或者副本集合发生变更时，Controller向其它broker发生的请求，broker在接收到请求后会看分区的新leader是否是当前broker的id

1. 如果是，则先暂停该分区本地副本的同步，因为它们从follower变为leader了，然后更新元数据，记录leader epoch checkpoint等，最终初始化当前副本为leader副本
2. 如果不是，则本地broker上的副本为follower副本,同样的更新本地缓存的元数据，此时按leader是否发生了改变分为2中情况
  1. leader改变了，那么移除当前同步线程对这些副本的同步，重新定位leader所在broker，以当前副本的HW为起始位移加入到副本同步线程中去
  2. leader没有变，什么都不做















