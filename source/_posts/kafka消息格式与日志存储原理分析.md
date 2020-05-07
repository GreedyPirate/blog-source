---
title: kafka消息格式与日志存储原理分析
date: 2020-02-01 14:35:29
categories: Kafka Tutorial
tags: [kafka,中间件,消息]
toc: true
comments: true
---

kafka自0.11.0.0版本之后消息体升级到了V2版本，本文从生产者消息发送，broker消息存储，消息读取等几个部分作为切入点，来分析kafka的消息流转


# 写入
producer通过PRODUCE请求将消息发送给broker，我们来看一下发送的内容是什么


```java
private final short acks;
private final int timeout;
private final String transactionalId;

private final Map<TopicPartition, Integer> partitionSizes;

private volatile Map<TopicPartition, MemoryRecords> partitionRecords;
private boolean transactional = false;
private boolean idempotent = false;

```
以上代码摘取自ProduceRequest类，它是producer发送消息的请求对象，其中消息载体为：Map<TopicPartition, MemoryRecords> partitionRecords

该map对象表示每一个分区对应的消息，那么MemoryRecords是如何包装消息的呢

MemoryRecords中一个重要的变量是: Iterable[MutableRecordBatch] batches， 如何理解这个对象呢，有个很取巧的办法，看MemoryRecords的toString方法

```java
@Override
public String toString() {
    StringBuilder builder = new StringBuilder();
    builder.append('[');

    // 自己定义的counter
    int batchCounter = 0, recordCounter = 0;

    Iterator<MutableRecordBatch> batchIterator = batches.iterator();
    while (batchIterator.hasNext()) {
        batchCounter++;

        RecordBatch batch = batchIterator.next();
        try (CloseableIterator<Record> recordsIterator = batch.streamingIterator(BufferSupplier.create())) {
            while (recordsIterator.hasNext()) {
                recordCounter++;
                Record record = recordsIterator.next();
                appendRecordToStringBuilder(builder, record.toString());
                if (recordsIterator.hasNext())
                    builder.append(", ");
            }
        } catch (KafkaException e) {
            appendRecordToStringBuilder(builder, "CORRUPTED");
        }
        if (batchIterator.hasNext())
            builder.append(", ");
    }
    builder.append(']');
    builder.append(", batch count is ").append(batchCounter).append(", record count is ").append(recordCounter);
    return builder.toString();
}
```
可以看到MemoryRecords中有一个RecordBatch(MutableRecordBatch是它的子接口)集合，而一个RecordBatch中又有一个Record集合

MemoryRecords，RecordBatch，Record三者的关系如下

