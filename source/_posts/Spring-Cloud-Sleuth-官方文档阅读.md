---
title: Spring Cloud Sleuth 官方文档阅读
date: 2018-08-29 11:49:54
categories: Spring Cloud
tags: [Spring Cloud,监控,官方文档阅读]
toc: true
comments: true
---
Spring Cloud Sleuth 版本 2.x

文档地址：[http://cloud.spring.io/spring-cloud-sleuth/2.0.x/single/spring-cloud-sleuth.html](http://cloud.spring.io/spring-cloud-sleuth/2.0.x/single/spring-cloud-sleuth.html)

## 遇到的英文单词

* reveals: 揭示
* network latency: 网络延迟

## 术语
### span

一次请求中每个微服务的处理过程叫一个span，可以理解为一次请求链路中的最小单元，用一个64位的唯一ID标识，span中有若干描述信息，如：ID，产生的时间戳，IP地址，服务名等

如果是入口服务，那么span的id等于trace id
### trace

一次请求经过若干个微服务，汇总每一个服务的span，最终形成一个树状的数据结构
### annotation
用来记录事件信息，表示请求的开始和结束，主要包含以下4个：

1. cs:client send 客户端发起请求，标识一个span的开始
2. sr:server received 服务端接收请求，开始处理请求，此时产生的ts(时间戳，以下统称为ts)减去cs的ts，可以计算出网络传输时间

3. ss:server send 服务端处理结束，开始响应客户端，此时的ts减去sr的ts，就是服务端请求处理时间
4. cr:client received 客户端接收到服务端的响应，此时的ts减去cs的ts，就是一次请求所消耗的时间





