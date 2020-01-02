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
batchIsFull的判断依据有两个：deque.size() > 1 || last.isFull()，意思是队列中至少有一个ProducerBatch已满，或者原来一个满的都没有，添加完这条消息就满了，两个条件满足其一就说明可以发送了
newBatchCreated表示新创建了一个ProducerBatch就发送，这里确实以前不知道
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

这样请求按broker分组合并，提升了效率，但是kafka有一个限定，同一个client对一个broker只能一个一个发请求，不能同时发送多个请求，这也是为了缓解broker端的压力
为了实现该方式，必然有个先进先出的请求队列，前一个请求拿到响应之后，才能出队，进行第二个请求

## sender发送消息

Sender类实现了Runnable接口，那么主逻辑就应该在run方法中了，关于一些判断，事务先不用管，来到Sender重载的run(long)方法，其中的两行关键代码

```java
/**
 * The main run loop for the sender thread
 */
public void run() {
    log.debug("Starting Kafka producer I/O thread.");

    // main loop, runs until close is called
    while (running) {
        try {
            run(time.milliseconds());
        } catch (Exception e) {
            log.error("Uncaught error in kafka producer I/O thread: ", e);
        }
    }
    // 省略关闭后的处理代码
}

// 进入到run方法
void run(long now) {
    // 省略事务相关代码 ....
    long pollTimeout = sendProducerData(now);
    client.poll(pollTimeout, now);
}
```

sendProducerData方法主要分以下三个步骤

### 步骤1
```java
RecordAccumulator.ReadyCheckResult result = this.accumulator.ready(cluster, now);
```
这行代码内部实现虽然有点复杂，但就一个作用，获取所有要发送分区的leader副本所在节点
然后就是对结果的处理及过滤, 不知道leader副本的topic交给metadata去更新，然后又根据NetworkClient过滤连接异常的节点
```java
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

### 步骤2
主要根据缓冲区对象的drain方法，把所有分区消息，按照Node id分组






















