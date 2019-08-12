---
title: Kafka生产者源码浅析(一)
date: 2019-01-28 15:27:22
categories: Kafka Tutorial
tags: [kafka,中间件,消息]
toc: true
comments: true
---


# Kafka生产者源码浅析(一)

>本文并没有直接使用原生的kafka-client，而是spring-kafka，版本为2.2.3.RELEASE。在当前以Spring-boot为首的潮流中，有必要学习Spring是如何集成kafka客户端的

# send方法
以KafkaTemplate#send方法为入口，使用debug方式跟进源码

```java
  @Override
  public ListenableFuture<SendResult<K, V>> send(String topic, K key, @Nullable V data) {
    ProducerRecord<K, V> producerRecord = new ProducerRecord<>(topic, key, data);
    return doSend(producerRecord);
  }
```
这里将消息封装为ProducerRecord对象，这是kafka-client原生对象，接下来进行发送操作
在doSend方法中，有很多事务相关，日志相关的代码，我们的目的是理清楚主流程，因此省略

```java
  protected ListenableFuture<SendResult<K, V>> doSend(final ProducerRecord<K, V> producerRecord) {
		  final Producer<K, V> producer = getTheProducer();
		  final SettableListenableFuture<SendResult<K, V>> future = new SettableListenableFuture<>();
		  producer.send(producerRecord, buildCallback(producerRecord, producer, future));
		  return future;
  }
```
可以看到首先通过getTheProducer获取生产者对象，那么Spring-kafka是如何创建该对象的呢？
## 构建生产者
代码只有一行，通过DefaultKafkaProducerFactory创建生产者
```java
return this.producerFactory.createProducer();
```
进入到DefaultKafkaProducerFactory#createProducer

```java
  @Override
  public Producer<K, V> createProducer() {
    // ...
    if (this.producer == null) {
      synchronized (this) {
        if (this.producer == null) {
          this.producer = new CloseSafeProducer<K, V>(createKafkaProducer());
        }
      }
    }
    return this.producer;
  }
```
我们知道kafka生产者是单例并且线程安全的，这里spring使用double-check构建了一个CloseSafeProducer对象，而它被volatile修饰，经典的懒汉单例模式
平常我们使用的都是KafkaProducer，这个CloseSafeProducer又是什么呢？
该类实现了Producer接口，这也是KafkaProducer的父接口，细心的同学发现了CloseSafeProducer在创建是调用了createKafkaProducer方法，该方法源码如下

```java
  protected Producer<K, V> createKafkaProducer() {
    return new KafkaProducer<K, V>(this.configs, this.keySerializer, this.valueSerializer);
  }
```
坑爹呢这是，这个不还是KafkaProducer对象吗，那么传入一个KafkaProducer是要干吗，对装饰者模式和代理模式熟悉的同学已经明白是怎么回事，spring也确实这样做的：具体功能实现都委托给KafkaProducer对象实现，spring对记录事务id等日志信息做了增强

```java
	protected static class CloseSafeProducer<K, V> implements Producer<K, V> {

		private final Producer<K, V> delegate;

		private final BlockingQueue<CloseSafeProducer<K, V>> cache;

		private final Consumer<CloseSafeProducer<K, V>> removeConsumerProducer;

		private final String txId;

		private volatile boolean txFailed;
  }
```
CloseSafeProducer的分析至此结束，在获取到包装后的KafkaProducer后，便是发送流程了

## 消息发送
回到doSend方法，发送的代码只有两行
```java
final SettableListenableFuture<SendResult<K, V>> future = 
													new SettableListenableFuture<>();
producer.send(producerRecord, buildCallback(producerRecord, producer, future));
```
SettableListenableFuture是一个可设置，可监听的Future对象，用它构建异步发送消息后的Callback对象，读者可以认为Spring使用SettableListenableFuture对象对返回结果和异常进行了封装，Callback的作用在下文揭晓。

接着send方法由CloseSafeProducer委托给KafkaProducer执行，KafkaProducer的send方法如下

```java
    @Override
    public Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback) {
        // intercept the record, which can be potentially modified; this method does not throw exceptions
        ProducerRecord<K, V> interceptedRecord = this.interceptors.onSend(record);
        return doSend(interceptedRecord, callback);
    }
```
该方法的官方文档翻译如下：
```java
异步发送一条消息到一个topic，并且在应答之后立即调用已提供的回调
发送是异步的，一旦消息存储到了等待发送的缓冲区，该方法会立即返回。这样就不用阻塞在等待每一次发送消息的响应，允许并行的发送大量消息。
发送后的结果对象RecordMetadata具体说明了消息被发送到了哪个分区，被分配的位移和时间戳。
如果topic使用了TimestampType#CREATE_TIME，那么使用用户指定的时间，如果未指定，则使用发送时间。
如果使用了TimestampType#LOG_APPEND_TIME，则使用消息在broker端追加到日志的时间
send方法会为RecordMetadata对象返回一个Future对象，调用Future#get将会阻塞到请求完成，返回消息的元数据，或者返回在发送请求期间的任何异常
如果你想模拟一下，你可以send之后立即调用get
producer.send(record).get()
完全非阻塞的用法是用Callback参数来提供一个回调，它将在请求结束之后被调用
producer.send(myRecord, new Callback(){...})

Callback将在producer的I/O线程中触发，所以它必须轻量，快速，否则其他线程的消息会延迟发送。如果你在Callback中有耗时的逻辑处理，建议使用你自己的Executor，在Callback体中并发的执行
```
Spring同样支持同步和异步，将结果和异常都保存在了SettableListenableFuture中
这里再提一下Callback和Producerinterceptor的使用

