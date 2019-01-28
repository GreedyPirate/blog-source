---
title: Kafka生产者源码浅析(二)
date: 2019-01-28 15:27:30
categories: Kafka Tutorial
tags: [kafka,中间件,消息]
toc: true
comments: true
---


# Kafka生产者源码浅析(二)

> 上篇文章中对Spring-kafka源码做了追踪，也对原生的KafkaProducer做了部门解析，对关键类事先说明，帮助读者理解源码，克服对源码的恐惧心理，一起品尝Kafka的饕餮盛宴

doSend的方法很长，我们分部拆解

## Part one

首先吐槽下这个tp变量，定义在外边没什么卵用

```java
TopicPartition tp = null;
try {
    throwIfProducerClosed();
    // first make sure the metadata for the topic is available
    ClusterAndWaitTime clusterAndWaitTime;
    try {
        clusterAndWaitTime = waitOnMetadata(record.topic(), record.partition(), maxBlockTimeMs);
    } catch (KafkaException e) {
        if (metadata.isClosed())
            throw new KafkaException("Producer closed while send in progress", e);
        throw e;
    }
}
```

1. throwIfProducerClosed做的很简单，看看Sender线程是否活着

2. waitOnMetadata返回一个ClusterAndWaitTime对象，里面是broker集群的元信息和获取信息的耗时，它不能大于max.block.ms，对该方法做个简单的分析：

   ```java
   metadata.add(topic);
   Cluster cluster = metadata.fetch();
   Integer partitionsCount = cluster.partitionCountForTopic(topic);
   if (partitionsCount != null && (partition == null || partition < partitionsCount))
       return new ClusterAndWaitTime(cluster, 0);
   ```

   将topic放入一个map中，value是topic的过期时间(如果开启topic过期功能)，从Cluster中获取topic的分区数，如果消息手动指定了分区，且小于分区数，就直接返回(说明手动指定的分区是有效的)

## Part two

这一部分比较简单，计算了下maxBlockTimeMs还剩多少时间，同时对key和value序列化

```java
long remainingWaitMs = Math.max(0, maxBlockTimeMs - clusterAndWaitTime.waitedOnMetadataMs);
Cluster cluster = clusterAndWaitTime.cluster;
byte[] serializedKey;
try {
    serializedKey = keySerializer.serialize(record.topic(), record.headers(), record.key());
} catch (ClassCastException cce) {
   // 省略...
}
byte[] serializedValue;
try {
    serializedValue = valueSerializer.serialize(record.topic(), record.headers(), record.value());
} catch (ClassCastException cce) {
    // 省略...
}
```

### 分区

接下来关注kafka是如何根据key为消息计算分区的

```java
int partition = partition(record, serializedKey, serializedValue, cluster);
tp = new TopicPartition(record.topic(), partition);
```

第二行代码是将topic和分区包装成一个TopicPartition类，重点关注第一行代码

partition方法会尝试获取消息中的partition，如果用户指定了分区，此时就不用计算了

```java
    private int partition(ProducerRecord<K, V> record, byte[] serializedKey, byte[] serializedValue, Cluster cluster) {
        Integer partition = record.partition();
        return partition != null ?
                partition :
                partitioner.partition(
                        record.topic(), record.key(), serializedKey, record.value(), serializedValue, cluster);
    }
```

#### DefaultPartitioner

partitioner.partition的具体实现在DefaultPartitioner#partition，其源码如下：

```java
public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
        int numPartitions = partitions.size();
        if (keyBytes == null) {
            int nextValue = nextValue(topic);
            List<PartitionInfo> availablePartitions = cluster.availablePartitionsForTopic(topic);
            if (availablePartitions.size() > 0) {
                int part = Utils.toPositive(nextValue) % availablePartitions.size();
                return availablePartitions.get(part).partition();
            } else {
                // no partitions are available, give a non-available partition
                return Utils.toPositive(nextValue) % numPartitions;
            }
        } else {
            // hash the keyBytes to choose a partition
            return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
        }
    }
```

回顾上一篇文章，Cluster封装了broker的很多信息，其中就用一个Map封装了topic的partition信息

```java
Map<String, List<PartitionInfo>> partitionsByTopic
```

此时要分区，首先要获取这个topic的PartitionInfo，第一行代码的作用就是这个，map.get(topic)，很简单

