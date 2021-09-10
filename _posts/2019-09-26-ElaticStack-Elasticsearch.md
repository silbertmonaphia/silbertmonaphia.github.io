---
published: true
category: DB
title: 日志分析:ElasticSearch
author: smona
date: '2019-09-26 12:52:47'
layout: post
---

- [前言](#前言)
- [数据类型](#数据类型)
- [索引](#索引)
  - [索引过程](#索引过程)
    - [物理上](#物理上)
    - [逻辑上](#逻辑上)
    - [Template vs Mapping](#template-vs-mapping)
    - [Analyzer](#analyzer)
    - [索引原理](#索引原理)
      - [倒排索引](#倒排索引)
      - [字典树Trie](#字典树trie)
      - [Lucence剖析](#lucence剖析)
- [查询](#查询)
  - [查询过程](#查询过程)
  - [查询语句](#查询语句)
  - [聚合与排序](#聚合与排序)
    - [聚合](#聚合)
    - [排序](#排序)
- [ES插件](#es插件)
- [ES工具](#es工具)
    - [Kibana](#kibana)
    - [Xpack](#xpack)
    - [ES-Head](#es-head)
    - [ES-SQL](#es-sql)
- [ES集群](#es集群)
  - [部署规划](#部署规划)
  - [查询优化](#查询优化)
- [REF](#ref)


# 前言  

之前讲过日志分析:ETL和ELK，但是主要是覆盖了日志集中式收集部分的内容，但是对ElasticSearch只是一笔带过，这篇就来好好聊聊ES，我会尽量少贴代码，因为写博客写到现在我发现，博文尽量减少代码会比较好，只做最低限度的代码讲解，这个启发来源于我自己去读别人的博客也是会跳过代码，先读文字段落  

当我们谈到后端存储的时候我们首先会聊到的是关系型数据库，大多数web应用最开始只是需要一个关系型数据库，缓存，消息队列，搜索引擎都是数据和并发规模上升之后带来新问题才引入的，当然再网上就是hadoop/spark的大数据存储和计算，这里有点脱离web应用的范畴而到了数据挖掘的范畴了，而就是搜索引擎而言，重要的就是全文索引，其实现代的关系型数据库都多多少少会支持全文索引，但是毕竟关系型数据实现全文索引还是捉襟见肘，比如现在主流的两大关系型数据库MySQL和PostgreSQL，都支持基于倒排索引的全文索引，但是就丰富性和健壮性来说，实在无法媲美今天要说的ElasticSearch  

ES本质上是Apapche Lucene，只是在其之上提供了RESTful API以及watcher监控套件，所以对ES研究深入的话还需要深入理解Lucence库的原理  
ES在处理节点发现与Master选举等方面没有选择Zookeeper等外部组件，而是自己实现的一套  

# 数据类型
其他数据类型基本和RDBMS差不多，唯一要注意的地方是关于字符类型的会要稍微理解一下，因为涉及到分词  
string:text(analyzed分词，只有分词了才能利用上全文索引)  
string:keyword(non-analyzed不分词)  
ES中文分词插件:ik 和 pinyin  

Q:ES中文分词和mysql5.7以后的ngram中文分词对比以及Postgresql的分词插件对比?  
A:  

# 索引
## 索引过程  
### 物理上
1.node节点代表计算机器，分为主节点master_node以及数据节点data_node，数据节点可以    
2.primary_shard主分片，，索引生成不可增加减少，这是因为document_id寻路到shard的方法是 hash(document_id) % number_of_primary_shards = shard_id
一个shard代表一个lucence索引,索引不可修改,增加的记录只是不断去增加lucence索引,或者叫段,ES会自动将小段合并为大段以提高搜索性能   
3.replica,主分片的副本数量,本质上也是一个分片  
4.总分片数 = 索引数 * (分片数 + 副本数)  

### 逻辑上
1. ES:_index/_type/_id(document) => RDBMS:db/table/row  
2. ES:document.field => RDBMS:row.field  
P.s. documents that have same mapping/template should be save in the same _type  
ES 6.0开始取消type的概念 https://github.com/elastic/elasticsearch/blob/6.5/docs/reference/mapping/removal_of_types.asciidoc  

- refresh api(still in memory)
    in-memory cache => segment => searchable segments
- flush api(fsync to disk-IO)
    translog => commit point

### Template vs Mapping
Template是ES模板，索引创建的时候会先去找保存在ES里面的符合索引名的Template，根据这个Template来完成自己的mapping创建，如果找不到就DynamicMapping了自动根据第一个录入的文档判断mapping中字段类型。  
Template是有优先级,比如"foo-bar-2019.04"这个索引,模板"for-bar-\*","foo-\*"都适用，那么就会选择优先级高的模板套上去，如果优先级一致，就会随机选一个套上去  
Mapping则是索引根据Template完成创建后属于索引自己的东西，如果录入到索引的文档字段和mapping的定义有冲突，ES会报mapping error(404)错误无法录入，这时候如果是Logstash的话可以考虑使用DLQ保存冲突的数据以便后面对这些数据修复后再次录入而不至于丢失数据  

### Analyzer
### 索引原理
#### 倒排索引
#### 字典树Trie
#### Lucence剖析

# 查询  
ES提供两种查询工具，一个是简单版本的`query string`(https://www.elastic.co/guide/en/elasticsearch/reference/5.5/query-dsl-query-string-query.html#query-string-syntax)，一种是完整版的`Query DSL`(https://www.elastic.co/guide/en/elasticsearch/reference/5.5/query-dsl.html)，但是其实`query string`是`Query DSL`中全文搜索`Full text queries`中的一种，`query string`常用在Kibana的Discover的手动的简单搜索中，但是一般做业务都会直接用完整版的`Query DSL`  

## 查询过程
分为两步"query_then_fetch":
1.Query阶段  

2.Fetch阶段  

## 查询语句
1.`Filter context`结构化查询  
这一块很像SQL查询，一般用来快速过滤数据，因为不用计算评分，所以会比全文索引要快。过滤器filter是如何计算的，其核心实际是采用一个 bitset 记录与过滤器匹配的文档。
单个查询内查找匹配文档 => 创建该查询的bitset(e.g. [1,0,0,1,0,0,0]) => 迭代该查询的bitset(s) => 增量使用计数(缓存bitsets)  
```json
{
  "query": { //这个就是Query context
      "filter": [ //这个就是Filter context,同样的还有must_not
        { "term":  { "status": "published" }}, 
        { "range": { "publish_date": { "gte": "2015-01-01" }}} 
      ]
  }
}
```

2.`Query context`全文索引查询  
这一块是ES的首要特性，也是强大过传统关系数据库的地方一般选型带上ES的原因都是有搜索按评分排序的需求  
关于相关度评分算法  
```json
{
    "query": { //这个就是Query context,同样的还有bool中的must, should 
        "match" : { //这个是全文检索的一种最常用的一个clause
            "message" : "this is a test" 
        }
    }
}
```

## 聚合与排序  
聚合Aggregation和排序OrderBy都是很消耗内存的活动，所以一般看到ES节点配置都是内存比较高，而CPU相对的小很多，比如2C8G  

### 聚合  
docValues给non-analyze未分词的字符串或者数值类型字段来做聚合的，是倒排索引的倒置，索引时生成  
fieldData给analyze分词后字符串字段来做聚合使用的，查询中生成  

### 排序

# ES插件  
`bin/elasticsearch-plugin list` 就可以看到自己已经装了啥插件  

# ES工具
### Kibana
Monitoring能监控ES各节点情况、索引情况、Logstah节点状况  
Dev Tools有`Console`可以像Curl一样直接调用ES的api，`Search Profiler`可以对ES查询调优,`Grok Debugger`类似`https://grokdebug.herokuapp.com/`，如果`https://grokdebug.herokuapp.com/`被墙了，还有Kibana的`Grok Debugger`顶着  
当然Kibana对数据展示能力才是它核心能力，初级入门`Discover+Visualize+Dashboard`三件套足够了，高级的可以搞搞`Timelion(时序数据)+MachineLearning(机器学习)+Graph(图算法)`，因为本篇是ES相关的这里就不铺开来说了  

### Xpack
Xpack主要还是扩展ES能力比如对索引的监控告警的`Watcher`模块就是在Xpack里面的，原本为收费的后面开源了  
Xpack有index lifecycle management (ILM)去基于你设定的规则去自动管理索引  
Xpack有Event Query Language (EQL)查询时序数据，为安全而生

### ES-Head
ES本身虽然已经包装简化了Lucence，提供了RESTful接口，但是用curl来操作ES实在是非常低效(curl一大段还经常忘记加header的Content-Type)，Kibana可以顶一点用，但是它主要的作用并不是维护和操作ES，而是可视化ES索引的数据，所以需要一款类似Navicat之于MySQL的数据库图形化界面，这里我推荐ES-Head  

ES-head可以直接装Chrome的插件，查询和操作ES还是挺方便的，就是导出不太方便，如果要导出csv/json可以选择Dejavu，同样也可以chrome装它的扩展，但是其实我觉得只是简单的操作和查询，开始ES-head已经够用了，虽然有对比嫌弃ES-head太老了  

ES-head:  
```shell
cd /data/joygames/elasticsearch-head
npm run start &
```

### ES-SQL
如果有认真研究过ES的人，会有ES和传统的关系型数据很像的感觉，其实官方也有把两者放在一起比较的文档，很多概念都是相通的，所以为什么不能用SQL来写ES的查询呢？毕竟ES的查询一坨可读性没有SQL好的json...
Xpack后期也自己搞了一个SQL的插件  

# ES集群
## 部署规划
脑裂(brain-split):两个主节点带来的状态不一致，触发选举节点设定的数目太少  
线程池:大小最好设置为`3 * cores/2 + 1`，最大不要超过`2 * cores`  
JVM:内存分配给JVM的不要超过32G,超过32G就不会启用内存指针压缩  
JVM GC堆:新生代(young)=>幸存空间(survivor)=>老生代(old)  
分配ES不要超过总内存的1/2  
shards在单节点2C8G前提下, 每个节点shards数目最好<200个  
每个索引的shard最好和节点数目保持一致，而replica最好设置为2，达到三备就足够了  

ES 单shard只能支持2^31(20亿)条记录，20 to 40 GB，是有限制的，那些企图只设置主分片shard=1的人要注意了，而且不用到达20亿，20G，由于机器原因，在一定量的时候查询和索引性能便出现明显的下滑，但是也不能主分片数量设置太大，因为ES查询需要把query分配到每个shard里面去查询，如果太多shard那么，合并每个shard查询结果的时候时间损耗很大，主分片数量的多寡需要实际压测才知道  
Ref:https://segmentfault.com/a/1190000011174694  

## 查询优化  
凡是有查询数据的地方必定涉及优化问题!我已经参透了，所以优化查询肯定是必备的  
然后也得打开慢查询，慢查询是我们优化的风向标，避免我们盲目找查询来优化，做到有的放矢  
和RDBMS比如MySQL一样，存储和查询数据的地方肯定有查询或者写入的问题，需要优化的地方，可以通过配置优化和开发优化两个方面思考，很多时候都是我们没有用好  

一个原则:能先过滤的过滤(`Filter context`)来避免不必要的`Query context`评分计算  

调优时候用到的ES api:  
search api  
analyze api => 查看分词结果  
explain api => 评分结果解析  
profile api => 查询性能分析  

# REF
[ES权威指南](https://www.elastic.co/guide/cn/elasticsearch/guide/2.x/index.html)