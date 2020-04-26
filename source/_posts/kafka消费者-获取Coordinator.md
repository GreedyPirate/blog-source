---
title: kafka消费者-获取Coordinator
date: 2019-11-08 15:50:55
categories: Kafka Tutorial
tags: [kafka,中间件,消息]
toc: true
comments: true
---

>本文主要介绍Consumer在第一次拉取消息前，获取Coordinator的过程

# 请求

FindCoordinatorRequest的主要参数是groupId，参数名为coordinatorKey，以下是Consumer发送FindCoordinatorRequest请求的源码

```java
private RequestFuture<Void> sendFindCoordinatorRequest(Node node) {
    // initiate the group metadata request
    log.debug("Sending FindCoordinator request to broker {}", node);
    FindCoordinatorRequest.Builder requestBuilder =
            new FindCoordinatorRequest.Builder(FindCoordinatorRequest.CoordinatorType.GROUP, this.groupId);
    return client.send(node, requestBuilder)
                 .compose(new FindCoordinatorResponseHandler());
}
```
json请求体如下：
```json
{
	"coordinatorKey": "your group id",
	"coordinatorType": 0, // 表示group
	"minVersion": 0, // group时为0
}
```

# server端处理

server获取Coordinator的过程大致分为以下3个步骤
1. 计算分区，groupId的hashcode%50，50是__consumer_offsets的分区个数
2. 获取__consumer_offsets所有分区的元信息
3. 从第2步所有分区的元数据里，过滤出用第1步计算好的分区的元数据

这里没有直接用第一步计算的分区去获取，是因为调用的方法具有通用性，个人认为这里优化下也不难

```java
def handleFindCoordinatorRequest(request: RequestChannel.Request) {
    val findCoordinatorRequest = request.body[FindCoordinatorRequest]

    if (认证失败...) {
    	// 省略...
    }
    else {
      // get metadata (and create the topic if necessary)
      val (partition, topicMetadata) = findCoordinatorRequest.coordinatorType match {
        case FindCoordinatorRequest.CoordinatorType.GROUP =>
          // 计算分区
          val partition = groupCoordinator.partitionFor(findCoordinatorRequest.coordinatorKey)
          // 这里拿到的是topic所有分区的元数据
          val metadata = getOrCreateInternalTopic(GROUP_METADATA_TOPIC_NAME, request.context.listenerName)
          (partition, metadata)

        case 如果是__transaction_state =>
          // 处理 ...
        case _ =>
          throw new InvalidRequestException("Unknown coordinator type in FindCoordinator request")
      }

      def createResponse(requestThrottleMs: Int): AbstractResponse = {
        val responseBody = if (topicMetadata.error != Errors.NONE) {
          new FindCoordinatorResponse(requestThrottleMs, Errors.COORDINATOR_NOT_AVAILABLE, Node.noNode)
        } else {
          // 获取消费者坐在分区的leader所在的node
          val coordinatorEndpoint = topicMetadata.partitionMetadata.asScala
            .find(_.partition == partition)
            .map(_.leader)
            .flatMap(p => Option(p))

          // 有则返回，没有报错
          coordinatorEndpoint match {
            case Some(endpoint) if !endpoint.isEmpty =>
              new FindCoordinatorResponse(requestThrottleMs, Errors.NONE, endpoint)
            case _ =>
              new FindCoordinatorResponse(requestThrottleMs, Errors.COORDINATOR_NOT_AVAILABLE, Node.noNode)
          }
        }
        responseBody
      }
      // 响应
      sendResponseMaybeThrottle(request, createResponse)
    }
}
```
这里返回给客户端的FindCoordinatorResponse对象，大致结构如下，主要是错误信息和Coordinator所在的节点信息
```json
{
	"error": {...},
	"node": {
		"id": 0,
		"host": "127.0.0.1",
		"port": "9092",
		"rack": "-1"
	}
}
```

## 获取topic分区元数据

经过getOrCreateInternalTopic方法调用，getPartitionMetadata用于获取topic所有分区的元数据，即上面的第2步

该方法大部分都是对maybeLeader异常的判断，直接看最后一行代码即可

