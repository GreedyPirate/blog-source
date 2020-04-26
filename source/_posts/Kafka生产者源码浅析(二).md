---
title: Kafka生产者源码浅析(二)
date: 2019-10-16 20:13:39
categories: Kafka Tutorial
tags: [kafka,中间件,消息]
toc: true
comments: true
---


# Kafka生产者源码浅析(二)

> 上篇文章中对Spring-kafka源码做了追踪，也对原生的KafkaProducer做了部分解析，对关键类事先说明，帮助读者理解源码，克服对源码的恐惧心理

doSend的方法很长，我们分部拆解
```java
// 省略部分代码，catch处理
private Future<RecordMetadata> doSend(ProducerRecord<K, V> record, Callback callback) {
    TopicPartition tp = null;
    try {
        throwIfProducerClosed();
        // first make sure the metadata for the topic is available
        ClusterAndWaitTime clusterAndWaitTime;
        clusterAndWaitTime = waitOnMetadata(record.topic(), record.partition(), maxBlockTimeMs);
        long remainingWaitMs = Math.max(0, maxBlockTimeMs - clusterAndWaitTime.waitedOnMetadataMs);
        Cluster cluster = clusterAndWaitTime.cluster;

        byte[] serializedKey;
        serializedKey = keySerializer.serialize(record.topic(), record.headers(), record.key());
    
        byte[] serializedValue;
        serializedValue = valueSerializer.serialize(record.topic(), record.headers(), record.value());
    
        int partition = partition(record, serializedKey, serializedValue, cluster);
        tp = new TopicPartition(record.topic(), partition);

        setReadOnly(record.headers());
        Header[] headers = record.headers().toArray();

        int serializedSize = AbstractRecords.estimateSizeInBytesUpperBound(apiVersions.maxUsableProduceMagic(),
                compressionType, serializedKey, serializedValue, headers);
        ensureValidRecordSize(serializedSize);
        long timestamp = record.timestamp() == null ? time.milliseconds() : record.timestamp();
        Callback interceptCallback = new InterceptorCallback<>(callback, this.interceptors, tp);

        if (transactionManager != null && transactionManager.isTransactional())
            transactionManager.maybeAddPartitionToTransaction(tp);

        RecordAccumulator.RecordAppendResult result = accumulator.append(tp, timestamp, serializedKey,
                serializedValue, headers, interceptCallback, remainingWaitMs);
        if (result.batchIsFull || result.newBatchCreated) {
            this.sender.wakeup();
        }
        return result.future;
    }
}
```

## Part one

```java
TopicPartition tp = null;
try {
    throwIfProducerClosed();
    // first make sure the metadata for the topic is available
    ClusterAndWaitTime clusterAndWaitTime;
    clusterAndWaitTime = waitOnMetadata(record.topic(), record.partition(), maxBlockTimeMs);
    long remainingWaitMs = Math.max(0, maxBlockTimeMs - clusterAndWaitTime.waitedOnMetadataMs);
    Cluster cluster = clusterAndWaitTime.cluster;
}
```
1. throwIfProducerClosed做的很简单，看看Sender线程是否活着

2. waitOnMetadata返回一个ClusterAndWaitTime对象，里面是broker集群的元信息和获取信息的耗时，这个耗时算在了max.block.ms中，它控制这send方法的最大执行时间

3. waitOnMetadata通过唤醒sender线程，依靠NetworkClient.poll()方法来更新元数据

Cluster 类信息如下
```java
public final class Cluster {
    private final boolean isBootstrapConfigured;
    // 所有的broker节点
    private final List<Node> nodes;
    private final Set<String> unauthorizedTopics;
    // 内部topic，如_consumer_offset
    private final Set<String> internalTopics;
    // controller节点
    private final Node controller;
    // 每个分区对应的分区信息
    private final Map<TopicPartition, PartitionInfo> partitionsByTopicPartition;
    // 每个topic所有分区的信息
    private final Map<String, List<PartitionInfo>> partitionsByTopic;
    // topic所有可用分区的信息
    private final Map<String, List<PartitionInfo>> availablePartitionsByTopic;
    // 每个broker节点的所有分区
    private final Map<Integer, List<PartitionInfo>> partitionsByNode;
    // 按照nodeId组成map
    private final Map<Integer, Node> nodesById;
    // 里面只有一个clusterId熟悉
    private final ClusterResource clusterResource;
}
```
具体是如何初始化的，可以看一下Cluster构造函数的源码

