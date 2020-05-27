---
title: kafka消费者-心跳流程全解析
date: 2020-05-15 22:12:28
categories: Kafka Tutorial
tags: [kafka,中间件,消息]
toc: true
comments: true
---


# 背景

Consumer需要和Coordinator保持心跳，来证明当前消费者线程存活，有消费消息的能力，但心跳又不止这么简单，它也是Coordinator下发rebalance请求的通道，同时Consumer利用心跳也可以主动离开消费者组

在Consumer端关于心跳的2个重要类为HeartbeatThread和Heartbeat，前者是心跳线程，后者用于记录心跳信息，Consumer poll时间，通过配置计算下一次心跳时间，计算Consumer两次poll的间隔是否超出maxPollIntervalMs

# 源码

## 心跳状态类

Heartbeat分为2部分，前面是配置参数相关，后面用于记录心跳与拉取过程中的时间点
```java
public final class Heartbeat {
    // 配置
    private final int sessionTimeoutMs;
    private final int heartbeatIntervalMs;
    private final int maxPollIntervalMs;
    private final long retryBackoffMs;

    // 上一次心跳发送时间，volatile用于监控读取
    private volatile long lastHeartbeatSend; // volatile since it is read by metrics
    // 上一次心跳响应接收时间
    private long lastHeartbeatReceive;
    // session重置/初始化的实际
    private long lastSessionReset;
    // consumer 上一次poll的实际
    private long lastPoll;
    private boolean heartbeatFailed;
}
```

下面先介绍它的几个重要方法

timeToNextHeartbeat用于计算距离下一次心跳的剩余时间，比如还有3秒就要做下一次心跳
```java
public long timeToNextHeartbeat(long now) {
    // 距离上一次心跳的时间
    long timeSinceLastHeartbeat = now - Math.max(lastHeartbeatSend, lastSessionReset);

    // 计划下一次心跳的时间
    final long delayToNextHeartbeat;
    if (heartbeatFailed)
        // 失败，就是重试间隔
        delayToNextHeartbeat = retryBackoffMs;
    else
        // 正常的interval时间
        delayToNextHeartbeat = heartbeatIntervalMs;

    // 已经超出了计划心跳时间，需要立即心跳
    if (timeSinceLastHeartbeat > delayToNextHeartbeat)
        return 0;
    else
        // 按计划还有几秒进行下一次心跳
        return delayToNextHeartbeat - timeSinceLastHeartbeat;
}
```

该方法用于Consumer端判断session是否过期，我想表达的意思是Coordinator端也会计算
```java
public boolean sessionTimeoutExpired(long now) {
    // 距离上次发送心跳成功的时间 是否大于sessionTimeout
    return now - Math.max(lastSessionReset, lastHeartbeatReceive) > sessionTimeoutMs;
}
```
该方法从字面意思理解为充值sessionTimeout起始时间点，大家可能觉得不好理解，我在这里直接说何时调用：获取到Coordinator，心跳线程暂停后重新启动(Consumer入组之后)
在这两个时间点重置起始时间点，大家应该好理解的多
```java
public void resetTimeouts(long now) {
    this.lastSessionReset = now;
    this.lastPoll = now;
    this.heartbeatFailed = false;
}
```

# 心跳线程

先看看HeartbeatThread类的全貌，它有enabled和closed两个变量，分别表示心跳线程的启动/暂停 和关闭，close只会在调用Consumer的close方法时触发，用于关闭资源

```java
private class HeartbeatThread extends KafkaThread {
    private boolean enabled = false;
    private boolean closed = false;
    private AtomicReference<RuntimeException> failed = new AtomicReference<>(null);

    // 设置线程名
    private HeartbeatThread() {
        super(HEARTBEAT_THREAD_PREFIX + (groupId.isEmpty() ? "" : " | " + groupId), true);
    }

    // 入组之后
    public void enable() {
        synchronized (AbstractCoordinator.this) {
            log.debug("Enabling heartbeat thread");
            this.enabled = true;
            heartbeat.resetTimeouts(time.milliseconds());
            AbstractCoordinator.this.notify();
        }
    }

    // 入组之前
    public void disable() {
        synchronized (AbstractCoordinator.this) {
            log.debug("Disabling heartbeat thread");
            this.enabled = false;
        }
    }

    // 关闭Consumer时
    public void close() {
        synchronized (AbstractCoordinator.this) {
            this.closed = true;
            AbstractCoordinator.this.notify();
        }
    }
}    
```
下面是重要的run方法

