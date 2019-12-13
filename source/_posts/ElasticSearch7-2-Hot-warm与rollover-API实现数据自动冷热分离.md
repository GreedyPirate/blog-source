---
title: ElasticSearch7.2 实现数据自动冷热分离
date: 2019-10-25 19:22:59
categories: ELK Stack
tags: [ElasticSearch, X-pack, kibana]
toc: true
comments: true
---

# 冷热分离
在基于时序数据中，我们总是关心最近产生的数据，例如查询订单通常只会查询最近三天，至多到最近一个月的，查询日志也是同样的情形，很少会去查询历史数据，也就是说类似的时序数据随着时间推移，价值在逐渐弱化。在es中经常按日或按月建立索引，我们很容易想到，历史索引被查询命中的概率越来越低，不应该占用高性能的机器资源(比如大内存，SSD)，可以将其迁移到低配置的机器上，从而实现冷热数据分离存储。

# 分片分配规则(shard allocation filtering)
假设我们有三个es节点，一台高性能机器(hot)和2个低配置机器(warm)，通常索引分片会均匀分布在集群节点中，但我们希望最新的数据由于其写入和查询频繁的特性，只能保存在hot节点上，而过期的数据保存在warm节点上。
实现该功能，首先要对节点人为的打个标签，然后在索引创建时指定要把分片分配给hot节点，在索引不再写入后，迁移到warm节点上

## 节点tag
依次启动三个节点，同时加入box_type和resource_level标签，box_type标记node1、node2为warm节点，node3为hot节点，resource_level标记机器资源的性能，分为高，中，低
```
bin/elasticsearch -d -p pid -E node.name=node1 -E node.max_local_storage_nodes=3 -E path.data=node1_data -E path.logs=node1_logs -E node.attr.box_type=warm -E node.attr.resource_level=high

bin/elasticsearch -d -p pid -E node.name=node2 -E node.max_local_storage_nodes=3 -E path.data=node2_data -E path.logs=node2_logs -E node.attr.box_type=warm -E node.attr.resource_level=mdeium

bin/elasticsearch -d -p pid -E node.name=node3 -E node.max_local_storage_nodes=3 -E path.data=node3_data -E path.logs=node3_logs -E node.attr.box_type=hot -E node.attr.resource_level=high
```

### 查看属性
kibana中输入以下命令
```
GET _cat/indices?v
```
得到以下结果，可以看到box_type和resource_level标签在每个节点的值
```
node  host      ip        attr              value
node3 127.0.0.1 127.0.0.1 ml.machine_memory 17179869184
node3 127.0.0.1 127.0.0.1 ml.max_open_jobs  20
node3 127.0.0.1 127.0.0.1 box_type          hot
node3 127.0.0.1 127.0.0.1 xpack.installed   true
node3 127.0.0.1 127.0.0.1 resource_level    high
node1 127.0.0.1 127.0.0.1 ml.machine_memory 17179869184
node1 127.0.0.1 127.0.0.1 box_type          warm
node1 127.0.0.1 127.0.0.1 xpack.installed   true
node1 127.0.0.1 127.0.0.1 ml.max_open_jobs  20
node1 127.0.0.1 127.0.0.1 resource_level    high
node2 127.0.0.1 127.0.0.1 ml.machine_memory 17179869184
node2 127.0.0.1 127.0.0.1 ml.max_open_jobs  20
node2 127.0.0.1 127.0.0.1 box_type          warm
node2 127.0.0.1 127.0.0.1 xpack.installed   true
node2 127.0.0.1 127.0.0.1 resource_level    mdeium
```

## 建立索引
假设当前时间为2019年9月1日，作为最新的数据存储在hot节点上，只需要在建立索引时指定allocation策略即可
```
PUT api_log_2019-09-01
{
  "settings": {
    "number_of_shards": 3, 
    "number_of_replicas": 0, 
    "index.routing.allocation.require.box_type": "hot"
  }
}
```
### index.routing.allocation详解

该配置支持include，require，exclude三种选项，它们的值都可以是多个，用逗号分隔
```
index.routing.allocation.include.{attribute}：将索引分配给具有至少一个值的节点。
index.routing.allocation.require.{attribute}：将索引分配给具有所有值的节点。
index.routing.allocation.exclude.{attribute}：将索引分配给没有该值的节点
```
es还提供了以下内置字段
* _name： 节点名称匹配
* _host_ip：主机名ip地址匹配
* _publish_ip：publish ip匹配，参考network.publish_host配置
* _ip：_host_ip或者_publish_ip匹配
* _host：主机名匹配

