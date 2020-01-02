---
title: kafka server端源码分析之接收消息(一)
date: 2019-11-02 19:23:42
categories: Kafka Tutorial
tags: [kafka,中间件,消息]
toc: true
comments: true
---

> 承接上篇搭建kafka源码环境之后，本文正式开始分析

# 入口
在开始之前，一个很重要的问题是从哪个类中看起。很简单，我们是从kafka.Kafka启动，跟随main方法到KafkaServer的startup方法中，看到了以下代码代码
```scala
def startup() {
	//省略 ...
    /* start processing requests */
    // 同样很重要的KafkaApis对象
    apis = new KafkaApis(...)

    requestHandlerPool = new KafkaRequestHandlerPool(config.brokerId, socketServer.requestChannel, apis, time,
      config.numIoThreads)
}
```
而KafkaRequestHandlerPool创建了KafkaRequestHandler对象
```scala
class KafkaRequestHandlerPool(...) extends Logging with KafkaMetricsGroup {

	def createHandler(id: Int): Unit = synchronized {
	    runnables += new KafkaRequestHandler(id, brokerId, aggregateIdleMeter, threadPoolSize, requestChannel, apis, time)
	    KafkaThread.daemon("kafka-request-handler-" + id, runnables(id)).start()
	}
}
```
numThreads的值就是配置文件中num.io.threads的值

# KafkaRequestHandler
KafkaRequestHandler实现了Runnable接口，根据注释，以及run方法中的部分代码，就可以看出这是在处理请求了
```scala
/**
 * A thread that answers kafka requests.
 */
class KafkaRequestHandler(id: Int,
                          brokerId: Int,
                          val aggregateIdleMeter: Meter,
                          val totalHandlerThreads: AtomicInteger,
                          val requestChannel: RequestChannel,
                          apis: KafkaApis,
                          time: Time) extends Runnable with Logging {
	def run() {
		while (!stopped) {
		  val req = requestChannel.receiveRequest(300)
		  req match {
		    case request: RequestChannel.Request =>
		      try {
		        apis.handle(request)
		      } 
		}
	}                          	
}                          	
```
具体的处理交给了KafkaApis对象的handle方法，该方法的处理也很容易理解，根据客户端的不同请求，用一个match case，类似java中的switch来处理，直接进去到handleProduceRequest方法中

