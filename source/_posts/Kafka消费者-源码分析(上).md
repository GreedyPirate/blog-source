---
title: Kafka消费者-源码分析(上)
date: 2020-03-10 15:38:34
categories: Kafka Tutorial
tags: [kafka,中间件,消息]
toc: true
comments: true
---

>本文从消费者拉取消息开始分析消费流程，但kafka并不是单纯的在poll方法中拉取消息，鉴于消费者组的存在，以及Rebalance动作，使整个消费流程的复杂度直线上升，因此需要比生产者花费更多的章节去讲解

# 准备

为了方便大家阅读源码，这里先对源码中经常出现的部分做一个解释，提示大家的阅读效率

## 名词解释

elapsedTime：已用时间，在一个带有超时时间的方法中，该变量用于记录部分已完成操作的已用时间，比如超时时间60s，其中访问数据库操作用了10s，那么elapsedTime就是10s

## 发送请求的一般模式

consumer向broker发送请求的一般模式是：
1. sendXxxRequest表示发生一个请求，通常返回一个RequestFuture
2. RequestFuture有几个方法，isDone表示请求结束，即获取到了broker端的响应，相反的表示无响应；succeeded表示请求成功，failed表示失败；可以对future注册一个Listener，执行成功和失败的回调
3. Listen通常是一个xxxResponseHandler，常见的代码如下：
```java
RequestFuture future = client.send(coordinator, requestBuilder, joinGroupTimeoutMs)
    .compose(new xxxResponseHandler());

if (future.succeeded()) {
  // 成功
} else {
    // 失败
    if (是可重试异常)
        continue;
    else if (!future.isRetriable()) 
        throw exception;
    // 重试的back off
    time.sleep(retryBackoffMs);
}
```

## consumer订阅

consumer订阅topic有3中方式：指定topic集合，指定topic正则，手动指定分区。前2中称之为AutoAssigned，因为是coordinator自动分配给消费者的，这三种方式分别对应下面3个api
```java
public void subscribe(Collection<String> topics);

public void subscribe(Pattern pattern);

public void assign(Collection<TopicPartition> partitions)
```
本文只讨论第一种，这也是我们开发中最常用的订阅方式

# poll方法

首先说下2个参数：timeoutMs和includeMetadataInTimeout
1. timeoutMs：整个poll调用的超时时间，第一次poll里面向broker发送了4个请求，该参数建议设置大于3s，
2. includeMetadataInTimeout：针对上面的超时时间，是否应该包含获取元数据的时间(向broker请求)

```java
public ConsumerRecords<K, V> poll(final Duration timeout) {
    return poll(timeout.toMillis(), true);
}
private ConsumerRecords<K, V> poll(final long timeoutMs, final boolean includeMetadataInTimeout) {
    acquireAndEnsureOpen();
    try {
        if (timeoutMs < 0) throw new IllegalArgumentException("Timeout must not be negative");

        if (this.subscriptions.hasNoSubscriptionOrUserAssignment()) {
            throw new IllegalStateException("Consumer is not subscribed to any topics or assigned any partitions");
        }

        // poll for new data until the timeout expires
        // 记录消耗的时间，防止超时
        long elapsedTime = 0L;
        do {

            client.maybeTriggerWakeup();

            final long metadataEnd;
            // 新版本的poll是true，就是说是否要把更新Metadata的时间，也算在poll的超时时间内
            if (includeMetadataInTimeout) {
                final long metadataStart = time.milliseconds(); // SystemTime
                /**
                 * 初始化Coordinator，初次rebalance，初始化每个分区的last consumed position
                 * 什么情况下返回false：
                 * 1. coordinator unknown
                 * 2. rebalance失败(长时间拿不到响应结果，发生不可重试的异常)
                 * 3. 获取不到分区的last consumed position (fetch offset)
                 */
                if (!updateAssignmentMetadataIfNeeded(remainingTimeAtLeastZero(timeoutMs, elapsedTime))) {
                    // coordinator不可用或者...
                    return ConsumerRecords.empty();
                }
                metadataEnd = time.milliseconds();
                elapsedTime += metadataEnd - metadataStart; // += (metadataEnd - metadataStart)
            } else {
                // 老版本的超时时间？
                while (!updateAssignmentMetadataIfNeeded(Long.MAX_VALUE)) {
                    log.warn("Still waiting for metadata");
                }
                metadataEnd = time.milliseconds();
            }

            final Map<TopicPartition, List<ConsumerRecord<K, V>>> records = pollForFetches(remainingTimeAtLeastZero(timeoutMs, elapsedTime));

            if (!records.isEmpty()) {
                /**
                 * 立即开始下一轮请求，和用户处理消息并行
                 */
                if (fetcher.sendFetches() > 0 || client.hasPendingRequests()) {
                    client.pollNoWakeup();
                }
                // 拦截器处理后的消息才交给用户
                return this.interceptors.onConsume(new ConsumerRecords<>(records));
            }
            // records为空
            final long fetchEnd = time.milliseconds();
            // 总的消耗时间 fetchEnd - metadataEnd是真正用来发fetch请求的所消耗的时间
            elapsedTime += fetchEnd - metadataEnd;

        } while (elapsedTime < timeoutMs);

        return ConsumerRecords.empty();
    } finally {
        release();
    }
}
```