假设建立索引时没有配置该选项也不要紧，动态修改即可
```
PUT api_log_2019-09-01/_settings
{
  "index.routing.allocation.require.box_type": "hot"
}
```


## 迁移索引
迁移历史索引到warm节点的方式也是采用动态修改请求的方式
```
PUT api_log_2019-09-01/_settings
{
    "index.routing.allocation.require.box_type": "warm",
    "index.routing.allocation.include.resource_level": "mdeium"
}
```
我们将api_log_2019-09-01迁移到了box_type为warm，resource_level为mdeium的节点，即node2
通过查询索引分片的分布情况
```
GET _cat/shards/api_log_2019-09-01?v
```
结果如下
```
index          shard prirep state   docs store ip        node
api_log_2019-09-01 1     p      STARTED 4711 4.1mb 127.0.0.1 node2
api_log_2019-09-01 2     p      STARTED 4656   4mb 127.0.0.1 node2
api_log_2019-09-01 0     p      STARTED 4707 4.1mb 127.0.0.1 node2
```

# Rollover API
大家应该也注意到了，迁移索引的步骤是手动完成的，有没有更智能的方式呢，答案是肯定的，rollover API可以很好地实现这个功能

## rollover
首先为索引建立别名, 由于多个index可以对应一个alias，为了让es知道往哪个索引中写，标记其中一个索引is_write_index为true
同时需要注意索引名以横杠+数字结尾的形式命名，这是为了让es自动生成索引
```
POST _aliases
{
  "actions": [
    {
      "remove": {
        "index": "api_log_2019-09-01",
        "alias": "api_logs",
        "is_write_index": true
      }
    }
  ]
}
```

rollover API会根据设置的条件(conditions)来生成新的索引
```
POST api_logs/_rollover
{
  "conditions": {
    "max_age": "1d",
    "max_docs": 10000,
    "max_size": "5gb"
  }
}
```
conditions的详细解释：

* max_age：索引是否创建大于1天
* max_docs：索引文档数是否超过10000
* max_size：索引大小是否超过5GB
max_size正在进行中的合并会产生大量的临时分片大小增长，而当合并结束后这些增长会消失掉，不稳定，max_age每个时间内不一定均衡，max_docs比较合适
即以上三个条件满足其一就会自动rollover

### 新索引配置
rollover也支持索引的settings设置

```
POST api_logs/_rollover
{
  "conditions": {
    "max_age": "1d",
    "max_docs": 10000,
    "max_size": "5gb"
  },
  "settings": {
    "index.number_of_shards": 2
  }
}
```

### 自定义索引名称
生成的索引名称为api_log_2019-09-000002, 以长度为6，序号+1，左填充0的格式命名，但es支持自定义名称，只需要在_rollover端点加入名称即可
```
POST api_logs/_rollover/api_log_2019-09-02
{...}
```




# 总结
shard allocation filtering赋予了索引选择节点的能力，但在迁移过程中需要手动触发，因此rollover API应运而生，它可以在索引满足一定的条件下自动迁移索引到warm节点，index lifecycle management从更高的角度定义了索引的声明周期，把每个节点定义为一个phase，在每个阶段要做的事定义为action，这个action可以让我们对索引rollover，delete等，这一系列的功能，都是为了更智能的管理时序数据，典型的场景就是每天产生的日志

# 参考
[https://www.elastic.co/guide/en/elasticsearch/reference/7.2/getting-started-index-lifecycle-management.html#ilm-gs-apply-policy](https://www.elastic.co/guide/en/elasticsearch/reference/7.2/getting-started-index-lifecycle-management.html#ilm-gs-apply-policy)
[https://www.elastic.co/guide/en/elasticsearch/reference/7.2/indices-rollover-index.html](https://www.elastic.co/guide/en/elasticsearch/reference/7.2/indices-rollover-index.html)
[https://www.elastic.co/guide/en/elasticsearch/reference/7.2/shard-allocation-filtering.html](https://www.elastic.co/guide/en/elasticsearch/reference/7.2/shard-allocation-filtering.html)
[https://www.elastic.co/cn/blog/managing-time-based-indices-efficiently](https://www.elastic.co/cn/blog/managing-time-based-indices-efficiently)
[https://juejin.im/post/5a990cdbf265da239b40de65](https://juejin.im/post/5a990cdbf265da239b40de65)







