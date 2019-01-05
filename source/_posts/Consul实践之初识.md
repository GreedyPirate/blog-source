---
title: Consul实践之初识
date: 2018-11-16 19:13:32
categories: 微服务注册中心
tags: [consul,注册中心]
toc: true
comments: true
---

>Consul是HashiCorp公司推出的开源工具，提供服务发现，健康检查，K/V存储，多数据中心，ACL等功能，同时也是Service Mesh解决方案。

# 与其它注册中心的对比
参考博主纯洁的微笑的文章：[springcloud(十三)：注册中心 Consul 使用详解](http://www.ityouknow.com/springcloud/2018/07/20/spring-cloud-consul.html)

# 术语
为什么有这一小节呢，本人刚接触到Consul时，对代理(agent)，client, server三者之间的关系没有搞清楚，以下对这几个概念做梳理，帮助新人快速理解

* 代理：从Consul官网下载的zip包中，解压后只有一个启动文件，启动之后会运行一个Consul服务，你可以把这个服务理解为agent。agent分为两种，server和client，在启动agent的时候，可以通过参数指定是server还是client
* agent集群：所有server和client节点形成的集群
* server：server主要维护应用服务信息，响应查询，参与一致性选举，与别的数据中心交换信息。
* server集群：server集群中的节点包括一个leader和多个follower，通过raft算法选举leader，保证一致性。server官方推荐的个数是一个数据中心有3或5个节点，一是为了高可用，二是奇数个方便选举，同时要保证server节点的存活数不低于（N/2）+1个，如3个server组成的集群，必须保证2个server存活，5个保证3个存活，否则server集群处于不可用状态
* client：agent的另一种，主要用于转发RPC请求，本身是无状态的，运行在后台维护gossip协议池
* datacenter：笔者对数据中心的理解还十分简陋，大概是机房的意思吧，将Consul部署到多个dc中，增加了可用性
* gossip协议：翻译为流言协议，取自人类社会中的谣言传播，在Consul中用于管理agent和广播消息，在一个agent集群中新加入一个节点，或离开一个节点时，会通过gossip协议告诉集群中的所有节点，Consul的这些gossip协议功能是通过自家的另一个开源产品Serf实现的，这里对Serf要有个印象
* LAN gossip与WAN gossip：分别代表一个数据中心中agent集群之间的的gossip协议，和多个数据中心之间的gossip协议


# Consul的架构

官网给出了两个数据中心的[俯视图](https://www.consul.io/docs/internals/architecture.html)，为了方便理解，笔者自己画了一个单数据中心的架构图，帮助大家理解

![Consul架构](https://ws2.sinaimg.cn/large/006tNbRwly1fxlh3wc72cj30mh0glwf0.jpg)

## 解读

1. 用3台服务器，部署3个server节点，形成server集群
2. 每一台应用服务器上部署一个client节点，同时可以部署应用服务，可以是一个，可以是多个，视运维部署规则而定，一般生产环境每台服务器只部署一个应用
3. 应用服务注册到本机的Consul client，通过它与server集群交互

# Consul原理

## 结束语

在写这一系列的文章时，笔者并不想一开始就实操，而是挈领提纲地介绍了Consul的原理，希望读者有个模糊的印象，在接下来的一步步操作中加深理解。