上面出现的PartitionInfo, 这些信息想必大家已经很熟悉
```java
public class PartitionInfo {
    // 主题
    private final String topic;
    // 分区
    private final int partition;
    // leader分区所在broker
    private final Node leader;
    // 副本所在broker
    private final Node[] replicas;
    // ISR副本所在broker
    private final Node[] inSyncReplicas;
    // 离线副本所在broker
    private final Node[] offlineReplicas;
}
```

## Part two

这一部分比较简单，对key和value序列化

```java
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

partition方法会尝试获取消息中的partition，如果用户指定了分区，此时就不用计算了，否则使用partitioner计算分区

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

回顾前文，Cluster封装了broker的很多信息，其中就用一个Map封装了topic的partition信息

```java
Map<String, List<PartitionInfo>> partitionsByTopic
```

此时要分区，首先要获取这个topic的PartitionInfo，第一行代码的作用就是这个，map.get(topic)，很简单

接下分两种情况：用户指定了key，和未指定key，我们知道旧版本的kafka在用户未指定key的情况下会默认将消息分配到某一个分区，
但这样会造成数据倾斜，官方后来对此作了优化，采用轮询(round-robin)的方式，简单提一下这块的代码

#### 随机分配

kafka会初始化一个很大的伪随机数放在AtomicInteger中：
```java
private int nextValue(String topic) {
    AtomicInteger counter = topicCounterMap.get(topic);
    if (null == counter) {
        counter = new AtomicInteger(ThreadLocalRandom.current().nextInt());
        AtomicInteger currentCounter = topicCounterMap.putIfAbsent(topic, counter);
        if (currentCounter != null) {
            counter = currentCounter;
        }
    }
    return counter.getAndIncrement();
}
```
以topic为key保存在一个ConcurrentHashMap中，每次用完counter自增并返回，这就是nextValue方法的作用

接下来从Cluster中获取可用的分区信息，获取分区数，使用counter对其取模，然后从可用分区列表中获取一个分区，由于counter的自增，达到了轮询(round-robin)的效果。但如果没有可用的分区，则从所有分区中挑选(有种破罐子破摔的味道)

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

首先思考下缓存区的数据结构是什么：每一个分区都有一个先进先出的队列，，而kafka真正使用的是一个双端队列

RecordAccumulator为topic的每一个分区都创建了一个ArrayDeque(thread unsafe)，里面存放的元素是ProducerBatch，它就是待批量发送的消息。
kafka使用一个CopyOnWriteMap保存分区和队列的关系，即只有在修改该map时把内容Copy出去形成一个新的map，然后配合volatile改变引用，这也是COW机制的常见用法

```java
ConcurrentMap<TopicPartition, Deque<ProducerBatch>> batches = new CopyOnWriteMap<>();
```
该map的模型如下
![模型](https://ae01.alicdn.com/kf/H409e050f5b184f7ebad5ecc5f12d9e41V.png)

append方法返回一个RecordAppendResult，它是消息在添加进内存缓冲区后的结果：Deque队列中是否有元素，是否有新的ProducerBatch创建，两个条件都可以去通知sender线程发送消息

这里的代码看似很多，其实并不难，我们还是逐步分析下

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

        // batch.size默认是16KB，但是即使超出也没事，通过Math.max函数取了二者最大值
        // we don't have an in-progress record batch try to allocate a new batch
        byte maxUsableMagic = apiVersions.maxUsableProduceMagic();
        int size = Math.max(this.batchSize, AbstractRecords.estimateSizeInBytesUpperBound(maxUsableMagic, compression, key, value, headers));

        // 这里在申请ByteBuffer缓存空间了
        buffer = free.allocate(size, maxTimeToBlock);
        synchronized (dq) {
            // Need to check if producer is closed again after grabbing the dequeue lock.
            // 我暂时没看懂意图，大概意思是在极端情况下，检查线程在获取到dequeue锁之后，producer又关闭
            if (closed)
                throw new KafkaException("Producer closed while send in progress");

            RecordAppendResult appendResult = tryAppend(timestamp, key, value, headers, callback, dq);
            if (appendResult != null) {
                // 万一这个时候又有了可用的ProducerBatch呢，我们就不用新建了呀，唉~这就很舒服
                // Somebody else found us a batch, return the one we waited for! Hopefully this doesn't happen often...
                return appendResult;
            }

            // 初始化MemoryRecordsBuilder，初始化DataOutputStream appendStream关键对象
            MemoryRecordsBuilder recordsBuilder = recordsBuilder(buffer, maxUsableMagic);
            // 新建一个ProducerBatch
            ProducerBatch batch = new ProducerBatch(tp, recordsBuilder, time.milliseconds());
            // 写入消息到appendStream关键对象
            FutureRecordMetadata future = Utils.notNull(batch.tryAppend(timestamp, key, value, headers, callback, time.milliseconds()));

            // 然后添加到队列尾部
            dq.addLast(batch);
            // incomplete对象就是个Set<ProducerBatch>, 用来保存还没有发送完成的，包括还没发送的
            incomplete.add(batch);

            // Don't deallocate this buffer in the finally block as it's being used in the record batch
            // 释放buffer资源，因为是HeapByteBuffer，等待GC回收即可
            buffer = null;

            // 返回结果
            return new RecordAppendResult(future, dq.size() > 1 || batch.isFull(), true);
        }
    } finally {
        if (buffer != null)
            free.deallocate(buffer);
        appendsInProgress.decrementAndGet();
    }
}
```
下面截取部分调用的代码进行讲解

