---
title: Consul实践之集群部署
date: 2018-11-23 18:09:22
categories: 微服务注册中心
tags: [consul,注册中心]
toc: true
comments: true
---



# 部署准备

根据上一章中的架构图，进行分布式部署

首先从[Consul官网](https://www.consul.io/downloads.html)下载最新的安装包, 本文采用1.3.0版本

## 服务器列表

| 节点类型 | 名称      | ip           |
| -------- | --------- | ------------ |
| server   | server-01 | 10.9.188.187 |
| server   | server-02 | 10.9.171.147 |
| server   | server-03 | 10.9.39.37   |
| client   | client-01 | 10.9.181.34  |
| client   | client-02 | 10.9.117.128 |

通过wget命令下载Consul到每台服务器中，可以在网页中右键"复制链接"获取下载地址

下载完成后，通过`unzip`命令解压

## 启动server节点

启动的命令如下

```bash
nohup ./consul agent -server -bind=10.9.188.187 -client=0.0.0.0 -bootstrap-expect=3 -data-dir=./data -datacenter=dc1 -node=server-01 &
```

新建`start.sh`，将以上命令拷贝进来，通过`sh start.sh`命令启动，接下来说明各个参数的含义

1. -server：表示以server的身份启动agent
2. -bind：对外提供的绑定地址，填写本机ip即可
3. -client：可以接受通信的客户端地址，`0.0.0.0`表示接收来自任意ip的客户端
4. -bootstrap-expect：预期的server节点数，只有达到了这个数目，才会形成server集群
5. -data-dir：保存数据的目录，建议事先新建
6. -datacenter：所在的数据中心，默认dc1
7. -node：节点名称，最终会显示在界面中



## 组建集群

启动3个server节点之后，只是3个孤立的节点，需要用gossip协议互相告知，在Consul中，使用`join`命令加入集群

启动第二台server节点之后，就可以join到第一台server，第三台server节点加入任意一个即可

```bash
consul join 第一台的ip
```

接着使用`members`检查集群成员

```bash
consul members
```

显示3个Type为server的节点集群，同时3台服务的status为alive

![](https://ws4.sinaimg.cn/large/006tNbRwly1fxjdws5ixuj326w050abp.jpg)

查看Server集群信息
```bash
consul operator raft list-peers
```
可以看到Leader节点的



## 启动client节点

启动client的命令如下

```bash
nohup ./consul agent -bind=10.9.181.34 -data-dir=./data -client=0.0.0.0 -node=client-01 -ui &
```

参数说明

1. -ui：启动ui界面，只需要有一个client加入这个参数即可
2. 其他参数同 server，

同样的使用join命令加入agent集群



## ui界面

打开浏览器，输入10.9.181.34:8500，注意ip是添加了`-ui`参数的client地址

![Consul控制台首页](https://ws3.sinaimg.cn/large/006tNbRwly1fxlfnff3vmj32780h2mze.jpg)
### 说明

1. Service显示我们的应用服务，其中默认显示了3个Server节点的Consul服务
2. Nodes显示所有的agent节点，包括server和client
3. K\V是一个key-vlaue存储，可用于分布式锁的实现
4. Consul提供了ACL功能，基于ip和port控制请求的畅通
5. Intention基于ACL，可以添加服务之间的通信规则，允许或者是禁止通信







