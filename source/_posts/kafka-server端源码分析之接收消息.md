---
title: kafka server端源码分析之接收消息
date: 2019-12-10 19:23:42
categories: Kafka Tutorial
tags: [kafka,中间件,消息]
toc: true
comments: true
---

> 承接上篇搭建kafka源码环境之后，本文正式开始分析

# 前文

在前文[kafka网络请求处理模型](https://greedypirate.github.io/2019/12/06/kafka%E7%BD%91%E7%BB%9C%E8%AF%B7%E6%B1%82%E5%A4%84%E7%90%86%E6%A8%A1%E5%9E%8B/)中提到, KafkaServer#startup方法涵盖了kafka server所有模块的初始化
KafkaRequestHandlerPool线程池中的KafkaRequestHandler对象通过调用KafkaApis的handle方法，处理各类网络请求

# 图文解析

![append消息流程](https://ae01.alicdn.com/kf/H46b57c5ef8eb44fbb533c5d808b49906v.png)

上图是kafka server追加消息到日志的整个流程，主要分为以下几步
1. handleProduceRequest首先过滤认证失败和leader未知的分区，定义响应回调。如果ack=0直接响应，否则ReplicaManager继续处理
2. 将生产者的相关参数，如超时时间，ack，以及第1步的响应回调函数传给ReplicaManager#appendRecords，appendRecords继续调用appendToLocalLog，完成后如果ack=-1时，第一次尝试结束请求
3. appendToLocalLog则遍历所有分区，获取该分区的本地leader副本Partition对象，调用它的appendRecordsToLeader方法，为每个分区追加消息
4. Partition#appendRecordsToLeader方法中，在校验完minIsr参数后，调用Log对象appendAsLeader->append方法，里面首先计算要追加的位移，消息CRC校验，截断无效消息等
5. Log#append方法之后会判断当前activeSegment是否需要roll(新建一个)，然后调用LogSegment#append->...->FileChannel#write将消息写入日志中
6. 层层返回，调用响应回调函数中的sendResponse，和[kafka网络请求处理模型](https://greedypirate.github.io/2019/12/06/kafka%E7%BD%91%E7%BB%9C%E8%AF%B7%E6%B1%82%E5%A4%84%E7%90%86%E6%A8%A1%E5%9E%8B/)一文承上启下，将Response对象放入Processor中的responseQueue，等待Processor轮询处理
	

# 源码

注: TopicPartition, 包含topic和partition的值，简称TP

## 生产者请求处理方法

KafkaApis#handle方法根据不同类型的请求，调用不同的handleXxx方法，生产者请求在handleProduceRequest方法中

该方法除了调用ReplicaManager#appendRecords,还对日志权限，事务，限流等做了处理，并且定义好了响应回调函数，一并作为参数传给了ReplicaManager#appendRecords方法

```java
// 省略部分代码
def handleProduceRequest(request: RequestChannel.Request) {
	  // 转换为具体的请求对象
    val produceRequest = request.body[ProduceRequest]
    val numBytesAppended = request.header.toStruct.sizeOf + request.sizeOfBodyInBytes

    val unauthorizedTopicResponses = mutable.Map[TopicPartition, PartitionResponse]()
    val nonExistingTopicResponses = mutable.Map[TopicPartition, PartitionResponse]()
    val authorizedRequestInfo = mutable.Map[TopicPartition, MemoryRecords]()

    for ((topicPartition, memoryRecords) <- produceRequest.partitionRecordsOrFail.asScala) {
      // 是否认证通过，是否有write权限
      if (!authorize(request.session, Write, Resource(Topic, topicPartition.topic, LITERAL)))
        // 忘了语法... +=是想集合添加元素，但是 ->呢？ 这是map的key->value 语法
        unauthorizedTopicResponses += topicPartition -> new PartitionResponse(Errors.TOPIC_AUTHORIZATION_FAILED)
      else if (!metadataCache.contains(topicPartition))
        // 元数据缓存中是否有该tp，元数据缓存是由controller直接更新的
        nonExistingTopicResponses += topicPartition -> new PartitionResponse(Errors.UNKNOWN_TOPIC_OR_PARTITION)
      else
        // 剩下的都是可用的消息
        authorizedRequestInfo += (topicPartition -> memoryRecords)
    }

    // the callback for sending a produce response
    // 嵌套方法，定义响应回调，可以先不看
    def sendResponseCallback(responseStatus: Map[TopicPartition, PartitionResponse]) {
      // ++表示集合合并
      val mergedResponseStatus = responseStatus ++ unauthorizedTopicResponses ++ nonExistingTopicResponses
      var errorInResponse = false

      // 先打个日志，不管
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

      // 省略配额限流相关代码

      // Send the response immediately. In case of throttling, the channel has already been muted.
      // ack=0表示发到broker就返回，不关心副本是否写入
      if (produceRequest.acks == 0) {
          sendNoOpResponseExemptThrottle(request)
      } else {
        // ack为-1或1的响应
        sendResponse(request, Some(new ProduceResponse(mergedResponseStatus.asJava, maxThrottleTimeMs)), None)
      }
    }

  // 只有__admin_client客户端才能写入内部topic，例如__consumer_offset
  val internalTopicsAllowed = request.header.clientId == AdminUtils.AdminClientId

  // call the replica manager to append messages to the replicas
  // 开始调用副本管理器追加消息
  replicaManager.appendRecords(
  	// 超时时间, 客户端Sender中的requestTimeoutMs，表示客户端请求超时
  	timeout = produceRequest.timeout.toLong,
  	// ack参数
  	requiredAcks = produceRequest.acks,
  	// 是否允许添加内部topic消息
  	internalTopicsAllowed = internalTopicsAllowed,
  	// 是否来自client，也有可能来自别的broker
  	isFromClient = true,
  	// 消息体
  	entriesPerPartition = authorizedRequestInfo,
  	// 响应函数
  	responseCallback = sendResponseCallback,
  	// 状态转换函数
  	recordConversionStatsCallback = processingStatsCallback
  )

  // if the request is put into the purgatory, it will have a held reference and hence cannot be garbage collected;
  // hence we clear its data here in order to let GC reclaim its memory since it is already appended to log
  // 如果需要被放入purgatory，清空引用让GC回收, 因为已经append到log了
  produceRequest.clearPartitionRecords()
}
```

ProduceRequest请求参数如下

```java
public class ProduceRequest extends AbstractRequest {
    private final short acks;
    private final int timeout;
    private final String transactionalId;

    private final Map<TopicPartition, Integer> partitionSizes;

    // This is set to null by `clearPartitionRecords` to prevent unnecessary memory retention when a produce request is
    // put in the purgatory (due to client throttling, it can take a while before the response is sent).
    // Care should be taken in methods that use this field.
    private volatile Map<TopicPartition, MemoryRecords> partitionRecords; // 每个分区待处理的消息
    private boolean transactional = false; // 事务
    private boolean idempotent = false; // 幂等性
}    
```

# ReplicaManager

ReplicaManager的主要功能是对分区副本层面做管理，包含日志写入，读取，ISR的变更，副本同步等。

appendRecords的方法注释如下：将消费追加到分区的leader副本，然后等待它们被follower副本复制，回调函数将会在超时或者ack条件满足时触发

该方法主要是在append消息之后，对当前请求的处理。ack=-1尝试完成当前请求，在ack=1时直接调用响应函数

```java
def appendRecords(... ) { //参数参考上面
    // 简单的校验ack合法性，-1，0，1才合法
    if (isValidRequiredAcks(requiredAcks)) {
      val sTime = time.milliseconds
      // 写入到本地broker中, 返回每个TPLogAppendResult => LogAppendInfo和异常
      val localProduceResults = appendToLocalLog(internalTopicsAllowed = internalTopicsAllowed,
        isFromClient = isFromClient, entriesPerPartition, requiredAcks)

      // produceStatus类型:Map[TopicPartition, ProducePartitionStatus]
      // 这个map保存的是每个TopicPartition append后的状态，状态包括：LEO和结果，结果里面有是否append出现错误等
      val produceStatus = localProduceResults.map { case (topicPartition, result) =>
        topicPartition ->
                ProducePartitionStatus(
                  result.info.lastOffset + 1, // required offset ， LEO 
                  new PartitionResponse(result.error, result.info.firstOffset.getOrElse(-1), result.info.logAppendTime, result.info.logStartOffset)) // response status
      }

      recordConversionStatsCallback(localProduceResults.mapValues(_.info.recordConversionStats))

      // ack为-1时需要follower同步，需要放入延迟队列中，等待条件满足后返回
      if (delayedProduceRequestRequired(requiredAcks, entriesPerPartition, localProduceResults)) {
        // create delayed produce operation
        // ack和消息append后的结果
        val produceMetadata = ProduceMetadata(requiredAcks, produceStatus)
        // 注意看里面的初始化语句块
        val delayedProduce = new DelayedProduce(timeout, produceMetadata, this, responseCallback, delayedProduceLock)

        // 就是TopicPartition集合
        val producerRequestKeys = entriesPerPartition.keys.map(new TopicPartitionOperationKey(_)).toSeq

        // 第一次尝试结束处理，否则丢入purgatory中，因为下一批消息可能已经到达将这批请求结束
        delayedProducePurgatory.tryCompleteElseWatch(delayedProduce, producerRequestKeys)
      } else {
        // 这是ack=1的时候，leader写入完了，就返回，之前已经处理过ack=0了
        val produceResponseStatus = produceStatus.mapValues(status => status.responseStatus)
        responseCallback(produceResponseStatus)
      }
    } else {
      // ack参数无效后直接返回错误
      val responseStatus = entriesPerPartition.map { case (topicPartition, _) =>
        topicPartition -> new PartitionResponse(Errors.INVALID_REQUIRED_ACKS,
          LogAppendInfo.UnknownLogAppendInfo.firstOffset.getOrElse(-1), RecordBatch.NO_TIMESTAMP, LogAppendInfo.UnknownLogAppendInfo.logStartOffset)
      }

      // 最后调用传进来的响应回调方法
      responseCallback(responseStatus)
    }
  }
```
追加日志都在appendToLocalLog中完成，后面的代码是对追加结果的处理

### appendToLocalLog方法实现

appendToLocalLog开始遍历分区消息集合Map[TopicPartition, MemoryRecords]对象，
```java
/**
* Append the messages to the local replica logs
* @param internalTopicsAllowed 是否允许操作内部topic
* @param isFromClient true，来自客户端
* @param entriesPerPartition 消息体
* @param requiredAcks ack参数
* @return Map[TopicPartition, LogAppendResult]
*/
private def appendToLocalLog(internalTopicsAllowed: Boolean,
                           isFromClient: Boolean,
                           entriesPerPartition: Map[TopicPartition, MemoryRecords],
                           requiredAcks: Short): Map[TopicPartition, LogAppendResult] = {

    // 遍历消息集合，追加消息 map里的case表示是个匿名偏函数
    entriesPerPartition.map { case (topicPartition, records) => 
      // 省略部分非核心代码
      // 如果是内部topic，但没有内部topic的操作权限，就报错，内部topic只有两个__consumer_offsets和__transaction_state

      // 获取当前tp的leader Partition对象
      val (partition, _) = getPartitionAndLeaderReplicaIfLocal(topicPartition)
      val info = partition.appendRecordsToLeader(records, isFromClient, requiredAcks)

      // 向一个tp中追加消息结束，返回结果
      (topicPartition, LogAppendResult(info))	    
    }
}
```

### Partition#appendRecordsToLeader方法实现

Partition对象的appendRecordsToLeader方法中检验ack=-1时，min.insync.replicas必须大于ISR个数，否则抛出NotEnoughReplicasException
然后调用Log对象的appendAsLeader->append方法，追加完消息后，第二次尝试完成生产者请求

```java
def appendRecordsToLeader(records: MemoryRecords, isFromClient: Boolean, requiredAcks: Int = 0): LogAppendInfo = {
    // inReadLock是一个柯里化函数，第二个参数是一个函数，返回值是LogAppendInfo和HW是否增加的bool值
    // 相当于给方法加了读锁
    val (info, leaderHWIncremented) = inReadLock(leaderIsrUpdateLock) {
      // leaderReplicaIfLocal表示本地broker中的leader副本
      leaderReplicaIfLocal match {
        //如果存在的话
        case Some(leaderReplica) =>
          // 获取Replica中的Log对象
          val log = leaderReplica.log.get
          // min.insync.replicas参数
          val minIsr = log.config.minInSyncReplicas
          // Set[Replica] ISR大小
          val inSyncSize = inSyncReplicas.size

          // Avoid writing to leader if there are not enough insync replicas to make it safe
          // 如果isr的个数没有满足min.insync.replicas就报错，需要知道的是min.insync.replicas是和ack=-1一起使用的
          if (inSyncSize < minIsr && requiredAcks == -1) {
            throw new NotEnoughReplicasException("Number of insync replicas for partition %s is [%d], below required minimum [%d]"
              .format(topicPartition, inSyncSize, minIsr))
          }

          // 真正的消息追加交给Log对象
          val info = log.appendAsLeader(records, leaderEpoch = this.leaderEpoch, isFromClient)

          // 写入完消息，尝试触发Fetch请求，比如满足消费者的fetch.max.bytes
          replicaManager.tryCompleteDelayedFetch(TopicPartitionOperationKey(this.topic, this.partitionId))
          // we may need to increment high watermark since ISR could be down to 1
          (info, maybeIncrementLeaderHW(leaderReplica))
      }
    }

    // some delayed operations may be unblocked after HW changed
    if (leaderHWIncremented)
      tryCompleteDelayedRequests()

    info
}
```

接下来是Log对象appendAsLeader的方法调用

### appendAsLeader->append实现

appendAsLeader方法直接调用了append方法

```java
private def append(records: MemoryRecords, isFromClient: Boolean, assignOffsets: Boolean, leaderEpoch: Int): LogAppendInfo = {
    maybeHandleIOException(s"Error while appending records to $topicPartition in dir ${dir.getParent}") {
      
      val appendInfo = analyzeAndValidateRecords(records, isFromClient = isFromClient)

      // return if we have no valid messages or if this is a duplicate of the last appended entry
      if (appendInfo.shallowCount == 0)
        return appendInfo

      // trim any invalid bytes or partial messages before appending it to the on-disk log
      var validRecords = trimInvalidBytes(records, appendInfo)


      lock synchronized {
      	// assignOffsets写死为true，就不看else了
        if (assignOffsets) {
          // assign offsets to the message set
          val offset = new LongRef(nextOffsetMetadata.messageOffset)
          // firstOffset又重新赋值了
          appendInfo.firstOffset = Some(offset.value)
          val now = time.milliseconds
          // 各种验证
          val validateAndOffsetAssignResult = LogValidator.validateMessagesAndAssignOffsets(validRecords,
              offset,
              time,
              now,
              appendInfo.sourceCodec,
              appendInfo.targetCodec,
              config.compact,
              config.messageFormatVersion.recordVersion.value,
              config.messageTimestampType,
              config.messageTimestampDifferenceMaxMs,
              leaderEpoch,
              isFromClient)

          // 验证通过后的消息
          validRecords = validateAndOffsetAssignResult.validatedRecords
          // 根据校验结果完善appendInfo对象
          appendInfo.maxTimestamp = validateAndOffsetAssignResult.maxTimestamp
          appendInfo.offsetOfMaxTimestamp = validateAndOffsetAssignResult.shallowOffsetOfMaxTimestamp
          appendInfo.lastOffset = offset.value - 1
          appendInfo.recordConversionStats = validateAndOffsetAssignResult.recordConversionStats
          if (config.messageTimestampType == TimestampType.LOG_APPEND_TIME)
            appendInfo.logAppendTime = now

          if (validateAndOffsetAssignResult.messageSizeMaybeChanged) {
            for (batch <- validRecords.batches.asScala) {
              // 每一批消息不能比max.message.bytes大
              if (batch.sizeInBytes > config.maxMessageSize) {
                throw new RecordTooLargeException(s"Message batch size is ${batch.sizeInBytes} bytes in append to" +
                  s"partition $topicPartition which exceeds the maximum configured size of ${config.maxMessageSize}.")
              }
            }
          }
        } 

        // update the epoch cache with the epoch stamped onto the message by the leader
        validRecords.batches.asScala.foreach { batch =>
          if (batch.magic >= RecordBatch.MAGIC_VALUE_V2)
            _leaderEpochCache.assign(batch.partitionLeaderEpoch, batch.baseOffset)
        }

        // check messages set size may be exceed config.segmentSize
        // MemoryRecords总消息不能比segment.bytes大
        if (validRecords.sizeInBytes > config.segmentSize) {
          throw new RecordBatchTooLargeException(s"Message batch size is ${validRecords.sizeInBytes} bytes in append " +
            s"to partition $topicPartition, which exceeds the maximum configured segment size of ${config.segmentSize}.")
        }

        // maybe roll the log if this segment is full
        // 是否需要生成一个新的segment，具体判断条件见下文
        val segment = maybeRoll(validRecords.sizeInBytes, appendInfo)

        // 保存位移的VO
        val logOffsetMetadata = LogOffsetMetadata(
          messageOffset = appendInfo.firstOrLastOffsetOfFirstBatch,
          segmentBaseOffset = segment.baseOffset,
          relativePositionInSegment = segment.size)

        // 真正append日志的是LogSegment对象
        segment.append(largestOffset = appendInfo.lastOffset,
          largestTimestamp = appendInfo.maxTimestamp,
          shallowOffsetOfMaxTimestamp = appendInfo.offsetOfMaxTimestamp,
          records = validRecords)

        // 更新LEO，lastOffset + 1
        updateLogEndOffset(appendInfo.lastOffset + 1)

        if (unflushedMessages >= config.flushInterval)
          flush()

        appendInfo
      }
    }
 }
```

这里必须要关注一下analyzeAndValidateRecords，因为它返回了LogAppendInfo对象,但是在讲解之前，需要和大家对kafka的消息结构所有了解，可以参考我之前的文章: [kafka消息格式与日志存储原理分析]()

#### analyzeAndValidateRecords

```java
private def analyzeAndValidateRecords(records: MemoryRecords, isFromClient: Boolean): LogAppendInfo = {
  var shallowMessageCount = 0
  var validBytesCount = 0
  var firstOffset: Option[Long] = None
  var lastOffset = -1L
  var sourceCodec: CompressionCodec = NoCompressionCodec
  var monotonic = true
  var maxTimestamp = RecordBatch.NO_TIMESTAMP
  var offsetOfMaxTimestamp = -1L
  var readFirstMessage = false
  var lastOffsetOfFirstBatch = -1L

  // MemoryRecords
  info(s"MemoryRecords is ${records}")

  for (batch <- records.batches.asScala) {
    // we only validate V2 and higher to avoid potential compatibility issues with older clients
    if (batch.magic >= RecordBatch.MAGIC_VALUE_V2 && isFromClient && batch.baseOffset != 0)
      throw new InvalidRecordException(s"The baseOffset of the record batch in the append to $topicPartition should " +
        s"be 0, but it is ${batch.baseOffset}")

    // update the first offset if on the first message. For magic versions older than 2, we use the last offset
    // to avoid the need to decompress the data (the last offset can be obtained directly from the wrapper message).
    // For magic version 2, we can get the first offset directly from the batch header.
    // When appending to the leader, we will update LogAppendInfo.baseOffset with the correct value. In the follower
    // case, validation will be more lenient.
    // Also indicate whether we have the accurate first offset or not
    // readFirstMessage就是想取第一批消息的数据
    if (!readFirstMessage) {
      if (batch.magic >= RecordBatch.MAGIC_VALUE_V2)
        firstOffset = Some(batch.baseOffset)
      lastOffsetOfFirstBatch = batch.lastOffset
      readFirstMessage = true
    }

    // check that offsets are monotonically increasing
    // offset是否单调递增
    if (lastOffset >= batch.lastOffset)
      monotonic = false

    // update the last offset seen
    lastOffset = batch.lastOffset

    // Check if the message sizes are valid.
    val batchSize = batch.sizeInBytes
    if (batchSize > config.maxMessageSize) {
      brokerTopicStats.topicStats(topicPartition.topic).bytesRejectedRate.mark(records.sizeInBytes)
      brokerTopicStats.allTopicsStats.bytesRejectedRate.mark(records.sizeInBytes)
      throw new RecordTooLargeException(s"The record batch size in the append to $topicPartition is $batchSize bytes " +
        s"which exceeds the maximum configured value of ${config.maxMessageSize}.")
    }

    // check the validity of the message by checking CRC
    batch.ensureValid()

    if (batch.maxTimestamp > maxTimestamp) {
      maxTimestamp = batch.maxTimestamp
      offsetOfMaxTimestamp = lastOffset
    }

    shallowMessageCount += 1
    validBytesCount += batchSize

    val messageCodec = CompressionCodec.getCompressionCodec(batch.compressionType.id)
    if (messageCodec != NoCompressionCodec)
      sourceCodec = messageCodec
  }

  // Apply broker-side compression if any
  val targetCodec = BrokerCompressionCodec.getTargetCompressionCodec(config.compressionType, sourceCodec)
   /*
    * @param firstOffset v2版本都是0
    * @param lastOffset 消息集(MemoryRecords)中最后一条消息的位移
    * @param maxTimestamp 消息集(MemoryRecords)中最大的Timestamp，一般就是最后一条消息的时间戳
    * @param offsetOfMaxTimestamp 最大时间戳对应的位移
    * @param logAppendTime -1，RecordBatch.NO_TIMESTAMP
    * @param logStartOffset 这是当前所有Segment的起始位移(过期的会清楚)
    * @param recordConversionStats assignOffsets=false时为null，此时为EMPTY
    * @param sourceCodec 生产者设置的压缩
    * @param targetCodec broker设置的压缩
    * @param shallowCount 浅层message的个数，一般都是1
    * @param validBytes 验证过的消息字节数
    * @param offsetsMonotonic 消息位移是否单调递增
    * @param lastOffsetOfFirstBatch 第一批消息的lastOffset
    */
  val appendInfo = LogAppendInfo(firstOffset, lastOffset, maxTimestamp, offsetOfMaxTimestamp, RecordBatch.NO_TIMESTAMP, logStartOffset,
    RecordConversionStats.EMPTY, sourceCodec, targetCodec, shallowMessageCount, validBytesCount, monotonic, lastOffsetOfFirstBatch)
  info(s"analyzeAndValidateRecords append info is ${appendInfo.toString}")
  appendInfo
}
```

### Roll Segment(滚动日志段)

maybeRoll方法用于判断是否需要roll一个新Segment，什么叫做roll可以参考[kafka消息格式与日志存储原理分析]()

```java
private def maybeRoll(messagesSize: Int, appendInfo: LogAppendInfo): LogSegment = {
    val segment = activeSegment
    val now = time.milliseconds

    val maxTimestampInMessages = appendInfo.maxTimestamp
    val maxOffsetInMessages = appendInfo.lastOffset

    if (segment.shouldRoll(messagesSize, maxTimestampInMessages, maxOffsetInMessages, now)) {
      // 省略了代码
      roll(maxOffsetInMessages - Integer.MAX_VALUE)
    } else {
      // 不需要Roll，就返回当前正在使用的Segment：activeSegment
      segment
    }
}
```
Roll方法其实就是在新建.log, .index, .timeIndex文件，如果用了事务，还会有.txnindex文件
```java
def roll(expectedNextOffset: Long = 0): LogSegment = {
    maybeHandleIOException(s"Error while rolling log segment for $topicPartition in dir ${dir.getParent}") {
      val start = time.hiResClockMs()
      lock synchronized {
        checkIfMemoryMappedBufferClosed()
        val newOffset = math.max(expectedNextOffset, logEndOffset)
        // 新建.log, .index, .timeIndex文件，如果用了事务，还会有.txnindex文件
        val logFile = Log.logFile(dir, newOffset)
        val offsetIdxFile = offsetIndexFile(dir, newOffset)
        val timeIdxFile = timeIndexFile(dir, newOffset)
        val txnIdxFile = transactionIndexFile(dir, newOffset)
        // 检查是否已存在以上文件，存在则先删除
        for (file <- List(logFile, offsetIdxFile, timeIdxFile, txnIdxFile) if file.exists) {
          warn(s"Newly rolled segment file ${file.getAbsolutePath} already exists; deleting it first")
          Files.delete(file.toPath)
        }

        // segments使用一个跳表构建的Map，说明Segment使用跳表组织的
        // key是Segment的baseOffset，value是Segment对象
        Option(segments.lastEntry).foreach(_.getValue.onBecomeInactiveSegment())

        // 创建LogSegment，添加到segments集合里
        val segment = LogSegment.open(dir,
          baseOffset = newOffset,
          config,
          time = time,
          fileAlreadyExists = false,
          initFileSize = initFileSize,
          preallocate = config.preallocate)
        val prev = addSegment(segment)
        // 说明已存在
        if (prev != null)
          throw new KafkaException(s"Trying to roll a new log segment for topic partition $topicPartition with " +
            s"start offset $newOffset while it already exists.")
        // 更新LEO
        updateLogEndOffset(nextOffsetMetadata.messageOffset)
        // 将recoveryPoint到新segment offset，也就是老的segment刷盘，包含4个文件：.log, .index, .timeIndex，.txnindex
        scheduler.schedule("flush-log", () => flush(newOffset), delay = 0L)

        segment
      }
    }
}
```



### 日志文件与索引文件的写入

Log#append方法在analyzeAndValidateRecords与maybeRoll操作之后，开始进行消息的写入，
Segment的append方法

```java
def append(largestOffset: Long,
             largestTimestamp: Long,
             shallowOffsetOfMaxTimestamp: Long,
             records: MemoryRecords): Unit = {
  if (records.sizeInBytes > 0) {
    val physicalPosition = log.sizeInBytes()
    if (physicalPosition == 0)
      rollingBasedTimestamp = Some(largestTimestamp)

    ensureOffsetInRange(largestOffset)

    // 追加消息
    val appendedBytes = log.append(records)
    // Update the in memory max timestamp and corresponding offset.
    if (largestTimestamp > maxTimestampSoFar) {
      maxTimestampSoFar = largestTimestamp
      offsetOfMaxTimestamp = shallowOffsetOfMaxTimestamp
    }
    // offsetIndex和timeIndex的添加
    // indexIntervalBytes = index.interval.bytes， 每4KB的消息，建立一个索引
    if (bytesSinceLastIndexEntry > indexIntervalBytes) {
      offsetIndex.append(largestOffset, physicalPosition)
      timeIndex.maybeAppend(maxTimestampSoFar, offsetOfMaxTimestamp)
      bytesSinceLastIndexEntry = 0
    }
    bytesSinceLastIndexEntry += records.sizeInBytes
  }
}
```
以下是日志的具体写入过程，可以看到就是java nio的操作了
```java
public int append(MemoryRecords records) throws IOException {
    int written = records.writeFullyTo(channel);
    size.getAndAdd(written);
    return written;
}
public int writeFullyTo(GatheringByteChannel channel) throws IOException {
    buffer.mark();
    int written = 0;
    while (written < sizeInBytes())
        written += channel.write(buffer);
    buffer.reset();
    return written;
}
```


# 总结
总的来说kafka在ReplicaManager,Partition,Log,LogSegment对象的层层调用来append消息。

在ack=-1时，因为follower同步之后，才算是消息提交(commit)，而在消息append过程中，并不知道什么时候follower完成同步

kafka的做法是多次尝试完成生产者请求，因此在源码中我们可以看到在append完成后，还会尝试完成生产者请求，否则放入Purgatory中监听(tryCompleteElseWatch)。

等待follower副本同步完成，再次尝试完成生产者请求(tryCompleteDelayedFetch)













