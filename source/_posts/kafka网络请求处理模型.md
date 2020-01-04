---
title: kafka网络请求处理模型
date: 2019-12-06 23:02:50
categories: Kafka Tutorial
tags: [kafka,中间件,消息]
toc: true
comments: true
---

众所周知，kafka是一款高性能，可伸缩的消息队列中间件，在一个庞大的kafka集群中，每秒能处理几十万条消息，那么必然存在着大量的网络请求，kafka是如何构建自己的网络请求模型的呢，答案就是Reactor

# Reactor模型
Reactor线程模型即为Java NIO中的selector模型。最简单的Reactor模型中，有多个client向服务端发送请求，首先请求会达到Dispatcher组件，它负责将不同的请求分发到多个线程中处理
![Reactor线程模型](https://ae01.alicdn.com/kf/H969ad8e2946e4fd284760f4868e91fcfD.png
)

Acceptor(Dispatcher)将变得十分轻量，因为它只负责分发，不用处理十分复杂的逻辑，而worker线程可以伸容、缩容，达到负载均衡的效果。
但是当处理读写任务的线程负载过高后，处理速度下降，事件会堆积，严重的会超时，可能导致客户端重新发送请求，性能越来越差

# Kafka
kafka server中当Acceptor将请求分发给Processor后，Processor并不处理，而是将请求放入到一个请求队列(requestQueue)中
同时还有一个IO线程池(KafkaRequestHandlerPoll)，该线程池大小由num.io.threads参数控制，默认为8，其中的KafkaRequestHandler线程，通过轮询的方式(300ms)从请求队列中取出请求，执行真正的处理。

KafkaRequestHandler背后真正负责处理的是一个叫做KafkaApis的类，它可以处理40多种请求，包含client与其它broker的请求，其中最重要的就是PRODUCE生产请求和FETCH消费拉取请求。PRODUCE将消息写入日志中；FETCH则从磁盘或页缓存中读取消息。

当IO线程处理完请求后，会将生成的响应发送到网络线程池的响应队列中，然后由对应的网络线程负责将Response返还给客户端。
请求队列是所有网络线程共享的，而响应队列则是每个网络线程专属的。这么设计的原因就在于，Acceptor只是用于请求分发而不负责响应回传，因此只能让每个网络线程自己发送Response给客户端，所以这些Response也就没必要放在一个公共的地方。

