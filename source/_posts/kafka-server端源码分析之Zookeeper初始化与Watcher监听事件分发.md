---
title: kafka-server端源码分析之Zookeeper初始化与Watcher监听事件分发
date: 2020-02-06 19:22:28
categories: Kafka Tutorial
tags: [kafka,中间件,消息]
toc: true
comments: true
---

这一篇比较简单，快速带大家过一下kafka如何连接ZooKeeper，以及kafka对节点事件监听的代码设计

# ZooKeeper大致介绍
kafka主要利用ZooKeeper选举Controller，这里先大致介绍下ZooKeeper的基本用法，仅用于学习Kafka

## ZNode

几乎所有的ZooKeeper教程都会告诉你ZooKeeper是一种类似文件系统目录结构的存储系统，但我不这么认为，文件系统中的目录本身无法存储数据，而ZooKeeper可以

ZooKeeper中的节点主要分为持久节点和临时节点，持久节点即使重启也会存在，因为它已经写入到磁盘文件了，而临时节点在ZooKeeper重启或是客户端会话超时后，就会消失

## zkVersion
简单的把它理解为乐观锁的版本号即可

## chroot

chroot的使用场景是一个zk集群管理了多套kafka集群，那么每个kafka集群需要一个根节点来区分
比如我们可以在kafka的sever.properties文件中这样配置: zookeeper.connect=localhost:2181/cluster_201

## zkCli

在ZooKeeper的bin目录下，可以启动zkCli.sh脚本，通过"ls 节点名"的方式获取子节点，通过"get 节点名"的方式获取该节点存储的数据
如果你已经有ZooKeeper的可视化管理工具，如zkui，shepher，查看起来就更方便了

## kafka选举Controller的原理

kafka是如何利用ZooKeeper的临时节点，来选举Controller的呢？
kafka集群的每个节点会在启动时创建/controller节点，如果该节点不存在，并且创建成功，那么该broker就成为Controller
其它broker创建时就会发现节点已存在，放弃成为Controller

# 初始化ZooKeeper

初始化ZooKeeper主要是建立连接，注册监听器，入口代码在KafkaServer的启动方法中(startup)，该方法调用了initZkClient
```java
private def initZkClient(time: Time): Unit = {
	// 方法定义，先不看
	def createZkClient(zkConnect: String, isSecure: Boolean) =
	  KafkaZkClient(zkConnect, isSecure, config.zkSessionTimeoutMs, config.zkConnectionTimeoutMs,
	    config.zkMaxInFlightRequests, time)

	// 获取chroot，没有返回None
	val chrootIndex = config.zkConnect.indexOf("/")
	val chrootOption = {
	  if (chrootIndex > 0) Some(config.zkConnect.substring(chrootIndex))
	  else None
	}

	// 安全配置相关
	val secureAclsEnabled = config.zkEnableSecureAcls // zookeeper.set.acl
	val isZkSecurityEnabled = JaasUtils.isZkSecurityEnabled()

	if (secureAclsEnabled && !isZkSecurityEnabled)
	  throw new java.lang.SecurityException(s"${KafkaConfig.ZkEnableSecureAclsProp} is true, but the verification of the JAAS login file failed.")

	// make sure chroot path exists
	// 确保chroot节点存在，没有则创建
	chrootOption.foreach { chroot =>
	  val zkConnForChrootCreation = config.zkConnect.substring(0, chrootIndex)
	  // 这里创建的是临时连接，仅为了创建chroot
	  val zkClient = createZkClient(zkConnForChrootCreation, secureAclsEnabled)
	  zkClient.makeSurePersistentPathExists(chroot)
	  info(s"Created zookeeper path $chroot")
	  zkClient.close()
	}

	// 调用前面定义的嵌套方法，创建KafkaZkClient对象，它是kafka对zk操作的封装类
	_zkClient = createZkClient(config.zkConnect, secureAclsEnabled)
	// 确保一些必须用到的节点存在，没有则创建，如: /brokers/ids, /brokers/topics, /config/changes等
	_zkClient.createTopLevelPaths()
}
```