## 心跳核心方法

首先，Consumer是可以以多线程运行的，为了保证线程安全，以AbstractCoordinator.this为锁，除去几个简单的if判断，我们看看最长的if...else判断

1. coordinator未知(为null或无法连接)时，会去查找coordinator，如果失败了，会等待，retryBackoffMs表示重试间隔
2. Consumer端计算sessionTimeout，标记coordinator未知
3. 如果Consumer的两次poll间隔超过了maxPollIntervalMs，发起Leave Group请求
4. Heartbeat#timeToNextHeartbeat返回的时间为0，表示还没到心跳的时间，等待

最后才会发起心跳请求

```java
public void run() {
    try {
        while (true) {
            synchronized (AbstractCoordinator.this) {
                // 已关闭Consumer
                if (closed)
                    return;

                // 离开了消费者组，或者被coordinator踢出了消费者组
                // 设置enabled=false
                if (!enabled) {
                    // 看notify
                    AbstractCoordinator.this.wait();
                    continue;
                }

                // 消费者组未到stable状态
                if (state != MemberState.STABLE) {
                    // the group is not stable (perhaps because we left the group or because the coordinator
                    // kicked us out), so disable heartbeats and wait for the main thread to rejoin.
                    disable();
                    continue;
                }

                client.pollNoWakeup();
                long now = time.milliseconds();

                if (coordinatorUnknown()) {
                    if (findCoordinatorFuture != null || lookupCoordinator().failed())
                        // the immediate future check ensures that we backoff properly in the case that no
                        // brokers are available to connect to.
                        // 重试
                        AbstractCoordinator.this.wait(retryBackoffMs);
                } else if (heartbeat.sessionTimeoutExpired(now)) {
                    // the session timeout has expired without seeing a successful heartbeat, so we should
                    // probably make sure the coordinator is still healthy.
                    // 标记Coordinator未知，也在告诉了其他操作
                    markCoordinatorUnknown();
                } else if (heartbeat.pollTimeoutExpired(now)) {
                    // 两次poll间隔超过了maxPollIntervalMs
                    // the poll timeout has expired, which means that the foreground thread has stalled
                    // in between calls to poll(), so we explicitly leave the group.
                    maybeLeaveGroup();
                } else if (!heartbeat.shouldHeartbeat(now)) {
                    // timeToNextHeartbeat返回的时间还没到
                    // poll again after waiting for the retry backoff in case the heartbeat failed or the
                    // coordinator disconnected
                    AbstractCoordinator.this.wait(retryBackoffMs);
                } else {
                    // 记录心跳发送时间
                    heartbeat.sentHeartbeat(now);
                    // 发送心跳
                    sendHeartbeatRequest().addListener(new RequestFutureListener<Void>() {
                        @Override
                        public void onSuccess(Void value) {
                            synchronized (AbstractCoordinator.this) {
                                // 记录心跳接收时间
                                heartbeat.receiveHeartbeat(time.milliseconds());
                            }
                        }

                        @Override
                        public void onFailure(RuntimeException e) {
                            synchronized (AbstractCoordinator.this) {
                                if (e instanceof RebalanceInProgressException) {
                                    // it is valid to continue heartbeating while the group is rebalancing. This
                                    // ensures that the coordinator keeps the member in the group for as long
                                    // as the duration of the rebalance timeout. If we stop sending heartbeats,
                                    // however, then the session timeout may expire before we can rejoin.
                                    // 在rebalance期间的心跳也算
                                    heartbeat.receiveHeartbeat(time.milliseconds());
                                } else {
                                    heartbeat.failHeartbeat();
                                    // 唤醒，找wait
                                    // wake up the thread if it's sleeping to reschedule the heartbeat
                                    AbstractCoordinator.this.notify();
                                }
                            }
                        }
                    });
                }
            }
        }
    } 
    // 省略各种异常处理
}
```

