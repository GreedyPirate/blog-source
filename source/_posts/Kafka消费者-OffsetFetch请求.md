---
title: Kafka消费者-OffsetFetch请求
date: 2020-03-13 19:06:03
categories: Kafka Tutorial
tags: [kafka,中间件,消息]
toc: true
comments: true
---

# 前言

本文聊聊消费者如何获取上次提交的位移

# OffsetFetch请求

在[Kafka消费者-源码分析(上)]()一文的最后，Consumer在refreshCommittedOffsetsIfNeeded方法发起了该请求，目的是获取消费者上次提交的位移，作为下次拉取请求的fetchOffset参数

```java
public boolean refreshCommittedOffsetsIfNeeded(final long timeoutMs) {
    // 既没有position，也没有resetStrategy，初始化时都没有
    final Set<TopicPartition> missingFetchPositions = subscriptions.missingFetchPositions();

    // 通过OffsetFetchRequest请求，向Coordinator获取last consumed位移
    final Map<TopicPartition, OffsetAndMetadata> offsets = fetchCommittedOffsets(missingFetchPositions, timeoutMs);
    if (offsets == null) return false;

    for (final Map.Entry<TopicPartition, OffsetAndMetadata> entry : offsets.entrySet()) {
        final TopicPartition tp = entry.getKey();
        final long offset = entry.getValue().offset();
        // seek 就是把offset保持到缓存中, 作为初始化的last consumed
        this.subscriptions.seek(tp, offset);
    }
    return true;
}
```

Consumer的初始是将该分区上次提交的位移保存到了TopicPartitionState的position变量，该类在前文我已经翻译了各个变量的意义

```java
TopicPartitionState {
        private Long position; // last consumed position
        private Long highWatermark; // the high watermark from last fetch
        private Long logStartOffset; // the log start offset
        private Long lastStableOffset;
        private boolean paused;  // whether this partition has been paused by the user
        private OffsetResetStrategy resetStrategy;  // the strategy to use if the offset needs resetting
        private Long nextAllowedRetryTimeMs;
}
```

# 源码

请求入口在KafkaApis#handleOffsetFetchRequest方法中，kafka之前的版本是从zk中获取，这部分代码省略
```java
def handleOffsetFetchRequest(request: RequestChannel.Request) {
    val header = request.header

    val offsetFetchRequest = request.body[OffsetFetchRequest]
  val (authorizedPartitions, unauthorizedPartitions) = offsetFetchRequest.partitions.asScala.partition(authorizeTopicDescribe)

  // 主要的核心逻辑
  val (error, authorizedPartitionData) = groupCoordinator.handleFetchOffsets(offsetFetchRequest.groupId,
    Some(authorizedPartitions))
  if (error != Errors.NONE)
    offsetFetchRequest.getErrorResponse(requestThrottleMs, error)
  else {
    val unauthorizedPartitionData = unauthorizedPartitions.map(_ -> OffsetFetchResponse.UNAUTHORIZED_PARTITION).toMap
    new OffsetFetchResponse(requestThrottleMs, Errors.NONE, (authorizedPartitionData ++ unauthorizedPartitionData).asJava)
  }
}
```

可以看到核心入口是 groupCoordinator.handleFetchOffsets

```java
def handleFetchOffsets(groupId: String, partitions: Option[Seq[TopicPartition]] = None):
  (Errors, Map[TopicPartition, OffsetFetchResponse.PartitionData]) = {

    // 先保证group是正常的
    validateGroupStatus(groupId, ApiKeys.OFFSET_FETCH) match {
      case Some(error) => error -> Map.empty
      case None =>
        (Errors.NONE, groupManager.getOffsets(groupId, partitions))
    }
}
```

getOffsets的源码如下，简而言之就是在offsets中获取，它代表分区提交的记录缓存，类型为Map[TopicPartition, CommitRecordMetadataAndOffset]
```java
def getOffsets(groupId: String, topicPartitionsOpt: Option[Seq[TopicPartition]]): Map[TopicPartition, OffsetFetchResponse.PartitionData] = {
    val group = groupMetadataCache.get(groupId)
    if (group == null) {
    	// 异常：INVALID_OFFSET ...
    } else {
      group.inLock {
        if (group.is(Dead)) {
          // 异常：INVALID_OFFSET ...
        } else {
          topicPartitionsOpt match {
            case None =>
              // Return offsets for all partitions owned by this consumer group. (this only applies to consumers
              // that commit offsets to Kafka.)
              // 返回所有分区的提交记录
              group.allOffsets.map { case (topicPartition, offsetAndMetadata) =>
                topicPartition -> new OffsetFetchResponse.PartitionData(offsetAndMetadata.offset, offsetAndMetadata.metadata, Errors.NONE)
              }

            case Some(topicPartitions) =>
              topicPartitions.map { topicPartition =>
                // 并不是去__consumer-offset里面取，group会缓存上一次提交的offset(第一次LeaderAndIsr的时候加载的)
                val partitionData = group.offset(topicPartition) match {
                  case None =>
                    // 异常：INVALID_OFFSET ...
                  case Some(offsetAndMetadata) =>
                    new OffsetFetchResponse.PartitionData(offsetAndMetadata.offset, offsetAndMetadata.metadata, Errors.NONE)
                }
                topicPartition -> partitionData
              }.toMap
          }
        }
      }
    }
}
```

# 总结

本文作为消费者第一次消费之前的一个准备动作，主要是为了获取上次消费的位置，GroupCoordinator从缓存的offsets Map中获取该消费者组对该分区上次提交的位移，Consumer在接收到响应后，保存到了TopicPartitionState的position变量中，作为下一次Fetch请求的fetchOffset参数