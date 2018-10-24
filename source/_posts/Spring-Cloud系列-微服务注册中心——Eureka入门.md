---
title: 'Spring Cloud系列:微服务注册中心——Eureka入门'
date: 2018-09-10 14:43:10
categories: Spring Cloud
tags: [Spring Cloud,微服务]
toc: true
comments: true
---

# 注册中心

作为一个没有经验的开发人员(捂脸），在了解Eureka之前，我更想让读者带着问题去思考

1. 什么是微服务注册中心，微服务为什么需要注册中心
2. 注册中心都实现了哪些功能
3. 开源的注册中心有哪些，为什么要选Eureka，优缺点有哪些
4. 生产环境中的注册中心如何部署

这些想法都是我敲完代码想要思考的，搭建一个注册中心几分钟的事，实在没什么技术含量，感觉收获到的东西太少，需要沉下来多思考



以下是Eureka server单机伪集群的配置方式：

1.首先Spring boot项目，加入依赖
```
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

2.配置文件如下
```
server:
  port: 8000
spring:
  profiles: master
  application:
    name: eureka-master
eureka:
  instance:
    hostname: peer1
  client:
    serviceUrl:
      defaultZone: http://peer2:8001/eureka/
    fetch-registry: false
    register-with-eureka: false
  server:
    eviction-interval-timer-in-ms: 10000 # 每10s就去清理无效的实例
    enable-self-preservation: false

---
server:
  port: 8001
spring:
  profiles: slave
  application:
    name: eureka-slave
eureka:
  instance:
    hostname: peer2
  client:
    serviceUrl:
      defaultZone: http://peer1:8000/eureka/
    fetch-registry: false
    register-with-eureka: false
  server:
    eviction-interval-timer-in-ms: 10000 # 每10s就去清理无效的实例
    enable-self-preservation: false
```
虽然复制粘贴到你的项目是件易事，不过有几个配置点笔者还是想要详细说一下：
1. hostname配置了peer1、peer2，这是在模拟集群环境，需要读者自己在hosts文件中添加映射，
`127.0.0.1     localhost peer1 peer2`.如果你的内存够大，也可以用两台虚拟机
2. 对于Eureka来说，无效的实例是通过定时任务去清除的，默认是60s，这里我设置为了10s
3. IDEA中通过一个项目启动Eureka集群,通过spring.profiles.active区分配置
![点击Edit Configurations](https://ws1.sinaimg.cn/large/006tNbRwly1fw8r9iqvrgj30u0044my9.jpg)
![点击加号新增Spring boot](https://ws3.sinaimg.cn/large/006tNbRwly1fw8rfa3p3ej31kw0asn18.jpg)