createZkClient初始化了KafkaZkClient对象，我们来看看它的apply初始化方法
```java
def apply(connectString: String,
        isSecure: Boolean,
        sessionTimeoutMs: Int,
        connectionTimeoutMs: Int,
        maxInFlightRequests: Int,
        time: Time,
        metricGroup: String = "kafka.server",
        metricType: String = "SessionExpireListener") = {
	// 创建了ZooKeeperClient对象
	val zooKeeperClient = new ZooKeeperClient(connectString, sessionTimeoutMs, connectionTimeoutMs, maxInFlightRequests,
	  time, metricGroup, metricType)
	// 封装成KafkaZkClient
	new KafkaZkClient(zooKeeperClient, isSecure, time)
```

KafkaZkClient最常用的2个方法是对zk请求的保证
```java
private def retryRequestUntilConnected[Req <: AsyncRequest](request: Req): Req#Response = {
	// 单个请求包装成集合
	retryRequestsUntilConnected(Seq(request)).head
}
// 批量请求
private def retryRequestsUntilConnected[Req <: AsyncRequest](requests: Seq[Req]): Seq[Req#Response] = {
	val remainingRequests = ArrayBuffer(requests: _*) // 待发送的请求集合
	val responses = new ArrayBuffer[Req#Response] // 响应集合
	while (remainingRequests.nonEmpty) {
	  val batchResponses = zooKeeperClient.handleRequests(remainingRequests)

	  // metric ...
	  batchResponses.foreach(response => latencyMetric.update(response.metadata.responseTimeMs))

	  // Only execute slow path if we find a response with CONNECTIONLOSS
	  // 发现连接丢失错误的处理，继续
	  if (batchResponses.exists(_.resultCode == Code.CONNECTIONLOSS)) {
	    // zip方法：合并集合 A(1,2,3), B(4,5,6)
	    // 合并结果: [(1,4),(2,5),(3,6)]
	    val requestResponsePairs = remainingRequests.zip(batchResponses)

	    remainingRequests.clear()
	    requestResponsePairs.foreach { case (request, response) =>
	      if (response.resultCode == Code.CONNECTIONLOSS)
	        // 相当于是重新放进请求队列了，怪不得要判断remainingRequests.nonEmpty
	        remainingRequests += request
	      else
	        responses += response
	    }

	    if (remainingRequests.nonEmpty)
	      // 无限等待直到zk达到CONNECTED状态，或者在AUTH_FAILED/CLOSED状态下抛出异常
	      zooKeeperClient.waitUntilConnected()
	  } else {
	    // 响应结果正常的处理，返回结果
	    remainingRequests.clear()
	    responses ++= batchResponses
	  }
	}
	responses
}
```

## ZooKeeperClient与ZooKeeperClientWatcher

ZooKeeperClient类中初始化了原生的ZooKeeper对象
```java
@volatile private var zooKeeper = new ZooKeeper(connectString, sessionTimeoutMs, ZooKeeperClientWatcher)
```

Watch永远都是ZooKeeper的核心对象
```java
private[zookeeper] object ZooKeeperClientWatcher extends Watcher {
    override def process(event: WatchedEvent): Unit = {
      Option(event.getPath) match {
        case None =>
          val state = event.getState
          stateToMeterMap.get(state).foreach(_.mark())
          inLock(isConnectedOrExpiredLock) {
            isConnectedOrExpiredCondition.signalAll()
          }
          if (state == KeeperState.AuthFailed) {
            error("Auth failed.")
            stateChangeHandlers.values.foreach(_.onAuthFailure())
          } else if (state == KeeperState.Expired) {
            scheduleSessionExpiryHandler()
          }
        case Some(path) =>
          (event.getType: @unchecked) match {
            case EventType.NodeChildrenChanged => zNodeChildChangeHandlers.get(path).foreach(_.handleChildChange())
            case EventType.NodeCreated => zNodeChangeHandlers.get(path).foreach(_.handleCreation())
            case EventType.NodeDeleted => zNodeChangeHandlers.get(path).foreach(_.handleDeletion())
            case EventType.NodeDataChanged => zNodeChangeHandlers.get(path).foreach(_.handleDataChange())
          }
      }
    }
  }
```
和java类似的写法，没有path时kafka对AuthFailed和Expired两种情况作了处理，不是重点
ZooKeeper使用EventType表示节点的4种事件，kafka针对不同节点的不同事件都有一组handler去处理，这里通过path获取handler并执行

