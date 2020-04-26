---
title: kafka源码环境搭建
date: 2019-11-18 22:36:18
categories: Kafka Tutorial
tags: [kafka,中间件,消息]
toc: true
comments: true
---

# 源码下载
从[kafka官网](http://kafka.apache.org/downloads)下载源码压缩包，以2.0.1版本为例，选择-src结尾的压缩包

![kafka 2.0.1](https://ae01.alicdn.com/kf/H210e4ddd7a6e484eb1187793e785a0c1w.png
)

# 依赖环境
kafka采用gradle构建，根据kafka的git提交记录，采用4.10.3版本构建，如果本地有别的gradle版本，可以尝试用[sdkman](https://sdkman.io/)这款工具来管理，一个命令即可切换版本
kafka使用scala语言开发，需要安装2.12版本的scala，同样的sdkman也可以快速安装
![提交日志](https://ae01.alicdn.com/kf/H2d59f09399574120b822f6d7c71729090.png)

# 修改配置及日志文件
修改config/server.properties文件相关参数
```
listeners=PLAINTEXT://localhost:9092
# 设置自己的目录路径
log.dirs=/Users/admin/app/other-kafka/kafka-2.0.1-src/logs
# 如果本地有别的kafka集群，设置zk的chroot
zookeeper.connect=localhost:2181/cluster_201
```

将config目录下的log4j文件复制到src/main/resources目录下，resources目录自己新建即可
![移动log4j文件](https://ae01.alicdn.com/kf/H97695e5cf22d4509b660f785e19ac77dE.png)


# 导入IDEA
解压后导入IDEA中，IDEA会自动开始构建，等待IDEA下载依赖

# 相关问题

如果在编译的过程中出现以下错误
```
You can't map a property that does not exist: propertyName=testClassesDir
```
调整build.gradle中的依赖版本为以下版本
![调整版本](https://ae01.alicdn.com/kf/H86c22cdb3be04652b06db45350080181y.png)


```
Cause: org.jetbrains.plugins.gradle.tooling.util.ModuleComponentIdentifierIm Lorg/gradle/api/artifacts/ModuleIdentifier
```
如果报以上错误,请检查IDEA是否配置了正确的gradle 4.10.3版本的home路径(尤其是IDEA 2018版本)

遇到下载超时的jar包，可以到maven中央仓库下载jar，通过命名手动安装

举例：
```
mvn install:install-file -DgroupId=org.scala-lang -DartifactId=scala-reflect -Dversion=2.10.6 -Dpackaging=jar -Dfile=/Users/admin/scala/scala-reflect-2.10.6.jar
```
然后继续等待IDEA编译，等待期间多祈祷能下载下来...

# 启动
首先启动Zookeeper，成功构建之后，调整kafka的启动参数
注：建议将jvm堆内存设置小一点，kafka默认是1G，本机机器完全没有必要 -Xmx512m -Xms512m 
![启动参数](https://ae01.alicdn.com/kf/H43d9e3cc1fad40fdbdba654de897c442J.png
)

启动成功之后控制台如下
![](https://ae01.alicdn.com/kf/H0dccf862d1364893ac367dce5856bc72l.png
)
注：日志不打印问题见我的另一篇博客-[kafka本地启动后不打印日志问题]()

# 测试
向topic中发送消息之后，logs目录产生了日志文件
![lgos目录](https://ae01.alicdn.com/kf/Ha8852196a2bc47c993dd79367104e549v.png
)

以上步骤已在MacOS和Windows系统上验证通过，遇到问题的同学请在评论区写下问题，我会尽量解答





