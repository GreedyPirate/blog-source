---
title: kafka缓冲池(BufferPool)原理剖析
date: 2020-05-2 15:46:35
categories: Kafka Tutorial
tags: [kafka,中间件,消息]
toc: true
comments: true
---

# 前言

本文主要聊聊kafka生产者端的缓冲池设计，不了解ByteBuffer的同学需要先看看我之前的文章[ByteBuffer浅显易懂的图解原理]()

# 生产者缓冲池的应用

kafka采用延迟批量发送的方式来提高了吞吐量，采用ByteBuffer来保存消息，你可以把ByteBuffer理解为一块内存，kafka内部实现了一套缓冲池技术，来缓存ByteBuffer，实现类为BufferPool。
缓冲池用的还是池化思想，尽可能的复用资源，避免频繁创建与销毁，增加申请空间的响应速度，同时也减轻了GC的负担

# BufferPool的基本思想

我们都知道Producer有2个关于缓冲池的参数：buffer.memory默认32MB，batch.size默认16KB(16384Byte)，前者表示缓冲池的总大小，后者表示一个缓冲区的大小

为了方便讲解，现在假设缓冲区总大小buffer.memory为1000Byte，每一个缓冲区为100Byte，对应BufferPool构造方法的totalMemory=nonPooledAvailableMemory=1000, poolableSize=100

```java
// 省略部分代码
public class BufferPool {
    // buffer.memory
    private final long totalMemory;
    // batch.size
    private final int poolableSize;

    // 访问nonPooledAvailableMemory，free，waiters时都要加锁
    private final ReentrantLock lock;

    // batch的内存块 队列
    private final Deque<ByteBuffer> free;

    // 等待队列
    private final Deque<Condition> waiters;

    // 未使用(未分配过)的大小
    private long nonPooledAvailableMemory;

    public BufferPool(long memory, int poolableSize) {
        this.poolableSize = poolableSize;
        this.lock = new ReentrantLock();
        this.free = new ArrayDeque<>();
        this.waiters = new ArrayDeque<>();
        this.totalMemory = memory;
        this.nonPooledAvailableMemory = memory;
    }
}
```
ArrayDeque的初始化大小是16，此时BufferPool的状态如下图

