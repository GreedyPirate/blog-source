---
title: KafkaController源码分析之Controller选举与初始化
date: 2020-02-12 15:32:10
categories: Kafka Tutorial
tags: [kafka,中间件,消息]
toc: true
comments: true
---

本文来分析下kafka的重要模块——Controller，主要介绍Controller的选举与初始化过程

# KafkaController

初始化的入口依然在KafkaServer#startup方法中
```java
// 创建BrokerInfo: {Broker{id,EndPoint,rack}, apiversion, jmxport}
val brokerInfo = createBrokerInfo 
// zk中注册 /brokers/ids/0 节点
zkClient.registerBrokerInZk(brokerInfo)

/* start kafka controller */
kafkaController = new KafkaController(config, zkClient, time, metrics, brokerInfo, tokenManager, threadNamePrefix)
kafkaController.startup()
```
在讲解KafkaController#startup之前，需要说明下KafkaController中有很多成员变量，主要分为

1. zk事件处理器(ZNodeChangeHandler，ZNodeChildChangeHandler)
2. StateMachine(有限状态机): 副本的状态机，分区的状态机，主要负责状态的维护及转换时的处理
3. ControllerContext：broker，topic，partition，replica相关的数据缓存
4. ControllerEventManager: zk事件管理器，详见[Zookeeper初始化与Watcher监听事件分发]()

# KafkaController启动

```java
def startup() = {
	// StateChangeHandler用于处理ZooKeeper AuthFailed事件，Zookeeper初始化与Watcher监听事件分发一文有提到
	zkClient.registerStateChangeHandler(new StateChangeHandler {
		// 非核心...
	})
	// Startup是一个ControllerEvent，ControllerEventThread会执行它的process方法
	eventManager.put(Startup)
	// 启动了ControllerEventManager
	eventManager.start()
}
```

Startup类定义如下
```java
case object Startup extends ControllerEvent {
    def state = ControllerState.ControllerChange

    override def process(): Unit = {
      // 如方法名所示：注册监听/controller节点的handler, 并检查/controller节点是否存在
      zkClient.registerZNodeChangeHandlerAndCheckExistence(controllerChangeHandler)
      // Controller选举
      elect()
    }

}
```
注册的ControllerChangeHandler主要监听/controller节点的创建，删除，以及数据改变事件，此处暂且不深入研究

