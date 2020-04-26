---
title: kafka本地启动后不打印日志问题
date: 2020-01-25 21:36:46
categories: Kafka Tutorial
tags: [kafka,中间件,消息]
toc: true
comments: true
---

>2020年的春节新冠状病毒肆虐，只能宅在家里(天赋异禀)，闲来无事再次打开kafka项目阅读源码，但是从一开始就有个小问题困扰着我，
kafka本地启动后不打印日志，虽然能运行，但是心里总是很难受，今日下定决心解决之

在[kafka源码环境搭建](https://greedypirate.github.io/2019/10/29/kafka%E6%BA%90%E7%A0%81%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA/)一文中，启动kafka之后，控制台如下
![](https://ae01.alicdn.com/kf/H0dccf862d1364893ac367dce5856bc72l.png
)
和我们用命令启动不一样，完全没有日志产生

虽然我通过百度，Stack Overflow等多个地方查找，想要解决这个问题，
但解决方案还是在图片中的链接里:[http://www.slf4j.org/codes.html#StaticLoggerBinder](http://www.slf4j.org/codes.html#StaticLoggerBinder)

根据下面的提示，只需要替换这三个jar的任意一个即可
```
Placing one (and only one) of slf4j-nop.jar slf4j-simple.jar, slf4j-log4j12.jar, slf4j-jdk14.jar
```

剩下的就简单了，通过本人的摸索(没有学习过gradle)，kafka的依赖管理在如下位置
![](https://ae01.alicdn.com/kf/He7f6efd2a0c64e20b1eaffc19ff905e8e.png)

log4j_bugfix是我新加的一个依赖，分别在dependencies.gradle的versions和libs数组的最后一行添加
```
versions += [
	...
	log4j_bugfix: "1.7.30"
]
libs += [
	... 
	log4jBugFix:"org.slf4j:slf4j-log4j12:$versions.log4j_bugfix"
]
```

完成以上步骤后，找到根目录下的build.gradle文件，大约来543行，添加刚才新增的依赖
![](https://ae01.alicdn.com/kf/Hba525c213131420088f4420f7bce7a90z.png)


最后在启动参数中加入log的目录配置:-Dkafka.logs.dir=
![](https://ae01.alicdn.com/kf/Hbf9d0dee5d5246c69b2febb82458e13ap.png)

启动kafka，日志开始正常打印，大功告成
```
[2020-01-25 21:34:51,570] INFO Registered kafka:type=kafka.Log4jController MBean (kafka.utils.Log4jControllerRegistration$)
[2020-01-25 21:34:52,824] INFO starting (kafka.server.KafkaServer)
[2020-01-25 21:34:52,826] INFO Connecting to zookeeper on localhost:2181/cluster_201 (kafka.server.KafkaServer)
[2020-01-25 21:34:52,920] INFO [ZooKeeperClient] Initializing a new session to localhost:2181. (kafka.zookeeper.ZooKeeperClient)
[2020-01-25 21:34:52,971] INFO Client environment:zookeeper.version=3.4.13-2d71af4dbe22557fda74f9a9b4309b15a7487f03, built on 06/29/2018 00:39 GMT (org.apache.zookeeper.ZooKeeper)
[2020-01-25 21:34:52,971] INFO Client environment:host.name=localhost (org.apache.zookeeper.ZooKeeper)
[2020-01-25 21:34:52,971] INFO Client environment:java.version=1.8.0_161 (org.apache.zookeeper.ZooKeeper)
[2020-01-25 21:34:52,971] INFO Client environment:java.vendor=Oracle Corporation (org.apache.zookeeper.ZooKeeper)
....
```

## 2020年02月19日更新

本人IDEA升级后，突然又不打印日志了，但是错误提示变为
```
No appenders could be found for logger (kafka.utils.Log4jControllerRegistration$)
```
首先保证log4j在classpath下，也就是在core/src/main/resources/log4j.properties下
然后找到File->Project Structure中的Global library，点击加号添加scala(2.11.12) sdk，Ctrl+A全选所有模块，确定后重启项目

如果还是不行，找到IDEA最右侧的gradle按钮，点击刷新按钮即可






