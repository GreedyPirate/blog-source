---
title: Spring boot实践之自定义starter
date: 2019-01-16 19:12:21
categories: Spring Boot
tags:
	- Spring Boot
toc: true
comments: true
---

>为何要自定义starter，使用场景是什么，又该如何去自定义呢？本文围绕这几个方面展示自定义starter的过程

# 使用场景
在Spring-Boot实践系列文章中，对日常开发中的许多功能做了统一封装，那么在分布式开发的组织架构下，开发组内个人单独使用是没有意义的，应该将其封装成一个SDK，发布到maven私服，供大家使用，分享精神先放一边，这样做的好处是统一标准，提升开发效率，同时又需要投入部分人力维护这个项目，不断更新和修复。

# 准备工作

## 项目结构
项目是一个多模块结构，不同的公共模块封装各自的公用功能，例如可以把统一的接口请求，返回，日志等放到web模块，对监控有要求的可以抽取一个监控模块，依赖了中间件时，对该中间的公用配置抽取一个模块。

项目名称可参考spring-cloud组件，例如叫base-starter-web, base-start-logger等

# 封装starter



TopicPartition