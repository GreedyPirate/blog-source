---
title: kafka-rebalance之SyncGroup
date: 2020-04-24 15:33:37
categories: Kafka Tutorial
tags: [kafka,中间件,消息]
toc: true
comments: true
---

# 前言
衔接上文[kafka-rebalance之JoinGroup](), 我们已经知道在JoinGroup请求的响应中，leader consumer会计算分区分配方案，并发起SyncGroup请求，本文讲解SyncGroup请求的处理过程

同样的思路，我们还是从请求发起看起

# 发送请求

```java
private RequestFuture<ByteBuffer> onJoinLeader(JoinGroupResponse joinResponse) {
    try {
        // perform the leader synchronization and send back the assignment for the group
        Map<String, ByteBuffer> groupAssignment = performAssignment(joinResponse.leaderId(), joinResponse.groupProtocol(),
                joinResponse.members());

        SyncGroupRequest.Builder requestBuilder =
                new SyncGroupRequest.Builder(groupId, generation.generationId, generation.memberId, groupAssignment);
        return sendSyncGroupRequest(requestBuilder);
    } catch (RuntimeException e) {
        return RequestFuture.failure(e);
    }
}
```
该请求的参数多用ByteBuffer表示，这里也是我通过源码反推出来的部分结构，不敢保证100%正确，但也相差不远，核心参数是每个consumer的分配方案

```json
{
	"groupId": "test-group",
	"generationId": 1,
	"memberId": "client-A625830A-86C6-4E10-809F-296297328FCA",
	"groupAssignment": [
		{
			"memberId": "client-FB86B927-DA68-4271-BB0C-2AA69879325D",
			"topic_partitions": {
				"topic": "test",
				"partitions": [0,1]
			},
			"userData": null
		}
		// 其他member ... 
	]
	
}
```

# GoupCoordinator处理请求

broker端处理入口同样的三步走：定义回调函数，认证(已省略)，处理，那么核心逻辑在GroupCoordinator的handleSyncGroup方法

```java
def handleSyncGroupRequest(request: RequestChannel.Request) {
    val syncGroupRequest = request.body[SyncGroupRequest]

    def sendResponseCallback(memberState: Array[Byte], error: Errors) {
      sendResponseMaybeThrottle(request, requestThrottleMs =>
        new SyncGroupResponse(requestThrottleMs, error, ByteBuffer.wrap(memberState)))
    }

	// 省略认证代码
	groupCoordinator.handleSyncGroup(
	  syncGroupRequest.groupId,
	  syncGroupRequest.generationId,
	  syncGroupRequest.memberId,
	  syncGroupRequest.groupAssignment().asScala.mapValues(Utils.toArray),
	  sendResponseCallback
	)
}
```

## handleSyncGroup

同样省略了大部分异常处理逻辑，可以看到直接调用了doSyncGroup
```java
def handleSyncGroup(groupId: String,
                      generation: Int,
                      memberId: String,
                      groupAssignment: Map[String, Array[Byte]],
                      responseCallback: SyncCallback): Unit = {

    groupManager.getGroup(groupId) match {
      case Some(group) => doSyncGroup(group, generation, memberId, groupAssignment, responseCallback)
    }
}
```

## doSyncGroup

通过[kafka-rebalance之JoinGroup]()我们一直当前消费者组处于CompletingRebalance状态，这里的分支我们只用看它即可

```java
private def doSyncGroup(group: GroupMetadata,
                          generationId: Int,
                          memberId: String,
                          groupAssignment: Map[String, Array[Byte]],
                          responseCallback: SyncCallback) {
    group.inLock {
        group.currentState match {
          case Empty | Dead => // 省略 ...
          case PreparingRebalance => // 省略 ...
          case CompletingRebalance =>
          	// 同样的暂存回调，在延迟任务完成时触发
            group.get(memberId).awaitingSyncCallback = responseCallback

            // 只处理leader consumer
            if (group.isLeader(memberId)) {
              // fill any missing members with an empty assignment
              val missing = group.allMembers -- groupAssignment.keySet
              val assignment = groupAssignment ++ missing.map(_ -> Array.empty[Byte]).toMap

              // 持久化保存到__consumer_offset
              groupManager.storeGroup(group, assignment, (error: Errors) => {
                group.inLock {
                  // another member may have joined the group while we were awaiting this callback,
                  // so we must ensure we are still in the CompletingRebalance state and the same generation
                  // when it gets invoked. if we have transitioned to another state, then do nothing
                  if (group.is(CompletingRebalance) && generationId == group.generationId) {
                    if (error != Errors.NONE) {
                      resetAndPropagateAssignmentError(group, error)
                      maybePrepareRebalance(group)
                    } else {
                      // 正常的逻辑
                      setAndPropagateAssignment(group, assignment)
                      group.transitionTo(Stable)
                    }
                  }
                }
              })
            }

          case Stable =>
            // if the group is stable, we just return the current assignment
            val memberMetadata = group.get(memberId)
            responseCallback(memberMetadata.assignment, Errors.NONE)
            completeAndScheduleNextHeartbeatExpiration(group, group.get(memberId))
        }
      }
    }
}
```

可以看到该方法只处理leader consumer的SyncGroupRequest，并将元数据存入了__consumer_offsets中，之后关键的处理在setAndPropagateAssignment方法，处理完成后将组状态转换为Stable

## setAndPropagateAssignment

该方法首先校验了状态，然后将每个member的分配方案保存到了allMemberMetadata，之后调用了propagateAssignment
```java
private def setAndPropagateAssignment(group: GroupMetadata, assignment: Map[String, Array[Byte]]) {
	assert(group.is(CompletingRebalance))
	group.allMemberMetadata.foreach(member => member.assignment = assignment(member.memberId))
	propagateAssignment(group, Errors.NONE)
}
```

## 响应客户端

该方法的处理逻辑也十分简单：调用回调响应客户端，开始执行定时任务监控member的心跳

```java
private def propagateAssignment(group: GroupMetadata, error: Errors) {
    for (member <- group.allMemberMetadata) {
      if (member.awaitingSyncCallback != null) {
        member.awaitingSyncCallback(member.assignment, error)
        member.awaitingSyncCallback = null

        completeAndScheduleNextHeartbeatExpiration(group, member)
      }
    }
}
```

响应的回调函数比较简单，可以回到最上面看handleSyncGroupRequest方法，就是把member的分配方案返回，而completeAndScheduleNextHeartbeatExpiration的作用是记录一次成功的心跳，并将下一次心跳的延迟任务放入Purgatory，同样的我们把它理解为延迟队列即可

# 总结





















