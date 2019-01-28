---
title: Kafka生产者源码浅析(三)
date: 2019-01-28 15:27:50
categories: Kafka Tutorial
tags: [kafka,中间件,消息]
toc: true
comments: true
---

# Kafka生产者源码浅析(三)

## 回顾

在doSend方法中，最后几行代码是在消息添加进内存缓冲区之后，判断是否有可发送的消息，并唤醒了Sender线程

```java
if (result.batchIsFull || result.newBatchCreated) {
    log.trace("Waking up the sender since topic {} partition {} is either full or getting a new batch", record.topic(), partition);
    this.sender.wakeup();
}
```

那么sender线程又是如何发送的呢

猜想：

1. 先拿到缓冲区中待发送的所有消息，找到每个partitions leader所在的broker
2. 然后按broker的地址分组
3. 和broker建立连接，发送消息

## sender发送消息

Sender类实现了Runnable接口，那么主逻辑就应该在run方法中了，关于一些判断，事务先不用关系，来到Sender重载的run(long)方法，其中的两行关键代码

```java
void run(long now) {
    long pollTimeout = sendProducerData(now);
    client.poll(pollTimeout, now);
}
```

sendProducerData方法有点长，分段分析

### part one

1. 获取集群信息
2. 从RecordAccumulator获取可以发送消息的kafka broker节点
3. 如果有partitions还没有leader，请求更新metadata
4. 检查网络是否符合发送条件，不符合则移除

```java
Cluster cluster = metadata.fetch();

// get the list of partitions with data ready to send
RecordAccumulator.ReadyCheckResult result = this.accumulator.ready(cluster, now);

// if there are any partitions whose leaders are not known yet, force metadata update
if (!result.unknownLeaderTopics.isEmpty()) {
    // The set of topics with unknown leader contains topics with leader election pending as well as
    // topics which may have expired. Add the topic again to metadata to ensure it is included
    // and request metadata update, since there are messages to send to the topic.
    for (String topic : result.unknownLeaderTopics)
        this.metadata.add(topic);

    log.debug("Requesting metadata update due to unknown leader topics from the batched records: {}", result.unknownLeaderTopics);

    this.metadata.requestUpdate();
}
// remove any nodes we aren't ready to send to
Iterator<Node> iter = result.readyNodes.iterator();
long notReadyTimeout = Long.MAX_VALUE;
while (iter.hasNext()) {
    Node node = iter.next();
    if (!this.client.ready(node, now)) {
        iter.remove();
        notReadyTimeout = Math.min(notReadyTimeout, this.client.pollDelayMs(node, now));
    }
}
```