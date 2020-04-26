---
title:  Kafka消费者-ListOffsets请求
date: 2019-11-09 11:53:23
categories: Kafka Tutorial
tags: [kafka,中间件,消息]
toc: true
comments: true
---

# 前言

本文聊聊消费者拉取消息时向kafka server发送LIST_OFFSETS的请求，这个请求的功能一言以蔽之:根据请求参数中的timeStamp获取消费者(或副本)能够fetch的位移

主要应用场景为消费者第一次拉取消息时，不知道从哪个offset拉取，这个拉取策略可以消费者通过auto.offset.reset指定，请求时翻译成timeStamp(ListOffsetRequest类常量)，
server端处理时从日志(LogSegment)中查找应该被fetch的offset(TimestampOffset)

在消费者之后的拉取中，记录了上次拉取的位置(TopicPartitionState@position)，不用再重新获取

# 源码解析

之前的文章中说过，server端通过KafkaApis#handle方法处理所有网络请求，LIST_OFFSETS请求如下

## handleListOffsetRequest

忽略别的代码，仅关注handleListOffsetRequestV1AndAbove方法, 它返回了每个TP对应的fetch offset

```java
private def handleListOffsetRequestV1AndAbove(request : RequestChannel.Request): Map[TopicPartition, ListOffsetResponse.PartitionData] = {
    val correlationId = request.header.correlationId
    val clientId = request.header.clientId
    val offsetRequest = request.body[ListOffsetRequest]

    val (authorizedRequestInfo, unauthorizedRequestInfo) = offsetRequest.partitionTimestamps.asScala.partition {
      case (topicPartition, _) => authorize(request.session, Describe, Resource(Topic, topicPartition.topic, LITERAL))
    }

    val unauthorizedResponseStatus = unauthorizedRequestInfo.mapValues(_ => {
      new ListOffsetResponse.PartitionData(Errors.TOPIC_AUTHORIZATION_FAILED,
                                           ListOffsetResponse.UNKNOWN_TIMESTAMP,
                                           ListOffsetResponse.UNKNOWN_OFFSET)
    })

    val responseMap = authorizedRequestInfo.map { case (topicPartition, timestamp) =>
		// 获取leader
		val localReplica = replicaManager.getLeaderReplicaIfLocal(topicPartition)

		// -1表示consumer
		val fromConsumer = offsetRequest.replicaId == ListOffsetRequest.CONSUMER_REPLICA_ID

		val found = if (fromConsumer) {
			// 根据事务隔离级别，获取可拉取的位移
			val lastFetchableOffset = offsetRequest.isolationLevel match {
			  case IsolationLevel.READ_COMMITTED => localReplica.lastStableOffset.messageOffset
			    // 默认没使用事务，返回的是highWatermark
			  case IsolationLevel.READ_UNCOMMITTED => localReplica.highWatermark.messageOffset
			}

			// 这里的if...else...就是if (fromConsumer)的返回值
			// reset到最新的
			if (timestamp == ListOffsetRequest.LATEST_TIMESTAMP)
			  // TimestampOffset，case class： -1 和 highWatermark
			  TimestampOffset(RecordBatch.NO_TIMESTAMP, lastFetchableOffset)
			else {
			  // 过滤函数：从log里查找出来的offset一定要比lastFetchableOffset小 或者是earliest
			  def allowed(timestampOffset: TimestampOffset): Boolean =
			    timestamp == ListOffsetRequest.EARLIEST_TIMESTAMP || timestampOffset.offset < lastFetchableOffset

			  // 获取offset
			  fetchOffsetForTimestamp(topicPartition, timestamp)
			    .filter(allowed).getOrElse(TimestampOffset.Unknown)
			}
		} 
		// 不是consumer的先不看
		
		// 这是map方法的返回，也就是在循环内
		(topicPartition, new ListOffsetResponse.PartitionData(Errors.NONE, found.timestamp, found.offset))
        
    }
    // 和未认证的TP并集，返回给客户端
    responseMap ++ unauthorizedResponseStatus
}
```

## Segment中获取

该方法就是根据客户端的reset policy(TimeStamp)来返回offset

```java
def fetchOffsetsByTimestamp(targetTimestamp: Long): Option[TimestampOffset] = {
    maybeHandleIOException(s"Error while fetching offset by timestamp for $topicPartition in dir ${dir.getParent}") {
    
      // 所有LogSegment的副本，共享变私有，避免锁竞争
      val segmentsCopy = logSegments.toBuffer
      // For the earliest and latest, we do not need to return the timestamp.
      if (targetTimestamp == ListOffsetRequest.EARLIEST_TIMESTAMP)
        // earliest返回logStartOffset：当前TP在日志自动清理后，目前最小的offset
        return Some(TimestampOffset(RecordBatch.NO_TIMESTAMP, logStartOffset))
      else if (targetTimestamp == ListOffsetRequest.LATEST_TIMESTAMP)
        // latest返回LEO 但是为什么返回LEO呢，万一一直没提交呢，返回HW不是更稳妥吗
        return Some(TimestampOffset(RecordBatch.NO_TIMESTAMP, logEndOffset))

      // earliest，latest之外的类型：Timestamp表示具体的时间戳，-1，-2只是表示了2个特殊的offset
      val targetSeg = {
        // Get all the segments whose largest timestamp is smaller than target timestamp
        // 先找segments，找第一个Segment的最大Timestamp大于请求中的Timestamp，可以看下takeWhile源码
        val earlierSegs = segmentsCopy.takeWhile(_.largestTimestamp < targetTimestamp) // takeWhile牛逼啊，一直循环，只要不满足表示式停止
        // We need to search the first segment whose largest timestamp is greater than the target timestamp if there is one.
        // 再找offset
        if (earlierSegs.length < segmentsCopy.length)
          Some(segmentsCopy(earlierSegs.length))
        else
          None
      }

      targetSeg.flatMap(_.findOffsetByTimestamp(targetTimestamp, logStartOffset))
    }
}
```








