---
title: 《快手万亿级别Kafka集群应用实践与技术演进之路》观后心得
date: 2019-11-21 15:08:09
categories: 架构随笔录
tags: [架构,演讲]
toc: true
comments: true
---

> 本文用于记录观看快手万亿级别Kafka集群应用实践与技术演进之路演讲后的心得，从中确实学到了很多，快手作为kafka的重度使用者，对kafka集群从不同角度优化，其中发现问题，解决问题的思路都值得学习

以下这张图是本次演讲的内容，分章节阐述具体内容
![](https://ae01.alicdn.com/kf/Hb4200382b3f448eeb7ebbda8d380768cP.jpg)


# 平滑扩容
kafka集群节点扩容时，要做副本迁移，但kafka是从副本最开始的offset同步的，并且消息是在不断写入的，那么就要去同步的速度要大于写入的速度，才有可能同步完，要么就会产生以下问题
## 磁盘读取
从最开始的offset开始迁移，就极有可能是在磁盘上，而不是在pageCache中，迁移过程将加大磁盘的负载，影响生产者的写入效率，造成不稳定

同时kafka有消息过期机制，假设同步完成后，消息过期了，就白同步了。而在大型集群中，一次同步动辄几个小时都是有可能的

## 解决方式
对最早的offset同步产生质疑，是否有可能从最新的offset开始同步，答案是肯定的，但是这样做会有消费者丢失数据的情况。
假设以下情形，消费者拉取消息offset为5，高水位线为8，LEO为10，此时集群扩容，新副本从10开始同步，那么新副本成为leader后，就丢失了6-9的数据

解决这个问题也不难，快手的解决方案是同步一段时间，等所有消费者都已经消费。
我个人的观点是从消费者已提交的位置开始同步，保险起见可以再往前一部分offset

# Mirror集群
MirrorMarker是用于同步多个kafka集群的工具，但它有很多缺点，不足以大规模应用
1. 无法同步消费者offset
2. 缺乏监控
3. topic是静态配置的，增加topic需要重启MirrorMarker
4. 只支持一个集群到另一个集群的同步，无法同步多个集群

# 资源隔离

在[kafka网络请求处理模型](https://greedypirate.github.io/2019/12/29/kafka%E7%BD%91%E7%BB%9C%E8%AF%B7%E6%B1%82%E5%A4%84%E7%90%86%E6%A8%A1%E5%9E%8B/)一文中，阐述了Processor是如何将请求放入requestQueue中的，
![kafka请求处理流程](https://ae01.alicdn.com/kf/H9d81a3a1afa14c2d945eef4dcce57ec8D.png)
该队列是一个有界队列，大小由queued.max.requests参数控制，默认是500，当某一个topic出现问题时，kafka处理速度变慢，导致队列满了，后续请求被阻塞，向上导致Processor和Acceptor接收网络请求被阻塞，这是不能接受的
快手的策略是多队列，每个队列后面跟一个worker线程池(KafkaRequestHandlerPoll), 同时发现队列满了以后，请求被丢弃

# cache改造
cache改造是我很感兴趣的一个部分，原来pageCache有很多稳定性问题，导致缓存命中率降低

在理想情况下，生产者发送的消费写入到pageCache中，消费者在缓存未失效的情况下，从pageCache中读取，整个过程是基于内存的，效率非常高
但是这以下两种情况下，pageCache会被污染，到时消息在pageCache中失效，需要从磁盘中读取，降低了效率

1. 线上环境消息积压是极有可能的情况，被积压的消息大概率是在磁盘中，此时消费者拉取消息就会将历史消息加载pageCache中
而正常topic写入的消息能够使用的pageCache就减少了，又会被写入到磁盘中，此时正常topic的消费者来消费时就要读取磁盘，降低了pageCache的命中率
2. follower副本从leader副本同步后，也要写入到broker磁盘中区，但是它也是要先写到pageCache中，我们都知道kafka中的follower副本是不提供读写能力的，那么它也写到pageCache中就很浪费内存了

## 解决方式
让kafka不要重度依赖pageCache，构建kafka自己的内存cache
![kafka-cache](https://ae01.alicdn.com/kf/H8509500f136e4f3f892e81b07e0cbd71B.png)

基于以上设计，consumer拉取消费时，先从cache中获取消息，没有再去pageCache中获取