# KafkaController选举
接下来的elect方法是关于Controller选举的核心方法，前文说过，选举很简单，负责的是里面各种变量的初始化
```java
private def elect(): Unit = {
    val timestamp = time.milliseconds
    // 获取zk /controller节点中的ControllerId，没有返回-1
    activeControllerId = zkClient.getControllerId.getOrElse(-1)

    // ControllerId=-1，表示当前broker已成为Controller，属于特殊场景下的防止死循环优化
    if (activeControllerId != -1) {
      debug(s"Broker $activeControllerId has been elected as the controller, so stopping the election process.")
      return
    }

    try {
      // 尝试去创建/controller节点，如果创建失败了(已存在)，会在catch里处理NodeExistsException
      zkClient.checkedEphemeralCreate(ControllerZNode.path, ControllerZNode.encode(config.brokerId, timestamp))
      info(s"${config.brokerId} successfully elected as the controller")
      activeControllerId = config.brokerId
      onControllerFailover()
    } catch {
      case _: NodeExistsException =>
        // If someone else has written the path, then
        activeControllerId = zkClient.getControllerId.getOrElse(-1)

        // 如果/controller已存在， brokerid就不会是-1
        // {"version":1,"brokerid":0,"timestamp":"1582610063256"}
        if (activeControllerId != -1)
          debug(s"Broker $activeControllerId was elected as controller instead of broker ${config.brokerId}")
        else
          // 上一届controller刚下台，节点还没删除的情况
          warn("A controller has been elected but just resigned, this will result in another round of election")

      case e2: Throwable =>
        error(s"Error while electing or becoming controller on broker ${config.brokerId}", e2)
        triggerControllerMove()
    }
}
```
整个选举过程并不复杂，选举流程如下图所示
![选举过程](https://ae01.alicdn.com/kf/H9597605bc9844f07abc7848ec538840cJ.png)

# KafkaController初始化

真正复杂的是broker在成为Controller之后，在onControllerFailover方法中进行的一系列初始化动作
下面是源码，接下来是对onControllerFailover方法的分段讲解

```java
private def onControllerFailover() {

    info("Reading controller epoch from ZooKeeper")
    readControllerEpochFromZooKeeper()
    info("Incrementing controller epoch in ZooKeeper")
    incrementControllerEpoch()
    info("Registering handlers")


    val childChangeHandlers = Seq(brokerChangeHandler, topicChangeHandler, topicDeletionHandler, logDirEventNotificationHandler,
      isrChangeNotificationHandler)
    childChangeHandlers.foreach(zkClient.registerZNodeChildChangeHandler)
    val nodeChangeHandlers = Seq(preferredReplicaElectionHandler, partitionReassignmentHandler)
    nodeChangeHandlers.foreach(zkClient.registerZNodeChangeHandlerAndCheckExistence)

    info("Deleting log dir event notifications")
    zkClient.deleteLogDirEventNotifications()
    info("Deleting isr change notifications")
    zkClient.deleteIsrChangeNotifications()
    info("Initializing controller context")
    initializeControllerContext()

    info("Fetching topic deletions in progress")
    val (topicsToBeDeleted, topicsIneligibleForDeletion) = fetchTopicDeletionsInProgress()
    info("Initializing topic deletion manager")
    topicDeletionManager.init(topicsToBeDeleted, topicsIneligibleForDeletion)

    // We need to send UpdateMetadataRequest after the controller context is initialized and before the state machines
    // are started. The is because brokers need to receive the list of live brokers from UpdateMetadataRequest before
    // they can process the LeaderAndIsrRequests that are generated by replicaStateMachine.startup() and
    // partitionStateMachine.startup().
    info("Sending update metadata request")
    sendUpdateMetadataRequest(controllerContext.liveOrShuttingDownBrokerIds.toSeq)

    replicaStateMachine.startup()
    partitionStateMachine.startup()

    info(s"Ready to serve as the new controller with epoch $epoch")

    maybeTriggerPartitionReassignment(controllerContext.partitionsBeingReassigned.keySet)

    topicDeletionManager.tryTopicDeletion()

    val pendingPreferredReplicaElections = fetchPendingPreferredReplicaElections()
    onPreferredReplicaElection(pendingPreferredReplicaElections)

    info("Starting the controller scheduler")
    kafkaScheduler.startup()
    if (config.autoLeaderRebalanceEnable) {
      scheduleAutoLeaderRebalanceTask(delay = 5, unit = TimeUnit.SECONDS)
    }

    if (config.tokenAuthEnabled) {
      info("starting the token expiry check scheduler")
      tokenCleanScheduler.startup()
      tokenCleanScheduler.schedule(name = "delete-expired-tokens",
        fun = tokenManager.expireTokens,
        period = config.delegationTokenExpiryCheckIntervalMs,
        unit = TimeUnit.MILLISECONDS)
    }
}
```

## controller.epoch

controller.epoch表示Controller的版本号，初始值为0，每次产生新的Controller都会自增1
它的作用类似乐观锁的版本号，在Controller操作zk相关节点时，需要用它来表示节点是被哪一个Controller更新的

以下是初始化Controller时，从zk中/controller_epoch节点读取epoch的值，加1设置回zk，并更新本地缓存中的epoch
```java
/**
  * 这一段代码就是获取controller.epoch，并自增+1设置回zk
  */
info("Reading controller epoch from ZooKeeper")
// 获取/controller_epoch节点数据，初始化ControllerContext的epoch和epochZkVersion字段
readControllerEpochFromZooKeeper()
info("Incrementing controller epoch in ZooKeeper")
incrementControllerEpoch()
info("Registering handlers")
```

## 注册节点监听器

```java
/**
  * 注册一组childrenChangeHandler，在NodeChildrenChange事件触发后，会分发给这些handler
  */
// before reading source of truth from zookeeper, register the listeners to get broker/topic callbacks
val childChangeHandlers = Seq(brokerChangeHandler, topicChangeHandler, topicDeletionHandler, logDirEventNotificationHandler,
  isrChangeNotificationHandler)
childChangeHandlers.foreach(zkClient.registerZNodeChildChangeHandler)
// 注册/admin/preferred_replica_election, /admin/reassign_partitions节点事件处理
// 也是注册，不过要检查节点是否存在(这里不对是否存在做处理，只是保证没有异常)
val nodeChangeHandlers = Seq(preferredReplicaElectionHandler, partitionReassignmentHandler)
nodeChangeHandlers.foreach(zkClient.registerZNodeChangeHandlerAndCheckExistence)

/**
  * 删除节点：/log_dir_event_notification/log_dir_event_xxx，/isr_change_notification/isr_change_xxx节点
  */
info("Deleting log dir event notifications")
zkClient.deleteLogDirEventNotifications()
info("Deleting isr change notifications")
zkClient.deleteIsrChangeNotifications()
```
这里注册了很多handler，先用一个表格大致介绍一下，后面会有详细讲解

| handler                         | 监听的zk节点                      | 事件        | ControllerEvent                | 功能                     |
| ------------------------------- | --------------------------------- | ----------- | ------------------------------ | ------------------------ |
| brokerChangeHandler             | /brokers/ids                      | childChange | BrokerChange                   |                          |
| topicChangeHandler              | /brokers/topics                   | childChange | TopicChange                    |                          |
| topicDeletionHandler            | /admin/delete_topics              | childChange | TopicDeletion                  |                          |
| logDirEventNotificationHandler  | /log_dir_event_notification       | childChange | LogDirEventNotification        |                          |
| isrChangeNotificationHandler    | /isr_change_notification          | childChange | IsrChangeNotification          |                          |
| partitionReassignmentHandler    | /admin/reassign_partitions        | create      | PartitionReassignment          | 执行副本重分配           |
| preferredReplicaElectionHandler | /admin/preferred_replica_election | create      | PreferredReplicaLeaderElection | Preferred leader副本选举 |



## 初始化ControllerContext

首先需要说下ControllerContext是什么，以及它的功能

ControllerContext是zk中broker，topic，partition，replica等元数据的缓存对象，它主要有以下几个缓存
```java
// controller epoch 在kafka中epoch就相当于乐观锁的version
var epoch: Int = KafkaController.InitialControllerEpoch - 1
// 这是zk自带的version，通用的用于更新节点数据
var epochZkVersion: Int = KafkaController.InitialControllerEpochZkVersion - 1

// Map[Topic, Map[Partition, Seq[Replica]]] 存储每个topic的每个分区的副本集合
private var partitionReplicaAssignmentUnderlying: mutable.Map[String, mutable.Map[Int, Seq[Int]]] = mutable.Map.empty

// LeaderIsrAndControllerEpoch: {"controller_epoch":19,"leader":0,"version":1,"leader_epoch":57,"isr":[0,1,2]}
val partitionLeadershipInfo: mutable.Map[TopicPartition, LeaderIsrAndControllerEpoch] = mutable.Map.empty

// 准备要重分配副本的分区
val partitionsBeingReassigned: mutable.Map[TopicPartition, ReassignedPartitionsContext] = mutable.Map.empty

// 这要等到向Controller发送LeaderAndIsr请求之后，才能初始化，key是brokerId
val replicasOnOfflineDirs: mutable.Map[Int, Set[TopicPartition]] = mutable.Map.empty

//存活的broker
private var liveBrokersUnderlying: Set[Broker] = Set.empty
private var liveBrokerIdsUnderlying: Set[Int] = Set.empty
```

比较重要的几个变量是：
1. partitionReplicaAssignmentUnderlying：表示topic-partition-replica之间的关系数据
2. partitionLeadershipInfo：每个分区对应的leader信息
3. LeaderIsr：表示分区的leader以及isr信息
4. LeaderIsrAndControllerEpoch：在LeaderIsr的基础上加上controller epoch，表示它是被哪一个Controller写入的
5. state节点：以test的第0个分区为例：/brokers/topics/test/partition/0/state中的样例数据为 
    {"controller_epoch":19,"leader":0,"version":1,"leader_epoch":57,"isr":[0,1,2]}

### initializeControllerContext

controller选举的第三步——ControllerContext初始化，源码如下
```java
private def initializeControllerContext() {
    // update controller cache with delete topic information
    // 更新controllerContext缓存中的liveBrokers和allTopics信息
    controllerContext.liveBrokers = zkClient.getAllBrokersInCluster.toSet  // /brokers/ids 下所有的Broker信息（id，ip，port等）
    controllerContext.allTopics = zkClient.getAllTopicsInCluster.toSet // brokers/topics下所有的topic名
    registerPartitionModificationsHandlers(controllerContext.allTopics.toSeq) // 为allTopics中的每个topic注册监听处理器
    /**
      * 通过allTopics获取Map[TopicPartition, Seq[Replica]]
      * 再讲该map保存到controllerContext的Map[Topic, Map[partition, Seq[replicas]]] partitionReplicaAssignmentUnderlying
      */
    zkClient.getReplicaAssignmentForTopics(controllerContext.allTopics.toSet).foreach {
      case (topicPartition, assignedReplicas) => controllerContext.updatePartitionReplicaAssignment(topicPartition, assignedReplicas)
    }
    // 初始化partitionLeadershipInfo和shuttingDownBrokerIds
    controllerContext.partitionLeadershipInfo.clear()
    controllerContext.shuttingDownBrokerIds = mutable.Set.empty[Int]
    // register broker modifications handlers
    // 注册监听 /brokers/ids/0节点的handler，endpoint字段变化会更新liveBrokers缓存
    registerBrokerModificationsHandler(controllerContext.liveBrokers.map(_.id))
    // update the leader and isr cache for all existing partitions from Zookeeper
    // 获取分区节点的数据，并缓存到controllerContext.partitionLeadershipInfo对象里
    updateLeaderAndIsrCache()
    // start the channel manager
    // 启动ControllerChannelManager中处理ControllerEvent的RequestSendThread线程
    // Zookeeper初始化与Watcher监听事件分发中有详细介绍
    startChannelManager()
    // 看有没有要分区重分配的操作，有就加到partitionsBeingReassigned缓存里
    initializePartitionReassignment()
}
```
该方法主要是为了初始化ControllerContext的各个缓存，调用的方法也很多，下面选几个重要变量初始化的过程

#### 分区改变事件处理器
上面的registerPartitionModificationsHandlers为每一个topic新建了PartitionModificationsHandler

```java
private def registerPartitionModificationsHandlers(topics: Seq[String]) = {
  // 每一个topic新建处理器，并且添加到partitionModificationsHandlers
  topics.foreach { topic =>
    val partitionModificationsHandler = new PartitionModificationsHandler(this, eventManager, topic)
    partitionModificationsHandlers.put(topic, partitionModificationsHandler)
  }
  // 注册到zk watcher的NodeChangeHandler里
  partitionModificationsHandlers.values.foreach(zkClient.registerZNodeChangeHandler)
}
```
PartitionModificationsHandler主要处理/brokers/topics/topicxxx节点的数据改变事件，首先看一下该节点存储的样例数据
```json
{"version":1,"partitions":{"4":[1,2,0],"5":[2,0,1],"1":[1,0,2],"0":[0,2,1],"2":[2,1,0],"3":[0,1,2]}}
```
就是topic每一个分区的副本映射

#### 初始化分区副本分配关系

```java
zkClient.getReplicaAssignmentForTopics(controllerContext.allTopics.toSet).foreach {
  case (topicPartition, assignedReplicas) => controllerContext.updatePartitionReplicaAssignment(topicPartition, assignedReplicas)
}
```
以上代码是在初始化ControllerContext的partitionReplicaAssignmentUnderlying缓存，它保存的是每个topic的每个分区的副本映射，因此它是一个嵌套map类型
```java
Map[Topic, Map[Partition, Seq[Replica]]] partitionReplicaAssignmentUnderlying
```

#### 分区leader缓存与分区reassign

```java
private def updateLeaderAndIsrCache(partitions: Seq[TopicPartition] = controllerContext.allPartitions.toSeq) {
  // 每个分区节点的数据对象 {"controller_epoch":19,"leader":0,"version":1,"leader_epoch":57,"isr":[0,1,2]}
  val leaderIsrAndControllerEpochs = zkClient.getTopicPartitionStates(partitions)
  leaderIsrAndControllerEpochs.foreach { case (partition, leaderIsrAndControllerEpoch) =>
    // Map[TopicPartition, LeaderIsrAndControllerEpoch]
    controllerContext.partitionLeadershipInfo.put(partition, leaderIsrAndControllerEpoch)
  }
}
```
updateLeaderAndIsrCache方法会遍历controllerContext.allPartitions，获取/brokers/topics/topicxxx/partitions/xxx/state节点的数据
该节点的样例数据如下
```json
{"controller_epoch":19,"leader":0,"version":1,"leader_epoch":57,"isr":[0,1,2]}
```

分区reassign即分区副本重分配，相关内容后续会说到，这里仅说初始化
从/admin/reassign_partitions(临时节点)获取重分配方案，并复制给controllerContext.partitionsBeingReassigned
```java
private def initializePartitionReassignment() {
  // read the partitions being reassigned from zookeeper path /admin/reassign_partitions
  val partitionsBeingReassigned = zkClient.getPartitionReassignment
  info(s"Partitions being reassigned: $partitionsBeingReassigned")

  controllerContext.partitionsBeingReassigned ++= partitionsBeingReassigned.iterator.map { case (tp, newReplicas) =>
    val reassignIsrChangeHandler = new PartitionReassignmentIsrChangeHandler(this, eventManager, tp)
    tp -> new ReassignedPartitionsContext(newReplicas, reassignIsrChangeHandler)
  }
}
```

### topic删除管理器

初始化ControllerContext之后，接下来是topicDeletionManager——topic删除管理器的初始化
注：topic删除只会在delete.topic.enable为true时才能进行，而且分阶段进行删除
```java
// 要删除的topics和删除失败的topics
info("Fetching topic deletions in progress")
val (topicsToBeDeleted, topicsIneligibleForDeletion) = fetchTopicDeletionsInProgress()
// 初始化topic删除管理器
info("Initializing topic deletion manager")
topicDeletionManager.init(topicsToBeDeleted, topicsIneligibleForDeletion)
```
fetchTopicDeletionsInProgress的源码分析如下，init方法比较简单，这里就不说了
```java
private def fetchTopicDeletionsInProgress(): (Set[String], Set[String]) = {
  // 获取/admin/delete_topics 删除的topic
  val topicsToBeDeleted = zkClient.getTopicDeletions.toSet
  // 存在不在线的副本的topic
  val topicsWithOfflineReplicas = controllerContext.allTopics.filter { topic => {
    val replicasForTopic = controllerContext.replicasForTopic(topic)
    replicasForTopic.exists(r => !controllerContext.isReplicaOnline(r.replica, r.topicPartition))
  }}
  // 要reassign的topic
  val topicsForWhichPartitionReassignmentIsInProgress = controllerContext.partitionsBeingReassigned.keySet.map(_.topic)
  // 求二者并集，即有副本不在线的，和要reassign副本的topic都不能删，标记为不能删除(Ineligible)
  val topicsIneligibleForDeletion = topicsWithOfflineReplicas | topicsForWhichPartitionReassignmentIsInProgress
  (topicsToBeDeleted, topicsIneligibleForDeletion)
}

// We need to send UpdateMetadataRequest after the controller context is initialized and before the state machines
// are started. The is because brokers need to receive the list of live brokers from UpdateMetadataRequest before
// they can process the LeaderAndIsrRequests that are generated by replicaStateMachine.startup() and
// partitionStateMachine.startup().
// 在处理LeaderAndIsrRequest请求之前，先更新所有broker以及所有partition的元数据
sendUpdateMetadataRequest(controllerContext.liveOrShuttingDownBrokerIds.toSeq)
```
最后一行代码是为后面的副本状态机，分区状态机的启动做准备，将元数据同步给其它broker，让它们可以处理LeaderAndIsrRequest请求

### 副本状态机与分区状态机的启动

Controller初始化接下来的动作是启动副本状态机和分区状态机，二者都比较复杂，在另一篇文章中分析
```java
replicaStateMachine.startup()
partitionStateMachine.startup()
```

# 小结
Controller的初始化代码很多，后面的操作主要依赖于ControllerContext里的缓存以及与zk的交互，代码虽然很多，但却不难
后面的源码分析见以下文章列表

[KafkaController源码分析之副本状态机与分区状态机的启动]()
[KafkaController源码分析之分区重分配(PartitionReassignment)]()






