![](https://pic.downk.cc/item/5ebbf05bc2a9a83be567e2e5.png)

下面我按照时间顺序，分3种情形来说明分配空间的过程，条件为：申请的空间大小 = poolableSize(batch.size)，正常消息的发送流程都符合此条件，也有内容太大的消息需要多分配，但并不多见

### 情形1

总的剩余空间大小，也就是可用于分配的空间大小由以下公式计算，其实只要理解了nonPooledAvailableMemory的含义就很好理解，它表示从未被使用过的空间，不包含分配后被回收的空间

总的剩余空间 = 未被使用过的空间大小(nonPooledAvailableMemory) + 空闲队列大小(free.size * poolableSize)

下图是第一个申请分配的请求， 此时总的剩余空间 > size(100)，会直接创建一个ByteBuffer缓冲区返回给外部调用；在程序使用结束后，会调用deallocate释放缓冲区，并将缓冲区添加到free队列，实现资源的回收

![](https://pic.downk.cc/item/5ebbf70fc2a9a83be56ed77f.png)

### 情形2

下图是第二个申请分配的请求，因为情形1中第一个个ByteBuffer缓冲区使用完后，归还到了free队列里，此时根据队列FIFO的性质，直接取队头的元素，返回给程序即可

![](https://pic.downk.cc/item/5ebbf725c2a9a83be56eec67.png)

### 情形3

情形3就比较复杂了，它是在总的剩余空间不够分配size(100Byte)时发生的，此时会创建一个ReentrantLock上的竞态条件(Condition)，并放入到等待队列的最末尾，进行超时等待。此处做以下几点说明

1. 为什么放在最末尾？ 因为保证了先来先得的公平竞争性，防止饥饿产生
2. 既然有超时等待，什么时候唤醒？在调用deallocate方法时，正常情况下是Sender线程发送完消息后调用，该方法中会释放ByteBuffer，并唤醒等待队列的第一个静态条件(Condition)
3. 等待超时会发生什么？ allocate会抛出TimeoutException

注意这里的超时时间并不完全等于max.block.ms配置，该配置包含了从broker端请求元数据的时间

![](https://pic.downk.cc/item/5ebbf74ec2a9a83be56f194b.png)

以上就是BufferPool的基本原理，其中nonPooledAvailableMemory变量表示totalMemory里从未被分配的空间大小，它是从被被使用过，干净的空间。下面来看看源码部分，核心是allocate方法

# 源码分析

下面的分析依然以buffer.memory默认32MB，batch.size默认16KB为准

## 分配缓冲区(ByteBuffer)

首先看看方法的参数与返回值，参数分为：要申请的大小(size)，缓冲池满时的最大等待时间(maxTimeToBlockMs); 返回值则是分配好的缓冲区(ByteBuffer)
源码如下，代码较长，但逻辑比较连贯，不便于分段解析，请根据前面的图文耐心看注释

```java
public ByteBuffer allocate(int size, long maxTimeToBlockMs) throws InterruptedException {
    if (size > this.totalMemory)
        throw new IllegalArgumentException("Attempt to allocate " + size
                                           + " bytes, but there is a hard limit of "
                                           + this.totalMemory
                                           + " on memory allocations.");

    ByteBuffer buffer = null;
    this.lock.lock();
    try {
        // check if we have a free buffer of the right size pooled
        // 如果申请的是一个标准的ByteBuffer，并且free队列里面有缓存的，直接取出并返回
        if (size == poolableSize && !this.free.isEmpty())
            return this.free.pollFirst();

        // now check if the request is immediately satisfiable with the
        // memory on hand or if we need to block
        // 空闲队列的总字节数大小 = 队列长度 * 单个缓冲区大小
        int freeListSize = freeSize() * this.poolableSize;
        // 未被使用的空间大小 + 空闲队列大小 >= 要申请的size
        if (this.nonPooledAvailableMemory + freeListSize >= size) {
            // we have enough unallocated or pooled memory to immediately
            // satisfy the request, but need to allocate the buffer
            // 确保现有的空间足够分配
            freeUp(size);
            // 并没有影响nonPooledAvailableMemory的语义
            this.nonPooledAvailableMemory -= size;
        }
        else {
            // 内存不够用了，就要阻塞
            // we are out of memory and will have to block

            // 记录已分配的内存，size有可能需要多次分配
            int accumulated = 0;
            // 获取竞态条件
            Condition moreMemory = this.lock.newCondition();
            try {
                // 阻塞时间
                long remainingTimeToBlockNs = TimeUnit.MILLISECONDS.toNanos(maxTimeToBlockMs);
                // 竞争队列，添加到了最末尾，说明这是公平锁，先来的请求先获取内存
                this.waiters.addLast(moreMemory);
                // loop over and over until we have a buffer or have reserved
                // enough memory to allocate one
                while (accumulated < size) {
                    // 本次获取缓冲区的开始时间
                    long startWaitNs = time.nanoseconds();
                    long timeNs;
                    // 等待是否超时
                    boolean waitingTimeElapsed;
                    try {
                        // 等待时，会在此停止运行
                        waitingTimeElapsed = !moreMemory.await(remainingTimeToBlockNs, TimeUnit.NANOSECONDS);
                    } finally {
                        long endWaitNs = time.nanoseconds();
                        // 记录阻塞时，等待的时间
                        timeNs = Math.max(0L, endWaitNs - startWaitNs);
                        // 收集指标
                        this.waitTime.record(timeNs, time.milliseconds());
                    }

                    // 等待超时
                    if (waitingTimeElapsed) {
                        throw new TimeoutException("Failed to allocate memory within the configured max blocking time " + maxTimeToBlockMs + " ms.");
                    }

                    // 未超时的情况，计算剩余时间
                    remainingTimeToBlockNs -= timeNs;

                    // 分两种情况，
                    // 1. size=batch.size, 从free里获取，结束
                    // 2. size > 或 < batch.size 都会判断下一次循环条件，来确定是否退出
                    // check if we can satisfy this request from the free list,
                    // otherwise allocate memory
                    if (accumulated == 0 && size == this.poolableSize && !this.free.isEmpty()) {
                        // just grab a buffer from the free list
                        // 从free队列里取出
                        buffer = this.free.pollFirst();
                        accumulated = size;
                    } else {
                        // size > poolableSize 或者 size < poolableSize
                        // we'll need to allocate memory, but we may only get
                        // part of what we need on this iteration
                        // 确保有足够的内存
                        freeUp(size - accumulated);
                        // accumulated=0，要分配的size为100，现有的nonPooledAvailableMemory只有80
                        // 就只能先分配80，然后在下一轮循环里等待
                        int got = (int) Math.min(size - accumulated, this.nonPooledAvailableMemory);
                        this.nonPooledAvailableMemory -= got;
                        //
                        accumulated += got;
                    }
                }
                // 这里很有意思, 这在下面的finally里表示分配过程中 是否有分配失败的内存
                // Don't reclaim memory on throwable since nothing was thrown
                accumulated = 0;
            } finally {
                // When this loop was not able to successfully terminate don't loose available memory
                // 分配成功就是0，分配失败就不是0了，这里是在回收分配失败的内存
                this.nonPooledAvailableMemory += accumulated;
                // 从队列里移除竞态条件
                this.waiters.remove(moreMemory);
            }
        }
    } finally {
        // signal any additional waiters if there is more memory left
        // over for them
        try {
            // 如果有空闲的空间，并且还有在等待的内存分配申请，就唤醒等待队列(第一个)
            if (!(this.nonPooledAvailableMemory == 0 && this.free.isEmpty()) && !this.waiters.isEmpty())
                this.waiters.peekFirst().signal();
        } finally {
            // Another finally... otherwise find bugs complains
            // 释放锁
            lock.unlock();
        }
    }

    // 分配并返回
    if (buffer == null)
        return safeAllocateByteBuffer(size);
    else
        return buffer;
}
```

下面是freeUp方法的源码，它的本意是确保现有的空间足够用于分配
```java
private void freeUp(int size) {
    // 如果nonPooledAvailableMemory < size，就看free队列是不是空的，
    // 如果不是空的，就free里的size加到nonPooledAvailableMemory > size为止
    while (this.nonPooledAvailableMemory < size && !this.free.isEmpty())
        this.nonPooledAvailableMemory += this.free.pollLast().capacity(); // poolLast取出并删除
}
```

最后看一下释放方法，它的重点是将释放的buffer放入了free队列，同时唤醒了队首节点，这是经典的多线程同步写法

```java
public void deallocate(ByteBuffer buffer, int size) {
    lock.lock();
    try {
        // size = batch.size 直接释放空间，并添加到空闲队列
        if (size == this.poolableSize && size == buffer.capacity()) {
            buffer.clear();
            this.free.add(buffer);
        } else {
            // buffer 没有被释放
            this.nonPooledAvailableMemory += size;
        }
        Condition moreMem = this.waiters.peekFirst();
        // 唤醒第一个待分配的
        if (moreMem != null)
            moreMem.signal();
    } finally {
        lock.unlock();
    }
}
```

# 总结

本文主要分析了生产者buffer.memory，batch.size参数背后的实现，总的来说是池化的思想，配合多线程的等待/唤醒机制来实现同步，相信大家对这两种技术有了一个感性的认识