### 心跳请求

HeartbeatRequest以groupId，generationId，消费者的memberId为参数，向coordinator发起请求
```java
synchronized RequestFuture<Void> sendHeartbeatRequest() {
    HeartbeatRequest.Builder requestBuilder =
            new HeartbeatRequest.Builder(this.groupId, this.generation.generationId, this.generation.memberId);
    return client.send(coordinator, requestBuilder)
            .compose(new HeartbeatResponseHandler());
}
```

# coordinator处理心跳请求

依然是KafkaApis为入口，具体是handleHeartbeatRequest方法，认证相关省略，核心在coordinator的handleHeartbeat方法，为了节省篇幅，以下省略部分源码

当消费者组为stable状态时，主要调用completeAndScheduleNextHeartbeatExpiration，这里我保留了一个异常：UNKNOWN_MEMBER_ID，因为我经常看见它

The coordinator is not aware of this member

```java
def handleHeartbeat(groupId: String,
                      memberId: String,
                      generationId: Int,
                      responseCallback: Errors => Unit) {
    // 处理异常

    groupManager.getGroup(groupId) match {
      case None =>

      case Some(group) => group.inLock {
        group.currentState match {
          case Dead =>

          case Empty =>
           
          case CompletingRebalance =>
           
          case PreparingRebalance =>

          case Stable =>
            if (!group.has(memberId)) {
              responseCallback(Errors.UNKNOWN_MEMBER_ID)
            } else if (generationId != group.generationId) {
              responseCallback(Errors.ILLEGAL_GENERATION)
            } else {
              val member = group.get(memberId)
              completeAndScheduleNextHeartbeatExpiration(group, member)
              responseCallback(Errors.NONE)
            }
        }
      }
    }
}
```
### 延迟队列

该方法首先记录了Consumer心跳时间，计算下次心跳的deadline，如果下次心跳没有按时发送，就会被踢出消费者组，然后以groupId和memberId为key创建了一个延迟任务：DelayedHeartbeat
```java
private def completeAndScheduleNextHeartbeatExpiration(group: GroupMetadata, member: MemberMetadata) {
    // complete current heartbeat expectation
    // 记录消费者组成员心跳时间
    member.latestHeartbeat = time.milliseconds()
    // delay key
    val memberKey = MemberKey(member.groupId, member.memberId)
    heartbeatPurgatory.checkAndComplete(memberKey)

    // reschedule the next heartbeat expiration deadline
    // 计算下次心跳的过期时间
    val newHeartbeatDeadline = member.latestHeartbeat + member.sessionTimeoutMs
    // 放入延迟队列
    val delayedHeartbeat = new DelayedHeartbeat(this, group, member, newHeartbeatDeadline, member.sessionTimeoutMs)
    heartbeatPurgatory.tryCompleteElseWatch(delayedHeartbeat, Seq(memberKey))
}
```

### 延迟任务DelayedHeartbeat

该延迟任务的tryComplete会判断Consumer的下次心跳时间还没到Deadline，就会取消该任务
重要的的是onExpireHeartbeat

```java
private[group] class DelayedHeartbeat(coordinator: GroupCoordinator,
                                      group: GroupMetadata,
                                      member: MemberMetadata,
                                      heartbeatDeadline: Long,
                                      sessionTimeout: Long)
  extends DelayedOperation(sessionTimeout, Some(group.lock)) {


  override def tryComplete(): Boolean = coordinator.tryCompleteHeartbeat(group, member, heartbeatDeadline, forceComplete _)
  // 过期处理，移出消费者组
  override def onExpiration() = coordinator.onExpireHeartbeat(group, member, heartbeatDeadline)
  // 空实现
  override def onComplete() = coordinator.onCompleteHeartbeat()
}
```

该方法首先从消费者组里移除了该Consumer，重新设置leader member，前文也说过了，leader member用于计算分区分配方案，
之后会触发rebalance操作，又回到了[kafka-rebalance之JoinGroup]()一文的步骤