![kafka请求处理流程](https://ae01.alicdn.com/kf/H9d81a3a1afa14c2d945eef4dcce57ec8D.png)

在有了上面对kafka的感性认知之后，再来看看它的源码是怎么写的

# 准备
希望大家在看源码之前对java nio代码有一定的熟悉程度，可以事先看看博客，参考我github上的[代码](https://github.com/GreedyPirate/Spring-Cloud-Stack/tree/master/java8/src/main/java/com/net/nio)
阅读源码时不要去关注边缘代码，这会影响我们的思路，大部分代码我都做了删减，只保留核心部分代码

# 源码分析

## Acceptor与Processor的初始化

从Kafka#main方法出发，跟踪到KafkaServer#startup方法，该方法涵盖kafka server所有模块的初始化，其中也包括了网络请求处理模块
```java
// Create and start the socket server acceptor threads so that the bound port is known.
// Delay starting processors until the end of the initialization sequence to ensure
// that credentials have been loaded before processing authentications.
socketServer = new SocketServer(config, metrics, time, credentialProvider)
socketServer.startup(startupProcessors = false)
```
### SocketServer#startup

SocketServer核心代码如下，第一个参数numNetworkThreads，这是num.network.threads配置的值
第二个参数则是listeners配置的值，它是一个EndPoint集合
```java
class SocketServer{
	def startup(startupProcessors: Boolean = true) {
		createAcceptorAndProcessors(config.numNetworkThreads, config.listeners)
	}
}
```
### 创建Acceptor与Processor

遍历EndPoint集合并在里面创建Acceptor，并启动Acceptor线程

Acceptor的个数和配置里listeners的个数有关，也就是一个服务用几个端口去监听网络请求

```java
private def createAcceptorAndProcessors(processorsPerListener: Int,
                                  endpoints: Seq[EndPoint]): Unit = synchronized {
	endpoints.foreach { endpoint =>
		val acceptor = new Acceptor(endpoint, sendBufferSize, recvBufferSize, brokerId, connectionQuotas)
		addProcessors(acceptor, endpoint, processorsPerListener)

		KafkaThread.nonDaemon(s"kafka-socket-acceptor-$listenerName-$securityProtocol-${endpoint.port}", acceptor).start()
		acceptor.awaitStartup()
		acceptors.put(endpoint, acceptor)
	}
}
```

### Acceptor类简要分析

进入到Acceptor类中，下面这行代码建立了socket。注：scala类中，非方法的语句在初始化之前都会先执行，而java中语句只能定义在方法中

Acceptor的初始化是一个标准的nio ServerSocketChannel创建
```java
private[kafka] class Acceptor extends AbstractServerThread(间接extends Runnable) {
	// 初始化语句
	val serverChannel = openServerSocket(endPoint.host, endPoint.port)
}

// 相信熟悉java nio代码的同学看到这里很熟悉
private def openServerSocket(host: String, port: Int): ServerSocketChannel = {
	val socketAddress =
	  if (host == null || host.trim.isEmpty)
	    new InetSocketAddress(port)
	  else
	    new InetSocketAddress(host, port)
	val serverChannel = ServerSocketChannel.open()
	serverChannel.configureBlocking(false)

	serverChannel.socket.bind(socketAddress)	 
	serverChannel
}
```

#### Acceptor#run方法
Acceptor本身也是一个线程，在run方法中通过select轮询，接收请求，并将数据通过round-robin(轮询)的方式分配给不同的Processor处理
```java
def run() {
	serverChannel.register(nioSelector, SelectionKey.OP_ACCEPT)
	var currentProcessor = 0
	while (isRunning) {
		val ready = nioSelector.select(500)
		if (ready > 0) {
			val keys = nioSelector.selectedKeys()
			val iter = keys.iterator()
			while (iter.hasNext && isRunning) {
		  
		    	val key = iter.next
		    	iter.remove()
		    	if (key.isAcceptable) {
		      		val processor = synchronized {
		        	currentProcessor = currentProcessor % processors.size
		        	processors(currentProcessor)
		      	}
		      	accept(key, processor)
		    	} else
		      	throw new IllegalStateException("Unrecognized key state for acceptor thread.")
		    	// round robin to the next processor thread, mod(numProcessors) will be done later
		    	currentProcessor = currentProcessor + 1
		  	} 
		}
	}    
}
```
注意accept(key, processor)这行代码是将请求保存到了Processor对象一个叫做newConnections队列对象中了，它的类型为ConcurrentLinkedQueue<SocketChannel>
这为Processor后续的处理埋下了伏笔

### Processor初始化

在初始化Acceptor之后，在创建Acceptor之后，[addProcessors](#创建Acceptor与Processor)方法会根据num.network.threads的个数循环去创建Processor线程
```java
  private def addProcessors(acceptor: Acceptor, endpoint: EndPoint, newProcessorsPerListener: Int): Unit = synchronized {
    val listenerProcessors = new ArrayBuffer[Processor]()

    for (_ <- 0 until newProcessorsPerListener) {
      val processor = newProcessor(nextProcessorId, connectionQuotas, listenerName, securityProtocol, memoryPool)
      listenerProcessors += processor
      requestChannel.addProcessor(processor)
      nextProcessorId += 1
    }
    listenerProcessors.foreach(p => processors.put(p.id, p))
    acceptor.addProcessors(listenerProcessors)
  }
```
其中有两行代码比较关键
requestChannel.addProcessor(processor)是将Processor添加到RequestChannel的缓存map中，为以后的请求处理做准备
acceptor.addProcessors(listenerProcessors)是为Acceptor和Processor做关系映射，说明该Processor属于该Acceptor，同时该方法内部启动了Processor线程

相关代码如下：
```java
private def startProcessors(processors: Seq[Processor]): Unit = synchronized {
	processors.foreach { processor =>
		KafkaThread.nonDaemon("...",processor).start()
	}
}
```

### Processor类简要分析 

既然Processor是一个线程，那么run方法是必须要看的, 在看run方法之前，需要知道Processor初始化了一个selector
虽然它叫KSelector(scala可以重命名import的类), 但熟悉kafka java客户端源码的同学应该知道，这就是kafka客户端中基于java nio Selector封装的Selector。
希望大家不要被绕晕了，简而言之，kafka在java nio Selector上封装了一个自己的Selector，而且是同名的
```java
import org.apache.kafka.common.network.{..., Selector => KSelector}

protected[network] def createSelector(channelBuilder: ChannelBuilder): KSelector = {
    new KSelector(
      maxRequestSize,
      connectionsMaxIdleMs,
      metrics,
      time,
      "socket-server",
      metricTags,
      false,
      true,
      channelBuilder,
      memoryPool,
      logContext)
}
```

```java
override def run() {
	startupComplete()
	while (isRunning) {
		// setup any new connections that have been queued up
		configureNewConnections()
		// register any new responses for writing
		processNewResponses()
		poll()
		processCompletedReceives()
		processCompletedSends()
		processDisconnected()
	}

}
```
在[Acceptor#run方法](#Acceptor#run方法)中，kafka将SocketChannel保存到了Processor的一个ConcurrentLinkedQueue<SocketChannel>队列中
这里依次调用的6个方法就是对队列轮询，取出SocketChannel并做处理
1. configureNewConnections：向selector注册OP_READ事件
2. processNewResponses：处理响应
3. poll：将网络请求放入到List<NetworkReceive>集合中
4. processCompletedReceives：遍历List<NetworkReceive>集合，构建Request对象，放入到RequestChannel的requestQueue队列中
5,6省略，不是现在关心的

#### processCompletedReceives详细
```java
private def processCompletedReceives() {
	selector.completedReceives.asScala.foreach { receive =>
	    openOrClosingChannel(receive.source) match {
	      case Some(channel) =>
	      	// 构建Request对象
	        val header = RequestHeader.parse(receive.payload)
	        val connectionId = receive.source
	        val context = new RequestContext(header, connectionId, channel.socketAddress,
	          channel.principal, listenerName, securityProtocol)
	        val req = new RequestChannel.Request(processor = id, context = context,
	          startTimeNanos = time.nanoseconds, memoryPool, receive.payload, requestChannel.metrics)
	        // 发送到RequestChannel的requestQueue队列中
	        requestChannel.sendRequest(req)
	    }
	} 
	
}
```
至此我们已经分析完了一半的流程
![处理流程](https://ae01.alicdn.com/kf/H5ba778ed69274f8380f75ff04b88d2234.png)


## KafkaRequestHandler

KafkaRequestHandler线程run方法如下, requestChannel.receiveRequest就是每300ms从requestQueue轮询请求，最后交给KafkaApis#handle方法处理

```java
def run() {
	while (!stopped) {
		val req = requestChannel.receiveRequest(300)
		req match {
			case request: RequestChannel.Request =>
				apis.handle(request)
		}
	} 
}
```

### RequestChannel

requestQueue是一个ArrayBlockingQueue有界队列，队列大小由queued.max.requests参数控制

```java
class RequestChannel(val queueSize: Int) extends KafkaMetricsGroup {
	private val requestQueue = new ArrayBlockingQueue[BaseRequest](queueSize)

	def receiveRequest(timeout: Long): RequestChannel.BaseRequest = 
		requestQueue.poll(timeout, TimeUnit.MILLISECONDS)
}

```

### KafkaApis处理请求
KafkaApis具体的请求处理流程以[kafka server端源码分析之接收消息(一)](https://greedypirate.github.io/2019/11/02/kafka-server%E7%AB%AF%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B9%8B%E6%8E%A5%E6%94%B6%E6%B6%88%E6%81%AF-%E4%B8%80/)为例，在每个请求处理结束后，都会调用RequestChannel的sendResponse方法，将Response放入LinkedBlockingDeque无界队列
```java
  private val responseQueue = new LinkedBlockingDeque[RequestChannel.Response]()

  /** Send a response back to the socket server to be sent over the network */
  def sendResponse(response: RequestChannel.Response) {
    val processor = processors.get(response.processor)
    // The processor may be null if it was shutdown. In this case, the connections
    // are closed, so the response is dropped.
    if (processor != null) {
      processor.enqueueResponse(response)
    }
  }
  // 放入队列
  private[network] def enqueueResponse(response: RequestChannel.Response): Unit = {
    responseQueue.put(response)
    wakeup()
  }
```

![sendResponse方法调用链](https://ae01.alicdn.com/kf/H4e58b30e781b4d2bbdc0a9010fe240dfa.png)

### 发送响应
在[Processor类简要分析](#Processor类简要分析)中，有6个方法，其中第2个方法用于处理新的响应，现在向responseQueue队列中放入了Response对象后，回过头来看看发送的源码
```java
private def processNewResponses() {
    var currentResponse: RequestChannel.Response = null
    while ({currentResponse = dequeueResponse(); currentResponse != null}) {
      val channelId = currentResponse.request.context.connectionId

      currentResponse match {
        case response: SendResponse =>
          sendResponse(response, response.responseSend)
      }
      
    }
}
```
处理响应的代码很清晰，通过dequeueResponse方法获取responseQueue中的Response，循环发送给客户端或其它broker
sendResponse方法的代码几乎只有一行，inflightResponses用于保存发送中的Response，表示还未收到客户端应答

注：带有inflight前缀的还有inflightRequest，含义也是类似的，常用于客户端请求
```java
selector.send(responseSend)
inflightResponses += (connectionId -> response)
```

至此整个kafka的网络请求处理流程分析结束，接下来将会分析kafka server如何处理生产者发送的消息，如何应答消费者的fetch请求，这些都是实际开发中常用的功能，必须要扎实掌握


















