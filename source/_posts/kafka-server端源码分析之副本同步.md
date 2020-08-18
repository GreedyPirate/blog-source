---
title: kafka server端源码分析之副本同步
date: 2020-03-08 17:26:44
categories: Kafka Tutorial
tags: [kafka,中间件,消息]
toc: true
comments: true
---

# 前言

为什么我现在才写副本同步的解析呢，因为它太复杂了，仅仅是什么时候触发的副本同步，就涉及到KafkaController，LeaderAndIsr请求等，经过前面文章的梳理，现在时机正好

# 正文

通常我们会为了提高系统并发能力、可伸缩性，为topic设置多个分区，每个分区副本数通常设置为3个，其中1个为leader副本，其余2个follower副本为冗余备份使用。在producer端为了保证消息不丢失，通常设置ack=-1，并搭配失败重试机制

本文主要讨论broker端写入leader副本后，follower副本如何同步消息，以及如何更新HighWatermark，并使Purgatory延迟队列中的PRODUCE请求完成(complete)，响应客户端


## 副本拉取管理器

在Kafka启动时，会初始化ReplicaManager副本管理器，同时该类中有一行初始化语句
```java
val replicaFetcherManager = createReplicaFetcherManager(metrics, time, threadNamePrefix, quotaManagers.follower)
```
其实就是new了一个ReplicaFetcherManager对象，该对象的功能十分简单，就是创建和关闭Fetch线程

```java
class ReplicaFetcherManager(brokerConfig: KafkaConfig, protected val replicaManager: ReplicaManager, metrics: Metrics,
                            time: Time, threadNamePrefix: Option[String] = None, quotaManager: ReplicationQuotaManager)
      extends AbstractFetcherManager("ReplicaFetcherManager on broker " + brokerConfig.brokerId,
        "Replica", brokerConfig.numReplicaFetchers) {

  override def createFetcherThread(fetcherId: Int, sourceBroker: BrokerEndPoint): AbstractFetcherThread = {
    val prefix = threadNamePrefix.map(tp => s"${tp}:").getOrElse("")
    val threadName = s"${prefix}ReplicaFetcherThread-$fetcherId-${sourceBroker.id}"
    new ReplicaFetcherThread(threadName, fetcherId, sourceBroker, brokerConfig, replicaManager, metrics, time, quotaManager)
  }

  def shutdown() {
    info("shutting down")
    closeAllFetchers()
    info("shutdown completed")
  }
}
```

# 副本同步

在分析副本同步过程之前，我们先想一想什么时候开始同步，也就是上面的createFetcherThread什么时候创建并启动的

## 何时同步

这里就要回顾[KafkaController源码分析之LeaderAndIsr请求]()一文了，这也是我先写LeaderAndIsr，然后才分析副本同步的原因

在前文中，提到了becomeLeaderOrFollower方法会将分区添加到副本同步线程中，具体实现就在addFetcherForPartitions方法中

```java
def addFetcherForPartitions(partitionAndOffsets: Map[TopicPartition, BrokerAndInitialOffset]) {
    lock synchronized {
      // partitionsPerFetcher = Map[BrokerAndFetcherId, Map[TopicPartition, BrokerAndInitialOffset]]
      // 分组的key是目标broker+同步线程，也就是同一个fetcher线程向同一个broker同步 为一组
      val partitionsPerFetcher = partitionAndOffsets.groupBy { case(topicPartition, brokerAndInitialFetchOffset) =>
        BrokerAndFetcherId(brokerAndInitialFetchOffset.broker, getFetcherId(topicPartition.topic, topicPartition.partition))}

      // 事先定义好创建并启动的方法
      def addAndStartFetcherThread(brokerAndFetcherId: BrokerAndFetcherId, brokerIdAndFetcherId: BrokerIdAndFetcherId) {
        val fetcherThread = createFetcherThread(brokerAndFetcherId.fetcherId, brokerAndFetcherId.broker)
        fetcherThreadMap.put(brokerIdAndFetcherId, fetcherThread)
        fetcherThread.start
      }

      for ((brokerAndFetcherId, initialFetchOffsets) <- partitionsPerFetcher) {
        val brokerIdAndFetcherId = BrokerIdAndFetcherId(brokerAndFetcherId.broker.id, brokerAndFetcherId.fetcherId)
        // fetcherThreadMap: Map[BrokerIdAndFetcherId, AbstractFetcherThread]
        // 这里的逻辑还是很清晰的
        fetcherThreadMap.get(brokerIdAndFetcherId) match {
            // 已存在对应的Thread，并且线程的broker和分区要同步的broker相同，直接复用就行了
          case Some(f) if f.sourceBroker.host == brokerAndFetcherId.broker.host && f.sourceBroker.port == brokerAndFetcherId.broker.port =>
            // reuse the fetcher thread
          case Some(f) =>
            // 如果前面的if不成立，就需要关闭，重新添加并启动
            f.shutdown()
            addAndStartFetcherThread(brokerAndFetcherId, brokerIdAndFetcherId)
          case None =>
            // 没有就创建
            addAndStartFetcherThread(brokerAndFetcherId, brokerIdAndFetcherId)
        }

        fetcherThreadMap(brokerIdAndFetcherId).addPartitions(initialFetchOffsets.map { case (tp, brokerAndInitOffset) =>
          tp -> brokerAndInitOffset.initOffset
        })
      }
    }
}
```

