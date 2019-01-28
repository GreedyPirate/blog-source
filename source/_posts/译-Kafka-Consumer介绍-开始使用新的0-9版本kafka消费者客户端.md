---
title: '[译]Kafka Consumer介绍:使用新的0.9版本kafka消费者'
date: 2019-01-21 10:41:46
categories: Kafka Tutorial
tags: [kafka,中间件,翻译]
toc: true
comments: true
---

>原文地址：[Introducing the Kafka Consumer: Getting Started with the New Apache Kafka 0.9 Consumer Client](https://www.confluent.io/blog/tutorial-getting-started-with-the-new-apache-kafka-0-9-consumer-client/)

Kafka创建之初，自带了用Scala编写的生产者和消费者客户端，随着时间的推移，我们开始认识到这些API的许多局限性。例如，我们有一个"high-level"消费者API，它支持消费者组和故障转移，但却不支持更多更复杂的使用场景。我们还有一个"简版"消费者API，它提供了完全的控制，但需要用户自己处理故障转移和错误。因此我们开始重新设计这些客户端，以开启许多旧客户端难以支持甚至不可能支持的用例，并建立一组我们能够长期支持的API

第一阶段是在0.8.1版本重写了的生产者API，最近发布的0.9版本完成了第二阶段，引入了新版消费者API，建立在一个kafka自身提供的新的组协调者协议之上，新的消费者带来了以下优势：

* 新的消费者结合了"简版"和"高级"API的功能，同时提供了组协调者和低级别访问，以构建你自己的消费策略
* 减少依赖：新版消费者API使用原生java，不依赖Scala运行时环境或者Zookeeper，这使得它以一个轻量库包含在你的项目中
* 更安全：kafka 0.9中的安全扩展只支持新版消费者
* 新的消费者还增加了一组用于管理消费者进程组容错的协议。以前这个功能在java客户端中的实现很笨重(有许多和ZooKeeper的重量交互)，这种复杂的逻辑使得用其他语言构建时变得十分困难，随着新协议的引进，这变得容易的多，实际上我们已经将C client迁移到新协议上了

即使消费者使用了重新设计的API和组协调者协议，这些概念并没有变，因此熟悉老消费者的用户理解它应该没有太多问题