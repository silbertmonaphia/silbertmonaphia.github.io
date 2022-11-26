---
published: false
category: PM
title: PM数据需求文档模板
author: smona
date: '2021-11-10 20:49:00'
layout: post
---

- [需求文档](#需求文档)
- [问题](#问题)
- [指标](#指标)
- [埋点](#埋点)
  - [事件:滑动-底部 SlideBottom(优先实现)](#事件滑动-底部-slidebottom优先实现)
  - [事件:滑动-滚动一屏 SlideScrollbarFullScreen](#事件滑动-滚动一屏-slidescrollbarfullscreen)
  - [事件:加载-在线统计卡片 LoadOnlineStatsCard](#事件加载-在线统计卡片-loadonlinestatscard)
  - [事件:点击-筛选-进入旧版概况 ClickOldSurveyButton](#事件点击-筛选-进入旧版概况-clickoldsurveybutton)
  - [事件:点击-充值转化卡片-更多 ClickPayConversionCardMore](#事件点击-充值转化卡片-更多-clickpayconversioncardmore)
  - [事件:悬停-新增玩家卡片同比-小问号 HoverCreateRoleTBCardExplanation](#事件悬停-新增玩家卡片同比-小问号-hovercreateroletbcardexplanation)
- [参考](#参考)
  - [产品-埋点命名设计](#产品-埋点命名设计)
  - [前端-上报数据](#前端-上报数据)
  - [后端-存储事件数据模型](#后端-存储事件数据模型)


在大部分互联网公司，规范的产品迭代流程是，业务侧产品经理在输出“产品需求文档（PRD）”的同时，数据产品经理或分析师等角色需要同步输出 DRD，双方的需求同步进入开发和测试验收。
REF: [神策文档-采集方案设计](https://manual.sensorsdata.cn/sa/latest/%E9%87%87%E9%9B%86%E6%96%B9%E6%A1%88%E8%AE%BE%E8%AE%A1-22249882.html#id-.%E9%87%87%E9%9B%86%E6%96%B9%E6%A1%88%E8%AE%BE%E8%AE%A1v2.1-%E4%BB%80%E4%B9%88%E6%98%AF%E9%87%87%E9%9B%86%E6%96%B9%E6%A1%88%E8%AE%BE%E8%AE%A1)

# 需求文档
BI-1.14.3-概览页提高信息密度-产品需求文档PRD。地址：

# 问题
(基于产品需求，如何确定迭代是有价值的)

[功能问题]1.相比旧版本概览页，用户会更喜欢新版本概览页？
[相关用户行为]：点击进入旧版本概览页，点击进入新版本概览页
产品假设：点击「进入旧版概况」的行为每天不超过10，且逐渐随着天数下降直到0

[功能问题]2.新概览排版是否满足一屏看完的需求，信息密度是否足够还是说太高了？
[相关用户行为]：下滑一屏幕、点击「更多」跳转到详情页
产品假设：

[功能问题]3.概览页新增了「充值转化」卡片，充值转化页面环比访问人数是否减少？
[相关用户行为]：访问「充值转化」功能
产品假设：

# 指标
(基于功能问题和用户行为，倒推出指标)

1.相比旧版本概览页，用户会更喜欢新版本概览页？
点击进入旧版本概览页 => 点击「进入旧版概况」次数/浏览页访问人数；
点击进入新版本概览页 => 点击「进入新版概况」次数/浏览页访问人数；

2.新概览排版是否满足一屏看完的需求，信息密度是否足够还是说太大？
下滑一屏幕 => 下滑次数；
点击「更多」跳转到详情页 => 点击「更多」跳转到详细页面的次数；

# 埋点
(由指标，基于指标的公式倒推出需要的埋点)

## 事件:滑动-底部 SlideBottom(优先实现)
触发时机：用户滑动到底部和底部-150px的区间的时候
前端埋点上报参数：
```
"behavior_type": "滑动", -- 行为类型
"area": "底部", -- 区域名
"behavior_content": "滑动-底部", -- 行为内容=「行为类型」+「区域名」+「子区域名」
```

## 事件:滑动-滚动一屏 SlideScrollbarFullScreen
触发时机：
前端埋点上报参数：
```
"behavior_type": "滑动",
"area": "",
"behavior_content": "",
```

## 事件:加载-在线统计卡片 LoadOnlineStatsCard
触发时机：
前端埋点上报参数：
```
"behavior_type": "",
"area": "",
"behavior_content": "",
```

## 事件:点击-筛选-进入旧版概况 ClickOldSurveyButton
触发时机：
前端埋点上报参数：
```
"behavior_type": "",
"area": "",
"behavior_content": "",
```

## 事件:点击-充值转化卡片-更多 ClickPayConversionCardMore
触发时机：
前端埋点上报参数：
```
"behavior_type": "",
"area": "",
"behavior_content": "",
```

## 事件:悬停-新增玩家卡片同比-小问号 HoverCreateRoleTBCardExplanation
触发时机：
前端埋点上报参数：
```
"behavior_type": "",
"area": "",
"behavior_content": "",
```

# 参考
## 产品-埋点命名设计
1.area区域 参数
根据BI页面，当时按照大多数相似页面，约定分成了从上到下为「搜索」、「内容」、「分页」三部分，但是却没有包含特殊的页面比如游戏概况

2.behavior_content行为内容 参数
点击-筛选-xxx
点击-内容-xxx
点击-分页-xxx

## 前端-上报数据
接口文档：
接收接口：  (golang)
```
{
  "data": [
    {
      "#event_time": 1669193474208,  -- WHEN
      "#ip": "", -- WHERE
      "#distinct_id": "{访客ID、浏览器指纹}", -- WHO，未登录状态
      "#user_id": "11061", -- WHO，登录状态
      "#lib": "js",  -- HOW
      "#lib_version": "0.0.1",
      "#screen_height": 900,
      "#screen_width": 1440,
      "#user_agent": "mozilla/5.0 (macintosh; intel mac os x 10_14_6) applewebkit/537.36 (khtml, like gecko) chrome/107.0.0.0 safari/537.36",
      "#os": 4,
      "#os_version": "Mac",
      "#browser": "chrome",
      "#browser_version": "107.0.0.0",
      "#log_level": "info",
      "#event": "behavior",
      "#event_type": "track",
      "#event_name": "behavior",   -- HOW
      "#project": "bi-api", -- WHAT
      "#game_name": "shqz", -- WHAT
      "properties": {
        "username": "test", -- WHO
        "nickname": "测试", -- WHO
        "module": "loss",  -- WHAT
        "module_cn": "流失(功能模块名)", -- WHAT
        "behavior_type": "点击", -- WHAT
        "area": "筛选", -- WHAT
        "behavior_content": "点击-筛选-时间" -- WHAT
      }
    }
  ]
}
```

## 后端-存储事件数据模型
```
CREATE TABLE `operation_behavior` (
`id` int(11) unsigned NOT NULL AUTO_INCREMENT COMMENT '自增ID',
`source_host` varchar(128) NOT NULL DEFAULT '' COMMENT '日志来源host',
`log_time` bigint(13) NOT NULL DEFAULT '-1' COMMENT '日志时间戳(13位毫秒级)',
`log_ymd` int(8) NOT NULL DEFAULT '-1' COMMENT '年月日',
`log_ym` int(6) NOT NULL DEFAULT '-1' COMMENT '日志年月',
`log_hour` tinyint(2) NOT NULL DEFAULT '-1' COMMENT '小时',
`log_minute` tinyint(2) NOT NULL DEFAULT '-1' COMMENT '分',
`event_time` bigint(13) NOT NULL DEFAULT '-1' COMMENT '事件时间戳(13位毫秒级)',  --WHEN
`ip` varchar(64) NOT NULL DEFAULT '' COMMENT 'IP地址',  --WHERE
`longitude` varchar(32) NOT NULL DEFAULT '' COMMENT '经度', --WHERE
`latitude` varchar(32) NOT NULL DEFAULT '' COMMENT '纬度', --WHERE
`province` varchar(64) NOT NULL DEFAULT '' COMMENT '省份', --WHERE
`city_name` varchar(64) NOT NULL DEFAULT '' COMMENT '城市名', --WHERE
`distinct_id` varchar(64) NOT NULL DEFAULT '' COMMENT '访客ID (浏览器指纹)', --WHO
`user_id` varchar(64) NOT NULL DEFAULT '' COMMENT '用户ID', --WHO
`username` varchar(64) NOT NULL DEFAULT '' COMMENT '用户账号', --WHO
`nickname` varchar(64) NOT NULL DEFAULT '' COMMENT '用户昵称', --WHO
`screen_height` smallint(4) NOT NULL DEFAULT '-1' COMMENT '屏幕高度', --HOW
`screen_width` smallint(4) NOT NULL DEFAULT '-1' COMMENT '屏幕宽度',
`os` tinyint(1) NOT NULL DEFAULT '-1' COMMENT '操作系统ID(1-IOS,2-Android,3-WinPhone,4-macOS,5-Windows,6-Linux)',
`os_version` varchar(32) NOT NULL DEFAULT '' COMMENT '操作系统版本',
`browser` varchar(64) NOT NULL DEFAULT '' COMMENT '浏览器',
`browser_version` varchar(64) NOT NULL DEFAULT '' COMMENT '浏览器版本',
`user_agent` text COMMENT 'UserAgent',
`lib` varchar(32) NOT NULL DEFAULT '' COMMENT '日志SDK类型',
`lib_version` varchar(32) NOT NULL DEFAULT '' COMMENT '日志SDK版本', --HOW
`game_name` varchar(32) NOT NULL DEFAULT '' COMMENT '游戏名', --WHAT
`project` varchar(64) NOT NULL DEFAULT '' COMMENT '项目', --WHAT
`module` varchar(64) NOT NULL DEFAULT '' COMMENT '模块标识',
`module_cn` varchar(64) NOT NULL DEFAULT '' COMMENT '模块名称',
`behavior_type` varchar(64) NOT NULL DEFAULT '' COMMENT '行为类型', --WHAT
`area` varchar(64) NOT NULL DEFAULT '' COMMENT '区域名', --WHAT
`behavior_content` text COMMENT '行为内容=「行为类型」+「区域名」+「子区域名」', --WHAT
PRIMARY KEY (`id`) USING BTREE,
KEY `idx_project_log_ymd_nickname_module_user_id` (
`log_ymd`,
`project`,
`nickname`,
`module`,
`user_id`
) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 35 DEFAULT CHARSET = utf8 ROW_FORMAT = COMPACT COMMENT = '用户行为日志'
```