#### 创建队列

```java
/**
 * Get the deque for the given topic-partition, creating it if necessary.
 */
private Deque<ProducerBatch> getOrCreateDeque(TopicPartition tp) {
    Deque<ProducerBatch> d = this.batches.get(tp);
    if (d != null)
        return d;
    d = new ArrayDeque<>();
    Deque<ProducerBatch> previous = this.batches.putIfAbsent(tp, d);
    if (previous == null)
        return d;
    else
        return previous;
}
```
从`ConcurrentMap<TopicPartition, Deque<ProducerBatch>> batches`中获取该主题分区对应的队列,如果不为空说明已经有了，直接返回，否者创建一个新的ArrayDeque，并放到map中，方便下次使用，
至于putIfAbsent方法，就是map中之前没有这个key，插入并返回新value，已经有了，就返回之前的value，即Deque


#### RecordAccumulator#tryAppend方法源码：

其实文档写的很清楚了，就是把消息追加到最后一个ProducerBatch中，但要是队列中一个都没有呢？ 很简单，直接返回null，在外层方法中会判断不为null在结束，否则会分配
吐槽下：一开始就看岔了，好几个tryAppend，如果是我，我会写成tryAppendInternal之类的方法名
RecordAppendResult构造方法的最后一个参数表示是否是新建的ProducerBatch，这里返回时也确实返回了false
```java
/**
 *  Try to append to a ProducerBatch.
 *
 *  If it is full, we return null and a new batch is created. We also close the batch for record appends to free up
 *  resources like compression buffers. The batch will be fully closed (ie. the record batch headers will be written
 *  and memory records built) in one of the following cases (whichever comes first): right before send,
 *  if it is expired, or when the producer is closed.
 */
private RecordAppendResult tryAppend(long timestamp, byte[] key, byte[] value, Header[] headers,
                                     Callback callback, Deque<ProducerBatch> deque) {
    ProducerBatch last = deque.peekLast();
    if (last != null) {
        FutureRecordMetadata future = last.tryAppend(timestamp, key, value, headers, callback, time.milliseconds());
        if (future == null)
            last.closeForRecordAppends();
        else
            return new RecordAppendResult(future, deque.size() > 1 || last.isFull(), false);
    }
    return null;
}
```

