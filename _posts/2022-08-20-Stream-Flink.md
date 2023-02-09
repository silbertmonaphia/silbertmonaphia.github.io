---
published: false
category: DB
title: 调研:流计算Flink
author: smona
date: '2022-08-20 23:49:00'
layout: post
---

- [需求场景](#需求场景)
  - [广告点击流](#广告点击流)
  - [用户画像事件流](#用户画像事件流)
- [实时数仓架构](#实时数仓架构)
- [流计算选型](#流计算选型)
- [成本](#成本)
  - [一.云商费用](#一云商费用)
    - [包年包月](#包年包月)
    - [按量付费](#按量付费)
  - [二.学习成本](#二学习成本)
- [测试](#测试)
  - [过程](#过程)
  - [问题](#问题)
    - [1.Flink Checkpoint需要配置么？Checkpoint作用是什么？](#1flink-checkpoint需要配置么checkpoint作用是什么)
    - [2.Flink JobManager和TaskManager是什么？并发度又是什么？](#2flink-jobmanager和taskmanager是什么并发度又是什么)
    - [3.Session集群和Per-Job集群之间有什么区别？怎么选择？](#3session集群和per-job集群之间有什么区别怎么选择)
    - [※4.那我怎么知道上线的FlinkJob有没有拉取source数据？sink出口没有数据怎么办？Job或者Task执行记录和错误在哪里看？](#4那我怎么知道上线的flinkjob有没有拉取source数据sink出口没有数据怎么办job或者task执行记录和错误在哪里看)
    - [5.做维表JOIN测试时候报错？](#5做维表join测试时候报错)
    - [6.临时表和永久表有什么区别？](#6临时表和永久表有什么区别)
    - [7. 6大窗口和4大JOIN？](#7-6大窗口和4大join)
    - [※8.Flink是怎么控制读取速度的？是一条条还是一批批？作为消费者是pull模式还是push模式？](#8flink是怎么控制读取速度的是一条条还是一批批作为消费者是pull模式还是push模式)
  - [结论](#结论)
- [参考](#参考)

# 需求场景
## 广告点击流
20220420
在「广告投放系统P0需求」评估中发现的，我们原本的SQL批处理计算方式(传统方式)会存在**重复计算**的数据，本质上广告场景下是希望通过**事件驱动**来更新指标才最贴近需求，这个就很符合流计算模型。  

所以数据组评估按照优先级，得一个月后才开始开发，但是对团队并没有半点积累的「流计算」，却最好先花2天时间去学习、调研、理解，看下会不会和自己想象中的存在偏差，毕竟「很符合流计算模型」只是存在脑海中。  

虽然用以前传统SQL批处理方法，然后把任务执行时间调整到每秒执行一次也能实现(保底方案)，但是流计算整个思维模式带来了一个创新，如果是有生产力提升的可能为何不试一下呢？  

## 用户画像事件流
20220714
用户属性值的最早fisrt、最近latest以及累计accumulate，分别用MIN、MAX、ON DUPLICATE UPDATE实现

# 实时数仓架构
![](https://help-static-aliyun-doc.aliyuncs.com/assets/img/zh-CN/1709238361/p97864.png)
P.S.上图中的维度表一般表示为dim，即dimension，一般作为翻译事件事实表中字段用，存放映射关系，比如存放渠道ID和渠道中文名字  
[阿里云-基于Flink的资讯场景实时数仓](https://help.aliyun.com/document_detail/161654.html)

数仓分层设计：自底向上为
=> ODS(什么都不处理，原汁原味)[最底层]
=> DWD(Data Warehouse Detail，只做去重、去空等简单的清洗。对应我们的各个事件事实表)
=> DWB(Data Warehouse Base，简单聚合和统计计算，目的是复用。对应我们的中间表mid_xxx)
=> DWS(Data Warehouse Service，为最终业务设计，方便快捷地让业务查询。对应我们的统计表stats_xxx)[最上层]

# 流计算选型
流计算当前业界流行的选择有几个：
[推荐]1.Flink(业界De-facto，真流式，基于事件驱动。2019年被阿里以9000万欧元约合1.03亿美元收购了其德国柏林的母公司DataArtisans) => 延时毫秒内的选择，对SQL支持不完善，但是不需要被Java/Python语言包裹，可以直接写纯SQL，国内使用选型较多
[备选]2.SparkSteaming(伪流式，用极小时间窗口做微批处理以逼近流式计算，优势在于原本有Spark集群的前提下可以直接使用) => 延时可以接受秒以上的选择，对SQL支持完善，但是需要被Java/Scala/Python语言包裹，海外使用选型较多
[x]3.Storm(和Flink一样也是基于事件驱动，但是Storm低级的API，不支持sql ,吞吐量、容错性、准确性问题，对开发不太友好)
[x]4.KafkaStreaming(仅仅是提供了简单的Client Library计算能力，只是想简单在Kafka基础上扩展简单计算可选)

P.S. Spark作为老前辈和Flink可以说是现在为止大数据的计算框架上唯二的选择，一开始是Flink抄Spark，然后以流计算去冲击Spark做的不好的Spark Streaming，后面Spark意识到了就发力做出了Spark Streaming的升级版Structured Streaming。除了底层Flink是用流做批，而Spark是用批做流的差别外，上层接口设计和功能特性已经越发靠近和相似。预言一波最后Spark+Flink会在某个新产品上完成融合

# 成本
## 一.云商费用
@阿里云Flink
一个月免费体验的是10CU=10CPU + 40GB内存，包年包月1800元/月，按量付费是2736元/月 => 体验过程中发现Flink中JVM比预想更吃资源，所以可能4CU无法满足，得10CU以上

1 CU=1核CPU+4 GB内存
简单业务:例如，单流过滤、字符串变换等操作。1 CU每秒可以处理30000~40000条数据。
复杂业务:例如，JOIN、GROUP BY或窗口函数等操作。1 CU每秒可以处理5000~10000条数据。

### 包年包月
一个实例的总价`=` `(`管控资源CU数`+`计算资源CU数 `)` `*` 单价 `*` 购买时长
注意: 管控资源CU数固定为2 CU
例: 以北京地域为例(华东2-上海，相同)，单价为180元/CU/月，创建一个实例需管控资源2 CU，计算资源2 CU，创建时长为1个月，则创建一个实例所需的费用为：（2 CU+2 CU）`*`180元/CU/月`*`1个月=720元/月

### 按量付费
一个实例的总价`=` `(`管控资源CU数`+`计算资源CU数`)` `*` 单价
注意: 管控资源CU数固定为2 CU
例: 以北京地域为例(华东2-上海，相同)，创建一个实例需管控资源2 CU，计算资源2 CU，创建时长为200个小时，则创建一个实例所需的费用为：（2 CU+2 CU）`*` 0.38元/CU/小时 `*` 720个小时(1个月30天，每天24小时)=1094.4元/月

## 二.学习成本
1.「流计算」和传统「批处理」区别在[官方文档-UserCase](https://flink.apache.org/zh/usecases.html)说的比较清楚了：简单来说就是
「流计算」是被动的事件触发所以可以做到微秒级别的延迟，而且没有事件的时候避免了批处理时候的空转；
「批处理」则是主动发起的查询一批的数据延迟都是秒起步，一般查询历史较大范围的数据，所以一般都是分钟到小时的任务  
2.初级使用:[Flink SQL](https://help.aliyun.com/document_detail/184279.html)
3.高级使用:Table API & DataStream/DataSet API(Java/Scala)
4.理解两大核心概念: [State-Checkpoint](https://nightlies.apache.org/flink/flink-docs-release-1.15/docs/concepts/stateful-stream-processing/)和[EventTime-Watermark](https://nightlies.apache.org/flink/flink-docs-release-1.15/docs/concepts/time/#event-time-and-watermarks)。

「关于Watermark」，简单来说就是基于EventTime计算，往前倒推N的时间以前的数据可以认为是全部事件都到达了Flink，这个概念是为了解决不知道要等待还未到达的事件到什么时候才可以触发计算，现在规定了N，换言之就是只等待事件数据N的时间，然后马上触发计算，如果仍然还有该时间前事件没到也不要了。Watermark的设计难点在于这个N要等多久，5s？10s？20s？

「关于内存资源」，内存的分配管理对性能也有重要影响，JVM垃圾回收在给开发人员带来便利的同时，也制约了内存的有效利用。另外，Java的对象创建及序列化也比较浪费资源。在内存优化方面做足功夫的代表是Flink。出于性能方面的考虑，Flink很多组件自行管理内存，无需依赖JVM垃圾回收机制。Flink还用到开辟内存池、用二进制数据代替对象、量身定制序列化、定制缓存友好的算法等优化手段。Flink还在任务的执行方面进行优化，包括多阶段并行执行和增量迭代。
REF：https://www.zhihu.com/question/37026972/answer/241772836

# 测试
调研后觉得实现的可能，可以上Flink，但是得待实际用「阿里云Flink」测试后，在评估各种成本，比如带入新技术的维护成本以及投入后产出的多寡  

## 过程
since 20220816

感悟一：
测试Flink过程中发现Flink自身去实现复杂逻辑比如更新update结果表只用SQL是很难实现的，需要下一层的DataStream接口才比较好实现，然后上网查询时候，想起阿里云DLA除了带了我们一直在用的「Serverless Presto」以外，还自带了[「Serverless Spark」](https://help.aliyun.com/document_detail/124303.html)，Spark也有流处理服务，而且Spark SQL比Flink SQL更适合做复杂动作；

Spark和Flink也是当前大数据计算唯二两个代表，前者代表了经典的批处理思维，后者代表的是流计算思维，现在两者都有相互融合成为「批流一体」的趋势

故有以下疑问：阿里云Flink SQL vs 阿里云 DLA Serverless Spark？

主要区别还是开发体验上，阿里云 DLA Serverless Spark需要用Scala/Java/Python包裹SQL实现，而Flink SQL只需要纯SQL即可，所以从后期团队维护成本来说，流计算场景下Flink会比Spark更好

感悟二：
1.Flink配置很复杂无法按照默认的开箱按照以往的思路写写SQL即用，调试很麻烦(默认配置下TaskManager运行日志输出一大堆，虽然黄色是warn，红色是error，但是还是得翻好久，阅读不友好)，哪怕是云化了的Flink，其文档也不完备有时候需要结合云文档和开源文档一起看，比如TEMPORARY表 JOIN维表必须得带上`FOR SYSTEM_TIME AS OF PROCTIME()`，这个这么重要的信息在[文档](https://help.aliyun.com/document_detail/185225.html?spm=a2cn1.draft.help.dexternal.16146635ws7N6Z)中只有一句轻描淡写的描述(不过Blink旧文档有提到)；
2.Flink就为了快(为了进到毫秒内)，所以牺牲了数据Durability，故数据具备易逝性，业务逻辑一旦修改，之前的数据不可重新计算；
3.Flink的SQL支持比较弱，写SQL需要小心翼翼，一不小心就报错语法不支持；

## 问题
### 1.Flink Checkpoint需要配置么？Checkpoint作用是什么？

Checkpoint是Flink的核心设计，用于解决流计算里面[「状态State管理」](https://zhuanlan.zhihu.com/p/119305376)问题，Flink开源方案中生产环境去维护这个状态的是在一个[RocksDB](https://flink.apache.org/2021/01/18/rocksdb.html)。
Flink在每次Checkpoint成功时，才会向Kafka提交当前读取Offset。如果未开启Checkpoint，或者Checkpoint设置的间隔过大，在Kafka端可能会查询不到当前读取的Offset。现在测试过程中我修改在「作业模板」修改了默认的时间180s压缩到30s，现在直接「作业开发」新建作业就可以用到「作业模板」的我调整过的配置

### 2.Flink JobManager和TaskManager是什么？并发度又是什么？

JobManager是Master节点完成任务管理、分派以及资源分派，TaskManager是Slave节点完成计算任务
并发度，其实应该叫并行度，阿里Flink的概念有点问题。并发度其实就是TaskManager的数量，提高并行度并不会增加JobManager数目，JobManager数目一直都是只有1个

故总的资源使用计算公式如下：
CPU=JobManagerCPU + TaskManagerCPU `*` 并发度
Memory=JobManagerMemory  + TaskManagerMemory `*` 并发度

P.S.研究这个原因是，第一次用免费体验一个月的阿里云Flink提供的10CPU和40Gi的资源，明明作业只是配置1C 2Gi低配置，期望顶多只占用2C 4Gi，作业跑起来却一下子吃完了所有资源，且启动不了作业，这就是没有理解JobManager、TaskManager以及并行度配置错了作业资源导致的

### 3.Session集群和Per-Job集群之间有什么区别？怎么选择？

参考[官方文档-Flink 架构](https://nightlies.apache.org/flink/flink-docs-master/zh/docs/concepts/flink-architecture/#flink-%e6%9e%b6%e6%9e%84)描述

Session 集群中，客户端连接到一个预先存在的、长期运行的集群，该集群可以接受多个作业提交，但是各个任务之间是「共享资源」，不做隔离，所以会有资源竞争情况存在，不建议生产环境采用Session模式。但是它的优势是拥有一个预先存在的集群可以节省大量时间申请资源和启动 TaskManager，所以用在测试「调试场景」非常合适，因为Flink部署一个Job启动真的有点久。

Per-Job集群的话就恰好相反，它能做到各个Job「独立资源」部署，不共享资源，做到任务级别的隔离，所以不存在资源竞争情况，适合「生产环境」部署模式。但是相对的，由于它并不像Session那样预先存在，所以无论是启动、暂停还是停止都需要大量时间申请资源和调动 TaskManager

### ※4.那我怎么知道上线的FlinkJob有没有拉取source数据？sink出口没有数据怎么办？Job或者Task执行记录和错误在哪里看？

在TaskManager有记录完整的运行日志，但是会很多，待解决

### 5.做维表JOIN测试时候报错？

因为是TEMPORARY表，所以JOIN时候少了`FOR SYSTEM_TIME AS OF PROCTIME()`是不行的，`FOR SYSTEM_TIME AS OF PROCTIME()`代表当左表(来源表source)的记录与右表(维表dim)JOIN时，只匹配当前处理时间(PROCTIME())，维表dim所对应的的快照数据

维表可能是会不断变化的，在维表JOIN时，需指明这条记录关联维表快照的时刻。需要注意是，目前Flink SQL的维表JOIN仅支持对当前时刻维表快照的关联(处理时间语义)，而不支持事实表rowtime所对应的的维表快照(事件时间语义)

### 6.临时表和永久表有什么区别？

永久表`Create Table`，需要 catalog（例如 Hive Metastore）以维护表的元数据。一旦永久表被创建，它将对任何连接到 catalog 的 Flink 会话可见且持续存在，直至被明确删除。

临时表`CREATE TEMPORARY TABLE`，通常保存于内存中并且仅在创建它们的 Flink 会话持续期间存在。这些表对于其它会话是不可见的。它们不与任何 catalog 或者数据库绑定但可以在一个命名空间（namespace）中创建。即使它们对应的数据库被删除，临时表也不会被删除。

### 7. 6大窗口和4大JOIN？

滚动窗口、滑动窗口、累计窗口、会话窗口、Over窗口、级联窗口
Regular JOIN、Interval JOIN、时态表 JOIN、维表 JOIN

P.S.可以看看阿里云Flink控制台-应用-模板中心给的例子，比文档提供的样例会更准确一些

### ※8.Flink是怎么控制读取速度的？是一条条还是一批批？作为消费者是pull模式还是push模式？

暂时没有找到答案，待解决

## 结论
当前BI 1.14.0中预估只有3天时间现在看来是不足以实现更新用户画像，测试下来比之前估计的要更复杂，且需求上可以接受小时级别更新，故还是用原本的批处理方法先实现了，流计算后续再以迭代的方式升级

# 参考
1.[开源Flink](https://flink.apache.org/flink-architecture.html)
2.[阿里云实时计算Flink版](https://help.aliyun.com/product/45029.html?spm=5176.15088477.J_5253785160.5.291717083ErDDP)
3.[阿里云数据湖DLA Serverless Spark](https://help.aliyun.com/document_detail/124303.html)
4.[技术选型：为什么批处理我们却选择了Flink](https://www.upyun.com/tech/article/587/%E6%8A%80%E6%9C%AF%E9%80%89%E5%9E%8B%EF%BC%9A%E4%B8%BA%E4%BB%80%E4%B9%88%E6%89%B9%E5%A4%84%E7%90%86%E6%88%91%E4%BB%AC%E5%8D%B4%E9%80%89%E6%8B%A9%E4%BA%86Flink.html)
5.[Flink 在米哈游的应用实践](https://developer.aliyun.com/article/1117565?spm=a2c6h.13148508.setting.14.20ae4f0eGoLvbT)