```scala
// 省略部分代码
def handleProduceRequest(request: RequestChannel.Request) {
	// Request对象类型转换
	val produceRequest = request.body[ProduceRequest]
	// 消息头和消息体大小
	val numBytesAppended = request.header.toStruct.sizeOf + request.sizeOfBodyInBytes

	// 定义三个可变map
	val unauthorizedTopicResponses = mutable.Map[TopicPartition, PartitionResponse]()
	val nonExistingTopicResponses = mutable.Map[TopicPartition, PartitionResponse]()
	val authorizedRequestInfo = mutable.Map[TopicPartition, MemoryRecords]()

	for ((topicPartition, memoryRecords) <- produceRequest.partitionRecordsOrFail.asScala) {
	  if (!authorize(request.session, Write, Resource(Topic, topicPartition.topic, LITERAL)))
	    unauthorizedTopicResponses += topicPartition -> new PartitionResponse(Errors.TOPIC_AUTHORIZATION_FAILED)
	  else if (!metadataCache.contains(topicPartition))
	    nonExistingTopicResponses += topicPartition -> new PartitionResponse(Errors.UNKNOWN_TOPIC_OR_PARTITION)
	  else
	    authorizedRequestInfo += (topicPartition -> memoryRecords)
	}

	// the callback for sending a produce response
	def sendResponseCallback(responseStatus: Map[TopicPartition, PartitionResponse]) {
	  val mergedResponseStatus = responseStatus ++ unauthorizedTopicResponses ++ nonExistingTopicResponses
	  var errorInResponse = false

	  mergedResponseStatus.foreach { case (topicPartition, status) =>
	    if (status.error != Errors.NONE) {
	      errorInResponse = true
	      debug("Produce request with correlation id %d from client %s on partition %s failed due to %s".format(
	        request.header.correlationId,
	        request.header.clientId,
	        topicPartition,
	        status.error.exceptionName))
	    }
	  }

	  // When this callback is triggered, the remote API call has completed
	  request.apiRemoteCompleteTimeNanos = time.nanoseconds

	  // Record both bandwidth and request quota-specific values and throttle by muting the channel if any of the quotas
	  // have been violated. If both quotas have been violated, use the max throttle time between the two quotas. Note
	  // that the request quota is not enforced if acks == 0.
	  val bandwidthThrottleTimeMs = quotas.produce.maybeRecordAndGetThrottleTimeMs(request, numBytesAppended, time.milliseconds())
	  val requestThrottleTimeMs = if (produceRequest.acks == 0) 0 else quotas.request.maybeRecordAndGetThrottleTimeMs(request)
	  val maxThrottleTimeMs = Math.max(bandwidthThrottleTimeMs, requestThrottleTimeMs)
	  if (maxThrottleTimeMs > 0) {
	    if (bandwidthThrottleTimeMs > requestThrottleTimeMs) {
	      quotas.produce.throttle(request, bandwidthThrottleTimeMs, sendResponse)
	    } else {
	      quotas.request.throttle(request, requestThrottleTimeMs, sendResponse)
	    }
	  }

	  // Send the response immediately. In case of throttling, the channel has already been muted.
	  if (produceRequest.acks == 0) {
	    // no operation needed if producer request.required.acks = 0; however, if there is any error in handling
	    // the request, since no response is expected by the producer, the server will close socket server so that
	    // the producer client will know that some error has happened and will refresh its metadata
	    if (errorInResponse) {
	      val exceptionsSummary = mergedResponseStatus.map { case (topicPartition, status) =>
	        topicPartition -> status.error.exceptionName
	      }.mkString(", ")
	      info(
	        s"Closing connection due to error during produce request with correlation id ${request.header.correlationId} " +
	          s"from client id ${request.header.clientId} with ack=0\n" +
	          s"Topic and partition to exceptions: $exceptionsSummary"
	      )
	      closeConnection(request, new ProduceResponse(mergedResponseStatus.asJava).errorCounts)
	    } else {
	      // Note that although request throttling is exempt for acks == 0, the channel may be throttled due to
	      // bandwidth quota violation.
	      sendNoOpResponseExemptThrottle(request)
	    }
	  } else {
	    sendResponse(request, Some(new ProduceResponse(mergedResponseStatus.asJava, maxThrottleTimeMs)), None)
	  }
	}

	def processingStatsCallback(processingStats: FetchResponseStats): Unit = {
	  processingStats.foreach { case (tp, info) =>
	    updateRecordConversionStats(request, tp, info)
	  }
	}

	if (authorizedRequestInfo.isEmpty)
	  sendResponseCallback(Map.empty)
	else {
	  val internalTopicsAllowed = request.header.clientId == AdminUtils.AdminClientId

	  // call the replica manager to append messages to the replicas
	  replicaManager.appendRecords(
	    timeout = produceRequest.timeout.toLong,
	    requiredAcks = produceRequest.acks,
	    internalTopicsAllowed = internalTopicsAllowed,
	    isFromClient = true,
	    entriesPerPartition = authorizedRequestInfo,
	    responseCallback = sendResponseCallback,
	    recordConversionStatsCallback = processingStatsCallback)

	  // if the request is put into the purgatory, it will have a held reference and hence cannot be garbage collected;
	  // hence we clear its data here in order to let GC reclaim its memory since it is already appended to log
	  produceRequest.clearPartitionRecords()
	}
}
```
该方法主要做了两件事
1. 检查topic是否存在，client是否有Desribe权限，是否有Write权限
2. 调用replicaManager.appendRecords()方法追加消息


# ReplicaManager
appendRecords的方法注释如下：
将消费追加到分区的leader副本，然后等待它们被follower副本复制，回调函数将会在超时或者ack条件满足是触发

