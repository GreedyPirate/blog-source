---
title: kafka-rebalance之JoinGroup
date: 2020-04-22 15:43:06
categories: Kafka Tutorial
tags: [kafka,中间件,消息]
toc: true
comments: true
---

# 前言

在AbstractCoordinator的initiateJoinGroup方法中，通过判断joinFuture为null，发起了JoinGroupRequest请求，本文主要讲解GroupCoordinator对该请求的处理。同样的，源码分为客户端发起请求时的参数，broker端的处理过程，以及consumer对响应的处理

# 发起请求

请求的发送代码在AbstractCoordinator的sendJoinGroupRequest方法在，方法比较简单，这里说点简单之外的事情

1. 首先确保已知coordinator节点，才能向它发起请求
2. generation.memberId初始化时为""
3. protocolType="consumer"
4. rebalanceTimeoutMs就是max.poll.interval.ms，这个结论可以从KafkaConsumer初始化ConsumerCoordinator得到
5. 4中的rebalanceTimeoutMs也是不最终客户端请求的超时时间，这里源码作者额外增加了5s

```java
// 省略部分代码
private synchronized RequestFuture<ByteBuffer> initiateJoinGroup() {
    if (joinFuture == null) {
    	// ...
        joinFuture = sendJoinGroupRequest();
        // ...
    }
    return joinFuture;
}
RequestFuture<ByteBuffer> sendJoinGroupRequest() {
	// 确保已知coordinator节点
    if (coordinatorUnknown())
        return RequestFuture.coordinatorNotAvailable();

    JoinGroupRequest.Builder requestBuilder = new JoinGroupRequest.Builder(
            groupId,
            this.sessionTimeoutMs,
            this.generation.memberId,
            protocolType(), // consumer
            metadata()).setRebalanceTimeout(this.rebalanceTimeoutMs);

    log.debug("Sending JoinGroup ({}) to coordinator {}", requestBuilder, this.coordinator);

    // Note that we override the request timeout using the rebalance timeout since that is the
    // maximum time that it may block on the coordinator. We add an extra 5 seconds for small delays.

    int joinGroupTimeoutMs = Math.max(rebalanceTimeoutMs, rebalanceTimeoutMs + 5000);
    return client.send(coordinator, requestBuilder, joinGroupTimeoutMs)
            .compose(new JoinGroupResponseHandler());
}
```

请求的格式的json形式如下：
```json
{
	"groupId": "test-group",
	"sessionTimeout": 30000,
	"memberId": "",
	"protocolType": "consumer",
	"groupProtocols": [
		{
			"name": "range", // PartitionAssignor的name
			"metadata": {
				"version": 0,
				"topic": "foo,bar", // 订阅的topic
				"user_data": null // 通常为null
			}
		}
	],
	"rebalanceTimeout": 10000
}
```

# GoupCoordinator处理请求

JoinGroupRequest由GoupCoordinator所在的broker处理，入口方法为handleJoinGroupRequest，下面的源码省略了认证相关，可以看出该方法做了2件事：定义响应回调，调用handleJoinGroup方法

```java
def handleJoinGroupRequest(request: RequestChannel.Request) {
    val joinGroupRequest = request.body[JoinGroupRequest]

    // the callback for sending a join-group response
    def sendResponseCallback(joinResult: JoinGroupResult) {
      val members = joinResult.members map { case (memberId, metadataArray) => (memberId, ByteBuffer.wrap(metadataArray)) }
      def createResponse(requestThrottleMs: Int): AbstractResponse = {
        val responseBody = new JoinGroupResponse(requestThrottleMs, joinResult.error, joinResult.generationId,
          joinResult.subProtocol, joinResult.memberId, joinResult.leaderId, members.asJava)

        trace("Sending join group response %s for correlation id %d to client %s."
          .format(responseBody, request.header.correlationId, request.header.clientId))
        responseBody
      }
      sendResponseMaybeThrottle(request, createResponse)
    }
  // let the coordinator handle join-group
  val protocols = joinGroupRequest.groupProtocols().asScala.map(protocol =>
    (protocol.name, Utils.toArray(protocol.metadata))).toList
  groupCoordinator.handleJoinGroup(
    joinGroupRequest.groupId,
    joinGroupRequest.memberId,
    request.header.clientId,
    request.session.clientAddress.toString,
    joinGroupRequest.rebalanceTimeout,
    joinGroupRequest.sessionTimeout,
    joinGroupRequest.protocolType, // consumer
    protocols,
    sendResponseCallback)
}
```

## handleJoinGroup