### ProducerBatch#tryAppend

注：一定注意和上面的同名方法的区分

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

#### 具体写入过程

recordsBuilder.append的主要实现过程如下：
首先key，value都会包装成ByteBuffer，写入时都是先写长度，再写内容
key，value，headers都写入DataOutputStream appendStream流对象中，返回写入的长度

```java
/**
 * Write the record to `out` and return its size.
 */
public static int writeTo(DataOutputStream out,
                          int offsetDelta,
                          long timestampDelta,
                          ByteBuffer key,
                          ByteBuffer value,
                          Header[] headers) throws IOException {
    int sizeInBytes = sizeOfBodyInBytes(offsetDelta, timestampDelta, key, value, headers);
    ByteUtils.writeVarint(sizeInBytes, out);

    byte attributes = 0; // there are no used record attributes at the moment
    out.write(attributes);

    ByteUtils.writeVarlong(timestampDelta, out);
    ByteUtils.writeVarint(offsetDelta, out);

    if (key == null) {
        ByteUtils.writeVarint(-1, out);
    } else {
        int keySize = key.remaining();
        ByteUtils.writeVarint(keySize, out);
        Utils.writeTo(out, key, keySize);
    }

    if (value == null) {
        ByteUtils.writeVarint(-1, out);
    } else {
        int valueSize = value.remaining();
        ByteUtils.writeVarint(valueSize, out);
        Utils.writeTo(out, value, valueSize);
    }

    if (headers == null)
        throw new IllegalArgumentException("Headers cannot be null");

    ByteUtils.writeVarint(headers.length, out);

    for (Header header : headers) {
        String headerKey = header.key();
        if (headerKey == null)
            throw new IllegalArgumentException("Invalid null header key found in headers");

        byte[] utf8Bytes = Utils.utf8(headerKey);
        ByteUtils.writeVarint(utf8Bytes.length, out);
        out.write(utf8Bytes);

        byte[] headerValue = header.value();
        if (headerValue == null) {
            ByteUtils.writeVarint(-1, out);
        } else {
            ByteUtils.writeVarint(headerValue.length, out);
            out.write(headerValue);
        }
    }

    return ByteUtils.sizeOfVarint(sizeInBytes) + sizeInBytes;
}
```


### append方法小结

append方法的具体实现过程还是很复杂的，这里说下笔者对这个过程的理解：

1. 尝试获取该TopicPartition下的队列，如果没有则创建
2. 获取队列的最后一个ProducerBatch元素，将消息添加至该ProducerBatch，该过程会对Deque加锁
3. 如果队列里没有ProducerBatch，或是最后一个ProducerBatch已经满了，就需要新建一个ProducerBatch
4. 分配一个ByteBuffer空间，该空间大小在batch.size和消息大小中取较大值
5. 再重新尝试步骤2一次，万一这时候刚好又有了呢(这时候Deque已经释放锁了)
6. 创建好ProducerBatch之后，继续尝试append，添加成功之后将future和callback放入一个Thunk对象中，并且添加到一个List<Thunk>集合，这是因为一批消息需要发送之后才有回调，所以先把回调统一放入一个集合中
7. 添加成功之后，返回future对象，将ProducerBatch添加至Deque队列，同时用一个集合IncompleteBatches持有住了ProducerBatch
8. 清理buffer空间，封装RecordAppendResult结果：Deque队列大小，新建的ProducerBatch对象是否已满


## 总结

kafka发送消息的步骤大致如下：

1. 获取broker上的元信息
2. key, value的序列化
3. 计算分区
4. 添加到缓存区
    1. 获取该分区对应的队列
    2. 尝试添加
    3. 如果添加成功返回