不用关注太多poll方法细节，仅关注超时时间内的循环，简单理解为3步：第一步为updateAssignmentMetadataIfNeeded，然后是pollForFetches，最后将拦截器处理后的消息返回给用户，大致的流程如下：

![poll方法](https://pic.downk.cc/item/5ea58054c2a9a83be509e638.png)

## updateAssignmentMetadataIfNeeded

updateAssignmentMetadataIfNeeded方法十分复杂，逻辑也很长，我这里直接说它的逻辑，让读者心里有个底。该方法主要做了3件事：
1. 初始化Coordinator，主要是节点信息(id,ip,port)
2. 初次rebalance，consumer启动时进入消费者组
3. 初始化每个分区的last consumed position，表示该消费者组上次消费到哪个位移了，Coordinator会缓存每个group最后消费的位移
4. 如果第3步获取不到，则根据auto.offset.reset获取

其次它的返回值是一个boolean，它在以下情况返回false：
1. coordinator unknown
2. rebalance失败(长时间拿不到响应结果，发生不可重试的异常)
3. 获取不到分区的last consumed position (fetch offset)

这里再科普一些知识点，Coordinator，即消费者组协调器，每一个broker启动时都初始化了一个GroupCoordinator对象，它负责消费者组的生命周期管理，以及消费者组，消费者组成员的元数据管理
而每个分区的last consumed position是指消费者每次poll，准确的说应该是发起fetch请求向broker拉取数据的时候，都要传递一个fetchOffset参数，表示从哪里开始拉消息
但也有一些特殊情况，比如消费者组过期被删除了，新消费者组第一次拉取时，此时coordinator没有该消费者组的信息，没法返回该消费者组上次消费的分区位移，那么auto.offset.reset就起作用了，coordinator会根据该配置返回相应的offset

```java
boolean updateAssignmentMetadataIfNeeded(final long timeoutMs) {
    final long startMs = time.milliseconds();
    // 返回false表示获取coordinator位置，初始化rebalance失败 (正则订阅暂不考虑)
    if (!coordinator.poll(timeoutMs)) {
        return false;
    }

    // 返回true,更新要fetch的Position
    return updateFetchPositions(remainingTimeAtLeastZero(timeoutMs, time.milliseconds() - startMs));
}
```

updateAssignmentMetadataIfNeeded分为2部分：coordinator.poll和updateFetchPositions，前者是rebalance的核心步骤，需要重点关注

## coordinator#poll

该方法位于ConsumerCoordinator类中，虽然源码看上去也不少(已删除部分)，但在消费者组已稳定(stable)的情况下，执行到下面这行代码就会返回了：
```java
pollHeartbeat(currentTime)
```
pollHeartbeat会尝试查看是否到了心跳时间，来发起心跳，同时还记录了一个lastPoll变量，它与maxPollIntervalMs参数息息相关，如果两次poll的间隔超出了maxPollIntervalMs，心跳线程会主动发起LeaveGroup请求，让consumer主动离开消费者组，触发一次rebalance，这也是大部分人看到的rebalance异常，因为业务逻辑处理的太慢，导致rebalance的原因

```java
public boolean poll(final long timeoutMs) {
    final long startTime = time.milliseconds();
    long currentTime = startTime;
    long elapsed = 0L;

    // 先执行队列里所有的OffsetCommitCompletion
    invokeCompletedOffsetCommitCallbacks();

    /**
     * 是否手动制定了TP,不用看else
     */
    if (subscriptions.partitionsAutoAssigned()) {
        // Always update the heartbeat last poll time so that the heartbeat thread does not leave the
        // group proactively due to application inactivity even if (say) the coordinator cannot be found.
        // 查看距离下一次心跳时间是否为0，唤醒心跳线程，发送心跳
        // 同时记录lastPoll，根据maxPollIntervalMs判断是否需要发起LeaveGroup请求(主动rebalance)
        /**
         * 如果不看下面coordinatorUnknown和rejoinNeededOrPending，正常步骤到这里就结束了
         */
        pollHeartbeat(currentTime);

        // coordinator节点为null，或不可用
        // 第一次poll时为null
        if (coordinatorUnknown()) {

            if (!ensureCoordinatorReady(remainingTimeAtLeastZero(timeoutMs, elapsed))) {
                // 直接返回了false
                return false;
            }
            currentTime = time.milliseconds();
            // elapsed 就是已用时间
            elapsed = currentTime - startTime;
        }

        if (rejoinNeededOrPending()) {
            // due to a race condition between the initial metadata fetch and the initial rebalance,
            // we need to ensure that the metadata is fresh before joining initially. This ensures
            // that we have matched the pattern against the cluster's topics at least once before joining.
            if (subscriptions.hasPatternSubscription()) { 
               // 一般不用正则订阅，省略代码...
            }

            // 直接看这，里面通过JoinGroup和SyncGroup进行rebalance，来保证达到STABLE状态
            if (!ensureActiveGroup(remainingTimeAtLeastZero(timeoutMs, elapsed))) {
                return false;
            }

            currentTime = time.milliseconds();
        }
    } else {
        // ... standalone方式 省略
    }

    // autoCommit时尝试提交
    maybeAutoCommitOffsetsAsync(currentTime);
    return true;
}
```
上面说的是消费者组已稳定的情况，那么在消费者启动时，相当于消费者组中新加入了一个成员，必然会触发一次rebalance，我称之为初始rebalance，此时consumer并不知道coordinator是哪台broker(coordinatorUnknown)，就会发起一次FindCoordinator请求，来初始化AbstractCoordinator.coordinator，此处的源码分析在[kafka消费者-获取Coordinator]()一文

在获取到Coordinator之后，进入下一个if，rejoinNeededOrPending方法初始化为true，接下里的ensureActiveGroup就是初始rebalance的核心步骤

## 开始rebalance

ensureActiveGroup源码如下：
```java
boolean ensureActiveGroup(long timeoutMs, long startMs) {
    // 前面已经获取到了Coordinator，这里确认一下
    if (!ensureCoordinatorReady(timeoutMs)) {
        return false;
    }

    startHeartbeatThreadIfNeeded(); // 启动心跳线程

    // join开始时间，和剩余的超时时间
    long joinStartMs = time.milliseconds();
    long joinTimeoutMs = remainingTimeAtLeastZero(timeoutMs, joinStartMs - startMs);
    return joinGroupIfNeeded(joinTimeoutMs, joinStartMs);
}
```
ensureActiveGroup会启动心跳线程，但并不会开始心跳，因为enabled参数默认为false,并利用线程的等待唤醒机制，让心跳线程在wait处等待
```java
if (!enabled) {
    AbstractCoordinator.this.wait();
    continue;
}
```
rebalance核心逻辑都在joinGroupIfNeeded方法中

### joinGroupIfNeeded

这里我们关注下onJoinPrepare，它会回调ConsumerRebalanceListener的onPartitionsRevoked方法，而之后就是典型的[客户端发送请求模式](#发送请求的一般模式)，只需要关注initiateJoinGroup方法即可

```java
boolean joinGroupIfNeeded(final long timeoutMs, final long startTimeMs) {
    long elapsedTime = 0L;

    while (rejoinNeededOrPending()) { // 第一次为true
        if (!ensureCoordinatorReady(remainingTimeAtLeastZero(timeoutMs, elapsedTime))) {
            return false;
        }
        elapsedTime = time.milliseconds() - startTimeMs;
        if (needsJoinPrepare) { // 第一次为true，generation=Generation.NO_GENERATION
            // 主要是触发ConsumerRebalanceListener，如果自动提交为true，尝试提交
            onJoinPrepare(generation.generationId, generation.memberId);
            needsJoinPrepare = false;
        }

        // 第一次加入组 future是JoinGroup请求返回的分配方案
        // initiateJoinGroup里面会把rejoinNeeded置为false，如果本次rebalance成功了，就会推出当前的while循环
        final RequestFuture<ByteBuffer> future = initiateJoinGroup();
        client.poll(future, remainingTimeAtLeastZero(timeoutMs, elapsedTime));
        // 无论请求成功还是失败，都还没拿到，说明超时了啊
        if (!future.isDone()) {
            // we ran out of time
            return false;
        }

        // 请求成功
        if (future.succeeded()) {
            // Duplicate the buffer in case `onJoinComplete` does not complete and needs to be retried.
            ByteBuffer memberAssignment = future.value().duplicate();

            // 初始化分区消费(拉取)状态，更新缓存数据
            // 执行回调 PartitionAssignor#onAssignment, ConsumerRebalanceListener#onPartitionsAssigned
            onJoinComplete(generation.generationId, generation.memberId, generation.protocol, memberAssignment);

            // We reset the join group future only after the completion callback returns. This ensures
            // that if the callback is woken up, we will retry it on the next joinGroupIfNeeded.
            resetJoinGroupFuture(); // joinFuture重置为null
            needsJoinPrepare = true;
        } else {
            resetJoinGroupFuture();
            final RuntimeException exception = future.exception();

            // 这三种异常会再次尝试rebalance
            if (exception instanceof UnknownMemberIdException ||
                    exception instanceof RebalanceInProgressException ||
                    exception instanceof IllegalGenerationException)
                continue;
            else if (!future.isRetriable()) // 其他的抛异常
                throw exception;
            // 重试的back off
            time.sleep(retryBackoffMs);
        }

        // 计算已用时间, 正常情况下进不到if，elapsedTime也只是为了计算多次失败的耗时
        if (rejoinNeededOrPending()) {
            elapsedTime = time.milliseconds() - startTimeMs;
        }
    }
    // 不需要rebalance，直接返回true
    return true;
}
```

### initiateJoinGroup

initiateJoinGroup中的sendJoinGroupRequest同样是[客户端发送请求模式](#发送请求的一般模式)的一种，可以看到在rebalance成功后，做了以下3件事
1. MemberState置为stable
2. rejoinNeeded置为false，它是退出外层循环的标志位
3. 启动心跳线程

而JoinGroupRequest的详细细节，请参考我的另外2篇文章[kafka-rebalance之JoinGroup]()和[kafka-rebalance之SyncGroup]()，里面完整的讲述了rebalance细节
```java
private synchronized RequestFuture<ByteBuffer> initiateJoinGroup() {
    if (joinFuture == null) {
        // 先暂停了心跳线程，其实本来就还没启动
        disableHeartbeatThread();

        state = MemberState.REBALANCING;
        joinFuture = sendJoinGroupRequest();
        joinFuture.addListener(new RequestFutureListener<ByteBuffer>() {
            @Override
            public void onSuccess(ByteBuffer value) {
                // handle join completion in the callback so that the callback will be invoked
                // even if the consumer is woken up before finishing the rebalance
                synchronized (AbstractCoordinator.this) {
                    log.info("Successfully joined group with generation {}", generation.generationId);
                    // 跟新2个很重要的
                    state = MemberState.STABLE; // 消费者Stable
                    rejoinNeeded = false; // 退出外层循环的标志位

                    // 前面停止的心跳线程也重新启动了
                    if (heartbeatThread != null)
                        heartbeatThread.enable();
                }
            }

            @Override
            public void onFailure(RuntimeException e) {
                // we handle failures below after the request finishes. if the join completes
                // after having been woken up, the exception is ignored and we will rejoin
                synchronized (AbstractCoordinator.this) {
                    state = MemberState.UNJOINED;
                }
            }
        });
    }
    return joinFuture;
}
```

该方法结束后，方法会层层返回到updateAssignmentMetadataIfNeeded，此时coordinator.poll已结束，接下来是updateFetchPositions方法
```java
boolean updateAssignmentMetadataIfNeeded(final long timeoutMs) {
    final long startMs = time.milliseconds();
    // 返回false表示获取coordinator位置，初始化rebalance失败 (正则订阅暂不考虑)
    if (!coordinator.poll(timeoutMs)) {
        return false;
    }

    // 返回true,更新要fetch的Position
    return updateFetchPositions(remainingTimeAtLeastZero(timeoutMs, time.milliseconds() - startMs));
}
```

## 预备知识

TopicPartitionState表示consumer在消费过程中的状态，它会在每一个拉取后更新，里面的参数都比较简单，不再细说
```java
TopicPartitionState {
        private Long position; // last consumed position
        private Long highWatermark; // the high watermark from last fetch
        private Long logStartOffset; // the log start offset
        private Long lastStableOffset;
        private boolean paused;  // whether this partition has been paused by the user
        private OffsetResetStrategy resetStrategy;  // the strategy to use if the offset needs resetting
        private Long nextAllowedRetryTimeMs;
}
```
## updateFetchPositions

这里首先判断了所有订阅的分区是否有last consumed position，它用于下一次消息拉取，consumer要从什么位置开始拉，初始化时为null，那么就会向coordinator发起OFFSET_FETCH请求，用于初始化TopicPartitionState的position，
但还有coordinator没有消费者组上次消费位置元数据的情况，比如消费者组过期，被管理员删除，第一次建立时，那么该如何初始化position呢？ 

答案是auto.offset.reset，根据重置offset策略，向分区leader所在的broker，注意不是coordinator，发送LIST_OFFSETS请求来初始化position，该请求的详细处理过程请参考[Kafka消费者-ListOffsets请求]()

```java
private boolean updateFetchPositions(final long timeoutMs) {
    // 是否所有订阅的TP都有记录的消费位移等状态，第一次poll肯定都是是null
    // 具体线索：ConsumerCoordinator#onJoinComplete->assignFromSubscribed->partitionToStateMap->new TopicPartitionState()
    cachedSubscriptionHashAllFetchPositions = subscriptions.hasAllFetchPositions();
    if (cachedSubscriptionHashAllFetchPositions) return true;

    // 向coordinator OFFSET_FETCH请求，初始化fetch offset
    // 这里指的是Coordinator会保存上一次提交的位移，而consumer拿到之后会作为fetch请求的fetch offset参数
    if (!coordinator.refreshCommittedOffsetsIfNeeded(timeoutMs)) return false;

    // 有offset Rest策略的，根据reset策略重置position，比如earliest或者latest
    subscriptions.resetMissingPositions();

    // 为没有position(last consumed)的分区发送LIST_OFFSETS请求
    // 这主要是group不存在的情况，消费者组过期，被删除，第一次建立
    fetcher.resetOffsetsIfNeeded();

    return true;
}
```

# 上半部分总结

本文主要从大家平时见到的poll方法开始分析，并在一开始就普及了源码中的难点，poll方法从流程图上看十分简单，主要分为：updateAssignmentMetadataIfNeeded，pollForFetches，返回消息给用户这三步，本文主要分析第一步就已花费了很多篇幅，由于内容过长，将一些核心逻辑放在单独的文章中分析:[获取Coordinator]()， [rebalance之JoinGroup]()， [rebalance之SyncGroup]()， [ListOffsets请求]()。

这些请求都是在consumer第一次拉取消息之前的准备工作，首先consumer要知道Coordinator的信息，并保证与之连接通畅。之后便开始了初次入组的rebalance，其中又可细分为入组，等待其他组员(非必需)，选举leader consumer，然后leader consumer根据分区策略制定分配方案，所有组员再次发送SyncGroup请求，由Coordinator来返回leader consumer制定的分配方案。

在有了分配方案之后，并不能立即开始拉取消息，因为consumer不知道每一个分区从哪里开始拉取，就要通过OffsetFetch请求向Coordinator获取fetchOffset，在有了fetchOffset之后理应可以拉取了，但又有2个特殊情况：当前是新消费者组，或是消费者组过期了(相关参数为offsets.retention.minutes)，此时Coordinator不知道consumer上一次消费到哪了，那么auto.offset.reset参数就起作用了，根据是它来获取最早或是最新的位移，到此，准备工作才算完成。