handleJoinGroup的核心逻辑是校验和调用doJoinGroup，关于校验这里说2点

1. groupId不能为null，也不能是""
2. sessionTimeoutMs默认必须在6000-300000 即6s-5min之间，当然你也可以修改group.min.session.timeout.ms，group.max.session.timeout.ms来调整区间

```java
def handleJoinGroup(groupId: String,
                      memberId: String,
                      clientId: String,
                      clientHost: String,
                      rebalanceTimeoutMs: Int,
                      sessionTimeoutMs: Int,
                      protocolType: String,
                      protocols: List[(String, Array[Byte])],
                      responseCallback: JoinCallback): Unit = {
    validateGroupStatus(groupId, ApiKeys.JOIN_GROUP).foreach { error =>
      responseCallback(joinError(memberId, error))
      return
    }

    // sessionTimeoutMs默认必须在6000-300000 即6s-5min之间
    if (sessionTimeoutMs < groupConfig.groupMinSessionTimeoutMs ||
      sessionTimeoutMs > groupConfig.groupMaxSessionTimeoutMs) {
      responseCallback(joinError(memberId, Errors.INVALID_SESSION_TIMEOUT))
    } else {
      // 第一个组消费者来这 group不存在并且member id=""，那么会创建group，然后调用doJoinGroup
      // 之后的组消费者之间走doJoinGroup
      groupManager.getGroup(groupId) match {
        case None =>
          // memberId已存在，group为空，只能说明是错误的请求
          if (memberId != JoinGroupRequest.UNKNOWN_MEMBER_ID) {
            responseCallback(joinError(memberId, Errors.UNKNOWN_MEMBER_ID))
          } else {
            // 如果是新的group，新建一个GroupMetadata，并且GroupState为Empty，之后添加到groupManager的groupMetadataCache
            val group = groupManager.addGroup(new GroupMetadata(groupId, initialState = Empty))
            doJoinGroup(group, memberId, clientId, clientHost, rebalanceTimeoutMs, sessionTimeoutMs, protocolType, protocols, responseCallback)
          }

        case Some(group) =>
          doJoinGroup(group, memberId, clientId, clientHost, rebalanceTimeoutMs, sessionTimeoutMs, protocolType, protocols, responseCallback)
      }
    }
}
```

## doJoinGroup

核心方法都在doJoinGroup方法中，此处省略了许多校验的代码，而group的状态此时为Empty，我们直接看该条件分支即可
此时memberId为空，也就是JoinGroupRequest.UNKNOWN_MEMBER_ID，因此这里仅调用addMemberAndRebalance方法
```java
private def doJoinGroup(group: GroupMetadata,
                          memberId: String,
                          clientId: String,
                          clientHost: String,
                          rebalanceTimeoutMs: Int,
                          sessionTimeoutMs: Int,
                          protocolType: String,
                          protocols: List[(String, Array[Byte])],
                          responseCallback: JoinCallback) {
    
    group.currentState match {
      case Dead =>
        responseCallback(joinError(memberId, Errors.UNKNOWN_MEMBER_ID))
      case PreparingRebalance =>
       	// 省略...
      case CompletingRebalance =>
        // 省略...
      case Empty | Stable =>
        if (memberId == JoinGroupRequest.UNKNOWN_MEMBER_ID) {
          // if the member id is unknown, register the member to the group
          addMemberAndRebalance(rebalanceTimeoutMs, sessionTimeoutMs, clientId, clientHost, protocolType,
            protocols, group, responseCallback)
        } else {
          val member = group.get(memberId)
          if (group.isLeader(memberId) || !member.matches(protocols)) {
            updateMemberAndRebalance(group, member, protocols, responseCallback)
          } else {

            responseCallback(JoinGroupResult(
              members = Map.empty,
              memberId = memberId,
              generationId = group.generationId,
              subProtocol = group.protocolOrNull,
              leaderId = group.leaderOrNull,
              error = Errors.NONE))
          }
        }
    }

    if (group.is(PreparingRebalance))
      joinPurgatory.checkAndComplete(GroupKey(group.groupId))

}
```

## addMemberAndRebalance

addMemberAndRebalance首先初始化了memberId，可以看到是clientId拼接一个UUID，然后封装成了一个MemberMetadata对象，这是组成员的元信息对象，之后添加到GroupMetadata中
注意这里的回调函数传给了awaitingJoinCallback变量，rebalance的处理在maybePrepareRebalance中