![消息类图](https://ae01.alicdn.com/kf/H69850c700b184507b2b6019442356429P.png)

其中大部分都是接口，具体实现就是MemoryRecords，DefaultRecordBatch，DefaultRecord三者的关系

小结一下：producer给broker发送一批消息：Map<TopicPartition, MemoryRecords>，每个分区对应的消息用MemoryRecords表示，它有一个batchs变量，表示一个DefaultRecordBatch集合
而每一个DefaultRecordBatch表示一批消息，里面的每一条消息用DefaultRecord对象表示

但是大家也应该发现一个问题了，一批消息用一个DefaultRecordBatch表示就好了，为什么要包装一层Iterable[MutableRecordBatch] batches
其实在上面的toString发送中，笔者已经做了测试，该集合的大小始终为1，只需要关心DefaultRecordBatch与DefaultRecord即可

## 消息批与消息格式

DefaultRecordBatch与DefaultRecord的结构在各自的类注释中已写明

```java
RecordBatch =>
	BaseOffset => Int64
	Length => Int32
	PartitionLeaderEpoch => Int32
	Magic => Int8
	CRC => Uint32
	Attributes => Int16
	LastOffsetDelta => Int32 // also serves as LastSequenceDelta
	FirstTimestamp => Int64
	MaxTimestamp => Int64
	ProducerId => Int64
	ProducerEpoch => Int16
	BaseSequence => Int32
	Records => [Record]

Record =>
	Length => Varint
	Attributes => Int8
	TimestampDelta => Varlong
	OffsetDelta => Varint
	Key => Bytes
	Value => Bytes
	Headers => [HeaderKey HeaderValue]
		HeaderKey => String
		HeaderValue => Bytes
```
为了方便观看，截取两张书中的图片
![单条消息Record结构](https://ae01.alicdn.com/kf/H9687116c6b684543acc6b46a5c217098H.png)
![批量消息RecordBatch结构](https://ae01.alicdn.com/kf/Hdad2d4d8764c434a8486ee90080aecbeM.png)

## 日志段Segment

消息真正落盘到文件系统是以日志存储，每一个分区，注意是分区级别，都对应了一个日志**目录**，下图是一个典型的分区目录，表示topic test-1的第0个分区
![分区日志目录](https://ae01.alicdn.com/kf/H6501d183d9804508ba634d0ef45947e8k.png)

而日志更细颗粒度的存储方式是一个个以.log结尾的Segment(日志段)，以下介绍几个关于日志的概念，对大家阅读源码有很大帮助

首先大家要回忆下log4j，它在线上服务器一般是这样使用的：不论何时，只有一个日志文件在写入，一到整点，就不再写入，新建下一个日志文件，继续写入，设置日志最大保留时间为30天，30天以前的日志自动删除

以上关于log4j的使用对大家理解kafka日志的运作大有帮助，二者有很多相似之处

1. activeSegment：活动的日志段，只有该Segment写入日志
2. roll Segment：在kafka中，每个Segment默认为1G，由log.segment.bytes参数控制，当达到1G时(已经写不下新消息了)，就会新建下一个Segment，文件名是offset.log，同时原来的文件变为只读
3. Segment的baseOffset：baseOffset就是Segment的第一条消息的位移，这也是Segment的文件名，读取消息时，只要我们知道了第一个比消息的offset大的baseOffset，那么它的前一个Segment就是消息所在的Segment，通过这样可以很快定位到消息在哪个Segment文件，所以见到Map<Long, Segment>的对象大家一看就懂了

4. logStartOffset: 它表示**第一个Segment**的起始offset，也是当前所有Segment的起始offset，在上图中它的值为203000。我们知道，kafka的日志是会过期的，也就是说logStartOffset在过期的Segment被删除之后是会变的

5. largestTimestamp: 每个Segment中最大的时间戳，也就是最后一条消息的时间戳

## 导出

.log文件是不能直接打开的，我们使用以下命令将其保存到本地文件中。注：Segment默认1G，不要在线上随意上使用，可先下载到内存较大的机器上执行
```sh
# 消息
./kafka-run-class.sh kafka.tools.DumpLogSegments --files /Users/admin/private/kafka/data/test-1-0/00000000000000203000.log --print-data-log > ~/message.log

#索引
./kafka-run-class.sh kafka.tools.DumpLogSegments --files /Users/admin/private/kafka/data/test-1-0/00000000000000203000.index --print-data-log > ~/index.log
```
截取部分日志内容如下
日志
```sh
Dumping /Users/admin/private/kafka/data/test-1-0/00000000000000203000.log
Starting offset: 203000
offset: 203000 position: 0 CreateTime: 1581597478310 isvalid: true keysize: -1 valuesize: 21 magic: 2 compresscodec: NONE producerId: -1 producerEpoch: -1 sequence: -1 isTransactional: false headerKeys: [] payload: local test -------- 0
offset: 203001 position: 0 CreateTime: 1581597478320 isvalid: true keysize: -1 valuesize: 21 magic: 2 compresscodec: NONE producerId: -1 producerEpoch: -1 sequence: -1 isTransactional: false headerKeys: [] payload: local test -------- 1
offset: 203002 position: 0 CreateTime: 1581597478321 isvalid: true keysize: -1 valuesize: 21 magic: 2 compresscodec: NONE producerId: -1 producerEpoch: -1 sequence: -1 isTransactional: false headerKeys: [] payload: local test -------- 2
offset: 203003 position: 0 CreateTime: 1581597478321 isvalid: true keysize: -1 valuesize: 21 magic: 2 compresscodec: NONE producerId: -1 producerEpoch: -1 sequence: -1 isTransactional: false headerKeys: [] payload: local test -------- 3
offset: 203004 position: 0 CreateTime: 1581597478321 isvalid: true keysize: -1 valuesize: 21 magic: 2 compresscodec: NONE producerId: -1 producerEpoch: -1 sequence: -1 isTransactional: false headerKeys: [] payload: local test -------- 4
```
索引
```sh
Dumping /Users/admin/private/kafka/data/test-1-0/00000000000000203000.index
offset: 203299 position: 8738
offset: 204657 position: 34953
offset: 205169 position: 51314
offset: 205681 position: 67695
offset: 206193 position: 84076
offset: 206705 position: 100457
offset: 207217 position: 116838
offset: 207729 position: 133219
offset: 208241 position: 149600
offset: 208753 position: 165981
offset: 209265 position: 182362
```

## 读取

在以上日志文件的基础上，我们来看看kafka是如何读取日志的。 假设我要们读取offset为203500的消息，查找过程如下

首先通过Segment的baseOffset确定在哪个Segment，只需要遍历segments对象，找到第一个baseOffset比206000小的Segment，也就是00000000000000203000这个Segment

这里不要狭隘的把Segment理解为.log日志文件，在新建LogSegment对象的时候，会创建.log, .index, .timeindex, .txn(事务)4个文件
```java
new LogSegment(
  FileRecords.open(Log.logFile(dir, baseOffset, fileSuffix), fileAlreadyExists, initFileSize, preallocate),
  new OffsetIndex(Log.offsetIndexFile(dir, baseOffset, fileSuffix), baseOffset = baseOffset, maxIndexSize = maxIndexSize),
  new TimeIndex(Log.timeIndexFile(dir, baseOffset, fileSuffix), baseOffset = baseOffset, maxIndexSize = maxIndexSize),
  new TransactionIndex(baseOffset, Log.transactionIndexFile(dir, baseOffset, fileSuffix)),
  baseOffset,
  indexIntervalBytes = config.indexInterval,
  rollJitterMs = config.randomSegmentJitter,
  maxSegmentMs = config.segmentMs,
  maxSegmentBytes = config.segmentSize,
  time)
}
```

查找Segment相关代码如下，segments类型为ConcurrentSkipListMap<baseOffset, Segment>，startOffset是消费时要拉取的起始offset
```java
var segmentEntry = segments.floorEntry(startOffset)
```

通过二分法，找出第一个比203500大的offset索引是[204657, 34953], 那么在.log文件中从物理位置34953开始查找，即可找到offset为203500的消息

### 时间索引

时间索引为.timeindex文件，它的原理是根据要查找的时间戳(targetTimestamp)，先找到相应的Segment，但是并没有一个Map保存了时间戳和Segment的映射关系，而Segment保存了当前分段中最大的时间戳(largestTimestamp)，所以需要遍历所有的Segment，找出第一个最大时间戳比targetTimestamp大的Segment
找到Segment后，通过查找.timeindex索引文件，查询先找到offset，然后再去.index文件找到相应的position，最后再去.log日志文件中查找

以上过程发生在Log类的fetchOffsetsByTimestamp方法，关键部分的代码如下

```java
def fetchOffsetsByTimestamp(targetTimestamp: Long): Option[TimestampOffset] = {
  val targetSeg = {
    // Get all the segments whose largest timestamp is smaller than target timestamp
    // 先找segments，找第一个Segment的最大Timestamp大于请求中的Timestamp，可以看下takeWhile源码
    val earlierSegs = segmentsCopy.takeWhile(_.largestTimestamp < targetTimestamp) //一直循环，只要不满足表示式停止
  }

  targetSeg.flatMap(_.findOffsetByTimestamp(targetTimestamp, logStartOffset))

}

def findOffsetByTimestamp(timestamp: Long, startingOffset: Long = baseOffset): Option[TimestampOffset] = {
  // Get the index entry with a timestamp less than or equal to the target timestamp
  val timestampOffset = timeIndex.lookup(timestamp)
  val position = offsetIndex.lookup(math.max(timestampOffset.offset, startingOffset)).position

  // Search the timestamp
  Option(log.searchForTimestamp(timestamp, position, startingOffset)).map { timestampAndOffset =>
    TimestampOffset(timestampAndOffset.timestamp, timestampAndOffset.offset)
  }
}
```

#### 二分查找过程

先说说3个参数，分别是：索引文件，要查找的目标值，查找类型，kafka将索引文件中的每一条数据抽象成一个entry，查找类型就是指按Key还是按Value查找

查找过程是一个十分简单的二分查找算法
```java
private def indexSlotRangeFor(idx: ByteBuffer, target: Long, searchEntity: IndexSearchEntity): (Int, Int) = {
    // check if the index is empty
    if(_entries == 0)
      return (-1, -1)

    // check if the target offset is smaller than the least offset
    if(compareIndexEntry(parseEntry(idx, 0), target, searchEntity) > 0)
      return (-1, 0)

    // binary search for the entry
    var lo = 0
    var hi = _entries - 1
    while(lo < hi) {
      val mid = ceil(hi/2.0 + lo/2.0).toInt
      val found = parseEntry(idx, mid)
      val compareResult = compareIndexEntry(found, target, searchEntity)
      if(compareResult > 0)
        hi = mid - 1
      else if(compareResult < 0)
        lo = mid
      else
        return (mid, mid)
    }

    (lo, if (lo == _entries - 1) -1 else lo + 1)
}
```














