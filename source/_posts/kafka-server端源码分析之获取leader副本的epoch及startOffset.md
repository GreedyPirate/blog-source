---
title: kafka server端源码分析之获取leader副本的epoch及startOffset
date: 2020-03-07 16:09:11
categories: Kafka Tutorial
tags: [kafka,中间件,消息]
toc: true
comments: true
---

### 前言

本文主要讲解follower副本发起用于同步的fetch请求之前，获取了leader副本的leader epoch及其startOffset，关于leader epoch的介绍，可以看看前面的LeaderAndIsr请求一文

# follower副本同步

在follower副本每次发起fetch请求之前，都会调用maybeTruncate方法，根据leader副本的leader epoch及startOffset和follower自己的作比较，判断是否需要截断日志(truncate)