```java
private def addMemberAndRebalance(rebalanceTimeoutMs: Int,
                                    sessionTimeoutMs: Int,
                                    clientId: String,
                                    clientHost: String,
                                    protocolType: String,
                                    protocols: List[(String, Array[Byte])],
                                    group: GroupMetadata,
                                    callback: JoinCallback) = {
    // memberId = clientId-UUID
    val memberId = clientId + "-" + group.generateMemberIdSuffix
    // 组成员的元信息
    val member = new MemberMetadata(memberId, group.groupId, clientId, clientHost, rebalanceTimeoutMs,
      sessionTimeoutMs, protocolType, protocols)
    member.awaitingJoinCallback = callback
    // update the newMemberAdded flag to indicate that the join group can be further delayed
    if (group.is(PreparingRebalance) && group.generationId == 0)
      group.newMemberAdded = true

    group.add(member)
    maybePrepareRebalance(group)
    member
}
```

add方法很简单，但要关注leaderId的赋值，它表示第一个consumer就是消费者组的leader，也就是第一个consumer为消费者组的leader member
```java
def add(member: MemberMetadata) {
	if (members.isEmpty)
	  this.protocolType = Some(member.protocolType)

	assert(groupId == member.groupId)
	assert(this.protocolType.orNull == member.protocolType)
	assert(supportsProtocols(member.protocols))

	if (leaderId.isEmpty)
	  leaderId = Some(member.memberId)   // 来的第一个就是leader ...
	members.put(member.memberId, member) // memberId为key MemberMetadata为value
}
```

## maybePrepareRebalance

maybePrepareRebalance仅仅是做了一个判断：当前组状态是Stable, CompletingRebalance, Empty其中之一，才可以开始rebalance，满足条件就调用prepareRebalance

prepareRebalance方法在第一个consumer入组时创建一个InitialDelayedJoin，它会等待group.initial.rebalance.delay.ms
这个参数也是为消费者启动时的rebalance优化，因为每启动一个consumer都相当于加入一个组成员，需要进行一次rebalance，这无疑很浪费，这里等待一段时间再开始PreparingRebalance
之后的消费者创建的是DelayedJoin，到期时间就是rebalanceTimeoutMs，即max.poll.interval.ms

此时group的状态由Empty转换为了PreparingRebalance

```java
private def maybePrepareRebalance(group: GroupMetadata) {
    group.inLock {
      if (group.canRebalance)
        prepareRebalance(group)
    }
}
private def prepareRebalance(group: GroupMetadata) {
    // if any members are awaiting sync, cancel their request and have them rejoin
    if (group.is(CompletingRebalance))
      resetAndPropagateAssignmentError(group, Errors.REBALANCE_IN_PROGRESS)

    val delayedRebalance = if (group.is(Empty))
      new InitialDelayedJoin(this,
        joinPurgatory,
        group,
        groupConfig.groupInitialRebalanceDelayMs,
        groupConfig.groupInitialRebalanceDelayMs,
        max(group.rebalanceTimeoutMs - groupConfig.groupInitialRebalanceDelayMs, 0))
    else
      new DelayedJoin(this, group, group.rebalanceTimeoutMs)
    // 从Empty转变为PreparingRebalance
    group.transitionTo(PreparingRebalance)

    val groupKey = GroupKey(group.groupId)
    joinPurgatory.tryCompleteElseWatch(delayedRebalance, Seq(groupKey))
}
```

## 延迟join

joinPurgatory可以理解为一个延迟队列，那么直接看InitialDelayedJoin的onComplete方法，大概意思就是在group.initial.rebalance.delay.ms时间内，它会一直等待消费者入组，超时后后调用父类的onComplete，而InitialDelayedJoin的父类是DelayedJoin，它的onComplete会调用GroupCoordinator的onCompleteJoin方法

```java
private[group] class InitialDelayedJoin(coordinator: GroupCoordinator,
                                        purgatory: DelayedOperationPurgatory[DelayedJoin],
                                        group: GroupMetadata,
                                        configuredRebalanceDelay: Int,
                                        delayMs: Int,
                                        remainingMs: Int) extends DelayedJoin(coordinator, group, delayMs) {

  override def tryComplete(): Boolean = false

  override def onComplete(): Unit = {
    group.inLock  {
      // 是继续等待还是直接结束DelayedJoin
      if (group.newMemberAdded && remainingMs != 0) {
        group.newMemberAdded = false
        val delay = min(configuredRebalanceDelay, remainingMs)
        val remaining = max(remainingMs - delayMs, 0)
        purgatory.tryCompleteElseWatch(new InitialDelayedJoin(coordinator,
          purgatory,
          group,
          configuredRebalanceDelay,
          delay,
          remaining
        ), Seq(GroupKey(group.groupId)))
      } else
        super.onComplete()
    }
  }

}

private[group] class DelayedJoin(coordinator: GroupCoordinator,
                                 group: GroupMetadata,
                                 rebalanceTimeout: Long) extends DelayedOperation(rebalanceTimeout, Some(group.lock)) {
  override def tryComplete(): Boolean = coordinator.tryCompleteJoin(group, forceComplete _)
  override def onExpiration() = coordinator.onExpireJoin()
  override def onComplete() = coordinator.onCompleteJoin(group)
}
```