注：不要被foreach迷惑，Option类的foreach表示对象不为空，就执行传入的函数

handlers定义的Map如下，此时Map是空的，会在后续的kafka启动程序中将handler添加进来(比如Controller启动)
```java
private val zNodeChangeHandlers = new ConcurrentHashMap[String, ZNodeChangeHandler]().asScala
private val zNodeChildChangeHandlers = new ConcurrentHashMap[String, ZNodeChildChangeHandler]().asScala
```

其实到现在ZooKeeper已经启动完毕了，但是事情没有这么简单，kafka对事件的处理又采用了经典的内存队列异步处理模式，这种模式在kafka中无处不在

## kafka内存队列异步处理zk事件

通过一个简单的例子来说明kafka是如何处理zk事件的

上述的ZNodeChildChangeHandler只是一个接口，我们看下其中一个实现类
```java
class ControllerChangeHandler(controller: KafkaController, eventManager: ControllerEventManager) extends ZNodeChangeHandler {
  override val path: String = ControllerZNode.path

  override def handleCreation(): Unit = eventManager.put(controller.ControllerChange)
  override def handleDeletion(): Unit = eventManager.put(controller.Reelect)
  override def handleDataChange(): Unit = eventManager.put(controller.ControllerChange)
}
```
### 原理

zk将不同的节点事件，转换成kafka内部的事件处理器，封装成了一个ControllerEvent对象，然后放到一个内存队列里，启动一个线程轮询处理zk事件

BrokerChange的源码如下, 主要处理放到了process中
```java
// 上面的put方法
eventManager.put(controller.BrokerChange)

case object BrokerChange extends ControllerEvent {
	override def state: ControllerState = ControllerState.BrokerChange

	override def process(): Unit = {
	  // 省略处理代码 ...
	}
}
```
那么eventManager#put做了什么呢
```java
private val queue = new LinkedBlockingQueue[ControllerEvent]

def put(event: ControllerEvent): Unit = inLock(putLock) {
	queue.put(event)
}
```
把BrokerChange放入到了一个LinkedBlockingQueue中

而在后续的eventManager启动过程中，启动了ControllerEventThread线程

```java
private val thread = new ControllerEventThread(ControllerEventManager.ControllerEventThreadName)
def start(): Unit = thread.start()

class ControllerEventThread(name: String) extends ShutdownableThread(name = name, isInterruptible = false) {
	override def doWork(): Unit = {
	  // 从队列中取出ControllerEvent
	  queue.take() match {
	    case KafkaController.ShutdownEventThread => initiateShutdown()
	    case controllerEvent =>
	      _state = controllerEvent.state

	      eventQueueTimeHist.update(time.milliseconds() - controllerEvent.enqueueTimeMs)

	      try {
	        rateAndTimeMetrics(state).time {
	          // 调用process方法
	          controllerEvent.process()
	        }
	      } catch {
	        case e: Throwable => error(s"Error processing event $controllerEvent", e)
	      }

	      // metric相关的监听
	      try eventProcessedListener(controllerEvent)
	      catch {
	        case e: Throwable => error(s"Error while invoking listener for processed event $controllerEvent", e)
	      }

	      _state = ControllerState.Idle
	  }
	}
}
```

# 总结

从kafka的网络请求处理模型开始，就遇见了内存队列来异步处理的模型，这种模型和mq类似，不过它是本地内存中的队列，kafka有很多地方使用了这种模式，这也是我们学习源码之后的收获
最后用流程图总结下
![](https://ae01.alicdn.com/kf/Hb9e64a8b20784f63b40228cb32fcf6adO.png)








