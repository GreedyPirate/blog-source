---
title: ElasticSearch7.2 X-pack安全认证
date: 2019-08-12 15:03:24
categories: ELK Stack
tags: [ElasticSearch, X-pack, kibana]
toc: true
comments: true
---

# 前言
ElasticSearch于6.8及7.1版本开始提供免费的x-pack, 并已默认集成，只需通过简单的配置即可开启。 [官方链接](https://www.elastic.co/cn/blog/security-for-elasticsearch-is-now-free)，主要包含以下特性:
1. TLS 功能，可对通信进行加密
2. 文件和原生 Realm，可用于创建和管理用户
3. 基于角色的访问控制，可用于控制用户对集群 API 和索引的访问权限；通过针对 Kibana Spaces 的安全功能，还可允许在 Kibana 中实现多租户

安全是个很大的话题,本章只针对用户权限方面做初步尝试，旨在为kibana添加用户认证。
通常我们的ES节点部署在内网当中，不对外暴露9200等端口，kibana是一款非常强大的可视化工具(由衷赞叹)，devTools使开发人员可以方便的操作集群，索引，
但是这个页面非开发人员也是可以看到的，因此第一步就是先要屏蔽非es使用方，提供一个登录认证功能。

## ES配置
首先在*elasticsearch.yml*中加入以下配置
```
xpack.security.enabled: true
xpack.license.self_generated.type: basic
xpack.security.transport.ssl.enabled: true
```

然后在bin目录执行以下命令
```
./elasticsearch-setup-passwords interactive
```
为各个组件设置密码

```
➜  bin ./elasticsearch-setup-passwords interactive
future versions of Elasticsearch will require Java 11; your Java version from [/Library/Java/JavaVirtualMachines/jdk1.8.0_161.jdk/Contents/Home/jre] does not meet this requirement
Initiating the setup of passwords for reserved users elastic,apm_system,kibana,logstash_system,beats_system,remote_monitoring_user.
You will be prompted to enter passwords as the process progresses.
Please confirm that you would like to continue [y/N]y


Enter password for [elastic]:
passwords must be at least [6] characters long
Try again.
Enter password for [elastic]:
Reenter password for [elastic]:
Enter password for [apm_system]:
Reenter password for [apm_system]:
Enter password for [kibana]:
Reenter password for [kibana]:
Enter password for [logstash_system]:
Reenter password for [logstash_system]:
Enter password for [beats_system]:
Reenter password for [beats_system]:
Enter password for [remote_monitoring_user]:
Reenter password for [remote_monitoring_user]:
Changed password for user [apm_system]
Changed password for user [kibana]
Changed password for user [logstash_system]
Changed password for user [beats_system]
Changed password for user [remote_monitoring_user]
Changed password for user [elastic]
```

ES的设置结束，接下来是kibana，切换至kibana/bin目录

## kibana

### 删除原有
先查看是否已经设置过，新安装的同学可以跳过
```
./kibana-keystore list
```
如果发现已安装，但忘记了密码，可以删除原有的配置
```
./kibana-keystore remove elasticsearch.username
./kibana-keystore remove elasticsearch.password
```

### 添加
最后添加ES密码，虽然可以在kibana.yml中配置，但是不安全，要考虑该目录的Linux权限，增加了复杂性，这也是官方提供keystore工具的原因之一
```
./kibana-keystore add elasticsearch.username
./kibana-keystore add elasticsearch.password
```

### 验证
打开kibana，输入账号密码，账号为elastic
效果图如下
![登录](https://ae01.alicdn.com/kf/H8359357d94e24907b332a72a2054dde4e.jpg)