### 延迟任务完成处理

GroupCoordinator的onCompleteJoin方法源码如下，它的大概意思是响应每一个consumer的请求，这里主要关注JoinGroupResult的第一个参数：leader member的元信息，是leader时才返回currentMemberMetadata(所有组成员的元信息)，也就是说GroupCoordinator只告诉了leader consumer组成员的元信息，原因在下文会揭晓
```java
def onCompleteJoin(group: GroupMetadata) {
    group.inLock {
      // remove any members who haven't joined the group yet
      group.notYetRejoinedMembers.foreach { failedMember =>
        removeHeartbeatForLeavingMember(group, failedMember)
        group.remove(failedMember.memberId)
        // TODO: cut the socket connection to the client
      }

      if (!group.is(Dead)) {
        group.initNextGeneration()
        if (group.is(Empty)) {
          info(s"Group ${group.groupId} with generation ${group.generationId} is now empty " +
            s"(${Topic.GROUP_METADATA_TOPIC_NAME}-${partitionFor(group.groupId)})")

          groupManager.storeGroup(group, Map.empty, error => {
            if (error != Errors.NONE) {
              warn(s"Failed to write empty metadata for group ${group.groupId}: ${error.message}")
            }
          })
        } else {
          info(s"Stabilized group ${group.groupId} generation ${group.generationId} " +
            s"(${Topic.GROUP_METADATA_TOPIC_NAME}-${partitionFor(group.groupId)})")

          for (member <- group.allMemberMetadata) {
            assert(member.awaitingJoinCallback != null)
            val joinResult = JoinGroupResult(
              members = if (group.isLeader(member.memberId)) {
                group.currentMemberMetadata
              } else {
                Map.empty
              },
              memberId = member.memberId,
              generationId = group.generationId,
              subProtocol = group.protocolOrNull,
              leaderId = group.leaderOrNull,
              error = Errors.NONE)

            member.awaitingJoinCallback(joinResult)
            member.awaitingJoinCallback = null
            completeAndScheduleNextHeartbeatExpiration(group, member)
          }
        }
      }
    }
}
```

这里还要关注一下initNextGeneration方法，generationId+1好理解，selectProtocol是因为每个consumer的分配策略可能不一样，selectProtocol用于投票选举一个PartitionAssignor
最后要关注的一点是组状态由PreparingRebalance转变为了CompletingRebalance，也就是所有消费者都进入组内了，等待GroupCoordinator分配分区

```java
def initNextGeneration() = {
    assert(notYetRejoinedMembers == List.empty[MemberMetadata])
    if (members.nonEmpty) {
      generationId += 1
      protocol = Some(selectProtocol)
      transitionTo(CompletingRebalance)
    } else {
      generationId += 1
      protocol = None
      transitionTo(Empty)
    }
    receivedConsumerOffsetCommits = false
    receivedTransactionalOffsetCommits = false
}
```

## 调用回调函数

上面调用的awaitingJoinCallback，其实就是sendResponseCallback，最终的响应返回值JoinGroupResponse如下
```java
def sendResponseCallback(joinResult: JoinGroupResult) {
  val members = joinResult.members map { case (memberId, metadataArray) => (memberId, ByteBuffer.wrap(metadataArray)) }
  def createResponse(requestThrottleMs: Int): AbstractResponse = {
    val responseBody = new JoinGroupResponse(requestThrottleMs, joinResult.error, joinResult.generationId,
      joinResult.subProtocol, joinResult.memberId, joinResult.leaderId, members.asJava)
    responseBody
  }
  sendResponseMaybeThrottle(request, createResponse)
}
```

# 客户端Consumer处理响应

consumer端处理响应的原理：在最开始的sendJoinGroupRequest方法中，除了发送请求，还定义了响应的处理器，我们只关注Errors为NONE的情况，主要做了2件事
1. 初始化了generation，它可以理解为rebalance的次数，版本号
2. 根据server返回的leader memberId判断，如果当前consumer就是leader，调用onJoinLeader，否则调用onJoinFollower

