---
published: false
category: QA
title: 调研:数据质量工程
author: smona
date: '2023-02-09 23:49:00'
layout: post
---

- [背景](#背景)
- [自研](#自研)
- [开源](#开源)
- [云商\[推荐\]](#云商推荐)
- [附录](#附录)


# 背景
1.每次埋点上线都会遇到人工核对日志字段低效的问题？ => 数据测试自动化
2.上线以后，会出现字段内容变化导致统计指标错误？=> 数据监控

# 自研
《有赞 埋点质量保障》 https://tech.youzan.com/mai-dian-zhi-liang-bao-zhang/
> "对于实时性，我们采用「Flink开发校验模块，实现秒级日志校验」；校验规则更新的及时性上，每分钟从埋点平台同步；可沉淀，校验结果除了推送给测试工具外，还会落到druid，用于后续分析"

《字节跳动 如何保障埋点数据的准确性？》https://k.sina.com.cn/article_2674405451_9f68304b019012zqy.html?from=tech
> 上报前通过注册埋点的信息开发完埋点后用验证工具去做「自动化测试」以保障埋点开发的准确性和质量，上报后我们也会用一些离线工具对埋点质量进行监控，但从我们的经验上来看，上报前埋点质量的把控会解决大部分的问题，上报后的埋点质量治理目前主要是离线的，也能解决一些场景的问题。

> 目前我们的处理链路是「基于 Flink 实现的」，大多数场景是没有开启 checkpoint，原因是我们的下游需求很多样化，有的下游不接受数据的大量重复，有的下游不接受数据延迟，如果开 checkpoint，会在任务 Failover 过程中有大量的数据重复和延迟，这个是下游没有办法接受的。我们通过 SLA 数据质量监控指标是可以观测出整个链路各个环节的「数据丢失率和重复率」，通过 SLA 指标来反映和保障数据准确性。

《知乎 埋点测试平台》 https://blog.51cto.com/u_15471709/4875862
> 埋点的质量是数据的生命线，一旦出现问题，则会导致整条大数据链路的数据价值出现问题。埋点异常不但影响决策，修复数据同样会消耗大量的精力和时间，最直接的后果就是虽然数据量越来越大，数据本身却无法有效的使用。

有空闲时间，推荐扩展阅读：
1.埋点实践 https://tech.youzan.com/track-1/
2.数据资产治理 https://tech.youzan.com/shu-ju-zi-chan-zan-zhi-zhi-li/
3.离线数据降本 https://tech.youzan.com/cong-liang-hua-dao-you-hua-xiang-jie-you-zan-chi-xian-shu-ju-jiang-ben-zhi-lu/

# 开源
1.Great Expectation【Python】
https://github.com/great-expectations/great_expectations

2.微众 Qualitis【Java，基于Spring Boot，Qualitis对它自家Linkis的依赖，灵活性要差一些，要用就得用全套的】
https://www.infoq.cn/article/gf5dydmv0eqyky90ylho 《微众银行如何在小团队规模下炼出一套一站式大数据平台》
https://github.com/WeBankFinTech/Qualitis/blob/master/README_zh.md

3.Apache Griffin【Java&Scala，Griffin是大数据质量监控领域唯一的Apache项目】
https://www.cnblogs.com/tree1123/p/16481069.html
https://www.cnblogs.com/tree1123/p/16489327.html 《数据质量管理工具预研——Griffin VS Deequ VS Great expectations VS Qualitis》

# 云商[推荐]
鉴于在用的ADB和DLA、Kafka都是阿里云的，所以首先调研阿里云的产品

1.[阿里云 DataWorks 数据质量CQC](https://help.aliyun.com/document_detail/73660.html)
费用：

[MaxCompute使用DataWorks的数据质量CQC监控](https://help.aliyun.com/document_detail/116897.htm?spm=a2c4g.11186623.0.0.2369b338MSwYFP#concept-221932)


# 附录
2022 数据工程技术图 https://www.cnblogs.com/tree1123/p/16520058.html
![](https://lakefs.io/wp-content/uploads/2022/06/State-of-Data-Engineering-2022-map-1920x1080_31.7-2048x1152.jpg)