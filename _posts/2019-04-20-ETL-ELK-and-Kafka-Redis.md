---
published: true
category: DB
title: 日志收集:ETL,ELK以及Kafka/Redis
author: smona
date: '2019-04-20 12:52:47'
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

      timeout: 60
      broker_timeout: 60

      # 如果输出到的Kafka开启了安全认证,就需要配置下面这些参数
      # username: "kafka_username"
      # password: "kafka_password"
      # ssl.enabled: true
      # ssl.certificate_authorities: ["/path_to/ca-cert"]
      # ssl.verification_mode: none
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
        kafka_key: foo_bar_kafka_key
        log_group: foo_bar_group
        type: foo_bar_type
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
作为消费者,如果是先处理再提交位点offset,那么就是at-least-once,如果先提交位点在处理就是at-most-once，而logstash则是at-least-once  

以下logstash配置示范都是基于Logstash5.5来说明,logstash编写比较繁琐,如果经常有新的的logstash配置要加入，建议脚本自动生成  

## 输入	 
logstash-input-beats:接受filebeat数据  
P.s.早期数据不多的时候，我们是filebeat=>logstash，跳过队列，这样调试比较方便，后面上了队列以后，调试的时候也是filebeat=>logstash，甚至只有filebeat或者只有logstash  

常用的logstash输入插件:
logstash-input-file:读取本地文件  
logstash-input-redis:拉取redis数据  
logstash-input-kafka:拉取kafka数据  

```ruby
input{
    beats {
        port => 5044
    }
    file {
        path => ["/path_to/logstash-plain.log"]
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
        bootstrap_servers => "100.200.200.100:9093","100.200.200.101:9093","100.200.200.102:9093"
        topics => ["foo_bar_kafka_key","topci_2"]
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

        # 如果输出到的Kafka开启了安全认证,就需要配置下面这些参数
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
logstash-filter-fingerprint:去重用事件指纹  
logstash-filter-ruby:自定制代码块  

```ruby
filter{
  if [log_group] == "foo_bar_group" {
    if [type] == "for_bar_type" {
        grok {
            keep_empty_captures => true
            match =>{
                "message" => "^\"(?<ymd>%{NUMBER}|null|)\",\"%{DATA:network}\",\"(?<active_time>%{NUMBER}|null|)\",\"%{DATA:ip}\""
            }
            remove_field=>["message"]
        }

        mutate {
            convert => ["ymd","integer","active_time","integer"]
        }

        uuid {
              target => 'log_uuid'
        }

        fingerprint {
            concatenate_sources => true
            method => "SHA256"
            key => "KEYKEYKEY"
            source => ["ymd","network", "active_time", "ip"]
            target => "log_uuid_fp"
        }

        geoip {
            source => "ip"
            target => "geoip"
            fields => [city_name, region_name,latitude,longitude]
        }

        ruby {
          code=>"
              path = event.get('source').split('/')
              event.set('src_path_prefix', path[-2])
              event.set('src_path_filename', path[-1])
          "
        }

        if !("_grokparsefailure" in [tags] or "_groktimeout" in [tags]){
          if [type] != "foo_bar_type"{
            ruby {
              code=>"
                log_time = Time.local(event.get('year'), event.get('month'), event.get('day'), event.get('hour'), event.get('minute'), '0').to_i
                event.set('log_time', log_time)
              "
            }
          }
          if [type] == "foo_bar_type"{
            ruby {
              code=>"
                require 'date';
                year = Date.parse(event.get('ymd').to_s).strftime('%Y')
                month = Date.parse(event.get('ymd').to_s).strftime('%m')
                day = Date.parse(event.get('ymd').to_s).strftime('%d')
                log_time = Time.local(year, month, day, event.get('hour'), 0, 0).to_i
                log_ym = Time.at(log_time).strftime('%Y%m').to_i
                event.set('log_time', log_time)
                event.set('log_ym', log_ym)
              "
            }
          }
          date {
              match => ["log_time", "UNIX"]
              target => "@timestamp"
          }
        }
    }
  }
}

