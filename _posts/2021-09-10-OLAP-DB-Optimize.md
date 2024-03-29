---
published: false
category: DB
title: OLAP型数据库的CPU资源优化思路     
author: smona
date: '2021-09-10 20:49:00'
layout: post
---

- [前言](#前言)
- [确认问题](#确认问题)
- [观察监控](#观察监控)
- [作出假设](#作出假设)
- [两个解法](#两个解法)
  - [挤掉总计算量上的水分](#挤掉总计算量上的水分)
  - [削峰填谷](#削峰填谷)
- [效果](#效果)
- [附录](#附录)
  - [接口的ad-hoc和报表的查询](#接口的ad-hoc和报表的查询)
  - [优化磁盘存储](#优化磁盘存储)

# 前言
OLAP型数据库的CPU资源优化思路，之前只做过MySQL这种OLTP数据库的SQL优化，这次遇到OLAP的CPU太高的问题就顺手记录下来  

# 确认问题

问题表象:  
CPU使用率很高(长时间>80%，偶尔出现峰值100%)导致一些页面很简单查询也会出现数据库无法响应?「BI概览页的数据库错误问题」  

进一步提问:  
1.是否真的当前的计算量的确有那么高？如果是，那么是否需要增加算力？  
2.现在的查询和插入操作是否还有优化空间，比如历史新增之类的计算转换为中间表，是否存在算子之间重复计算的情况？  

----

# 观察监控

现在有的监控指标数据:  
1.CPU使用率  
2.每分钟插入次数  
3.每分钟查询次数  
4.查询耗时  
x5.磁盘使用率  
6.TOP10耗时SQL  
x7.TOP5表空间  
x8.表分区数据倾斜  

这次问题可以用的监控数据除了磁盘使用率、TOP5表空间以及表分区数据倾斜以外都用上了  
磁盘使用率、TOP5表空间以及表分区数据倾斜在之前「优化磁盘存储」(见附录)，切割冷数据到离线库DLA的时候用到了  

监控数据解读:  
1.CPU使用率 => 看不出是哪个查询引起的，光是看这个没办法找到高CPU占用原因，我们需要知道每个查询占用了多少  
2.每分钟插入次数 => 插入后的索引会导致CPU使用率上升，但是插入后并不是实时反映到CPU上升上，会有一段时间延时  
3.每分钟查询次数 => 查询次数大可能会导致CPU使用率高，但是并非绝对，如果都是简单查询倒不会导致CPU升高，反倒是就算是只有一条SQL，但是这一条SQL运算量特别大的会导致CPU升高  
4.查询耗时 => 查询耗时和慢查询一样，可能是高磁盘IO引起或者高计算量引起的  
6.TOP10耗时SQL => 慢查询可能是高磁盘IO引起或者高计算量引起的  

根据监控解读可知，并不能从这一堆监控数据中一眼就看出哪些查询是需要优化的，其实我们是希望像Linux `ps`命令一样，可以一眼就看出每个进程消耗的CPU，内存的量是多少，如果可以有ADB `ps`这样类似的工具就好了(ADB3.0有，2.0版本是没有的)，如果有这种ADB `ps`工具看得出每一个查询的CPU和内存占用那就可以针对TOP N的查询优先做针对性优化，而不用凭借经验来猜哪些很有可能需要优化  

----

# 作出假设

那既然没办法像`ps`那样一眼就看出TOP N最消耗计算资源的查询的计算资源占用情况，那我们就只能靠经验做假设了  

凭借对项目的经验，我们判断造成CPU高可能原因:  
1.离线统计项目里面的离线算子查询？ => 亲自排查  
- 非常有可能
- 首先，因为离线算子基本上都是一些留存、LTV等非常复杂，且有大量JOIN连表和GROUPBY分组操作的运算，所以很容易造成CPU高的情况
- 其次，就是从CPU监控角度来看，占用率是「一直」维持高占用状态，很符合我们离线算子查询的定时任务的设定，因为我们定时任务是每小时计算一次
- 终上所述，可以放在第一个来排查

2.接口的ad-hoc和报表的查询？ => 同事排查(见附录)  
- 可能性中等，因为不会呈现一直CPU高的情况，可以在离线算子查询排查后再排查，原因如下
- 首先，我们是TOB应用，如果是这块导致的问题，那么在下班后时间内应该会有下降，但是监控并没有呈现上班时间段占用高，而下班时间段内CPU占用的下降
- 其次，ad-hoc和报表的查询都是不定时，如果有用户点开报表或者提交SQL查询才会开始占用CPU资源，所以如果是这个原因引起的，那么CPU资源升高应该是「一阵一阵」的，而不是长时间一直的70%-80%的高占用状况

3.ETL配置项目里面的Logstash的实时写入以及写入后的索引导致? => 1和2以后看看效果再考虑
- 可能性比较低，按照经验一般无论是慢查询还是现在这种CPU高的原因都会是查询引起的，而极少是插入引起的，所以我们可以放到最后来排查  

----

# 两个解法

想起之前做过一次优化，但是那是对接口的优化，也就是用户体验的优化，通过「缓存」以及「中间表预计算」俩办法，让接口的查询直接读数据拿结果，而不用做多余的计算，所有耗时的计算都搬到了预计算去了。虽然用户体验是快了，但是对OLAP库的压力有增无减，而这一次就是在维持用户体验还是那么快的同时，还需要把OLAP库的压力降下来！  

因此我们从「挤掉总计算量上的水分」以及「削峰填谷」  两条路来解决这个「OLAP库的压力降下来」问题

## 挤掉总计算量上的水分
压榨每个查询计算量  
=> 如果当初编写的时候按照最佳实践来写的话，一般可以压榨的空间不大，而且工程量比较大，修改查询很容易导致算法出错，需要反复测试，但是是唯一可以降低总计算量的方法  

查询编写俩原则「简单」「减少数据加载」  
(0)从TOP10耗时SQL开始;  
(1)去掉不必要加载SELECT字段;  
(2)去掉不必要的JOIN联表/GROUP-BY分组/ORDER-BY排序/LIMIT分页，这些都非常吃计算资源;  
(3)检查算子之间是否存在的「重复」计算，考虑用中间表代替(由于原本开发可能赶工期，后期优化只是做了部分，所以会导致比如有些中间表只用在了部分算子上，还有部分算子没用上的情况);  

## 削峰填谷
调整算子执行频率  
=> 和压榨每个查询计算量不同，在任务调度系统上调整执行频率不会破坏原本算法的正确性，但是也只能让CPU占用更加平均，降低出现极端峰值的几率，本质上并没有降低总的计算量，只是起到「削峰填谷」的效果，适合CPU占用监控数据出现多峰多谷的情况  

(1)确定每个游戏业务能忍受的最长更新频率，基于这个极限来规划离线任务的执行频率  
一般地
「前期」刚上线的游戏需要更频繁更新但是计算数据量不会很大  
「中期」稳定的游戏更新频率不会像新上线的游戏那样需要频繁更新但是计算量会变大  
「后期」衰退时候会接受T+1的更新但是计算数据量会变得很大  
(2)最好每个小时不要塞两个游戏的算子簇  

# 效果
一.挤掉总计算量上的水分  

重新检查我们原本的一些算子发现历史新增(i.e. 以前没有，第一次出现)计算类的算子之间存在比较大的计算重复，而且存在扫描全表去确定是否在历史上存在，在查询维度造成很多不必要的扫描，因为我们在业务上只是想要历史上首次的记录，而事实表里面除了首次记录还有后面每次出现该行为的记录，所以我们决定新增中间表去挤掉事实表上的"水分"，并把原本相关计算的算子都切换到依赖这张表计算，去除算子之间对「历史是否存在」这一全表扫描行为的重复计算  

1.新老玩家留存  
jobs.retention_player.retention_player 14s => 10s(-28.57%)  
jobs.retention_player_no_server.retention_player_no_server 12s => 6s(-50%)  

2.剔除广告 新玩家留存  
jobs.retention_player_without_ad.retention_player_without_ad 7s => 4s(-42.86%)  
jobs.retention_player_no_server_without_ad.retention_player_no_server_without_ad 6s => 3s(-50%)  

3.LTV
jobs.ltv_no_server.ltv_no_server 89s => 57s(-35.95%)  
jobs.ltv_no_server_without_ad.ltv_no_server_without_ad 93s => 47s(-49.46%)  

4.新增充值玩家留存 => 考虑到充值表比较少，而且用到的地方不多，判断优化价值不大，看下上面三个算子计算的优化能带来多少效益，再结合同事对接口的ad-hoc和报表的查询的优化再决定  

上述可以看出去除算子之间对「历史是否存在」这一全表扫描行为的重复计算，可以把查询时间基本都是优化幅度在28 ~ 50%，因为没办法追踪单条SQL的CPU和内存的计算资源使用情况，只能通过查询耗时推断，查询耗时越低，资源使用情况越少  

二.削峰填谷  
针对ADB上的两个游戏重新规划了在任务调度平台上的计算频率  

三.「离线统计项目里面的离线算子查询」以及「接口的ad-hoc和报表的查询优化」后监控结果
优化前:

优化后:


---

# 附录
## 接口的ad-hoc和报表的查询
优化基础数据模块SQL，两类特征，一个是对全量计算优化，中间表去除重复计算；另外一个是对循环查询优化，原本写法是PHP中的循环拼接出多次IO查询的数据，现在通过SQL DATEDIFF这种实现去除多次IO往返，变成1次IO即可  
1、全量计算 ：滚服、新玩家注册  
2、循环查询：流失分布、回流统计、首充分布、持续付费  


## 优化磁盘存储
1.关于ADB存储表空间分析，得出TOP3的三张表的冷数据迁移切到DLA结论  

2.关于DLA的查询速度优化Parquet和ORC两种新格式的对比，原本csv文件查询要13s，不满足查询速度要求，新格式可以去到1s内  