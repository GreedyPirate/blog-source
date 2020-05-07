---
title: kafka server端源码分析之副本同步
date: 2020-03-08 17:26:44
categories: Kafka Tutorial
tags: [kafka,中间件,消息]
toc: true
comments: true
---

# 前言

为什么我现在才写副本同步的解析呢，因为它太复杂了，仅仅是什么时候触发的副本同步，就涉及到KafkaController，LeaderAndIsr请求等，经过前面文章的梳理，现在时机正好

# 正文

通常我们会为了提高系统并发能力、可伸缩性，为topic设置多个分区，每个分区副本数通常设置为3个，其中1个为leader副本，其余2个follower副本为冗余备份使用。在producer端为了保证消息不丢失，通常设置ack=-1，并搭配失败重试机制

本文主要讨论broker端写入leader副本后，follower副本如何同步消息，以及如何更新HighWatermark，并使Purgatory延迟队列中的PRODUCE请求完成(complete)，响应客户端

## 副本拉取管理器
在启动Kafka时，会初始化ReplicaManager副本管理器，同时该类中有一行初始化语句
```java
val replicaFetcherManager = createReplicaFetcherManager(metrics, time, threadNamePrefix, quotaManagers.follower)
```
其实就是new了一个ReplicaFetcherManager对象，该对象的功能十分简单，就是创建和关闭Fetch线程

```java
class ReplicaFetcherManager(brokerConfig: KafkaConfig, protected val replicaManager: ReplicaManager, metrics: Metrics,
                            time: Time, threadNamePrefix: Option[String] = None, quotaManager: ReplicationQuotaManager)
      extends AbstractFetcherManager("ReplicaFetcherManager on broker " + brokerConfig.brokerId,
        "Replica", brokerConfig.numReplicaFetchers) {

  override def createFetcherThread(fetcherId: Int, sourceBroker: BrokerEndPoint): AbstractFetcherThread = {
    val prefix = threadNamePrefix.map(tp => s"${tp}:").getOrElse("")
    val threadName = s"${prefix}ReplicaFetcherThread-$fetcherId-${sourceBroker.id}"
    new ReplicaFetcherThread(threadName, fetcherId, sourceBroker, brokerConfig, replicaManager, metrics, time, quotaManager)
  }

  def shutdown() {
    info("shutting down")
    closeAllFetchers()
    info("shutdown completed")
  }
}
```

# 副本同步

在分析副本同步过程之前，我们先想一想什么时候开始同步，也就是上面的createFetcherThread什么时候创建并启动的

## 何时同步

这里就要回顾[KafkaController源码分析之LeaderAndIsr请求]()一文了，这也是我先写LeaderAndIsr，然后才分析副本同步的原因

在前文中，提到了becomeLeaderOrFollower方法会将分区添加到副本同步线程中，具体实现就在addFetcherForPartitions方法中

```java
def addFetcherForPartitions(partitionAndOffsets: Map[TopicPartition, BrokerAndInitialOffset]) {
    lock synchronized {
      // partitionsPerFetcher = Map[BrokerAndFetcherId, Map[TopicPartition, BrokerAndInitialOffset]]
      // 分组的key是目标broker+同步线程，也就是同一个fetcher线程向同一个broker同步 为一组
      val partitionsPerFetcher = partitionAndOffsets.groupBy { case(topicPartition, brokerAndInitialFetchOffset) =>
        BrokerAndFetcherId(brokerAndInitialFetchOffset.broker, getFetcherId(topicPartition.topic, topicPartition.partition))}

      // 事先定义好创建并启动的方法
      def addAndStartFetcherThread(brokerAndFetcherId: BrokerAndFetcherId, brokerIdAndFetcherId: BrokerIdAndFetcherId) {
        val fetcherThread = createFetcherThread(brokerAndFetcherId.fetcherId, brokerAndFetcherId.broker)
        fetcherThreadMap.put(brokerIdAndFetcherId, fetcherThread)
        fetcherThread.start
      }

      for ((brokerAndFetcherId, initialFetchOffsets) <- partitionsPerFetcher) {
        val brokerIdAndFetcherId = BrokerIdAndFetcherId(brokerAndFetcherId.broker.id, brokerAndFetcherId.fetcherId)
        // fetcherThreadMap: Map[BrokerIdAndFetcherId, AbstractFetcherThread]
        // 这里的逻辑还是很清晰的
        fetcherThreadMap.get(brokerIdAndFetcherId) match {
            // 已存在对应的Thread，并且线程的broker和分区要同步的broker相同，直接复用就行了
          case Some(f) if f.sourceBroker.host == brokerAndFetcherId.broker.host && f.sourceBroker.port == brokerAndFetcherId.broker.port =>
            // reuse the fetcher thread
          case Some(f) =>
            // 如果前面的if不成立，就需要关闭，重新添加并启动
            f.shutdown()
            addAndStartFetcherThread(brokerAndFetcherId, brokerIdAndFetcherId)
          case None =>
            // 没有就创建
            addAndStartFetcherThread(brokerAndFetcherId, brokerIdAndFetcherId)
        }

        fetcherThreadMap(brokerIdAndFetcherId).addPartitions(initialFetchOffsets.map { case (tp, brokerAndInitOffset) =>
          tp -> brokerAndInitOffset.initOffset
        })
      }
    }
}
```

可以看到Fetcher线程的启动是通过addAndStartFetcherThread方法实现的，createFetcherThread刚好调用了前面的ReplicaFetcherManager

同时我们注意一下createFetcherThread方法的第二个参数传入的是broker，那么我们可以得出以下结论
1. 一个fetcher线程只会向一个broker同步
2. 一个fetcher线程管理了本地broker多个分区的同步，它和消费者一样都是发送的FETCH请求，此时我们就把它看做一个消费者，我们的业务代码中就是一个消费者线程可以拉取多个分区的消息

ReplicaFetcherThread的类图如下，执行的主体在它的父类AbstractFetcherThread的doWork方法中，具体的fetch逻辑由子类实现，典型的模板模式
![fetch-thread](https://pic.downk.cc/item/5e96a9c6c2a9a83be5876153.png)

## 同步线程doWork

AbstractFetcherThread的doWork方法是副本同步的入口，其中maybeTruncate是0.11版本之后，副本恢复的截断协议从HW改为leader epoch方式，过程较为复杂，后续会单独分析

```java
override def doWork() {
  maybeTruncate()
  // 构建fetch请求
  val fetchRequest = inLock(partitionMapLock) {
    val ResultWithPartitions(fetchRequest, partitionsWithError) = buildFetchRequest(states)
    if (fetchRequest.isEmpty) {
      trace(s"There are no active partitions. Back off for $fetchBackOffMs ms before sending a fetch request")
      // replica.fetch.backoff.ms
      partitionMapCond.await(fetchBackOffMs, TimeUnit.MILLISECONDS)
    }
    handlePartitionsWithErrors(partitionsWithError)
    fetchRequest
  }
  if (!fetchRequest.isEmpty)
    processFetchRequest(fetchRequest)
}
```

