```java
def onExpireHeartbeat(group: GroupMetadata, member: MemberMetadata, heartbeatDeadline: Long) {
    group.inLock {
      if (!shouldKeepMemberAlive(member, heartbeatDeadline)) {
        info(s"Member ${member.memberId} in group ${group.groupId} has failed, removing it from the group")
        removeMemberAndUpdateGroup(group, member)
      }
    }
}
private def removeMemberAndUpdateGroup(group: GroupMetadata, member: MemberMetadata) {
    group.remove(member.memberId)
    group.currentState match {
      case Dead | Empty =>
      case Stable | CompletingRebalance => maybePrepareRebalance(group)
      case PreparingRebalance => joinPurgatory.checkAndComplete(GroupKey(group.groupId))
    }
}
```

## 回到Consumer端的心跳线程

下面是Consumer对心跳请求的响应处理，这里主要是判断各类异常
```java
private class HeartbeatResponseHandler extends CoordinatorResponseHandler<HeartbeatResponse, Void> {
    @Override
    public void handle(HeartbeatResponse heartbeatResponse, RequestFuture<Void> future) {
        sensors.heartbeatLatency.record(response.requestLatencyMs());
        Errors error = heartbeatResponse.error();
        if (error == Errors.NONE) {
            log.debug("Received successful Heartbeat response");
            future.complete(null);
        } else if (error == Errors.COORDINATOR_NOT_AVAILABLE
                || error == Errors.NOT_COORDINATOR) {
            log.info("Attempt to heartbeat failed since coordinator {} is either not started or not valid.",
                    coordinator());
            markCoordinatorUnknown();
            future.raise(error);
        } else if (error == Errors.REBALANCE_IN_PROGRESS) {
            log.info("Attempt to heartbeat failed since group is rebalancing");
            requestRejoin();
            future.raise(Errors.REBALANCE_IN_PROGRESS);
        } else if (error == Errors.ILLEGAL_GENERATION) {
            log.info("Attempt to heartbeat failed since generation {} is not current", generation.generationId);
            resetGeneration();
            future.raise(Errors.ILLEGAL_GENERATION);
        } else if (error == Errors.UNKNOWN_MEMBER_ID) {
            log.info("Attempt to heartbeat failed for since member id {} is not valid.", generation.memberId);
            resetGeneration();
            future.raise(Errors.UNKNOWN_MEMBER_ID);
        } else if (error == Errors.GROUP_AUTHORIZATION_FAILED) {
            future.raise(new GroupAuthorizationException(groupId));
        } else {
            future.raise(new KafkaException("Unexpected error in heartbeat response: " + error.message()));
        }
    }
}
```

但这并没有结束, 在callback里，更新Heartbeat状态类的lastHeartbeatReceive变量，如果失败了则判断是否是在rebalance期间的心跳失败，如果是也算心跳成功，因为这属于被动失败，否则记为心跳失败，唤醒阻塞在AbstractCoordinator.this的线程
```java
sendHeartbeatRequest().addListener(new RequestFutureListener<Void>() {
    @Override
    public void onSuccess(Void value) {
        synchronized (AbstractCoordinator.this) {
            // 记录心跳接收时间
            heartbeat.receiveHeartbeat(time.milliseconds());
        }
    }

    @Override
    public void onFailure(RuntimeException e) {
        synchronized (AbstractCoordinator.this) {
            if (e instanceof RebalanceInProgressException) {
                // it is valid to continue heartbeating while the group is rebalancing. This
                // ensures that the coordinator keeps the member in the group for as long
                // as the duration of the rebalance timeout. If we stop sending heartbeats,
                // however, then the session timeout may expire before we can rejoin.
                // 在rebalance期间的心跳也算
                heartbeat.receiveHeartbeat(time.milliseconds());
            } else {
                heartbeat.failHeartbeat();
                // 唤醒，找wait
                // wake up the thread if it's sleeping to reschedule the heartbeat
                AbstractCoordinator.this.notify();
            }
        }
    }
});
```

