```java
private def getOrCreateInternalTopic(topic: String, listenerName: ListenerName): MetadataResponse.TopicMetadata = {
    val topicMetadata = metadataCache.getTopicMetadata(Set(topic), listenerName)
    // 取集合第一个，因为getTopicMetadata是批量的，Set(topic)
    topicMetadata.headOption.getOrElse(createInternalTopic(topic))
}

def getTopicMetadata(topics: Set[String], listenerName: ListenerName, errorUnavailableEndpoints: Boolean = false,
                       errorUnavailableListeners: Boolean = false): Seq[MetadataResponse.TopicMetadata] = {
    topics.toSeq.flatMap { topic =>
        getPartitionMetadata(topic, listenerName, errorUnavailableEndpoints, errorUnavailableListeners).map { partitionMetadata =>
          new MetadataResponse.TopicMetadata(Errors.NONE, topic, Topic.isInternal(topic), partitionMetadata.toBuffer.asJava)
        }
    }
}

private def getPartitionMetadata(topic: String, listenerName: ListenerName, errorUnavailableEndpoints: Boolean,
                                   errorUnavailableListeners: Boolean): Option[Iterable[MetadataResponse.PartitionMetadata]] = {
    cache.get(topic).map { partitions =>
      partitions.map { case (partitionId, partitionState) =>
        val topicPartition = TopicAndPartition(topic, partitionId)
        val leaderBrokerId = partitionState.basePartitionState.leader
        val maybeLeader = getAliveEndpoint(leaderBrokerId, listenerName) // 可能为空
        val replicas = partitionState.basePartitionState.replicas.asScala.map(_.toInt)
        val replicaInfo = getEndpoints(replicas, listenerName, errorUnavailableEndpoints) // 副本所在的Node
        // 离线副本的Node信息
        val offlineReplicaInfo = getEndpoints(partitionState.offlineReplicas.asScala.map(_.toInt), listenerName, errorUnavailableEndpoints)

        maybeLeader match {
          case None =>
            val error = if (!aliveBrokers.contains(brokerId)) { // we are already holding the read lock
              debug(s"Error while fetching metadata for $topicPartition: leader not available")
              Errors.LEADER_NOT_AVAILABLE
            } else {
              debug(s"Error while fetching metadata for $topicPartition: listener $listenerName not found on leader $leaderBrokerId")
              if (errorUnavailableListeners) Errors.LISTENER_NOT_FOUND else Errors.LEADER_NOT_AVAILABLE
            }
            new MetadataResponse.PartitionMetadata(error, partitionId, Node.noNode(),
              replicaInfo.asJava, java.util.Collections.emptyList(), offlineReplicaInfo.asJava)

          case Some(leader) =>
            val isr = partitionState.basePartitionState.isr.asScala.map(_.toInt)
            val isrInfo = getEndpoints(isr, listenerName, errorUnavailableEndpoints) // isr的Node信息

            // 副本信息不全
            if (replicaInfo.size < replicas.size) {
              debug(s"Error while fetching metadata for $topicPartition: replica information not available for " +
                s"following brokers ${replicas.filterNot(replicaInfo.map(_.id).contains).mkString(",")}")

              new MetadataResponse.PartitionMetadata(Errors.REPLICA_NOT_AVAILABLE, partitionId, leader,
                replicaInfo.asJava, isrInfo.asJava, offlineReplicaInfo.asJava)
            } else if (isrInfo.size < isr.size) {
              // isr 信息不全
              debug(s"Error while fetching metadata for $topicPartition: in sync replica information not available for " +
                s"following brokers ${isr.filterNot(isrInfo.map(_.id).contains).mkString(",")}")
              new MetadataResponse.PartitionMetadata(Errors.REPLICA_NOT_AVAILABLE, partitionId, leader,
                replicaInfo.asJava, isrInfo.asJava, offlineReplicaInfo.asJava)
            } else {
              // 分区的leader，isr，replica，offline replica信息
              new MetadataResponse.PartitionMetadata(Errors.NONE, partitionId, leader, replicaInfo.asJava,
                isrInfo.asJava, offlineReplicaInfo.asJava)
            }
        }
      }
    }
}
```

# 小结
FindCoordinatorRequest请求是consumer在拉取消息时的前置步骤，用于确保coordinator的存在，broker具体的做法是根据consumer的groupId确定其所在的__consumer_offsets分区，之后再获取该分区的元数据，主要的元信息为分区leader，replica，isr集合，离线副本集合


