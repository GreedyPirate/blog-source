---
title: Guava实现大数据量sql分段批量插入
date: 2019-01-18 10:56:28
categories: 技术积累
tags:
	- [积累]
toc: true
comments: true
---

>最近做一个数据拉取的需求，由于有上万的数据量，想到分段批量插入数据库，经同事推荐，Guava有好的工具类，特此记录并分享给大家

## 代码实现

一开始在网上搜索，基本用到的都是List接口的sublist方法，第一版自己实现了一遍，功能没问题，但很啰嗦，下面介绍guava的partition方法

```java
    /**
     * 分段批量插入
     * @param ticketLists
     */
    private void batchSegmentInsertList(List<TicketList> ticketLists) {
        if(CollectionUtils.isEmpty(ticketLists)){
            return;
        }

        List<List<TicketList>> lists = Lists.partition(ticketLists,1000);
        lists.forEach(tickets ->{
            ticketListDao.batchInsert(tickets);
        });
    }
```

上述代码是笔者实际开发中的代码，首先需要自己在mybatis中实现批量插入，然后使用Lists.partition方法对原有的集合做分段，1000代表分段大小

## 源码分析

进入partition源码，省略中间多余过程，最终发现一个静态类

```java
static class Partition<T> extends AbstractList<List<T>>
```

其实现了AbstractList，有趣的是泛型参数是List<T>，结合ArrayList类的实现，该泛型参数代表集合的一个元素，说明Partition本身是一个集合，里面的元素是一个个的小集合，结合我们要实现的功能，说明这就是我们要的对大集合切割之后的每一个小集合，而下面的get方法也佐证了我们的猜想

```java
    @Override public List<T> get(int index) {
      checkElementIndex(index, size());
      int start = index * size;
      int end = Math.min(start + size, list.size());
      return list.subList(start, end);
    }
```
同时还可以看出Partition底层还是用的sublist方法，不过最后我想说的还是那句老话，不要重复造轮子，遇到这种大批量sql分段插入，你肯定不是第一个人遇到，多用大牛写好的轮子，同时去看看他们是如何实现的，解放生产力的同时，也提升自己的技术