```
## 输出
常用的logstash输出插件:  
logstash-output-file:输出到本地文件  
logstash-output-jdbc:插入(`INSERT`)/更新(`UPDATE`)支持jdbc的关系型数据库(**注意!这个需要另外安装,非自带**)  
REF:[Github:logstash-output-jdbc](https://github.com/theangryangel/logstash-output-jdbc)

logstash-output-kafka:输出到Kafka  
logstash-output-elasticsearch:索引到ES  
logstash-output-webhdfs:写到hdfs  

```ruby
output{
  if [log_group] == 'foo_bar_group' {
    if "_grokparsefailure" in [tags] or "_groktimeout" in [tags]{
      file {
        path => "/path_to_error_log/%{src_path_}/%{src_path_filename}"
        codec => line { format => "%{message}"}
      }
    } else {
      if [type] == "foo_bar_type"{
        jdbc {
          connection_string => "jdbc:mysql://db_host:3306/for_bar_db?user=username&password=password&useUnicode=true&characterEncoding=UTF-8&autoReconnect=true"
          unsafe_statement => true
          max_flush_exceptions => 10
          retry_initial_interval => 2
          retry_max_interval => 128
          retry_sql_states => ["100","HY00","2015"]
          statement => ["INSERT INTO for_bar_%{ymd} (log_uuid_fp,log_uuid,log_time,ymd,network,active_time,ip,longitude,latitude,region_name,city_name,source_host) VALUES(?,?,?,?,?,?,?,?,?,?,?,?)","log_uuid_fp","log_uuid","log_time","ymd","network","active_time","ip","geoip[longitude]","geoip[latitude]","geoip[region_name]","geoip[city_name]","beat[hostname]"]
        }
        kafka {
          codec => json
          bootstrap_servers => "100.200.200.100:9093","100.200.200.101:9093","100.200.200.102:9093"
          topic_id => "for_bar_json"

          # 如果输出到的Kafka开启了安全认证,就需要配置下面这些参数
          # sasl_mechanism => "ONS"
          # jaas_path => "/path_to/jaas.conf"
          # ssl_truststore_password => "KafkaOnsClient"
          # ssl_truststore_location => "/path_to/kafka.client.truststore.jks"
        }
        elasticsearch {
          codec => "json"
          hosts => ["es_host:9200"]
          user => "user_name"
          password => "password"
          index => "foo-bar-%{+YYYY.MM}"
          document_type=>"foobar_doc_type"
          manage_template => true
          template_name => "foo-bar"
          template_overwrite => true
          template => "/path_to/for_bar.json"
        }
      }
    }
  }
}
```

ES模板:
1.一旦索引根据模板生成索引，那么即便修改模板内某字段类型，也不会影响索引的该字段类型，除非重建索引，或者删除重新生成;  

2.模板有优先级,比如"foo-bar-2019.04"这个索引,模板"for-bar-\*","foo-\*"都适用，那么就会选择优先级高的模板套上去，如果优先级一致，就会随机选一个套上去  

3.关于shards,replicas以及字段类型等ElasticSearch相关的配置可以看到下面的ES篇，这里就不展开叙述,此处只是补充logstash-output-elasticsearch配置中template指向的模板配置  

```json
{
  "template": "for-bar-*",
  "settings": {
    "index.refresh_interval": "2s",
    "number_of_shards": "3",
    "number_of_replicas": "1",
    "max_result_window": "10000"
  },
  "mappings": {
    "info": {
      "properties": {
        "log_uuid_fp": {
          "type": "keyword"
        },
        "log_uuid": {
          "type": "keyword"
        },
        "log_time": {
          "type": "integer"
        },
        "ymd":{
          "type": "integer"
        },
        "network":{
          "type": "keyword"
        },
        "active_time":{
          "type": "integer"
        },
        "ip":{
          "type": "ip"
        },
        "longitude":{
          "type": "float"
        },
        "latitude":{
          "type": "float"
        },
        "region_name":{
          "type": "keyword"
        },
        "city_name":{
          "type": "keyword"
        },
        "source_host":{
          "type": "keyword"
        }
      }
    }
  }
}
```

P.s.
经常遇到一个问题就是，和同事上传logstash新配置如果有问题，会影响线上其他logstash正常的配置,logstash(6.0+) mutiple pipeline,让pipeline之间不互相影响，这个待测试?


# 队列  

## Redis  

- Message Queue(lpush + brpop with timeout)

Redis除了做主要作用是缓存(在缓存上对比也是内存型的Memcached只有KV一种类型,而Redis支持的类型则丰富得多)以外也可以做简单的消息队列(Message Queue),Redis做消息队列一般都是用list数据类型，用Redis做消息队列看中的是Redis的简单易用，而且速度快，然而Redis的list一旦有消费者消费了一条信息，那么这条信息就没有办法让其他消费者消费了，但是Kafka可以通过不同的group_id来给不同的消费者消费  

Redis本身比较适合缓存，针对消息队列，Redis作者另外开了个项目Disque  

顺便简单扯点Redis缓存问题:  
缓存问题本质上就是数据库压力问题，尽量不要让本来在缓存的压力达到数据库  
1.缓存穿透(查询不存在key)  
解决方案:
(缓存方案)如果查询不存在key，返回null  
(过滤方案)bloom filter  
2.缓存击穿(多线程查key,key失效,并发打到数据)  
解决方案:mutex后做缓存，解决db并发思路一致  
3.缓存雪崩(redis cluster不可用，本质上违背了高可用)  
解决方案:本地缓存+限流  

ACK机制:  
**RabbitMQ**
1.Publisher把消息通知给Consumer，如果Consumer已处理完任务，那么它将向Broker发送ACK消息，告知某条消息已被成功处理，可以从队列中移除。如果Consumer没有发送回ACK消息，那么Broker会认为消息处理失败，会将此消息及后续消息分发给其他Consumer进行处理(redeliver flag置为true)。
2.这种确认机制和TCP/IP协议确立连接类似。不同的是，TCP/IP确立连接需要经过三次握手，而RabbitMQ只需要一次ACK。
3.值的注意的是，RabbitMQ当且仅当检测到ACK消息未发出且Consumer的连接终止时才会将消息重新分发给其他Consumer，因此不需要担心消息处理时间过长而被重新分发的情况。

**REDIS**
1.维护两个队列:pending队列和doing表(hash表)。
2.workers定义为ThreadPool
3.由pending队列出队后，workers分配一个线程（单个worker）去处理消息——给目标消息append一个当前时间戳和当前线程名称，将其写入doing表，然后该worker去消费消息，完成后自行在doing表擦除信息
4.启用一个定时任务，每隔一段时间去扫描doing队列，检查每隔元素的时间戳，如果超时，则由worker的ThreadPoolExecutor去检查线程是否存在，如果存在则取消当前任务执行，并把事务rollback。最后把该任务从doing队列中pop出，再重新push进pending队列
5.在worker的某线程中，如果处理业务失败，则主动回滚，并把任务从doing队列中移除，重新push进pending队列

- Pub/Sub

当然Redis除了list以外还有Pub/Sub，这种模式是MessageQueue的兄弟，它和观察者设计模式极为类似，支持多消费者的模式。Publish的消息在Redis中不存储，因此必须先执行订阅再等待消息发布，如果消息订阅者中途加入订阅，那么通道中此前的消息将无从获得。  

Publishers =(push)=> Channels =(push)=> Subscribers  

发布者和订阅者之间解耦可以带来更好的扩展能力和更灵活的网络拓扑。  
Pub/Sub下通道名称是全局的,和客户端连接的Redis数据库没有关系，比如你在db10 发布到channel，在db1订阅这个channel的消费者也可以拿到数据。 

关于Redis持久化:  
1.RDB 持久化可以在指定的时间间隔内生成数据集的时间点快照(point-in-time snapshot)  
2.AOF 持久化记录服务器执行的所有写操作命令，并在服务器启动时，通过重新执行这些命令来还原数据集  
3.Redis可以同时使用AOF持久化和RDB持久化。在这种情况下，当Redis重启时， 它会优先使用AOF文件来还原数据集,因为AOF文件保存的数据集通常比RDB文件所保存的数据集更完整  
4.RDB在恢复大数据集时的速度比AOF的恢复速度要快  
REF:http://redisdoc.com/topic/persistence.html  

## Kafka  

Redis其实不适合正儿八经拿来当消息队列的，一些基本的要求比如顺序保证，EOS(Exaclty Once Semantics)语义，数据可靠性(就上面介绍的无论是list还是pub/sub都是即发即失)等都没有，我们如果要正儿八经的消息队列，可以看看RabbitMQ,Kafka  

相对于Redis，Kafka提供了更多的保证，比如0.10.0的at-least-once到0.11.0的exactly-once,以及内部包含了ZooKeeper，让它具备了强数据一致性，高可用的特性。虽然使用了ZooKeeper,但是Kafka所使用的leader选举算法不是ZK的ZAB，而更像是微软的PacificA算法  

和Redis的Pub/Sub主动消息推给消费者不同，消费者从kafka中pull数据，因为如果是由kafka主动推给消费者，容易造成消费者负载突然增高  

kafka是MessageQueue和Pub/Sub的集合体,通过consumer_group和offset保证消费者能任何时候从任何位置开始读取topic的消息;
通过partitions分区保证分区内的消息是有序的,并能提升并发消费能力;
通过副本(replica)带来容错(fault-tolence);
通过优化过的写入磁盘策略(structured commit-log storage)让它和redis相比能更久地保存更多的消息，而且天然地支持持久化,一般可以设置为保存长达2周的消息

Kafka 0.11.0实现了EOS语义:  
在0.11.0之前Kafka只是支持at-least-once，不能保证不重复，只能保证不丢(生产者设置request.required.acks=1/0/-1)，如果想要系统EOS，那么就必须在系统层面在下游做去重  

Kafka的ack机制:  
当 producer向leader发送数据时，可以通过request.required.acks参数来设置数据可靠性的级别：
1(default):这意味着producer在ISR中的leader已成功收到的数据并得到确认后发送下一条message。如果leader宕机了，则会丢失数据。
0:这意味着producer无需等待来自broker的确认而继续发送下一批消息。这种情况下数据传输效率最高，但是数据可靠性确是最低的。
-1:producer需要等待ISR中的所有follower都确认接收到数据后才算一次发送完成，可靠性最高。但是这样也不能保证数据不丢失,比如当ISR中只有leader时(前面 ISR那一节讲到，ISR中的成员由于某些情况会增加也会减少，最少就只剩一个 leader),这样就变成了acks=1的情况。

EOS是流式处理实现正确性的基石,主流的流式处理框架基本都支持EOS(如Storm Trident, Spark Streaming, Flink),Kafka streams肯定也要支持的。  
0.11版本通过3个大的改动支持EOS:
1.幂等的producer(这也是千呼万唤始出来的功能)
2.支持事务;
3.支持EOS的流式处理(保证读-处理-写全链路的EOS)

Kafka设计:
1.PageCache:
aka DiskCache,rather than maitain as much as possible in memory,and flush it all at once to the filesystem in a panic when run out of space, we invert that:All data is immediately written to a persistent log.

2.Small IO operations and excessive byte copying(low meesage rate) by using sendfile()

关于ZooKeeper:
ByzantineFault数据一致性问题(Paxos/ZAB/Raft)
分布式锁
分布式事务(solution:2pc,3pc,tcc)

# ElasticSearch  
本质上是Apapche Lucene，只是在其之上提供了RESTful API以及watcher监控套件
master_node / data_node  
2节点的脑裂(brain-split)问题  
shards & replica  
keyword vs text  
倒排索引原理
字段数据类型
ES-DSL
ES在处理节点发现与Master选举等方面没有选择Zookeeper等外部组件，而是自己实现的一套

ES-head:
```shell
cd /data/joygames/elasticsearch-head
npm run start &
```

ES sql