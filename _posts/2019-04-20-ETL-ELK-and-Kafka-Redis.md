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
	* [补充](#补充)
* [队列](#队列)
	* [Redis](#redis)
	* [Kafka](#kafka)
* [ElasticSearch](#elasticsearch)

<!-- /code_chunk_output -->

# 前言
其实一直都想写ELK的，毕竟在公司做了一年的日志ETL的工作，而且经历了上个世纪遗留的日志收集方案到现在流行的日志收集方案的变更，但是一直都没有找到合适的时间和机会写这一篇文章，趁着寒冬需求量下降没有那么忙碌就做了  

ELK是Elastic公司的产品，elastic公司最远近闻名的就是他的ElasticSearch，这也是ELK中的'E'，其他'L'和'K'，分别是指Logstash以及Kibana  

但是现在日志收集其实不止ELK，更多是指Beats => Kafka/Redis => Logstash => ElasticSearch/RDB 这种组合  

当然收集日志还有Flume方案:  
"Flume使用 tail -f 的方式实时收集日志文件中追加的文本内容。通过Flume配置文件中定义的正则表达式对日志文本进行字段分割...Logstash在进行文本数据收集时并没有使用 tail -f 这种简单粗暴的方式，而是在本地文件中记录了日志文件被读取到的位置，完美解决了flume升级重启时丢失数据的问题"  
REF:[有哪些基于ELK的亿级实时日志分析平台实践的案例？ - 小猫助手的回答 - 知乎](https://www.zhihu.com/question/59957272/answer/170694929)  

其实无论是Flume还是Logstash，都是是实现了收集 => 处理 => 保存这三部分(重写前的Flume组件很混乱根本看不出这三部分，而且配置麻烦，虽然重写后简单不到哪里去，但是重写后还是明确地往这三部分靠)，可以说这三部分已经是成熟的日志收集系统模型标准。  

# Filebeat  

Beats家族有很多不同的种类:Filebeat(日志文件),Metricbeat(指标),Packetbeat(网络数据),Winlogbeat(Windows 事件日志),Auditbeat(审计数据),Heartbeat(运行时间监控),Functionbeat(无需服务器的采集器)，我们这里用的是filebeat收集一些用于统计的游戏玩家数据(比如创角充值或者是玩法购买掉落)。  

ELK是ES,Logstash,Kibana的首字母缩写，并没有Filebeat，这是因为Filebeat是logstash的轻量版，基于go写的，因为基于jvm的logstash在资源消耗方面很大，在最初版发布后，用户发现在日志产生的服务的服务器中用Logstash收集日志会显著地消耗资源，影响了原来的业务，所以后面才有filebeat的出世，filebeat没有logstash这么多的解析日志、组装字段等对日志内容处理的插件，他只是单纯地做日志运送，最大的优点是资源占用极小(配置得当的话1%的资源消耗都不到,相较于Flume的Source来说是完胜)，不会影响正常业务，缺点就是没有丰富的解析插件支持。所以现在实践都是日志产生源一般都会用filebeat，然后传输到logstash集群做集中地日志处理，以免影响正常业务

值得注意的点:  
1.注册表文件(deb/rpm包装的路径/var/lib/filebeat),  
REF:[Directory Layout](https://www.elastic.co/guide/en/beats/filebeat/5.5/directory-layout.html#_deb_and_rpm)  

注册表文件样例:  
```json
[
    {
        "source":"/foo/bar.log",
        "offset":702,
        "FileStateOS":{"inode":535737,"device":64529},
        "timestamp":"2019-04-29T16:32:59.514463586+08:00",
        "ttl":-2
    },
    {
        "source":"/foo/foo_bar.log",
        "offset":190,
        "FileStateOS":{"inode":540909,"device":64529},
        "timestamp":"2019-04-29T16:32:59.514466675+08:00",
        "ttl":-2
    }
]
```

source:日志文件完整路径  
offset:上次读到位置(字节数)  
inode:linux inode,linux会有inode重用问题  
device:日志所在磁盘的磁盘编号,和inode一起确定一个唯一的文件  
timestamp:上次更新时间戳  
ttl:采集失效时间。-1表示只要日志存在，就一直采集该日志  

REF:[filebeat相关registry文件内容解析](https://www.cnblogs.com/micmouse521/p/8085229.html)

2.linux inode重用问题:  
Linux文件系统是通过inode和device来辨认文件的。当一个文件从磁盘移除的时候，它原本的inode有几率重新分配到一个新的文件，比如File Rotation,即一个文件删除后，马上创建一个文件，那么新的文件的inode和刚被删除的文件的inode很可能一模一样，这种情况下，filebeat会认为这俩文件是一个文件，然后从上次offset位置继续读取数据，这很明显是错误的
解决inode重用问题,推荐的做法是利用好clean_* 配置,特别是clean_inactive(清理指定时间外的注册表记录),比如如果你文件24小时rotate一次，然后旧文件就不再更新，那么你可以设置ignore_older48h以及clean_inactive设置为72h  
REF:[Inode reuse causes Filebeat to skip lines?](https://www.elastic.co/guide/en/beats/filebeat/master/faq.html#inode-reuse-issue)

3.重试机制(blocked/not confirm):  
publisher从spooler中接收事件，转换其类型。@timestamp和其他跟日志内容无关的tag就是在这一步打入到发送数据中的，详情看input/event.go。它接着调用PublishEvents把数据发送出去，并注册了通知函数和标志位Guaranteed。**正是由于这个Guaranteed标志位，filebeat的数据会在发送时一直重试到成功为此**。当数据被发送时，发送方调用通知函数，publisher知道可以把这个事件划入到“已确认”的队列中  

```yaml
# Filebeat全局配置
# spool_size:后台事件计数阈值，超过后强制发送，默认2048
filebeat.spool_size: 2048
# idle_timeout:后台刷新超时时间，超过定义时间后强制发送，不管spool_size是否达到，默认5秒
filebeat.idle_timeout: 5s

# shutdown_timeout:filebeat关闭前最长等待publisher发送事件成功时间
filebeat.shutdown_timeout: 5s
filebeat.config_dir: /etc/filebeat/conf.d

# 日志路径
logging.to_files: true
logging.level: info
logging.files:
  path: /data/filebeat/logs/

output:
    logstash:
      hosts: ["10.10.15.100:5044", "10.10.16.101:5044"]
      # loadbalance事实上filebeat5.5只会选hosts中一个logstash连接,而不是测量所有logstash状态来平均发送,只有当前连接中的logstash连接不上,才会重新尝试hosts里面其他logstash
      loadbalance: true

    redis:
      hosts: ["redis_host_1","redis_host_2"]
      port: "redis_port"
      db: "7"
      password: "password"
      # key:这里%{[log_group]}我们是取log_group变量的值,这个值具体会在每个日志收集配置里面设置，通过log_group变量区分redis上的key
      key: '%{[log_group]}'
      data_type : "list"
      # 启用线程数量
      threads : 10
      # 返回的事件数量,此属性仅在list模式下起作用
      batch_count : 2500

    kafka:
      hosts: ["100.200.200.100:9093","100.200.200.101:9093","100.200.200.102:9093"]
      topic: '%{[kafka_key]}'
      version: 0.10.0.0
      compression: gzip
      worker: 1
      required_acks: 1
      max_retries: 3
      max_message_bytes: 1000000
      partition.round_robin:
        reachable_only: false
      keep_alive: 86400

      ssl.enabled: true
      ssl.certificate_authorities: ["/etc/filebeat/ca-cert"]
      ssl.verification_mode: none
      timeout: 60
      broker_timeout: 60

      username: "kafka_username"
      password: "kafka_password"
      ssl.enabled: true
      ssl.certificate_authorities: ["/etc/filebeat/ca-cert"]
      ssl.verification_mode: none
```

```yaml
# Filebeat每个日志配置
filebeat.prospectors:
  - input_type: log
    paths:
      - /foo/bar.log
    # enabled:每个prospectors的开关，默认true
    enabled: true
    # scan_frequency:prospect指定的目录下面检测文件更新
    # 如果你发现filebeat占用CPU过高，可以调低scan_frequency扫描频率
    scan_frequency: 10s
    # close_inactive:如果在指定时间没有被读取，将关闭文件句柄
    # Solve Too many open files handlers problem
    close_inactive: 5m
    # close_renamed:如果文件被重命名和移动，关闭文件的处理读取
    close_renamed: true
    # close_removed:文件被删除时，关闭文件的处理读取
    # Solve file rotation problem
    close_removed: true
    # ignore_older:忽略指定时间段以外修改的日志内容
    ignore_older: 24h
    # clean_inactive: >=ignore_older+scan_frequency
    # 以确保在文件仍在收集时没有删除任何状态
    clean_inactive: 48h
    # clean_removed:如果文件在磁盘上找不到，将从注册表中清除记录
    # 若关闭close_removed 必须关闭clean_removed
    clean_removed: true
    fields:
        kafka_key: foo_bar
        log_group: foo_bar
        type: foo_bar
    fields_under_root: true
```

Filebeat层引起数据重复问题的原因:  
1.技术上,上面全局配置的参数filebeat.shutdown_timeout如果没有配置的话是默认关闭的,filebeat默认马上重启不会等待发送的事件的ack来更新注册表,重启时候会造成重复发送数据  

> filebeat默认配置重启有几率会产生重复，因为默认shutdown_timeout没有打开，Filebeat会马上关闭而不会等待publisher完成发送事件，这就意味着任何发送出去却没有被acknowledged的事件(即没有更新注册表),在重启之后都可能重新发送一次，可以通过设置全局变量filebeat.shutdown_timeout强制等待

2.业务上，假如clean_inactive设置为24h,即昨天日志以前在filebeat注册表中的记录会被清理掉，在你的估算下昨天以前的日志本来不应该更新，但是一旦更新就会造成重复数据，对于这种我们可以:  
- 根据业务,增大安全范围,比如clean_inactive调整到48h以上  
- 日志源及时清理收集过的日志,转移到另外的地方备份  
- 下游做好去重机制  

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

## 补充
经常遇到一个问题就是，和同事上传logstash新配置如果有问题，会影响线上其他logstash正常的配置,logstash(6.0+) mutiple pipeline,让pipeline之间不互相影响，这个待测试?


# 队列  

## Redis  
终于找到除了做缓存以后Redis的第二种使用场景:消息队列,Redis做消息队列一般都是用list，用Redis做消息队列看中的是Redis的简单易用，而且速度快，然而Redis的list一旦有消费者消费了一条信息，那么这条信息就没有办法让其他消费者消费了，但是Kafka可以通过不同的goup_id来给不同的消费者消费
pub-sub
redis能做到exactly-once么?

扯点Redis缓存问题:
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