```java
private class JoinGroupResponseHandler extends CoordinatorResponseHandler<JoinGroupResponse, ByteBuffer> {
    @Override
    public void handle(JoinGroupResponse joinResponse, RequestFuture<ByteBuffer> future) {
        Errors error = joinResponse.error();
        if (error == Errors.NONE) {
            sensors.joinLatency.record(response.requestLatencyMs());

            synchronized (AbstractCoordinator.this) {
                if (state != MemberState.REBALANCING) {
                    // if the consumer was woken up before a rebalance completes, we may have already left
                    // the group. In this case, we do not want to continue with the sync group.
                    future.raise(new UnjoinedGroupException());
                } else {
                    AbstractCoordinator.this.generation = new Generation(joinResponse.generationId(),
                            joinResponse.memberId(), joinResponse.groupProtocol());
                    if (joinResponse.isLeader()) {
                        onJoinLeader(joinResponse).chain(future);
                    } else {
                        onJoinFollower().chain(future);
                    }
                }
            }
        } else if (error == xxx) {
        	// 其他错误处理
        }
    }
}
```
onJoinLeader和onJoinFollower都是发送了一个SyncGroupRequest请求，唯一的区别是，onJoinLeader会计算分配方案，传给SyncGroupRequest请求，而onJoinFollower传入的是一个emptyMap

```java
private RequestFuture<ByteBuffer> onJoinLeader(JoinGroupResponse joinResponse) {
    try {
        // perform the leader synchronization and send back the assignment for the group
        Map<String, ByteBuffer> groupAssignment = performAssignment(joinResponse.leaderId(), joinResponse.groupProtocol(),
                joinResponse.members());

        SyncGroupRequest.Builder requestBuilder =
                new SyncGroupRequest.Builder(groupId, generation.generationId, generation.memberId, groupAssignment);
        log.debug("Sending leader SyncGroup to coordinator {}: {}", this.coordinator, requestBuilder);
        return sendSyncGroupRequest(requestBuilder);
    } catch (RuntimeException e) {
        return RequestFuture.failure(e);
    }
}
private RequestFuture<ByteBuffer> onJoinFollower() {
    // send follower's sync group with an empty assignment
    SyncGroupRequest.Builder requestBuilder =
            new SyncGroupRequest.Builder(groupId, generation.generationId, generation.memberId,
                    Collections.<String, ByteBuffer>emptyMap());
    log.debug("Sending follower SyncGroup to coordinator {}: {}", this.coordinator, requestBuilder);
    return sendSyncGroupRequest(requestBuilder);
}
```

# 总结

## 过程分析

消费者发送JoinGroupRequest请求的主要作用是向GoupCoordinator上报的订阅信息，而GoupCoordinator处理的核心逻辑就是addMemberAndRebalance，具体是封装消费者组成员的信息为MemberMetadata，将其添加到GroupMetadata中，
并在此时确定第一个消费者为leader，之后的操作可以理解为将rebalance操作封装成一个DelayedJoin任务，放入延迟队列中，此时消费者组状态由Empty转变为PreparingRebalance，在延迟任务完成时，才返回给客户响应，响应的主要内容主要是leader member的元信息，消费者组的元信息。
客户端(consumer)接收到响应后，主要查看broker返回的leader memberId是不是就是自己，如果是，调用onJoinLeader，它会按分区分配算法计算每个消费者的分区，并再次发送一个SyncGroupRequest请求；相反，如果自己不是leader member，调用的是onJoinFollower，虽然它也发送了SyncGroupRequest请求，但是它的分配方案是空的。

那么为什么要发送一个空的SyncGroupRequest呢？ 这是因为GoupCoordinator只认可leader member的分配方案，其他consumer发送空的SyncGroupRequest只是为了让GoupCoordinator返回leader member的分配方案，即对非leader member的consumer来说，空的SyncGroupRequest不是重点，该请求的响应里包含的分区分配才是重点

## 难点分析

JoinGroupRequest最难理解的地方是对InitialDelayedJoin和DelayedJoin的理解，InitialDelayedJoin在早期的kafka源码并不存在，是后来考虑到项目启动时会触发多次rebalance，因此在kafka的server.properties配置文件中最后一行配置：group.initial.rebalance.delay.ms，设置一定的延迟时间是有意义的，而在改时间超时后，会触发父类DelayedJoin的onComplete，它会调用GroupCoordinator的onCompleteJoin，这里面才会返回给所有consumer JoinGroupRequest请求的响应

## 流程图

![join_group](https://pic.downk.cc/item/5ea294a8c2a9a83be5578794.png)














