---
title: kafka-server端源码分析之拉取消息
date: 2019-12-17 23:03:53
categories: Kafka Tutorial
tags: [kafka,中间件,消息]
toc: true
comments: true
---

>发送fetch请求的对象有2类：client和follower，client拉取时有高水位线的限制，follower则没有，本文仅介绍client，
follower拉取时涉及到副本同步，以后单独分析

# 术语回顾

![](https://ae01.alicdn.com/kf/H7a61234572984b95a98cc61ef9abf1f4C.png)
在kafka消息中有2个重要的术语：HW(HighWatermark)，LEO(Log End Offset)

HW会在生产者发送的消息写入后，等待follower副本同步完成后更新, HW之前的消息称之为committed(已提交的消息)，消费者只能消费HW之前的消息
而LEO则是在消息写入到本地leader副本后立即更新，它的值是最后一条消息的下一个位移，图中15被虚线标注，表示LEO处没有消息


# 源码分析

## client端发送的FETCH请求

截止到2.0.1版本，Fetch请求已经是V8版本了，client端发送的FetchRequest在Fetcher#sendFetches方法中初始化

```java
final FetchSessionHandler.FetchRequestData data = entry.getValue();
final FetchRequest.Builder request = FetchRequest.Builder
        // fetch.max.wait.ms: 拉取的等待时间
        // fetch.min.bytes: 至少拉取的字节数，没有达到则等待
        .forConsumer(this.maxWaitMs, this.minBytes, data.toSend())
        // 事务隔离级别，read_uncommited
        .isolationLevel(isolationLevel)
        // fetch.max.bytes: 拉取的最大字节
        .setMaxBytes(this.maxBytes)
        .metadata(data.metadata())
        .toForget(data.toForget());
```

该请求体略微复杂，首先关注下data.toSend方法，它返回的是一个Map<TopicPartition, PartitionData>，表示一个消费者可以消费多个topic的多个分区
```java
public static final class PartitionData {
    // 拉取的offset
    public final long fetchOffset;
    // 
    public final long logStartOffset;
    // max.partition.fetch.bytes：每个分区拉取的最大值
    public final int maxBytes;
}
```
data.metadata方法返回的FetchMetadata主要包含epoch和sessionId两个字段，data.toForget返回的是一个分区数组
相信大部分参数大家都是熟悉的，而epoch，sessionId和toForget是专门用于FetchSession的实现，它是1.1.0版本新增的功能，这里我只说它出现的背景

### FetchSession背景

FetchSession出现是为了解决什么问题？
在kafka集群中的topic和partition达到一定规模后，会产生大量的Fetch请求，既包含消息拉取，也包含副本同步，而后者的请求量会很大。
假设100个topic，每个topic有3个分区，每个分区有3个副本，那么同时就有600个follower副本发送Fetch请求，再加上活动期间的业务量猛增的消费者的请求，Fetch请求的QPS将会很高
并且Fetch的请求体本身就很大，通常有几十KB，但是大部分参数都是不变的，比如订阅的分区，拉取参数等，因此可以将这些参数缓存在server端，client用一个session id来代替一次会话
这对Fetch请求的性能将是一个瓶颈，因此需要对请求体优化

其中大部分的参数大家都很熟悉，主要说2个不常见的参数：metadata和toForget
这两个参数是kafka 1.1.0版本之后新加的，用于FetchSession的实现，主要解决了在server端没有接收到消息时，消费者会空轮询，在topic分区较多时，FetchSession为Fetch请求体起到了瘦身的作用

想象一下每个client不止订阅一个topic，也会不止分配到一个TopicPartition，消费者在发送FETCH请求之前，要知道每个partition的leader副本在哪个broker上，然后按照broker分组，fetch请求体很大并不是空穴来风，kafka对此进行优化是很有必要的

以下是Fetch请求体格式，红框内的参数先不必关注，之后分析FetchSession相关内容
![FetchRequest](https://ae01.alicdn.com/kf/Haf611a59880e479d953217e41e26eaf47.png)

## FETCH请求

FETCH请求同样也是在KafkaApis类中处理，此处省略部分代码，如FetchSession相关，关注拉取的核心流程

```java
def handleFetchRequest(request: RequestChannel.Request) {
    val versionId = request.header.apiVersion
    val clientId = request.header.clientId
    val fetchRequest = request.body[FetchRequest]

    // FetchSession来做增量的fetch请求
    val fetchContext = fetchManager.newContext(fetchRequest.metadata(),
          fetchRequest.fetchData(),
          fetchRequest.toForget(),
          // isFromFollower： replicaId是否大于0表示是follower
          fetchRequest.isFromFollower())

    // 异常响应方法：传入一个Error，返回发送异常时的Response
    def errorResponse[T >: MemoryRecords <: BaseRecords](error: Errors): FetchResponse.PartitionData[T] = {
      new FetchResponse.PartitionData[T](error, FetchResponse.INVALID_HIGHWATERMARK, FetchResponse.INVALID_LAST_STABLE_OFFSET,
        FetchResponse.INVALID_LOG_START_OFFSET, null, MemoryRecords.EMPTY)
    }

    val erroneous = mutable.ArrayBuffer[(TopicPartition, FetchResponse.PartitionData[Records])]()
    val interesting = mutable.ArrayBuffer[(TopicPartition, FetchRequest.PartitionData)]()

    // 筛选TP, 放入interesting集合中
    if (fetchRequest.isFromFollower()) {
    	// 暂不关心follower fetch
    } else {
      // foreachPartition由FullFetchContext和IncrementalFetchContext实现
      fetchContext.foreachPartition { (topicPartition, data) =>
      	// consumer要有READ读权限，而且在metadata里有记录
        if (!authorize(request.session, Read, Resource(Topic, topicPartition.topic, LITERAL)))
          erroneous += topicPartition -> errorResponse(Errors.TOPIC_AUTHORIZATION_FAILED)
        // 元数据中要有该topicPartition
        else if (!metadataCache.contains(topicPartition))
          erroneous += topicPartition -> errorResponse(Errors.UNKNOWN_TOPIC_OR_PARTITION)
        else
          interesting += (topicPartition -> data)
      }
    }

    // 响应的回调函数，最后分析
    def processResponseCallback(responsePartitionData: Seq[(TopicPartition, FetchPartitionData)]): Unit = {
    }

    if (interesting.isEmpty)
      processResponseCallback(Seq.empty)
    else {
      // replicaManager为入口调用
      // call the replica manager to fetch messages from the local replica
      replicaManager.fetchMessages(
        fetchRequest.maxWait.toLong,
        fetchRequest.replicaId,
        fetchRequest.minBytes,
        fetchRequest.maxBytes,
        versionId <= 2, // 从后面的代码看，version <= 2时，至少返回第一条消息，哪怕它的大小超出了maxBytes
        interesting,
        replicationQuota(fetchRequest),
        processResponseCallback,
        fetchRequest.isolationLevel)
    }
}
```
handleFetchRequest方法主要是过滤请求中可用的TopicPartition作为interesting参数，最后连带响应的回调函数一起传给replicaManager的fetchMessages方法，processResponseCallback响应回调最终再分析

# ReplicaManager#fetchMessages

fetchMessages方法中主要调用了readFromLog->readFromLocalLog方法来读取消息，readFromLocalLog返回的是一个LogReadResult对象，如果当前是follower副本发送的用于同步的fetch请求，还会调用updateFollowerLogReadResults更新同步状态，这一部分内容在[kafka server端源码分析之副本同步]()中做了详细阐述

由于Consumer拉取消息有一系列的参数控制，如fetch.max.wait.ms，fetch.min.bytes，fetch.max.wait.ms等，让本次fetch不能立即完成，需要新建一个DelayedOption对象，放入Purgatory中，等待后续操作触发本次请求的完成(complete)。Purgatory可以简单理解为一个延迟队列

```java
def fetchMessages(timeout: Long,
                    replicaId: Int,
                    fetchMinBytes: Int,
                    fetchMaxBytes: Int,
                    hardMaxBytesLimit: Boolean,
                    fetchInfos: Seq[(TopicPartition, PartitionData)],
                    quota: ReplicaQuota = UnboundedQuota,
                    responseCallback: Seq[(TopicPartition, FetchPartitionData)] => Unit,
                    isolationLevel: IsolationLevel) {
    val isFromFollower = Request.isValidBrokerId(replicaId)
    val fetchOnlyFromLeader = replicaId != Request.DebuggingConsumerId && replicaId != Request.FutureLocalReplicaId // 还有不从leader同步的？
    // follower fetch时没有高水位线的限制
    val fetchOnlyCommitted = !isFromFollower && replicaId != Request.FutureLocalReplicaId

    // 先定义后调用
    def readFromLog(): Seq[(TopicPartition, LogReadResult)] = {
      val result = readFromLocalLog(
        replicaId = replicaId,
        fetchOnlyFromLeader = fetchOnlyFromLeader,
        readOnlyCommitted = fetchOnlyCommitted,
        fetchMaxBytes = fetchMaxBytes,
        // vision<=2，目前=8，说明为false，v2版本有最大字节限制吗？
        hardMaxBytesLimit = hardMaxBytesLimit,
        // fetchInfos是读取的关键，这是fetch参数，注意要读取多个分区，这是个
        readPartitionInfo = fetchInfos,
        // 配额
        quota = quota,
        // 事务隔离级别，默认read_uncommited
        isolationLevel = isolationLevel)
      // 这里是follower的fetch结果处理
      if (isFromFollower) updateFollowerLogReadResults(replicaId, result)
      else result
    }

	// 调用并返回一个(TopicPartition, LogReadResult)集合
    val logReadResults = readFromLog()

    // check if this fetch request can be satisfied right away
    // LogReadResult集合
    val logReadResultValues = logReadResults.map { case (_, v) => v }
    // 读取的消息大小之和
    val bytesReadable = logReadResultValues.map(_.info.records.sizeInBytes).sum
    // 结果中是否有错误
    val errorReadingData = logReadResultValues.foldLeft(false) ((errorIncurred, readResult) =>
      errorIncurred || (readResult.error != Errors.NONE))

    /**
      * 能够立即返回给客户端的4种情况
      * 1. fetch请求没有大于0的wait时间,参考fetch.max.wait.ms设置
      * 2. fetch请求要拉取的分区为空
      * 3. 根据fetch.min.bytes的设置，有足够的数据返回
      * 4. 出现异常
      */
    if (timeout <= 0 || fetchInfos.isEmpty || bytesReadable >= fetchMinBytes || errorReadingData) {
      // fetchPartitionData是一个TopicPartition -> FetchPartitionData 的map集合
      val fetchPartitionData = logReadResults.map { case (tp, result) =>
        tp -> FetchPartitionData(result.error, result.highWatermark, result.leaderLogStartOffset, result.info.records,
          result.lastStableOffset, result.info.abortedTransactions)
      }
      // 调用响应回调函数
      responseCallback(fetchPartitionData)
    } else { // 创建响应的DelayOption，放入purgatory中，等待完成

      // construct the fetch results from the read results
      val fetchPartitionStatus = logReadResults.map { case (topicPartition, result) =>
        // collectFirst：根据function find first element
        // fetchInfos是请求参数(TopicPartition, PartitionData)集合，
        // 意思就是从读取结果logReadResults里的TopicPartition和fetchInfos里的TopicPartition匹配
        // 找出该TopicPartition的PartitionData请求参数
        val fetchInfo = fetchInfos.collectFirst {
          case (tp, v) if tp == topicPartition => v
        }.getOrElse(sys.error(s"Partition $topicPartition not found in fetchInfos"))

        // 赋值给外层的fetchPartitionStatus
        (topicPartition, FetchPartitionStatus(result.info.fetchOffsetMetadata, fetchInfo))
      }
      val fetchMetadata = FetchMetadata(fetchMinBytes, fetchMaxBytes, hardMaxBytesLimit, fetchOnlyFromLeader,
        fetchOnlyCommitted, isFromFollower, replicaId, fetchPartitionStatus)
      val delayedFetch = new DelayedFetch(timeout, fetchMetadata, this, quota, isolationLevel, responseCallback)

      // create a list of (topic, partition) pairs to use as keys for this delayed fetch operation
      // 以分区为delay的watchKey
      val delayedFetchKeys = fetchPartitionStatus.map { case (tp, _) => new TopicPartitionOperationKey(tp) }

      // try to complete the request immediately, otherwise put it into the purgatory;
      // this is because while the delayed fetch operation is being created, new requests
      // may arrive and hence make this operation completable.
      // 先尝试一次，不行就放入Purgatory中
      delayedFetchPurgatory.tryCompleteElseWatch(delayedFetch, delayedFetchKeys)
    }
}
```

接下来就从readFromLocalLog方法看看如何读取消息

## readFromLocalLog方法

首先明确性该方法的入参和返回值，入参前文有详细注释，返回值则是一个(TopicPartition, LogReadResult)集合，前文也已提到

```java
/**
   * Read from multiple topic partitions at the given offset up to maxSize bytes
   */
def readFromLocalLog(replicaId: Int,
                       fetchOnlyFromLeader: Boolean,
                       readOnlyCommitted: Boolean,
                       fetchMaxBytes: Int,
                       hardMaxBytesLimit: Boolean,
                       readPartitionInfo: Seq[(TopicPartition, PartitionData)],
                       quota: ReplicaQuota,
                       isolationLevel: IsolationLevel): Seq[(TopicPartition, LogReadResult)] = {

	/**
      * 又是先定义后调用，从后面的代码块这是在遍历请求参数中的TopicPartition集合
      * 作用是读取一个分区里的消息
      * @param tp 要读取的分区
      * @param fetchInfo 读取的参数，如从哪里开始读，读多少
      * @param limitBytes fetchMaxBytes参数
      * @param minOneMessage 是否至少读第一条后立即返回，即使它比fetchMaxBytes大，true
      * @return 读取的结果
      */
	def read(tp: TopicPartition, fetchInfo: PartitionData, limitBytes: Int, minOneMessage: Boolean): LogReadResult = {
		val offset = fetchInfo.fetchOffset //从哪fetch
		val partitionFetchSize = fetchInfo.maxBytes // fetch多少
		val followerLogStartOffset = fetchInfo.logStartOffset // 这应该是针对follower的，consumer始终为-1

		// decide whether to only fetch from leader
		val localReplica = if (fetchOnlyFromLeader)
		  //先找leader副本
		  getLeaderReplicaIfLocal(tp)
		else
		  getReplicaOrException(tp)

		// hw
		val initialHighWatermark = localReplica.highWatermark.messageOffset
		// 事务相关
		val lastStableOffset = if (isolationLevel == IsolationLevel.READ_COMMITTED)
		  Some(localReplica.lastStableOffset.messageOffset)
		else
		  None

		// decide whether to only fetch committed data (i.e. messages below high watermark)
		val maxOffsetOpt = if (readOnlyCommitted)
		  // 没开启事务时lastStableOffset应该为None
		  // 这里返回的还是initialHighWatermark
		  Some(lastStableOffset.getOrElse(initialHighWatermark))
		else
		  None
		val initialLogEndOffset = localReplica.logEndOffset.messageOffset // LEO
		// 这应该是副本目前所有Segment的初始位移(第一个Segment的baseOffset),会随着日志清理改变
		val initialLogStartOffset = localReplica.logStartOffset
		val fetchTimeMs = time.milliseconds // 当前时间
		val logReadInfo = localReplica.log match {
		  case Some(log) =>
		    // TODO 目前还没搞清楚PartitionData里的maxBytes和FetchRequest的maxBytes什么区别
		    // limitBytes是请求参数中的maxBytes
		    val adjustedFetchSize = math.min(partitionFetchSize, limitBytes)

		    // Try the read first, this tells us whether we need all of adjustedFetchSize for this partition
		   	// 从Log对象中读取
		    val fetch = log.read(offset, adjustedFetchSize, maxOffsetOpt, minOneMessage, isolationLevel)

		    // If the partition is being throttled, simply return an empty set.
		    // 超出配额(被限流)时返回一个空消息
		    if (shouldLeaderThrottle(quota, tp, replicaId))
		      FetchDataInfo(fetch.fetchOffsetMetadata, MemoryRecords.EMPTY)
		    // For FetchRequest version 3, we replace incomplete message sets with an empty one as consumers can make
		    // progress in such cases and don't need to report a `RecordTooLargeException`
		    else if (!hardMaxBytesLimit && fetch.firstEntryIncomplete)
		      FetchDataInfo(fetch.fetchOffsetMetadata, MemoryRecords.EMPTY)
		    else fetch // 返回正常的结果给logReadInfo变量

		  case None =>
		    error(s"Leader for partition $tp does not have a local log")
		    FetchDataInfo(LogOffsetMetadata.UnknownOffsetMetadata, MemoryRecords.EMPTY)
		}

		// 返回结果
		LogReadResult(info = logReadInfo,
		              highWatermark = initialHighWatermark,
		              leaderLogStartOffset = initialLogStartOffset,
		              leaderLogEndOffset = initialLogEndOffset,
		              followerLogStartOffset = followerLogStartOffset,
		              fetchTimeMs = fetchTimeMs,
		              readSize = partitionFetchSize,
		              lastStableOffset = lastStableOffset,
		              exception = None)
	  
	}

	var limitBytes = fetchMaxBytes
	val result = new mutable.ArrayBuffer[(TopicPartition, LogReadResult)]
	var minOneMessage = !hardMaxBytesLimit // true
	readPartitionInfo.foreach { case (tp, fetchInfo) => // 遍历每个tp，按照消费者的参数读取日志
	  val readResult = read(tp, fetchInfo, limitBytes, minOneMessage)

	  // 拿到读取结果后，更新limitBytes，添加到result集合中
	  val recordBatchSize = readResult.info.records.sizeInBytes
	  // Once we read from a non-empty partition, we stop ignoring request and partition level size limits
	  if (recordBatchSize > 0)
	    minOneMessage = false
	  // fetchMaxBytes 减去 已读取的消息大小
	  limitBytes = math.max(0, limitBytes - recordBatchSize)
	  result += (tp -> readResult)
	}
	result
}
```
readFromLocalLog主要是遍历请求中的分区，调用事先定义好的嵌套方法read，read方法会先找到leader副本，并且准备好读取的各种参数，最终调用Log对象的read方法

而外层的readFromLocalLog在拿到结果之后，会在循环中从fetchMaxBytes里减去已读取的消息大小

## Log#read

我们知道Log只是个逻辑上的概念，本质是一个个Segment文件，每个Segment文件都有自己的起始位移(baseOffset)，
fetch请求要从fetchOffset处开始读取消息，我们常规的做法是先找到要读取的Segment文件，kafka为了加快寻找速度，增加了索引文件的概念，找到后根据fetchMaxBytes参数(当前在循环中，会一直变化)， 在高水位线的限制下调用Segment对象read方法读取消息，返回FetchDataInfo结果对象

以上就是该方法要做的事，Log对象的read方法源码如下
```java
/**
   * Read messages from the log.
   *
   * @param startOffset 从哪里fetch，fetch请求中的fetchOffset参数:
    *                    The offset to begin reading at
   * @param maxLength fetch的maxBytes-已读取的消息大小:
    *                  The maximum number of bytes to read
   * @param maxOffset fetch的上限，即高水位线:
    *                  The offset to read up to, exclusive. (i.e. this offset NOT included in the resulting message set)
   * @param minOneMessage 是否至少fetch一条，即使的大小它已经超出了maxBytes:
    *                      If this is true, the first message will be returned even if it exceeds `maxLength` (if one exists)
   */
  def read(startOffset: Long, maxLength: Int, maxOffset: Option[Long] = None, minOneMessage: Boolean = false,
           isolationLevel: IsolationLevel): FetchDataInfo = {
    maybeHandleIOException(s"Exception while reading from $topicPartition in dir ${dir.getParent}") {
      trace(s"Reading $maxLength bytes from offset $startOffset of length $size bytes")

      // Because we don't use lock for reading, the synchronization is a little bit tricky.
      // We create the local variables to avoid race conditions with updates to the log.
      // 使用局部变量来避免并发锁竞争，nextOffsetMetadata.messageOffset就是LEO
      val currentNextOffsetMetadata = nextOffsetMetadata
      val next = currentNextOffsetMetadata.messageOffset

      // 事务部分，先不关心
      if (startOffset == next) {
        val abortedTransactions =
          if (isolationLevel == IsolationLevel.READ_COMMITTED) Some(List.empty[AbortedTransaction])
          else None
        return FetchDataInfo(currentNextOffsetMetadata, MemoryRecords.EMPTY, firstEntryIncomplete = false,
          abortedTransactions = abortedTransactions)
      }
      // segments是一个跳表做的map，key为Segment的baseOffset，value是LogSegment对象
      // floorEntry是干嘛的？看哪个LogSegment的baseOffset <= startOffset，其实就是在找要读取的LogSegment
      // segmentEntry是一个entry: <baseOffset,LogSegment>
      var segmentEntry = segments.floorEntry(startOffset)

      // return error on attempt to read beyond the log end offset or read below log start offset
      // 异常处理，大于LEO肯定不对，没找到合适的LogSegment也是不对的，至于startOffset < logStartOffset感觉很多余
      if (startOffset > next || segmentEntry == null || startOffset < logStartOffset)
        throw new OffsetOutOfRangeException(s"Received request for offset $startOffset for partition $topicPartition, " +
          s"but we only have log segments in the range $logStartOffset to $next.")

      /**
        * 从baseOffset小于指定offset的Segment里读取消息，但如果Segment里没有消息，
        * 就继续往后面的Segment读,直到读取到了消息，或者到达了log的末尾
        */
      // Do the read on the segment with a base offset less than the target offset
      // but if that segment doesn't contain any messages with an offset greater than that
      // continue to read from successive segments until we get some messages or we reach the end of the log
      while (segmentEntry != null) {
        // 取出LogSegment
        val segment = segmentEntry.getValue

        // 如果fetch读取了active Segment(最后一个正在写入的LogSegment)，在LEO更新前，发生了两次fetch会产生并发竞争，
        // 那么第二次fetch可能会发生OffsetOutOfRangeException，因此我们限制读取已暴露的位置(下面的maxPosition变量)，而不是active Segment的LEO

        // If the fetch occurs on the active segment, there might be a race condition where two fetch requests occur after
        // the message is appended but before the nextOffsetMetadata is updated. In that case the second fetch may
        // cause OffsetOutOfRangeException. To solve that, we cap the reading up to exposed position instead of the log
        // end of the active segment.
        /**
          * maxPosition大概是说segmentEntry如果是最后一个(active Segment)就返回LEO，
          * 否则返回当前Segment的大小
          */
        val maxPosition = {
          if (segmentEntry == segments.lastEntry) {
            val exposedPos = nextOffsetMetadata.relativePositionInSegment.toLong
            // 这个check again真的有用吗，为了解决bug？有点low
            // Check the segment again in case a new segment has just rolled out.
            if (segmentEntry != segments.lastEntry)
            // New log segment has rolled out, we can read up to the file end.
              segment.size
            else
              exposedPos
          } else {
            segment.size
          }
        }
        /**
          * 总结一下这几个入参
          * startOffset：从哪个位置开始读
          * maxOffset：读取的上限，高水位线
          * maxLength：读取的maxBytes
          * maxPosition：目前不知道什么用，LEO或者Segment的size
          * minOneMessage：是否至少读第一条
          */
        val fetchInfo = segment.read(startOffset, maxOffset, maxLength, maxPosition, minOneMessage)
        if (fetchInfo == null) {
          segmentEntry = segments.higherEntry(segmentEntry.getKey)
        } else {
          return isolationLevel match {
            // 默认是READ_UNCOMMITTED，这里的fetchInfo作为返回值
            case IsolationLevel.READ_UNCOMMITTED => fetchInfo
            case IsolationLevel.READ_COMMITTED => addAbortedTransactions(startOffset, segmentEntry, fetchInfo)
          }
        }
      }

      // 上面的while执行到最后一个Segment都还没return，说明我们要读取的消息都被删除了，这种情况返回空消息
      // okay we are beyond the end of the last segment with no data fetched although the start offset is in range,
      // this can happen when all messages with offset larger than start offsets have been deleted.
      // In this case, we will return the empty set with log end offset metadata
      FetchDataInfo(nextOffsetMetadata, MemoryRecords.EMPTY)
    }
}
```

## LogSegment#read

LogSegment#read属于接近底层的方法了，上一小节已经根据一个<baseOffset, Segment>的map找到了相应的Segment，但是要知道默认一个Segment大小为1G，想要在这么大的文件中查询数据，必须依赖索引。

kafka的读取逻辑是先根据二分法找到相应的offset和position，最终通过FileRecords.slice读取区间内的消息
```java
/**
   * @param startOffset 从哪个位置开始读：
    *                    A lower bound on the first offset to include in the message set we read
   * @param maxOffset 读取的上限，高水位线：
    *                  An optional maximum offset for the message set we read
   * @param maxSize fetch的maxBytes-已读取的消息大小：
    *                The maximum number of bytes to include in the message set we read
   * @param maxPosition 目前不知道什么用，LEO或者Segment的size：
    *                    The maximum position in the log segment that should be exposed for read
   * @param minOneMessage 是否至少读第一条：
    *                      If this is true, the first message will be returned even if it exceeds `maxSize` (if one exists)
   *
   * @return The fetched data and the offset metadata of the first message whose offset is >= startOffset,
   *         or null if the startOffset is larger than the largest offset in this log
   */
  @threadsafe
  def read(startOffset: Long, maxOffset: Option[Long], maxSize: Int, maxPosition: Long = size,
           minOneMessage: Boolean = false): FetchDataInfo = {
   
    // 日志文件字节数大小
    val logSize = log.sizeInBytes // this may change, need to save a consistent copy
    // 从index文件里查找offset,position
    val startOffsetAndSize = translateOffset(startOffset)

    val startPosition = startOffsetAndSize.position
    val offsetMetadata = new LogOffsetMetadata(startOffset, this.baseOffset, startPosition)

    val adjustedMaxSize =
      if (minOneMessage) math.max(maxSize, startOffsetAndSize.size)
      else maxSize

    // calculate the length of the message set to read based on whether or not they gave us a maxOffset
    val fetchSize: Int = maxOffset match {
      case Some(offset) =>
        val mapping = translateOffset(offset, startPosition)
        val endPosition =
          if (mapping == null)
            logSize // the max offset is off the end of the log, use the end of the file
          else
            mapping.position
        min(min(maxPosition, endPosition) - startPosition, adjustedMaxSize).toInt
    }

    // log.slice方法是在真正的获取消息
    FetchDataInfo(offsetMetadata, log.slice(startPosition, fetchSize),
      firstEntryIncomplete = adjustedMaxSize < startOffsetAndSize.size)
}
```

## 返回结果处理

经过层层返回，回到最初的的handleFetchRequest方法中，看看processResponseCallback方法中是如何对读取结果进行处理并返回给consumer的

省略配额限流相关代码...

该方法的入参是一个(TopicPartition, FetchPartitionData)，表示每个分区对应的读取结果
```java
// the callback for process a fetch response, invoked before throttling
def processResponseCallback(responsePartitionData: Seq[(TopicPartition, FetchPartitionData)]): Unit = {
  // FetchPartitionData转PartitionData
  val partitions = new util.LinkedHashMap[TopicPartition, FetchResponse.PartitionData[Records]]
  responsePartitionData.foreach { case (tp, data) =>
    val abortedTransactions = data.abortedTransactions.map(_.asJava).orNull
    val lastStableOffset = data.lastStableOffset.getOrElse(FetchResponse.INVALID_LAST_STABLE_OFFSET)
    partitions.put(tp, new FetchResponse.PartitionData(data.error, data.highWatermark, lastStableOffset,
      data.logStartOffset, abortedTransactions, data.records))
  }

  // 错误的分区也要返回各自的错误信息
  erroneous.foreach { case (tp, data) => partitions.put(tp, data) }

  // When this callback is triggered, the remote API call has completed.
  // Record time before any byte-rate throttling.
  request.apiRemoteCompleteTimeNanos = time.nanoseconds

  var unconvertedFetchResponse: FetchResponse[Records] = null
  
  // follower同步的fetch
  if (fetchRequest.isFromFollower) {
    // We've already evaluated against the quota and are good to go. Just need to record it now.
    unconvertedFetchResponse = fetchContext.updateAndGenerateResponseData(partitions)
    val responseSize = sizeOfThrottledPartitions(versionId, unconvertedFetchResponse, quotas.leader)
    quotas.leader.record(responseSize)
    trace(s"Sending Fetch response with partitions.size=${unconvertedFetchResponse.responseData().size()}, " +
      s"metadata=${unconvertedFetchResponse.sessionId()}")
    sendResponseExemptThrottle(request, createResponse(0), Some(updateConversionStats))
  } else {
    // Fetch size used to determine throttle time is calculated before any down conversions.
    // This may be slightly different from the actual response size. But since down conversions
    // result in data being loaded into memory, we should do this only when we are not going to throttle.
    //
    // Record both bandwidth and request quota-specific values and throttle by muting the channel if any of the
    // quotas have been violated. If both quotas have been violated, use the max throttle time between the two
    // quotas. When throttled, we unrecord the recorded bandwidth quota value

    // 大部分都是限流先关代码，先忽略
    val responseSize = fetchContext.getResponseSize(partitions, versionId)
    val timeMs = time.milliseconds()
    val requestThrottleTimeMs = quotas.request.maybeRecordAndGetThrottleTimeMs(request)
    val bandwidthThrottleTimeMs = quotas.fetch.maybeRecordAndGetThrottleTimeMs(request, responseSize, timeMs)

    val maxThrottleTimeMs = math.max(bandwidthThrottleTimeMs, requestThrottleTimeMs)
    if (maxThrottleTimeMs > 0) {
      // Even if we need to throttle for request quota violation, we should "unrecord" the already recorded value
      // from the fetch quota because we are going to return an empty response.
      quotas.fetch.unrecordQuotaSensor(request, responseSize, timeMs)
      if (bandwidthThrottleTimeMs > requestThrottleTimeMs) {
        quotas.fetch.throttle(request, bandwidthThrottleTimeMs, sendResponse)
      } else {
        quotas.request.throttle(request, requestThrottleTimeMs, sendResponse)
      }
      // If throttling is required, return an empty response.
      unconvertedFetchResponse = fetchContext.getThrottledResponse(maxThrottleTimeMs)
    } else {
      // Get the actual response. This will update the fetch context.
      // 这是很关键的一行代码，创建了Response对象，全量和增量的方式有所不同，后续的FetchSession再说
      unconvertedFetchResponse = fetchContext.updateAndGenerateResponseData(partitions)
    }

    // Send the response immediately.
    // 发送响应到Processor的responseQueue中
    sendResponse(request, Some(createResponse(maxThrottleTimeMs)), Some(updateConversionStats))
  }

  /**
    * 很简单的一个方法，里面用maybeConvertFetchedData方法处理版本兼容引起的消息降级转换
    * 然后统计了下bytes out的metric，最终返回FetchResponse
    */
  def createResponse(throttleTimeMs: Int): FetchResponse[BaseRecords] = {
    // Down-convert messages for each partition if required
    val convertedData = new util.LinkedHashMap[TopicPartition, FetchResponse.PartitionData[BaseRecords]]
    unconvertedFetchResponse.responseData().asScala.foreach { case (tp, unconvertedPartitionData) =>
      if (unconvertedPartitionData.error != Errors.NONE)
        debug(s"Fetch request with correlation id ${request.header.correlationId} from client $clientId " +
          s"on partition $tp failed due to ${unconvertedPartitionData.error.exceptionName}")
      convertedData.put(tp, maybeConvertFetchedData(tp, unconvertedPartitionData))
    }

    // Prepare fetch response from converted data
    val response = new FetchResponse(unconvertedFetchResponse.error(), convertedData, throttleTimeMs,
      unconvertedFetchResponse.sessionId())
    response.responseData.asScala.foreach { case (topicPartition, data) =>
      // record the bytes out metrics only when the response is being sent
      brokerTopicStats.updateBytesOut(topicPartition.topic, fetchRequest.isFromFollower, data.records.sizeInBytes)
    }
    info(s"fetch response is ${response}")
    response
  }

  def updateConversionStats(send: Send): Unit = {
    send match {
      case send: MultiRecordsSend if send.recordConversionStats != null =>
        send.recordConversionStats.asScala.toMap.foreach {
          case (tp, stats) => updateRecordConversionStats(request, tp, stats)
        }
      case _ =>
    }
  }
}
```

# 总结
fetch请求处理流程调用的对象基本和produce请求类似，需要注意的几点是：
1. fetch请求分为consumer和follower，server端用一个replicaId字段判断，consumer为-1
2. consumer读取有高水位线的限制，follower则没有
3. consumer受限于各种参数，不会立即响应，需要放入purgatory延迟队列中等待完成
4. 响应回调中遇到了限流，FetchSession，消息降级等过程

![consumer fetch流程](https://ae01.alicdn.com/kf/Hfe405813f8c148df945920680455868dt.png)

部分图片引用：
https://www.cnblogs.com/huxi2b/p/9335064.html
https://www.cnblogs.com/huxi2b/p/7453543.html

