---
title: Kafka生产者源码浅析(三)
date: 2019-10-28 19:03:41
categories: Kafka Tutorial
tags: [kafka,中间件,消息]
toc: true
comments: true
---

# Kafka生产者源码浅析(三)

## 回顾

在doSend方法中，最后几行代码是在消息添加进内存缓冲区之后，判断是否有可发送的消息，并唤醒了Sender线程

那么sender线程又是如何发送的呢

猜想：

1. 先拿到缓冲区中待发送的所有消息，找到每个partition leader所在的broker
2. topic的分区可能很多，按照分区的leader所在的broker id分组
3. 和broker建立连接，发送消息

为什么要按broker分组？
假设broker集群有3台，一批消息要发送给5个分区，那么我们可以按照broker分区，一次请求就把所有在这台broker上的分区leader的消息发送完，这样请求按broker分组合并，提升了效率。

但是kafka有一个限定，同一个client对一个broker只能一个一个发请求，不能同时发送多个请求，这也是为了缓解broker端的压力
为了实现该方式，必然有个先进先出的请求队列，前一个请求拿到响应之后，才能出队，进行第二个请求

## sender发送消息

Sender类实现了Runnable接口，那么主逻辑就应该在run方法中了，关于一些判断，事务先不用管，来到Sender重载的run(long)方法，其中的两行关键代码

```java
void run(long now) {
    // 省略事务相关代码 ....
    long pollTimeout = sendProducerData(now);
    client.poll(pollTimeout, now);
}
```

## 消息分组并构建请求对象

sendProducerData方法用于构建PRODUCE请求对象，主要分以下三个步骤

1. 获取可发送的分区所在的broker节点集合
2. 按照broker id对消息集合分组
3. 构建请求对象

```java
private long sendProducerData(long now) {
    Cluster cluster = metadata.fetch();

    // get the list of partitions with data ready to send
    // 获取可发送的分区所在的broker节点集合
    RecordAccumulator.ReadyCheckResult result = this.accumulator.ready(cluster, now);

    // if there are any partitions whose leaders are not known yet, force metadata update
    if (!result.unknownLeaderTopics.isEmpty()) {
        // The set of topics with unknown leader contains topics with leader election pending as well as
        // topics which may have expired. Add the topic again to metadata to ensure it is included
        // and request metadata update, since there are messages to send to the topic.
        // 有分区leader未知的topic，等待下次元数据更新
        for (String topic : result.unknownLeaderTopics)
            this.metadata.add(topic);

        log.debug("Requesting metadata update due to unknown leader topics from the batched records: {}", result.unknownLeaderTopics);

        this.metadata.requestUpdate();
    }

    // remove any nodes we aren't ready to send to
    // 移除网络通信异常的node
    Iterator<Node> iter = result.readyNodes.iterator();
    long notReadyTimeout = Long.MAX_VALUE;
    while (iter.hasNext()) {
        Node node = iter.next();
        if (!this.client.ready(node, now)) {
            iter.remove();
            notReadyTimeout = Math.min(notReadyTimeout, this.client.pollDelayMs(node, now));
        }
    }

    // create produce requests
    // 按照broker id分组
    Map<Integer, List<ProducerBatch>> batches = this.accumulator.drain(cluster, result.readyNodes,
            this.maxRequestSize, now);
    if (guaranteeMessageOrder) {
        // Mute all the partitions drained
        for (List<ProducerBatch> batchList : batches.values()) {
            for (ProducerBatch batch : batchList)
                this.accumulator.mutePartition(batch.topicPartition);
        }
    }

    // 省略事务，监控相关
    sendProduceRequests(batches, now);

    return pollTimeout;
}
```

### 构建请求对象

可以看到最终发送给broker的是一个Map<TopicPartition, MemoryRecords>对象

```java
private void sendProduceRequest(long now, int destination, short acks, int timeout, List<ProducerBatch> batches) {
    if (batches.isEmpty())
        return;

    Map<TopicPartition, MemoryRecords> produceRecordsByPartition = new HashMap<>(batches.size());
    final Map<TopicPartition, ProducerBatch> recordsByPartition = new HashMap<>(batches.size());

    // find the minimum magic version used when creating the record sets
    byte minUsedMagic = apiVersions.maxUsableProduceMagic();
    for (ProducerBatch batch : batches) {
        if (batch.magic() < minUsedMagic)
            minUsedMagic = batch.magic();
    }

    for (ProducerBatch batch : batches) {
        TopicPartition tp = batch.topicPartition;
        MemoryRecords records = batch.records();

        produceRecordsByPartition.put(tp, records);
        recordsByPartition.put(tp, batch);
    }

    ProduceRequest.Builder requestBuilder = ProduceRequest.Builder.forMagic(minUsedMagic, acks, timeout,
            produceRecordsByPartition, transactionalId);
    RequestCompletionHandler callback = new RequestCompletionHandler() {
        public void onComplete(ClientResponse response) {
            handleProduceResponse(response, recordsByPartition, time.milliseconds());
        }
    };

    String nodeId = Integer.toString(destination);
    ClientRequest clientRequest = client.newClientRequest(nodeId, requestBuilder, now, acks != 0,
            requestTimeoutMs, callback);
    client.send(clientRequest, now);
}   
```

# 发送流程

最后总结下生产者发送流程
![](https://ae01.alicdn.com/kf/H85cc7061b8514ba2bcb36773714a49276.png)


