### Callback
这里提一下Callback类，这是一个函数式接口，仅有一个onCompletion方法
```java
 public void onCompletion(RecordMetadata metadata, Exception exception);
```
两个参数分别为成功之后的消息元数据对象，和失败之后的异常对象，两者总是只有一个不为空(要么成功，要么失败)，而Exception分为两类异常：可重试异常，不可重试异常
可重试

```java
CorruptRecordException
InvalidMetadataException
NotEnoughReplicasAfterAppendException
NotEnoughReplicasException
OffsetOutOfRangeException
TimeoutException
UnknownTopicOrPartitionException
```
不可重试
```java
InvalidTopicException
OffsetMetadataTooLargeException
RecordBatchTooLargeException
RecordTooLargeException
UnknownServerException
```
可重试异常都继承自RetriableException，常见的判断方式如下：
```java
if(e instanceof RetriableException){
    ...
} else {
    ...
}
```
### 拦截器
在发送消息之前，开发者都可以自定义拦截器，实现Producerinterceptor即可
```java
// 每条消息发送之前调用
public ProducerRecord<K, V> onSend(ProducerRecord<K, V> record);
//发送请求应答之后调用
public void onAcknowledgement(RecordMetadata metadata, Exception exception);
public void close();
```
#### 添加拦截器
kafka原生配置方式

```java
List<String> interceptors = new ArrayList<>();
interceptors.add("your class"); 
props.put(ProducerConfig.INTERCEPTOR_CLASSES_CONFIG, interceptors);
```
在spring-kafka中配置更加简单
```
spring.kafka.producer.properties.interceptor.classes=your class
```

### 消息发送(doSend)

经过拦截器拦截后，发送消息的流程又是如何呢

![发送流程](https://ae01.alicdn.com/kf/H0d7dcdd1533945eda1d032ad9b7b7c5e8.png)

上图摘自胡夕老师的《Apache kafka实战》，十分形象的描绘了消息发送流程，正如上图所示，doSend方法只有有一个入参ProducerRecord，用于封装消息，一个出参RecordMetadata，它是broker应答之后的返回信息。二者的源码如下：

```java
// key长度，value长度可计算
public class ProducerRecord<K, V> {
    private final String topic;
    private final Integer partition;
    private final Headers headers;
    private final K key;
    private final V value;
    private final Long timestamp;
}

public final class RecordMetadata {
    private final long offset;
    private final long timestamp;
    private final int serializedKeySize;
    private final int serializedValueSize;
    private final TopicPartition topicPartition;
    private volatile Long checksum;
}
```

#### 预备知识

为了便于理解接下来的流程，有几个类需要为大家介绍清楚

在KafkaProducer的构造函数中初始化了以下几个关键类，有兴趣的读者可自行研究，可省略JMX和事务相关的内容

* Partitioner：分区选择器，你要发送的这条消息应该分配到哪个分区？

  ```java
  this.partitioner = 
          config.getConfiguredInstance(ProducerConfig.PARTITIONER_CLASS_CONFIG,Partitioner.class);
  ```

* KafkaClient：用于和broker做网络交互的客户端

  ```
  KafkaClient client = kafkaClient != null ? kafkaClient : new NetworkClient(...);
  ```

* Sender：用于批量发送消息的I/O线程，也称sender线程

  ```java
  this.sender = new Sender(...);
  ```

  sender.wakeup()的作用是：消息达到了batch.size了，起来干活

* KafkaThread：继承了Thread，构造函数可以传入线程名，和设置守护线程

  ```java
  this.ioThread = new KafkaThread(ioThreadName, this.sender, true);
  this.ioThread.start();
  ```

* RecordAccumulator：消息累加器，其实也就是常说的消息的内存缓冲区

在前期基本工作做好后，kafka便可以开始发送了，发送过程比较复杂，首先要获取broker端集群信息，broker到底是个什么情况，地址是什么，有几台服务器，里面已有的topic，topic已有的分区，分区在broker的分布，ISR列表，OLR列表等等信息，这些都是发送之前要关心的

* Metadata：这些元信息都封装在了Metadata类中，Metadata还负责这些元信息的缓存及刷新
* Cluster: Metadata中持有一个Cluster对象，kafka每一个broker都保存了topic的leader副本分区信息，producer只需要随机向一个broker发送请求就可以获取获取到，同时该对象还有每一个kafka broker节点的元信息，如ip端口等

* TopicPartition：将topic和计算好的分区封装到一起
* InterceptorCallback：如果没有拦截器，它就是Callback回调



本文至此结束，接下来发送的具体源码将在下文揭晓

