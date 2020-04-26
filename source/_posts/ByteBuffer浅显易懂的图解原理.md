---
title: ByteBuffer浅显易懂的图解原理
date: 2019-12-01 12:21:14
categories: 技术积累
tags: [nio]
toc: true
comments: true
---

本文希望通过图解的形式帮助更多新手理解ByteBuffer的使用

# ByteBuffer的作用

我们知道在java中读写文件都是通过流操作的，那么想象一下要读取一个大文件，每次都从流中一个字节一个字节的读取，效率有多低下，缓存是必须存在的，就好比搬一堆沙子，总不能一粒一粒的搬，得要用个小推车来搬运

那么在java nio中，都是通过channel读写，ByteBuffer作为缓存

读：从channel中读取到ByteBuffer中，程序再从ByteBuffer中读取
写：程序先写到ByteBuffer中，然后ByteBuffer写入到channel

# ByteBuffer

ByteBuffer没有大家想象的那么复杂，即使它的方法、变量很多，大家就把它理解为一个普通的字节数组(byte[])就好了。
为数不多的难点是里面的4个变量：mark，position，limit，capacity

第一次看见就要理解4个变量确实比较多，精简一下，capacity就是数组的初始化长度，它是不变的，mark暂时不用看
剩下的position，limit是读写的关键

## 初始化

首先来说下ByteBuffer的初始化，既然它本质是一个byte[], 那么我们指定长度就可以了，常用的初始化方式如下

```java
ByteBuffer buffer = ByteBuffer.allocate(8);
```

或者自己初始化好一个byte[]，交给ByteBuffer用
```java
byte[] arr = new byte[8];
ByteBuffer buffer = ByteBuffer.wrap(arr);
```

以上代码都是创建了一个8字节长度的缓冲, 此时内部的变量状态如下
![](https://ae01.alicdn.com/kf/Hc6075479cfe345cd8f692defb2ff0dben.png)

为了方便观察，定义一个打印方法

```java
public void printBuffer(ByteBuffer buffer) {
    System.out.println("position :" + buffer.position() + ",limit :" + buffer.limit() + ",capacity" + buffer.capacity());
}
```

## 写入

接下来我们写入一个int类型，还记得int类型在内存中占几个字节吗，4个

```java
ByteBuffer buffer = ByteBuffer.allocate(8);
buffer.putInt(1);

printBuffer(buffer);
```

此时的内部状态如下: position :4,limit :8,capacity8

![](https://ae01.alicdn.com/kf/Ha16d3139a96049af8a7239ee1e234effJ.png)

### limit

在写入时，position的作用就是一个指针，写入一个byte就加1，就和我们常用的List的size作用一样
同时我们也应该知道，写入"1"之后，ByteBuffer只剩4个字节了，如果我们再写入2个数字，就出超出ByteBuffer的容量限制(limit)

```java
buffer.putInt(2);
buffer.putInt(3);
```

报错：

```java
java.nio.BufferOverflowException
	at java.nio.Buffer.nextPutIndex(Buffer.java:527)
	at java.nio.HeapByteBuffer.putInt(HeapByteBuffer.java:372)
```
由此引出，limit表示在写入时position的上限


## 读取

首先考虑一个问题，我们读取的起始位置，结束位置是什么。承接上文，在写入数字1之后，position :4,limit :8
我们要做的是从0开始读取，读到4为止，ByteBuffer为我们提供了一个很方便的方法：flip，读之前要翻转，很巧妙的比喻

调用flip之后，position置为0， limit置为4，刚好是读取的区间. 而flip的源码也是按照这个思路来的
```java
public final Buffer flip() {
    limit = position;
    position = 0;
    mark = -1;
    return this;
}
```
flip之后的状态如下图：

![flip之后的状态](https://ae01.alicdn.com/kf/Ha80a7ce86b1a4411975447fd515df539L.png)

### 读取ByteBuffer

```java
buffer.flip();
while (buffer.hasRemaining()) {
    int anInt = buffer.getInt();
    System.out.println("anInt = " + anInt);
}
```
操作读取的代码如上，每读取一个int，position+4, limit就是读取的上限，hasRemaining方法就是为了防止越界


## 小结
相信通过上面的图文，大家对position和limit这两个变量不在陌生了，也能理解为什么有flip方法

## api

ByteBuffer还有很多的api，其实都无须解释它们的作用，源码都是在操纵那4个变量

### clear

clear方法有点类似重置，position，limit，mark都回归到了初始状态
```java
public final Buffer clear() {
    position = 0;
    limit = capacity;
    mark = -1;
    return this;
}
```

### rewind

听名字很难理解，感觉很抽象，但是源码及其简单，重置position=0
```java
public final Buffer rewind() {
    position = 0;
    mark = -1;
    return this;
}
```

### slice

就是字面意思的截取，假设长度为8的ByteBuffer，写了4个字节，目前position为4

```java
public ByteBuffer slice() {
    return new HeapByteBuffer(hb,
                                    -1,
                                    0,
                                    this.remaining(), // limit - position，即要截取的字节数
                                    this.remaining(),
                                    this.position() + offset);
}

protected HeapByteBuffer(byte[] buf,
                         int mark, int pos, int lim, int cap,
                         int off){
    super(mark, pos, lim, cap, buf, off);
}
```

### mark与reset
摘取kafka MemoryRecords中的一段代码, 可以看到为了重置position，先进行了mark
```java
public int writeFullyTo(GatheringByteChannel channel) throws IOException {
    buffer.mark();
    int written = 0;
    while (written < sizeInBytes())
        written += channel.write(buffer);
    buffer.reset();
    return written;
}
```

## HeapByteBuffer与MappedByteBuffer

HeapByteBuffer与MappedByteBuffer是ByteBuffer的两个子类，二者通俗的区别在于：

HeapByteBuffer创建在jvm heap堆上，创建，回收都由jvm控制，适合需要频繁创建的情形，不适合需要大缓存的场景，毕竟jvm内存有限
MappedByteBuffer使用的是系统内存，也就是堆外内存，创建与回收对于jvm来说都是比较重的操作，适合申请之后不会立即回收，长时间重复利用的场景

最后我想说的是，学习ByteBuffer最快速最简单的方式就是看源码，网上很多文档对api的表述方式反而让人看得云里雾里


