---
title: ElasticSearch7.2 父子文档
date: 2019-09-27 16:21:59
categories: ELK Stack
tags: [ElasticSearch]
toc: true
comments: true
---
>写这篇文章的目的是为了帮助大家了解7.2版本中的父子文档，之前希望通过百度的博客快速了解一下，然而大失所望，建立索引的语法在7.2版本没有一个能通过，决定仔细看一遍[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/7.2/parent-join.html#_searching_with_parent_join)

## 建立父-子文档语法

首先看一下如何建立父子文档，明显和网上"_parent"的方式不一样，说明es后期版本已经修改了语法
```
PUT my_index
{
  "mappings": {
    "properties": {
      "my_join_field": { 
        "type": "join",
        "relations": {
          "question": "answer" 
        }
      }
    }
  }
}
```
这段代码建立了一个my_index的索引，其中my_join_field是一个用于join的字段，type为join，关系relations为：父为question, 子为answer
至于建立一父多子关系，只需要改为数组即可：` "question": ["answer", "comment"] `

## 插入数据

插入两个父文档，语法如下
```
PUT my_index/_doc/1?refresh
{
  "text": "This is a question",
  "my_join_field": {
    "name": "question" 
  }
}
```
同时也可以省略name
```
PUT my_index/_doc/1?refresh
{
  "text": "This is a question",
  "my_join_field": "question"
}
```

### 插入子文档
子文档的插入语法如下，注意routing是父文档的id，平时我们插入文档时routing的默认就是id
此时name为answer，表示这是个子文档
```
PUT /my_index/_doc/3?routing=1
{
  "text": "This is an answer",
  "my_join_field": {
    "name": "answer", 
    "parent": "1" 
  }

```
### 通过parent_id查询子文档
通过parent_id query传入父文档id即可
```
GET my_index/_search
{
  "query": {
    "parent_id": { 
      "type": "answer",
      "id": "1"
    }
  }
}
```



## 父-子文档的性能及限制性
父-子文档主要适用于一对多的实体关系，将其反范式存入文档中

父-子文档主要由以下特性：

* Only one join field mapping is allowed per index. 
每个索引只能有一个join字段
* Parent and child documents must be indexed on the same shard. This means that the same routing value needs to be provided when getting, deleting, or updating a child document.
父-子文档必须在同一个分片上，也就是说增删改查一个子文档，必须使用和父文档一样的routing key(默认是id)
* An element can have multiple children but only one parent.
每个元素可以有多个子，但只有一个父
* It is possible to add a new relation to an existing join field.
可以为一个已存在的join字段添加新的关联关系
* It is also possible to add a child to an existing element but only if the element is already a parent.
可以在一个元素已经是父的情况下添加一个子

### 总结
es中通过父子文档来实现join，但在一个索引中只能有一个一父多子的join

## 关系字段
es会自动生成一个额外的用于表示关系的字段：field#parent
我们可以通过以下方式查询
```
POST my_index/_search
{
 "script_fields": {
    "parent": {
      "script": {
         "source": "doc['my_join_field#question']" 
      }
    }
  }
}
```
部分响应为
```
{
"_index" : "my_index",
"_type" : "_doc",
"_id" : "8",
"_score" : 1.0,
"fields" : {
  "parent" : [
    "8"
  ]
}
},
{
"_index" : "my_index",
"_type" : "_doc",
"_id" : "4",
"_score" : 1.0,
"_routing" : "10",
"fields" : {
  "parent" : [
    "10"
  ]
}
}
```
有_routing字段的说明是子文档，它的parent字段是父文档id，如果没有_routing就是父文档，它的parent指向当前id

## 全局序列
父-子文档的join查询使用一种叫做全局序列(Global ordinals)的技术来加速查询，它采用预加载的方式构建，防止在第一次查询或聚合时出现太长时间的延迟，但在索引元数据改变时重建，父文档越多，构建时间就越长，重建在refresh时进行，这会造成refresh大量延迟时间(在refresh时也是预加载).
如果join字段很少用，可以关闭这种预加载模式:`"eager_global_ordinals": false`

### 全局序列的监控
```
# 每个索引
curl -X GET "localhost:9200/_stats/fielddata?human&fields=my_join_field#question&pretty"
# 每个节点上的每个索引
curl -X GET "localhost:9200/_nodes/stats/indices/fielddata?human&fields=my_join_field#question&pretty"
```

## 一父多子的祖孙结构
考虑以下结构
```
   question
    /    \
   /      \
comment  answer
           |
           |
          vote
```
建立索引
```
PUT my_index
{
  "mappings": {
    "properties": {
      "my_join_field": {
        "type": "join",
        "relations": {
          "question": ["answer", "comment"],  
          "answer": "vote" 
        }
      }
    }
  }
}
 ```

### 插入孙子节点
注意这里的routing和parent值不一样,routing指的是祖父字段，即question,而parent指的就是字面意思answer
```
PUT my_index/_doc/3?routing=1&refresh 
{
  "text": "This is a vote",
  "my_join_field": {
    "name": "vote",
    "parent": "2" 
  }
}
 ```

## has-child查询
查询包含特定子文档的父文档，这是一种很耗性能的查询，尽量少用。它的查询标准格式如下
```
GET my_index/_search
{
    "query": {
        "has_child" : {
            "type" : "child",
            "query" : {
                "match_all" : {}
            },
            "max_children": 10, //可选，符合查询条件的子文档最大返回数
            "min_children": 2, //可选，符合查询条件的子文档最小返回数
            "score_mode" : "min"
        }
    }
}
```

## 测试代码
部分测试代码如下
```
DELETE my_index

PUT /my_index?pretty
{
  "mappings": {
    "properties": {
      "my_join_field": { 
        "type": "join",
        "relations": {
          "question": "answer" 
        }
      }
    }
  }
}


# 插入父
PUT /my_index/_doc/8?refresh&pretty
{
  "text": "This is a question",
  "my_join_field": {
    "name": "question" 
  }
}

PUT /my_index/_doc/10?refresh&pretty
{
  "text": "This is a new question",
  "my_join_field": {
    "name": "question"
  }
}

PUT /my_index/_doc/12?refresh&pretty
{
  "text": "This is a new question",
  "my_join_field": {
    "name": "question"
  }
}

# 插入子
PUT /my_index/_doc/3?routing=8&refresh&pretty
{
  "text": "This is an answer",
  "my_join_field": {
    "name": "answer", 
    "parent": "8" 
  }
}


PUT /my_index/_doc/4?routing=10&refresh&pretty
{
  "text": "This is another answer",
  "my_join_field": {
    "name": "answer",
    "parent": "10"
  }
}

# 通过parent_id查询子文档
GET my_index/_search
{
  "query": {
    "parent_id": { 
      "type": "answer",
      "id": "8"
    }
  }
}

# 查询relation
POST my_index/_search
{
 "script_fields": {
    "parent": {
      "script": {
         "source": "doc['my_join_field#question']" 
      }
    }
  }
}
```












