可以看到Fetcher线程的启动是通过addAndStartFetcherThread方法实现的，createFetcherThread刚好调用了前面的ReplicaFetcherManager

同时我们注意一下createFetcherThread方法的第二个参数传入的是broker，那么我们可以得出以下结论
1. 一个fetcher线程只会向一个broker同步
2. 一个fetcher线程管理了本地broker多个分区的同步，它和消费者一样都是发送的FETCH请求，此时我们就把它看做一个消费者，和消费者线程一样可以拉取多个分区的消息

ReplicaFetcherThread的类图如下，执行的主体在它的父类AbstractFetcherThread的doWork方法中，具体的fetch逻辑由子类实现，典型的模板模式
![fetch-thread](https://pic.downk.cc/item/5e96a9c6c2a9a83be5876153.png)

## 同步过程分解

副本同步表面看只是follower单向地向leader发送fetch请求，但不要忘了ISR这个概念，follower同步不及时会触发ISR的shrink，那么怎么判断follower同步是否及时能？很简单，在leader副本端维护一个时间戳，记录follower副本每次同步的时间，超出replica.lag.time.max.ms(默认10s)就代表follower副本同步太慢

那么我们在看副本同步时，就要站在更高的一个视角去看，一边是follower，一边是leader。

# follower副本端同步

通过前面的信息我们知道同步是从AbstractFetcherThread的doWork方法开始的，需要说明的是该方法是在一个while循环中一直执行，也就是说副本同步是一个不间断的操作，下面就从它的源码开始分析

## 同步线程doWork

AbstractFetcherThread的doWork方法是副本同步的入口，其中maybeTruncate是0.11版本之后，副本恢复的截断协议从HW改为leader epoch方式，过程较为复杂，后续会单独分析
剩下的步骤就是构建fetch请求，然后调用processFetchRequest进行请求发送及响应处理
```java
override def doWork() {
  maybeTruncate()
  // 构建fetch请求
  val fetchRequest = inLock(partitionMapLock) {
    val ResultWithPartitions(fetchRequest, partitionsWithError) = buildFetchRequest(states)
    if (fetchRequest.isEmpty) {
      trace(s"There are no active partitions. Back off for $fetchBackOffMs ms before sending a fetch request")
      // replica.fetch.backoff.ms
      partitionMapCond.await(fetchBackOffMs, TimeUnit.MILLISECONDS)
    }
    handlePartitionsWithErrors(partitionsWithError)
    fetchRequest
  }
  if (!fetchRequest.isEmpty)
    processFetchRequest(fetchRequest)
}
```

## 发送fetch请求

processFetchRequest表示follower向leader发送fetch请求，然后对响应结果处理。而在leader副本端是如何处理该请求的，在[kafka-server端源码分析之拉取消息]()一文中已基本描述，但是留下了一个ReplicaManager的updateFollowerLogReadResults方法没有讲解，我们按照顺序，先补一下leader端的处理，看看updateFollowerLogReadResults到底做了什么

```java
private def processFetchRequest(fetchRequest: REQ) {
    val partitionsWithError = mutable.Set[TopicPartition]()
    var responseData: Seq[(TopicPartition, PD)] = Seq.empty

    responseData = fetch(fetchRequest)
    
    fetcherStats.requestRate.mark()

    // 响应结果处理下文讲解 ...
}
```

# leader记录follower副本的同步状态

updateFollowerLogReadResults的作用就是leader端记录follower副本的同步状态，例如上一次达到同步状态的时间点，上一次follower副本发送fetch请求的时间点等，依据这些信息，leader副本才能判断出follower副本能否在ISR列表中。
下面看看源码是如何实现的

```java
private def updateFollowerLogReadResults(replicaId: Int,
                                           readResults: Seq[(TopicPartition, LogReadResult)]): Seq[(TopicPartition, LogReadResult)] = {
    debug(s"Recording follower broker $replicaId log end offsets: $readResults")
    readResults.map { case (topicPartition, readResult) =>
      var updatedReadResult = readResult
      nonOfflinePartition(topicPartition) match {
        case Some(partition) =>
          partition.getReplica(replicaId) match {
            case Some(replica) =>
              partition.updateReplicaLogReadResult(replica, readResult)
            case None =>
              // 如果副本不存在则不更新
              updatedReadResult = readResult.withEmptyFetchInfo
          }
        case None =>
          warn(s"While recording the replica LEO, the partition $topicPartition hasn't been created.")
      }
      topicPartition -> updatedReadResult
    }
}
```

updateFollowerLogReadResults方法比较简单，但也有不少的细节。先回顾下它的两个参数:
1. replicaId表示follower副本的id，也就是我一直强调的follower副本所在的broker id，二者等价
2. readResults是本次fetch请求读取的结果，和消费者一样，可以拉取多个分区

真正的调用是以下代码，表示Partition更新某一个follower的同步状态，该方法的难点在于replica参数，结合Partition类的allReplicasMap来看，此处的replica代表了在leader端，follower副本对应的Replica对象，根据后面的代码来看，leader会维护每一个follower副本的同步状态
```java
// 成员变量
private val allReplicasMap = new Pool[Int, Replica]

// updateFollowerLogReadResults里的
partition.getReplica(replicaId) match {
  case Some(replica) =>
    partition.updateReplicaLogReadResult(replica, readResult)
```

## Partition更新同步状态

Partition对象的updateReplicaLogReadResult方法，它主要做了3件事：

1. 调用Replica对象的updateLogReadResult，更新该follower副本的同步状态
2. 尝试扩充ISR列表
3. 尝试完成一些延迟操作: produce,fetch,deleteRecords

```java
def updateReplicaLogReadResult(replica: Replica, logReadResult: LogReadResult): Boolean = {
    // 此处的replica就是远程的follower副本
    val replicaId = replica.brokerId
    
    // No need to calculate low watermark if there is no delayed DeleteRecordsRequest
    // LW就是所有副本logStartOffset的最小值
    val oldLeaderLW = if (replicaManager.delayedDeleteRecordsPurgatory.delayed > 0) lowWatermarkIfLeader else -1L
    // 更新同步信息
    replica.updateLogReadResult(logReadResult)
    // 新的LW
    val newLeaderLW = if (replicaManager.delayedDeleteRecordsPurgatory.delayed > 0) lowWatermarkIfLeader else -1L
    // check if the LW of the partition has incremented
    // since the replica's logStartOffset may have incremented
    val leaderLWIncremented = newLeaderLW > oldLeaderLW
    // check if we need to expand ISR to include this replica
    // if it is not in the ISR yet
    // 扩充ISR列表
    val leaderHWIncremented = maybeExpandIsr(replicaId, logReadResult)

    val result = leaderLWIncremented || leaderHWIncremented
    // some delayed operations may be unblocked after HW or LW changed
    if (result)
      // 尝试完成一些延迟操作:produce,fetch,deleteRecords
      tryCompleteDelayedRequests()

    debug(s"Recorded replica $replicaId log end offset (LEO) position ${logReadResult.info.fetchOffsetMetadata.messageOffset}.")
    result
}
```

首先看下第一步，follower同步状态的更新

### Replica更新同步状态

大体思路很清晰，首先更新_lastCaughtUpTimeMs，它记录的follower达到同步状态的时间，至于如何判定达到了同步状态，该方法的2个if给出了答案，
 而lastFetchTimeMs仅仅是leader收到fetch请求的时间
```java
def updateLogReadResult(logReadResult: LogReadResult) {

    // fetchOffsetMetadata就是fetch请求中的fetchOffset，表示从哪里开始拉取，leaderLogEndOffset就是LEO
    // 通过debug，大部分情况是走第一个if，二者是相等的，表示生产消息和follower同步消息的速率在一个水平线上
    if (logReadResult.info.fetchOffsetMetadata.messageOffset >= logReadResult.leaderLogEndOffset)
      // _lastCaughtUpTimeMs更新为fetchTimeMs，表示拉取时的当前时间
      _lastCaughtUpTimeMs = math.max(_lastCaughtUpTimeMs, logReadResult.fetchTimeMs)
    else if (logReadResult.info.fetchOffsetMetadata.messageOffset >= lastFetchLeaderLogEndOffset)
      _lastCaughtUpTimeMs = math.max(_lastCaughtUpTimeMs, lastFetchTimeMs)

    // followerLogStartOffset是fetch请求中的，表示的是follower副本的LogStartOffset
    // 注意里面有if local的判断，这里其实更新的是follower副本的LogStartOffset
    // _logStartOffset = followerLogStartOffset
    logStartOffset = logReadResult.followerLogStartOffset
    // 和上面一样，这个LEO表示的是follower副本的LEO
    logEndOffset = logReadResult.info.fetchOffsetMetadata
    // 记录fetch时， leader的LEO
    lastFetchLeaderLogEndOffset = logReadResult.leaderLogEndOffset
    // 记录fetch的时间
    lastFetchTimeMs = logReadResult.fetchTimeMs
}
```
logStartOffset，logEndOffset表示的是_logStartOffset和_logEndOffset，即记录的是follower的logStartOffset和logEndOffset

logStartOffset变量之前的文章也经常提到，这里再解释一遍，副本对应一个Log对象，一个日志用多个Segment存储，第一个Segment的第一条消息的offset就是logStartOffset，因为kafka会定时删除日志，所以它是会变的，也就可以简单理解为目前副本的第一个消息的offset；至于logEndOffset就不再解释了
```java
def logStartOffset: Long =
  if (isLocal)
    log.get.logStartOffset
  else
    _logStartOffset
```

### maybeExpandIsr扩充ISR列表

在看完Partition更新同步状态的第一步后，接下来看第二步maybeExpandIsr，首先判断是否需要添加到ISR副本中，有以下4个条件
1. follower副本目前不在ISR列表中
2. 是已分配的副本
3. follower的LEO > Leader的HW，从前面看follower的LEO就是本次fetch请求的fetchOffset
4. follower的fetchOffset至少比一个leader epoch的start offset大
前2个条件很好理解，第3个条件表示已达到同步，第4个条件则是确保fetchOffset的正确性，防止数据丢失

更新的过程也十分简单，就是将新的Isr更新到zk的/brokers/topics/xxxTopic/partitions/0/state节点，并更新到本地缓存isrChangeSet中

最后在ISR新加入了一个副本之后，有可能触发leader副本的HW更新

```java
def maybeExpandIsr(replicaId: Int, logReadResult: LogReadResult): Boolean = {
    inWriteLock(leaderIsrUpdateLock) {
      // check if this replica needs to be added to the ISR
      leaderReplicaIfLocal match {
        case Some(leaderReplica) =>
          val replica = getReplica(replicaId).get
          val leaderHW = leaderReplica.highWatermark
          val fetchOffset = logReadResult.info.fetchOffsetMetadata.messageOffset

          // 目前不在ISR列表中 && 是已分配的副本 && follower的LEO > Leader的HW && follower的fetchOffset至少比一个leader epoch的start offset大
          if (!inSyncReplicas.contains(replica)
            && assignedReplicas.map(_.brokerId).contains(replicaId)
            && replica.logEndOffset.offsetDiff(leaderHW) >= 0
            && leaderEpochStartOffsetOpt.exists(fetchOffset >= _)) {

            // 添加到集合
            val newInSyncReplicas = inSyncReplicas + replica
  
            // update ISR in ZK and cache
            // 新的Isr更新到zk的state节点，并更新到本地缓存isrChangeSet中
            updateIsr(newInSyncReplicas)
            // metrics
            replicaManager.isrExpandRate.mark()
          }
          // 尝试增加leader的HW，因为有follower进入到ISR了
          // check if the HW of the partition can now be incremented
          // since the replica may already be in the ISR and its LEO has just incremented
          maybeIncrementLeaderHW(leaderReplica, logReadResult.fetchTimeMs)
        case None => false // nothing to do if no longer leader
      }
    }
}
```
leader端的处理结束了，再看看follower副本对fetch请求响应的处理

# follower副本端处理响应

重新回到processFetchRequest方法，该方法通过fetch方法发送请求，在上面已经讲过了leader端是如何处理follower的fetch的，下面看看follower如何处理fetch请求的响应

```java
// 篇幅原因，仅保留核心代码
private def processFetchRequest(fetchRequest: REQ) {
    val partitionsWithError = mutable.Set[TopicPartition]()
    var responseData: Seq[(TopicPartition, PD)] = Seq.empty
   
    responseData = fetch(fetchRequest)
    
    fetcherStats.requestRate.mark()

    if (responseData.nonEmpty) {

      inLock(partitionMapLock) {

        responseData.foreach { case (topicPartition, partitionData) =>
          val topic = topicPartition.topic
          val partitionId = topicPartition.partition
          Option(partitionStates.stateValue(topicPartition)).foreach(currentPartitionFetchState =>
            // It's possible that a partition is removed and re-added or truncated when there is a pending fetch request.
            // In this case, we only want to process the fetch response if the partition state is ready for fetch and the current offset is the same as the offset requested.
            if (fetchRequest.offset(topicPartition) == currentPartitionFetchState.fetchOffset &&
                currentPartitionFetchState.isReadyForFetch) {
              partitionData.error match {
                case Errors.NONE =>
                  try {

                    // ===================== 核心部分 =============================
                    val records = partitionData.toRecords
                    // 获取最后一个消息的nextOffset，作为下次新的fetchOffset，没有则依然以当前的为准
                    val newOffset = records.batches.asScala.lastOption.map(_.nextOffset).getOrElse(
                      currentPartitionFetchState.fetchOffset)

                    // 更新metric lag(FetcherLagStats)，如果lag<=0，说明是inSync的；   HW-lastOffset
                    fetcherLagStats.getAndMaybePut(topic, partitionId).lag = Math.max(0L, partitionData.highWatermark - newOffset)
                    // Once we hand off the partition data to the subclass, we can't mess with it any more in this thread
                    // 参数解释：分区，拉取时的fetchOffset，拉取的结果数据
                    processPartitionData(topicPartition, currentPartitionFetchState.fetchOffset, partitionData)

                    val validBytes = records.validBytes
                    // ReplicaDirAlterThread may have removed topicPartition from the partitionStates after processing the partition data
                    if (validBytes > 0 && partitionStates.contains(topicPartition)) {
                      // 更新分区的PartitionState(newOffset, 0, false)
                      // Update partitionStates only if there is no exception during processPartitionData
                      partitionStates.updateAndMoveToEnd(topicPartition, new PartitionFetchState(newOffset))
                      // metrics ...
                      fetcherStats.byteRate.mark(validBytes)
                    }
                  } 
                } 
            })
        }
      }
    }
}
```

核心代码的逻辑是：
1. 将拉取到的消息的最一条的offset，作为下一次拉取的fetchOffset参数，保存在partitionStates中
2. 

processPartitionData除去校验和限流相关代码，主要做了3件事：

1. 将消息追加到本地副本中(appendRecordsToFollowerOrFutureReplica)
2. 取本地follower副本的LEO(append之后已更新)和响应中leader HW的较小值，作为follower的HW
3. 根据leader的logStartOffset来判断是否需要截断自己的leader epoch startOffset，此处暂不用关心

```java
def processPartitionData(topicPartition: TopicPartition, fetchOffset: Long, partitionData: PartitionData) {
    val replica = replicaMgr.getReplicaOrException(topicPartition)
    val partition = replicaMgr.getPartition(topicPartition).get
    val records = partitionData.toRecords

    // 老版本没有第一条消息大于replica.fetch.max.bytes时，至少取一条的处理，目前fetchRequestVersion=8,不用关心
    maybeWarnIfOversizedRecords(records, topicPartition)

    // 说明请求的fetchOffset就是当前的LEO
    if (fetchOffset != replica.logEndOffset.messageOffset)
      throw new IllegalStateException("Offset mismatch for partition %s: fetched offset = %d, log end offset = %d.".format(
        topicPartition, fetchOffset, replica.logEndOffset.messageOffset))

    // Append the leader's messages to the log
    // 就是Log append 不过调用的是Log#appendAsFollower
    partition.appendRecordsToFollowerOrFutureReplica(records, isFuture = false)

    // 取本地follower副本的LEO(append之后已更新) 和响应中leader HW的较小值，作为follower的HW
    val followerHighWatermark = replica.logEndOffset.messageOffset.min(partitionData.highWatermark)
    replica.highWatermark = new LogOffsetMetadata(followerHighWatermark)

    // for the follower replica, we do not need to keep
    // its segment base offset the physical position,
    // these values will be computed upon making the leader
    val leaderLogStartOffset = partitionData.logStartOffset
    replica.maybeIncrementLogStartOffset(leaderLogStartOffset)

    // Traffic from both in-sync and out of sync replicas are accounted for in replication quota to ensure total replication
    // traffic doesn't exceed quota.
    if (quota.isThrottled(topicPartition))
      quota.record(records.sizeInBytes)
    replicaMgr.brokerTopicStats.updateReplicationBytesIn(records.sizeInBytes)
}
```

# 总结

至此，副本同步的过程结果，相关流程用以下一张图解释
![副本同步流程](https://pic.downk.cc/item/5eb61275c2a9a83be5467f7e.png)


