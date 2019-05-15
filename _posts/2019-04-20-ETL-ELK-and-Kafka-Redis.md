---
published: true
category: DB
title: 日志收集:ETLELK以及Kafka/Redis
author: smona
date: '2019-04-14 12:52:47'
layout: post
---

<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [前言](#前言)
* [Filebeat](#filebeat)
* [Logstash](#logstash)
	* [输入](#输入)
	* [过滤](#过滤)
	* [输出](#输出)
* [ElasticSearch](#elasticsearch)
* [队列](#队列)
	* [Redis](#redis)
	* [Kafka](#kafka)
* [EOS语义](#eos语义)

<!-- /code_chunk_output -->

# 前言
其实一直都想写ELK的，毕竟在公司做了一年的日志ETL的工作，而且经历了上个世纪遗留的日志收集方案到现在流行的日志收集方案的变更，但是一直都没有找到合适的时间和机会写这一篇文章，趁着寒冬需求量下降没有那么忙碌就做了  

ELK是Elastic公司的产品，elastic公司最远近闻名的就是他的ElasticSearch，这也是ELK中的'E'，其他'L'和'K'，分别是指Logstash以及Kibana  

但是现在日志收集其实不止ELK，更多是指Beats => Kafka/Redis => Logstash => ElasticSearch/RDB 这种组合  

当然收集日志还有Flume方案:  
"Flume使用 tail -f 的方式实时收集日志文件中追加的文本内容。通过Flume配置文件中定义的正则表达式对日志文本进行字段分割...Logstash在进行文本数据收集时并没有使用 tail -f 这种简单粗暴的方式，而是在本地文件中记录了日志文件被读取到的位置，完美解决了flume升级重启时丢失数据的问题"  
REF:[有哪些基于ELK的亿级实时日志分析平台实践的案例？ - 小猫助手的回答 - 知乎](https://www.zhihu.com/question/59957272/answer/170694929)  

beats有很多不同的种类:Filebeat(日志文件),Metricbeat(指标),Packetbeat(网络数据),Winlogbeat(Windows 事件日志),Auditbeat(审计数据),Heartbeat(运行时间监控),Functionbeat(无需服务器的采集器)，我们这里用的是filebeat收集一些用于统计的游戏玩家数据  

其实无论是Flume还是Logstash，都是是实现了收集=>处理=>保存日志系统原理上的这三部分，可以说这三部分是任何日志收集系统的模型了。  

# Filebeat
ELK是ES,Logstash,Kibana的首字母缩写，并没有Filebeat，这是因为Filebeat是logstash的轻量版，基于go写的，因为基于jvm的logstash在资源消耗方面很大，在最初版发布后，用户发现在日志产生的服务的服务器中用Logstash收集日志会显著地消耗资源，影响了原来的业务，所以后面才有filebeat的出世，filebeat没有logstash这么多的解析日志、组装字段等对日志内容处理的插件，他只是单纯地做日志运送，最大的优点是资源占用极小(配置得当的话1%的资源消耗都不到,相较于Flume的Source来说是完胜)，不会影响正常业务，缺点就是没有丰富的解析插件支持。所以现在实践都是日志产生源一般都会用filebeat，然后传输到logstash集群做集中地日志处理，以免影响正常业务

1.注册表文件
2.重试机制(blocked/not confirm)
3.重启会产生重复，因为restart don't wait for ack:filebeat.shutdown_timeout: 5s
4.ignore_older & clean_inactive & close_inactive(in case too many open files)
5.inode reuse problem
6.scan_frequency:如果你发现filebeat占用CPU过高，可以调节扫描频率

# Logstash  
作为消费者,如果是先处理再提交位点offset,那么就是at-least-once,如果先提交位点在处理就是at-most-once
一下都是Logstash5.5来说明  

## 输入	 
logstash-input-beats:接受filebeat数据(filebeat=>logstash跳过队列)
logstash-input-file:读取本地文件
logstash-input-redis:拉取redis数据
logstash-input-kafka:拉取kafka数据

```ruby
input{
    beats {
        port => 5044
    }
    file {
        path => ["/data/logstash/logs/logstash-plain.log"]
        add_field => { "log_group"=>"logstash-log" }
    }
    redis {
        host => "redis_host"
        port => "redis_port"
        db => "redis_db"
        password => "password"
        key => "list_name"
        data_type => "list"
        threads => 1 #启用线程数量
        batch_count => 500 #返回的事件数量，此属性仅在list模式下起作用
    }
    kafka {
        bootstrap_servers => "10.10.28.130:9092,10.10.28.128:9092,10.10.28.129:9092"
        topics => ["topic_2","topci_2"]
        group_id => "GoupID"

        consumer_threads => 3
        decorate_events => true
        codec => json
        type => "access"

        poll_timeout_ms => 10000
        # request_timeout_ms should be greater than session_timeout_ms and fetch_max_wait_ms
        request_timeout_ms => "300000"
        session_timeout_ms => "150000"
        fetch_max_wait_ms => "10000"
        max_poll_records => "500"
        auto_commit_interval_ms => "5000"

        # fetch_max_bytes(3*1024k=3145728 advice from alibaba) is a new option start from logstash6.0
        # fetch_max_bytes => "3145728"
        # logstash5.5 doesn't have fetch_max_bytes option,use max_partition_fetch_bytes(3*1024k/24 = 131072 advice from alibaba) instead
        max_partition_fetch_bytes => "131072"

        # Security Config
        # security_protocol => "SASL_SSL"
        # sasl_mechanism => "PLAIN"
        # jaas_path => "/path_to/jaas.conf"
        # ssl_truststore_password => "KafkaOnsClient"
        # ssl_truststore_location => "/path_to/kafka.client.truststore.jks"
    }
}
```

## 过滤
常用的logstash过滤插件:  
logstash-filter-grok:非结构化日志文本转化为结构化事件  
logstash-filter-mutate:强制转换
logstash-filter-uuid:生成uuid
logstash-filter-date:日期插件
logstash-filter-geoip:可以根据IP转化出经纬度和地理位置
logstash-filter-fingerprint:去重用指纹
logstash-filter-ruby:自定制代码块

## 输出
常用的logstash输出插件:
logstash-output-file
logstash-output-kafka
logstash-output-elasticsearch
logstash-output-webhdfs

logstash-output-jdbc:插入/更新实现jdbc的关系型数据库(**注意!这个需要另外安装,非自带**)


# 队列  

## Redis  
终于找到除了做缓存以后Redis的第二种使用场景:消息队列,Redis做消息队列一般都是用list，用Redis做消息队列看中的是Redis的简单易用，而且速度快，然而Redis的list一旦有消费者消费了一条信息，那么这条信息就没有办法让其他消费者消费了，但是Kafka可以通过不同的goup_id来给不同的消费者消费
pub-sub
redis能做到exactly-once么?

扯点缓存问题:
缓存问题本质上就是数据库压力问题，尽量不要让本来在缓存的压力达到数据库  

1.缓存穿透(查询不存在key)  
解决方案:
(缓存方案)如果查询不存在key，返回null
(过滤方案)bloom filter

2.缓存击穿(多线程查key,key失效,并发打到数据)  
解决方案:mutex后做缓存，解决db并发思路一致

3.缓存雪崩(redis cluster不可用，本质上违背了高可用)  
本地缓存+限流

## Kafka  
相对于Redis，Kafka提供了更多的保证，比如0.10.0的at-least-once到0.11.0的exactly-once,以及
有ZooKeeper(分布式锁和ByzantineFault一致性问题&ZAB算法)，分布式事务(solution:2pc,3pc,tcc)

如果是由kafka推给消费者，容易造成消费者过载，所以消费者和kafka之间是pull模式
kafka是queue和pub-sub的集合体(consumer_group & offset),in-order-garantee(partitions),fault-tolence(replica),write-to-disk(structured commit-log storage),long retention policy(often set to  2 weeks)

kafka 0.11.0实现了EOS语义:
EOS是流式处理实现正确性的基石,主流的流式处理框架基本都支持EOS(如Storm Trident, Spark Streaming, Flink)
Kafka streams肯定也要支持的,0.11版本通过3个大的改动支持EOS(exactly-once semantics)：
1.幂等的producer（这也是千呼万唤始出来的功能)
2.支持事务;
3.支持EOS的流式处理(保证读-处理-写全链路的EOS)

EOS = Exaclty Once Semantics
EOS = at-least-once + deduplicate

ack = 0,1,2

Kafka设计:
1.PageCache:
aka DiskCache,rather than maitain as much as possible in memory,and flush it all at once to the filesystem in a panic when run out of space, we invert that:All data is immediately written to a persistent log.

2.small IO operations and excessive byte copying(low meesage rate) by using sendfile()

# ElasticSearch  
本质上是Apapche Lucene，只是在其之上提供了RESTful API以及watcher监控套件
master_node / data_node
2节点的脑裂(brain-split)问题
shards & replica
keyword vs text
倒排索引原理