接下分两种情况：用户指定了key，和未指定key，我们知道旧版本的kafka在用户未指定key的情况下会默认将消息分配到某一个分区，但这样会造成数据倾斜，官方后来对此作了优化，采用随机的方式，简单提一下这块的代码

#### 随机分配

kafka会初始化一个很大的伪随机数放在AtomicInteger中：
```java
counter = new AtomicInteger(ThreadLocalRandom.current().nextInt())
```
以topic为key保存在一个ConcurrentHashMap中，每次用完counter自增并返回，这就是nextValue方法的作用

接下来从Cluster中获取可用的分区信息，获取分区数，使用counter对其取模，然后从可用分区列表中获取一个分区，由于counter的自增，达到了轮询(round-robin)的效果。但如果没有可用的分区，则从所有分区中挑选

Utils.toPositive用于取绝对值，kafka选择了一个cheap way: 与运算

以上是对消息中没有key的情况下如何分配分区的分析，至于有key的情况就比较简单了：对key做[murmur2 hash](https://sites.google.com/site/murmurhash/)运算，然后对分区数取模

### 自定义分区策略
实现Partitioner接口即可，配置方式参考拦截器，二者同理，参数名称为: partitioner.class

## Part three

```java
setReadOnly(record.headers());
Header[] headers = record.headers().toArray();

int serializedSize = 	AbstractRecords.estimateSizeInBytesUpperBound(apiVersions.maxUsableProduceMagic(),
                    compressionType, serializedKey, serializedValue, headers);
ensureValidRecordSize(serializedSize);
long timestamp = record.timestamp() == null ? time.milliseconds() : record.timestamp();
log.trace("Sending record {} with callback {} to topic {} partition {}", record, callback, record.topic(), partition);
Callback interceptCallback = new InterceptorCallback<>(callback, this.interceptors, tp);
```

先把不重要的说了，这几行代码的可读性很好，设置消息头只读，然后**估算**消息的总大小，确保不会超出max.request.size和buffer.memory的大小，获取消息的时间戳，用户指定的优先，最后构建一个InterceptorCallback回调对象，它会先指定拦截器的onAcknowledgement回调，然后执行用户指定的Callback#onCompletion

### 追加至缓存并发送

```java
RecordAccumulator.RecordAppendResult result = accumulator.append(tp, timestamp, serializedKey,serializedValue, headers, interceptCallback, remainingWaitMs);
if (result.batchIsFull || result.newBatchCreated) {
    log.trace("Waking up the sender since topic {} partition {} is either full or getting a new batch", record.topic(), partition);
    this.sender.wakeup();
}
return result.future;
```

首先思考下缓存区的数据结构是什么：它应该有个先来后到的顺序，即先进先出(FIFO)，用一个队列实现即可

而kafka真正使用的是一个双端队列，基于"工作密取"模式减少队列竞争，提高效率

RecordAccumulator为topic的每一个分区都创建了一个ArrayDeque(thread unsafe)，里面存放的元素是ProducerBatch，它就是待批量发送的消息。
kafka使用一个CopyOnWriteMap保存分区和队列的关系，即只有在修改该map时把内容Copy出去形成一个新的内容然后再改

```java
ConcurrentMap<TopicPartition, Deque<ProducerBatch>> batches = new CopyOnWriteMap<>();
```
该map的模型如下
![模型](https://ws4.sinaimg.cn/large/006tNc79ly1fzmdhav2ifj311q0gsgmu.jpg)

append方法返回一个RecordAppendResult，它是消息在添加进内存缓冲区后的结果：Deque队列中是否有元素，是否有新的ProducerBatch创建，两个条件都可以去通知sender线程发送消息

append方法的具体实现过程还是很复杂的，这里说下笔者对这个过程的理解：

1. 尝试获取该TopicPartition下的队列，如果没有则创建
2. 获取队列的最后一个ProducerBatch元素，将消息添加至该ProducerBatch，该过程会对Deque加锁
3. 如果队列里没有ProducerBatch，或是最后一个ProducerBatch已经满了，就需要新建一个ProducerBatch
4. 分配一个ByteBuffer空间，该空间大小在batch.size和消息大小中取较大值
5. 再重新尝试步骤2一次，万一这时候刚好又有了呢(这时候Deque已经释放锁了)
6. 创建好ProducerBatch之后，继续尝试append，添加成功之后将future和callback放入一个Thunk对象中，并且添加到一个List<Thunk>集合，这是因为一批消息需要发送之后才有回调，所以先把回调统一放入一个集合中
7. 添加成功之后，返回future对象，将ProducerBatch添加至Deque队列，同时用一个集合IncompleteBatches持有住了ProducerBatch
8. 清理buffer空间，封装RecordAppendResult结果：Deque队列大小，新建的ProducerBatch对象是否已满

完整的源码如下，其中的调用栈还可以深入，有兴趣的同学可以顺着我的思路继续看

```java
    public RecordAppendResult append(TopicPartition tp,
                                     long timestamp,
                                     byte[] key,
                                     byte[] value,
                                     Header[] headers,
                                     Callback callback,
                                     long maxTimeToBlock) throws InterruptedException {
        // We keep track of the number of appending thread to make sure we do not miss batches in
        // abortIncompleteBatches().
        appendsInProgress.incrementAndGet();
        ByteBuffer buffer = null;
        if (headers == null) headers = Record.EMPTY_HEADERS;
        try {
            // check if we have an in-progress batch
            Deque<ProducerBatch> dq = getOrCreateDeque(tp);
            synchronized (dq) {
                if (closed)
                    throw new KafkaException("Producer closed while send in progress");
                RecordAppendResult appendResult = tryAppend(timestamp, key, value, headers, callback, dq);
                if (appendResult != null)
                    return appendResult;
            }

            // we don't have an in-progress record batch try to allocate a new batch
            byte maxUsableMagic = apiVersions.maxUsableProduceMagic();
            int size = Math.max(this.batchSize, AbstractRecords.estimateSizeInBytesUpperBound(maxUsableMagic, compression, key, value, headers));
            log.trace("Allocating a new {} byte message buffer for topic {} partition {}", size, tp.topic(), tp.partition());
            buffer = free.allocate(size, maxTimeToBlock);
            synchronized (dq) {
                // Need to check if producer is closed again after grabbing the dequeue lock.
                if (closed)
                    throw new KafkaException("Producer closed while send in progress");

                RecordAppendResult appendResult = tryAppend(timestamp, key, value, headers, callback, dq);
                if (appendResult != null) {
                    // 万一这个时候又有了可用的ProducerBatch呢，我们就不用新建了呀，唉~这就很舒服
                    // Somebody else found us a batch, return the one we waited for! Hopefully this doesn't happen often...
                    return appendResult;
                }

                MemoryRecordsBuilder recordsBuilder = recordsBuilder(buffer, maxUsableMagic);
                ProducerBatch batch = new ProducerBatch(tp, recordsBuilder, time.milliseconds());
                FutureRecordMetadata future = Utils.notNull(batch.tryAppend(timestamp, key, value, headers, callback, time.milliseconds()));

                dq.addLast(batch);
                incomplete.add(batch);

                // Don't deallocate this buffer in the finally block as it's being used in the record batch
                buffer = null;

                return new RecordAppendResult(future, dq.size() > 1 || batch.isFull(), true);
            }
        } finally {
            if (buffer != null)
                free.deallocate(buffer);
            appendsInProgress.decrementAndGet();
        }
    }
```

ProducerBatch#tryAppend方法源码：

```java
    public FutureRecordMetadata tryAppend(long timestamp, byte[] key, byte[] value, Header[] headers, Callback callback, long now) {
        if (!recordsBuilder.hasRoomFor(timestamp, key, value, headers)) {
            return null;
        } else {
            Long checksum = this.recordsBuilder.append(timestamp, key, value, headers);
            this.maxRecordSize = Math.max(this.maxRecordSize, AbstractRecords.estimateSizeInBytesUpperBound(magic(),
                    recordsBuilder.compressionType(), key, value, headers));
            this.lastAppendTime = now;
            FutureRecordMetadata future = new FutureRecordMetadata(this.produceFuture, this.recordCount,
                                                                   timestamp, checksum,
                                                                   key == null ? -1 : key.length,
                                                                   value == null ? -1 : value.length);
            // we have to keep every future returned to the users in case the batch needs to be
            // split to several new batches and resent.
            thunks.add(new Thunk(callback, future));
            this.recordCount++;
            return future;
        }
    }

```













