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

即使消费者使用了重新设计的API和组协调者协议，这些概念并没有变，因此熟悉老消费者的用户理解它应该没有太多问题。但是，在组管理和线程模型方面有一些细微的细节需要特别注意。这篇教程的目的是覆盖new consumer的基本用法，并解释这些细节

# 开始
深入代码之前，我们回顾一下基本概念。在kafka中，topic以partition为维度划分为一组日志，producer追加写入这些日志的尾部，消费者按自己的速度读取日志，kafka在消费者组间通过分布式分区扩展消息消费，消费者组是一组消费者共享的组标识，下图展示了一个topic，它有三个分区，还有一个消费者组，它有两个消费者成员，topic里的每一个分区只分配给组内的一个成员
![](https://www.confluent.io/wp-content/uploads/2016/08/New_Consumer_figure_1.png)

老的消费者依赖Zookeeper管理组，新消费者用一个建立在kafka本身之上的组协调者协议。对每一个消费者组，一个kafka broker被选为组协调者。该协调者负责管理消费者组的状态，它主要的工作是在新消费者加入组时，原有消费者离开时，和topic元信息发生改变时调节分区分配，重新分配分区的行为称之为重平衡组

当组首次初始化，消费者通常会从分区的最早或最近位移开始读取消息，然后按顺序读取每个分区。随着消费者的运行，它会提交它已经成功处理的消息的位移，如下图，消费者消费的位置在6，它上一次提交的位移是1
![](https://www.confluent.io/wp-content/uploads/2016/08/New_Consumer_Figure_2.png)

当一个分区重新分配给了组内的另一个消费者，初始的位移被设置到上一次提交的位移。如果上图中的消费者突然挂了，消费者组成员接管这个分区，位移从1开始消费。这种情况下，必须重新处理崩溃消费者分区的第6个位置

这张图里还展示了日志里的2个重点，日志末端位移(LEO)表示最后一条被写入日志的消息的位移，high watermark是最后一条被成功复制到副本的消息的位移。从消费者的角度来看，最重要的事情是你只能从high watermark的位置处开始读取，防止了消费者读取副本未同步完成，有可能丢失的消息

# 配置与初始化
在开始消费者学习之前，添加kafka-client依赖到你的项目，maven片段如下
```xml
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>0.9.0.0-cp1</version>
</dependency>
```
和其它kafka-client一样，使用properties文件构造consumer，在下面的例子中，我们提供了一个使用消费者组的最小配置
```
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("group.id", "consumer-tutorial");
props.put("key.deserializer", StringDeserializer.class.getName());
props.put("value.deserializer", StringDeserializer.class.getName());
KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props); 
```
和原来的消费者，生产者一样，我们需要为consumer配置一个broker初始化列表，让consumer发现集群的其余部分, 不需要提供集群里的所有server地址，消费者会从给定列表的broker中确定所有存活的broker集合，这里我们假设broker运行在本地。consumer还需要知道如何反序列化key和value。最终为了加入一个消费者组，我们需要指定gruop id。在接下来的学习中，我们将介绍更多的配置

# topic 订阅
开始消费之前，你必须首先订阅你的应用要读取的topic，在下面的例子中，我们订阅了主题"foo"和"bar"
```java
consumer.subscribe(Arrays.asList(“foo”, “bar”)); 
```
订阅之后，消费者可以和组内其它成员协调以获取分区分配，这些都是在你消费数据之后自动处理的，之后我们会展示如何通过assign API手动分配，但要记住不能自动和手动混合分配。

订阅方法是不可增加的：你必须在列表中包含所有的你想消费的topic。你可以随时改变你已经订阅过的topic，以前订阅的topic都会在你重新调用subscribe方法之后覆盖

# 基本的轮询
消费者需要能够并行的获取数据，可能来自多个broker上的多个topic的多个分区。consumer使用类似unix poll或select风格的API来做这件事：一旦topic被注册，所有以后的协调，重平衡和数据获取都由一个基于在事件循环调用的poll方法来驱动的，这是一个能够用单线程处理IO的简单，高效的实现。

订阅topic之后，你需要开始时间循环，以分配到一个分区并开始获取数据，这听起来很复杂，但你只需要在循环里调用poll方法，然后consumer处理剩下的事情。每次调用poll方法，它会返回一组来自分配的分区里的消息(可能为空)，下面的例子展示了poll循环的基本用法，打印了offset和vlaue：
```java
try {
  while (running) {
    ConsumerRecords<String, String> records = consumer.poll(1000);
    for (ConsumerRecord<String, String> record : records)
      System.out.println(record.offset() + ": " + record.value());
  }
} finally {
  consumer.close();
}
```
poll方法基于当前位置获取数据，当消费者组首次创建，会根据reset策略设置位置(通常为每个分区设置成earliest或latest位移)。一旦消费者开始提交位移，然后每次rebalance会重置到上次提交的位置。poll方法的参数控制了当前位置的消息消费者需要阻塞的最大时间，一旦有消息可用，消费者立即返回，但如果没有任何消息，则会等待到给定的超时时间后返回

consumer对象被设计成在自己的线程中允许，这在内部没有同步机制的情况下对于多线程是不安全的，这也不是个好主意，在这个例子中，我们使用一个标志位，当应用关闭时终止poll循环。当标志位在另一个线程中被设置为false时，一旦poll返回就会终止，不管返回了什么消息，应用也会停止处理。

你应该在完成消费后关闭它，不仅仅是清理它使用的socket连接，它还提醒了消费者组它要从组中离开了

这里例子使用了一个相对较小的超时时间，来保证关闭consumer时没有太多的延迟。可选的，你可以使用一个很大的超时时间，并用wakeup方法结束循环。

```java
        try {
            while (true) {
                ConsumerRecords<String, String> records = consumer.poll(Long.MAX_VALUE);
                for (ConsumerRecord<String, String> record : records) 
                    System.out.println(record.offset() + ": "+record.value());
            }
        } catch (WakeupException e) {   
            // ignore for shutdown
        } finally {  
        	consumer.close(); 
        }
        return value;
```
我们将超时时间改为Long.MAX_VALUE，这基本意味着consumer无限的阻塞，直到返回下一批消息。不像前面的例子设置一个标志位那样，线程可以通过consumer.wakeup()触发一个shutdown事件来中断运行中的poll，并抛出WakeupException，这个方法是线程安全的。注意如果当前没有运行中的poll，这个异常将会延续到下次请求，在这个例子中我们捕获了这个异常防止它传播

# 汇总到一起
接下来的例子，我们将所有东西都放在一起构建一个简单的Runnable任务，该任务初始化消费者，订阅一组topic，无限循环地执行poll，直到在外面关闭

```java
public class ConsumerLoop implements Runnable {
  private final KafkaConsumer<String, String> consumer;
  private final List<String> topics;
  private final int id;

  public ConsumerLoop(int id,
                      String groupId, 
                      List<String> topics) {
    this.id = id;
    this.topics = topics;
    Properties props = new Properties();
    props.put("bootstrap.servers", "localhost:9092");
    props.put(“group.id”, groupId);
    props.put(“key.deserializer”, StringDeserializer.class.getName());
    props.put(“value.deserializer”, StringDeserializer.class.getName());
    this.consumer = new KafkaConsumer<>(props);
  }
 
  @Override
  public void run() {
    try {
      consumer.subscribe(topics);

      while (true) {
        ConsumerRecords<String, String> records = consumer.poll(Long.MAX_VALUE);
        for (ConsumerRecord<String, String> record : records) {
          Map<String, Object> data = new HashMap<>();
          data.put("partition", record.partition());
          data.put("offset", record.offset());
          data.put("value", record.value());
          System.out.println(this.id + ": " + data);
        }
      }
    } catch (WakeupException e) {
      // ignore for shutdown 
    } finally {
      consumer.close();
    }
  }

  public void shutdown() {
    consumer.wakeup();
  }
}
```
要测试这个用例，你要运行release 0.9.0.0版本的kafka，和一个用字符串数据的topic，最简单的方式使用kafka-verifiable-producer.sh监本写一批数据到一个topic。为了更有意思，我们应该确保topic有多个分区，这样一个消费者就不需要做所有事（？？）。例如，一个broker和Zookeeper都运行在本地，你可能会做以下的事：

```bash
# bin/kafka-topics.sh --create --topic consumer-tutorial --replication-factor 1 --partitions 3 --zookeeper localhost:2181
```
```bash
# bin/kafka-verifiable-producer.sh --topic consumer-tutorial --max-messages 200000 --broker-list localhost:9092
```

然后我们创建一个小的驱动来创建一个含有三个消费者的消费者组，都订阅了我们刚刚创建的topic
```java
public static void main(String[] args) { 
  int numConsumers = 3;
  String groupId = "consumer-tutorial-group"
  List<String> topics = Arrays.asList("consumer-tutorial");
  ExecutorService executor = Executors.newFixedThreadPool(numConsumers);

  final List<ConsumerLoop> consumers = new ArrayList<>();
  for (int i = 0; i < numConsumers; i++) {
    ConsumerLoop consumer = new ConsumerLoop(i, groupId, topics);
    consumers.add(consumer);
    executor.submit(consumer);
  }

  Runtime.getRuntime().addShutdownHook(new Thread() {
    @Override
    public void run() {
      for (ConsumerLoop consumer : consumers) {
        consumer.shutdown();
      } 
      executor.shutdown();
      try {
        executor.awaitTermination(5000, TimeUnit.MILLISECONDS);
      } catch (InterruptedException e) {
        e.printStackTrace;
      }
    }
  });
}
```
这个例子向线程池提交了3个consumer线程，每个线程都分配了一个id，这样你可以观察到是哪个线程在接收数据，当你关闭应用时，将会执行shutdown hook，它会用wakeup暂停三个线程，并等待它们关闭。运行之后，你讲看到许多来自线程中的数据，下面是一个样例：
```java
2: {partition=0, offset=928, value=2786}
2: {partition=0, offset=929, value=2789}
1: {partition=2, offset=297, value=891}
2: {partition=0, offset=930, value=2792}
1: {partition=2, offset=298, value=894}
2: {partition=0, offset=931, value=2795}
0: {partition=1, offset=278, value=835}
2: {partition=0, offset=932, value=2798}
0: {partition=1, offset=279, value=838}
1: {partition=2, offset=299, value=897}
1: {partition=2, offset=300, value=900}
1: {partition=2, offset=301, value=903}
1: {partition=2, offset=302, value=906}
1: {partition=2, offset=303, value=909}
1: {partition=2, offset=304, value=912}
0: {partition=1, offset=280, value=841}
2: {partition=0, offset=933, value=2801}
```

这些展示了三个分区交叉消费情况，每个分区分配给了一个消费者线程，在每个分区里，正如预期的显示了offset的增加，你可以用Ctrl+C或者通过你的IDE来终止进程。

# 消费者活力(求生欲?)
当作为消费者组的一部分时，每个消费者都会从其订阅的主题中分配分区的一个子集。通常是分区上的一组锁，只要持有锁，就没有其他消费者读取它们。当你的消费者是健康的，这就是你想要的结果。这是你避免重复消费的唯一方式。但如果消费者由于机器或应用死亡，你需要锁被释放，以便分配给其它的健康消费者

笔者解读：对于一个消费者组来说，一个分区只能被一个消费者消费，作者想表达的是读取时加锁，防止别的消费者读取，实现了消费者之间避免重复消费

kafka的组协调者协议使用心跳机制来解决这个问题，每次rebalance之后，当代所有的消费者开始发送周期性的心跳给组协调者，只要协调者继续接收心跳，它就会假定消费者是健康的，每收到一次心跳，协调者就会开启或重置一个计时器，如果计时器过期了还没有心跳，协调者就会标记这个消费者死亡了，给组内其余成员发送重新入组的信号，以便分区被重新分配，这个计时器的间隔被称为session timeOut，在客户端通过以下方式配置：session.timeout.ms
```java
props.put(“session.timeout.ms”, “60000”);
```
会话过期保证在机器，应用崩溃，或消费者与协调者之间的网络被隔离的情况下释放锁，但是应用失败有点棘手，因为消费者仍在发送心跳给协调者，并不表明应用时健康的

consumer的poll方法旨在解决这个问题，当你调用poll或者其它阻塞API时，所有的网络IO都在前台完成(？？)。consumer从不使用后台线程，这表示在你调用poll时只有心跳发送给了协调者，当你的应用停止poll(抛出异常或是下有系统挂了)时，就不再发送心跳，会话就会过期，组内开始rebalance。
唯一的问题是如果消费者处理消息的时间超过了会话过期时间就会触发一次假的rebalance，你应该因此将session timeout设置的足够大，来使这不太可能发送，默认是30秒，但设置成几分钟也是没道理的。更大的session timeout的缺点是，协调者将会更多的时间检测到消费者真的挂了的情况

# 消息传递语义

当消费者组首次创建，根据auto.offset.reset设置的值来初始化位移，一旦消费者开始执行，它会根据应用需要定期的提交位移。在每个后来的rebalance之后，分区的位移会被设置到上一次组提交的位置上，如果consumer在成功处理消息，却又在提交之前崩溃了，结果是另一个consumer会重复复工作。你提交的越频繁，你在崩溃期间看到的重复消费就越少。

在目前为止的例子中，我们假设自动提交是开启的。当前enable.auto.commit为true(默认值),consumer会根据auto.commit.interval.ms设置的值，周期性的自动提交位移。通过较少提交间隔的方式，你可以限制在consumer崩溃时的重复消费数量。

为了使用提交的API，首先你应该禁止位移自动提交，设置enable.auto.commit为false














