---
published: false
category: DB
title: 行为分析:事件式日志EVENT
author: smona
date: '2022-07-07 20:49:00'
layout: post
---

- [参考](#参考)
- [格式](#格式)
- [路径](#路径)
- [DDL设计](#ddl设计)
  - [应用运行日志](#应用运行日志)
  - [用户行为日志](#用户行为日志)
  - [玩家行为日志](#玩家行为日志)
- [埋点分工](#埋点分工)

# 参考  
Ref: [数据模型 - 神策分析](https://manual.sensorsdata.cn/sa/latest/page-1573771.html#id-.%E6%95%B0%E6%8D%AE%E6%A8%A1%E5%9E%8Bv1.13-Event%E5%AE%9E%E4%BD%93)
神策的事件式日志设计(4W1H=When+Where+Who+How+What)

# 格式  
「发行平台日志」应用运行日志和用户行为日志统一用JSON方便核对核查  
「游戏日志」只有玩家行为日志，考虑到量大的问题仍然延用CSV-like(e.g.CSV,TSV)，新游戏采用JSON
P.S. 
1.JSON格式天然就对CSV-like麻烦的字段缺失、多了导致的错位问题有抵抗力，故如果存储成本允许第一份数据应当以JSON格式存放
2.CSV-like日志增加字段需要在最后加，这样不会影响前边历史日志查询，如果中间加的话会导致历史日志查询时候错位

# 路径  
「发行平台日志」
- 应用运行日志(接口日志):
  - 路径:$logs_prefix/$project_name/$log_type/$log_ym/$log_ymd.$log_level.log 
  - e.g. $logs_prefix/sdk/monitor/202102/20210202.ERROR.log

- 用户行为日志(统计日志): 
  - 路径:$logs_prefix/$project_name/$log_type/$log_ym/$log_secondary_type/$log_ymd.$log_level.log 
  - e.g. $logs_prefix/sdk/stat/202102/login/20210202.INFO.log

「游戏日志」
- 玩家行为日志:
  - 路径:$logs_prefix/$log_ymd/$server_id/$log_type.txt 
  - e.g. $logs_prefix/20160511/1101/LoginRole.txt

# DDL设计
## 应用运行日志  

PIPELINE(数据管道信息)
  `source_host` varchar COMMENT '日志来源host',
  `fingerprint` varchar COMMENT '指纹',
  `log_uuid` varchar COMMENT '日志uuid',

LOG(日志元信息)
  `log_time` int COMMENT '日志时间戳',
  `ymd` int COMMENT '日志年月日',
  `ym` bigint COMMENT '日志年月',
  `hour` tinyint COMMENT '日志小时',
  `minute` tinyint COMMENT '日志分',

WHEN
	`event_time` bigint COMMENT '事件时间戳(13位毫秒级)',

WHERE
	`ip` varchar COMMENT ‘IP地址’,

WHO
	`device_uid` varchar COMMENT '设备唯一标识符',
	`mac` varchar COMMENT 'MAC地址',
	`imei` varchar COMMENT 'Android设备标识',
	`idfa` varchar COMMENT 'IOS设备标识',

HOW
  `device` varchar COMMENT '设备型号',
  `os` tinyint COMMENT '操作系统ID(1-IOS,2-Android,5-macOS,6-Windows)',
  `os_version` varchar COMMENT '操作系统版本',
  `network_type` varchar COMMENT '网络类型',
  `operators` varchar COMMENT '网络运营商',
  `sdk_version` varchar COMMENT 'SDK客户端版本号',
  `browser` varchar COMMENT '浏览器',
  `browser_version` varchar COMMENT '浏览器版本',
  `user_agent` varchar  COMMENT 'UserAgent',

WHAT
	`log_level` varchar COMMENT '日志级别',
	`process_id` bigint COMMENT '进程ID',
	`request_id` varchar COMMENT '请求ID',
	`request_url`  varchar COMMENT '请求地址,
	`request_method` varchar COMMENT '请求方法',
	`request_params` varchar COMMENT '请求参数',
	`response_code` int COMMENT '响应码',
	`response_msg` int COMMENT '响应信息',
	`response_data` varchar COMMENT '响应数据',
	`content` varchar COMMENT '错误内容',
	`backtrace` varchar COMMENT '调用栈'

## 用户行为日志  

PIPELINE(数据管道信息)
  `source_host` varchar COMMENT '日志来源host',
  `fingerprint` varchar COMMENT '指纹',
  `log_uuid` varchar COMMENT '日志uuid',
  `longitude` varchar COMMENT '经度',
  `latitude` varchar COMMENT '纬度',
  `province` varchar COMMENT '省份',
  `city_name` varchar COMMENT '城市名',

LOG(日志元信息)
  `log_time` int COMMENT '日志时间戳',
  `ymd` int COMMENT '日志年月日',
  `ym` bigint COMMENT '日志年月',
  `hour` tinyint COMMENT '日志小时',
  `minute` tinyint COMMENT '日志分',

WHEN
  `event_time` bigint COMMENT '事件时间戳(13位毫秒级)',

WHERE(ip可以推导出经纬度以及省份城市的信息)
  `ip` varchar COMMENT 'IP地址',

WHO(包括账号和设备两种标识信息)
  `channel_user_id` bigint COMMENT '用户ID',
  `account_id_old` varchar COMMENT '游戏账号(channelId_channelUserId)',
  `account_id` varchar COMMENT '游戏账号(channelUserId.channelName)',
  `device_uid` varchar COMMENT '设备唯一标识符',
  `mac` varchar COMMENT 'MAC地址',
  `imei` varchar COMMENT 'Android设备标识',
  `idfa` varchar COMMENT 'IOS设备标识',

HOW
  `device` varchar COMMENT '设备型号',
  `screen_length` smallint COMMENT '屏幕长度',
  `screen_width` smallint COMMENT '屏幕宽度',
  `os` tinyint COMMENT '操作系统ID(1-IOS,2-Android,5-macOS,6-Windows)',
  `os_version` varchar COMMENT '操作系统版本',
  `network_type` varchar COMMENT '网络类型',
  `operators` varchar COMMENT '网络运营商'
  `sdk_version` varchar COMMENT 'SDK客户端版本号',
  `browser` varchar COMMENT '浏览器',
  `browser_version` varchar COMMENT '浏览器版本',
  `user_agent` varchar  COMMENT 'UserAgent',
  `url` varchar  COMMENT '当前页面',
  `referer` varchar  COMMENT '页面来源',

WHAT(业务特有信息，不同业务定义不同，这里只是样例)
  `source` varchar COMMENT '来源',
  `event` varchar COMMENT '用户事件类型(短信发送,改密,找密,手机绑定,手机改绑,邮箱绑定,邮箱改绑,身份证绑定)',
  `event_content` varchar COMMENT '用户事件内容JSON(短信内容,手机号码,邮箱号,身份证号)',
  `channel_group_id` int COMMENT '渠道类型(1-IOS,2-Android)',
  `channel_id` int COMMENT '渠道ID',
  `sub_channel_id` int COMMENT '子渠道ID',
  
## 玩家行为日志  

PIPELINE(数据管道信息)
  `source_host` varchar COMMENT '日志来源host',
  `fingerprint` varchar COMMENT '指纹',
  `log_uuid` varchar COMMENT '日志uuid',
  `longitude` varchar COMMENT '经度',
  `latitude` varchar COMMENT '纬度',
  `province` varchar COMMENT '省份',
  `city_name` varchar COMMENT '城市名',

LOG(日志元信息)
 `log_time` int COMMENT '日志时间戳(10位秒级)',
 `log_ymd` int COMMENT '日志年月日',
 `log_ym` bigint COMMENT '日志年月',
 `hour` int COMMENT '日志小时',
 `minute` int COMMENT '日志分',

WHEN
 `event_time` bigint COMMENT '事件时间戳(13位毫秒级)',

WHERE(ip可以推导出经纬度以及省份城市的信息)
 `ip` varchar COMMENT 'Ipv4地址',

WHO(包括设备和账号两种标识信息)
 `udid` varchar COMMENT '设备唯一标示符',
 `mac` varchar COMMENT '设备mac地址',
 `imei` varchar COMMENT 'android设备标识',
 `idfa` varchar COMMENT 'iOS设备标识',
 `account_id` varchar COMMENT '账号唯一标识符,取值SDK值',

HOW
 `device_model` varchar COMMENT '设备型号',
 `screen_length` smallint COMMENT '屏幕长度',
 `screen_width` smallint COMMENT '屏幕宽度',
 `group_id` int COMMENT '操作系统ID',
 `os_version` varchar COMMENT '操作系统版本',
 `network_type` varchar COMMENT '网络连接',
 `app_channel` varchar COMMENT '运营渠道',
 `app_version` varchar COMMENT '客户端版本号',
 `platform_tag` varchar COMMENT '发行平台标记',
 `channel_id` int COMMENT '运营渠道ID',
 `server` int COMMENT '游戏服ID',

WHAT(业务特有信息，不同业务定义不同，这里只是以登录为例)
 `create_time` int COMMENT '角色创建时间',
 `role_id` varchar COMMENT '角色唯一标识符',
 `role_name` varchar COMMENT '角色名称',
 `role_job` int COMMENT '角色职业',
 `role_level` int COMMENT '角色等级',
 `login_time` int COMMENT '登录时间',
 `last_logout_time` int COMMENT '上次登出时间',
 `total_pay` float COMMENT '角色总充值',
 `vip_level` int COMMENT '角色VIP等级'

# 埋点分工  

| BI数据库字段 | 数据类型 | 术语 | 规范及demo | 前台字段 | 后台字段 |
| ------- | ---- | --- | ------- | ---- | ---- |
| 日志元信息字段，数据组负责 | 7个 |  |  |  |  |
| log_uuid | varchar | 日志uuid | 9692d03d-00e4-48c6-b7bc-4bfb497a8fcc |  |  |
| source_host | varchar | 日志来源host | afjf\_bk\_gamedb\_0002\_183\-60\-252\-147 |  |  |
| fingerprint | varchar | 指纹(用户管道实现EOS语义) | 336344e8aaa7a67f3263b50c0022524e7f864e425c534725d69f7943451434f8 |  |  |
| longitude | varchar | 经度 | 127.79 |  |  |
| latitude | varchar | 纬度 | 42.02 |  |  |
| province | varchar | 省份 | Guangdong |  |  |
| city_name | varchar | 城市 | Guangzhou |  |  |
| 前台通用字段 | 5个 |  |  |  |  |
| app_id | bigint | 产品ID | 1712080001 | appID |  |
| game_name | varchar | 游戏唯一标识 | shqz | gameName |  |
| channel_id | int | 主渠道ID | 1002 | channelID |  |
| sub\_channel\_id | int | 子渠道ID | 10020000 | subChannelID |  |
| event_time | bigint | 事件时间戳 | 13位毫秒 | timestamp |  |
| os | tinyint | 操作系统 | 1 IOS，2 Android，3 WinPhone，4 Mac， 5 Windows | os |  |
| group_id | tinyint | 操作系统(BI) | 1 IOS; 2 安卓; 3 IOS越狱 |  |  |
| os_version | varchar | 操作系统版本 | android：9，iOS：11.3 | osVersion |  |
| device_uid/udid | varchar | 设备唯一标识符 | android：4d2cd5dc-be71-7726-fffb-0458562fc6f5，iOS：E1A1D218-0610-4397-B47F-BC897ECAA50F | deviceUID，WEB： |  |
| 移动端通用字段 |  |  |  |  |  |
| manufacturer | varchar | 设备制造商 | android：vivo，iOS：Apple | manufacture |  |
| device | varchar | 设备型号 | android：NX616J，iOS：iPhone 6 | device |  |
| screen_height | smallint | 屏幕的高度 | 1920x1080 | screen |  |
| screen_width | smallint | 屏幕的宽度 | 1920x1080 | screen |  |
| operators | varchar | 运营商名称 | 中国联通 | operators |  |
| network_type | varchar | 网络类型 | 2G，3G，4G，5G，WIFI | networkType |  |
| channel\_group\_id | tinyint | 渠道分类 | BI专用：1 IOS，2 IOS越狱，3 安卓，4 安卓内测 | channelGroupID |  |
| sdk_version | varchar | sdk版本 | 3.3.2 | sdkVersion |  |
| package_version | varchar | 游戏包版本 | 0.9.0 | packageVersion |  |
| 移动端Android专属字段 |  |  |  |  |  |
| imei | varchar | IMEI | 862728033239891 | imei |  |
| 移动端IOS专属字段 |  |  |  |  |  |
| idfa | varchar | IDFA | E1A1D218-0610-4397-B47F-BC897ECAA50F | idfa |  |
| Web前端专属字段 |  |  |  |  |  |
| browser | varchar | 浏览器名 |  |  |  |
| browser_version | varchar | 浏览器版本 |  |  |  |
| user_agent | varchar | UA |  |  |  |
| url | varchar | 当前页面 |  |  |  |
| referer | varchar | 页面来源 |  |  |  |
| 后台专属字段 |  |  |  |  |  |
| ip | varchar | 客户端IP |  |  |  |
| log_time | int | 日志时间戳 | 13位毫秒 |  |  |
| ymd/log_ymd | int | 年月日 |  |  |  |
| ym/log_ym | bigint | 年月（BI需要bigint类型） |  |  |  |
| hour | tinyint | 时 |  |  |  |
| minute | tinyint | 分 |  |  |  |
| channel\_user\_id/username | varchar | 渠道用户ID | afsdk channel_user_id(uid) :14790537， joysdk username :9999015.xiao7ios |  |  |
| account_id | varchar | 非水Q游戏账号 | 渠道ID.渠道标识（9999015.xiao7ios）  | 取SDK值 |  |
| account\_id\_old | varchar | 水Q游戏账号 | 渠道ID或子渠道ID\_ channel\_user\_id |  |  |
| lib | varchar | 日志SDK类型 | PHP、Openresty、Python |  |  |
| lib_version | tinyint | 日志SDK版本 | 整数自增1 |  |  |
| event | varchar | 事件名 | 事件名由团队负责人跟相关人员统筹分配，切勿自行添加，参考：apiInit、apiResponseJson、apiResponseString |  |  |
| event_type | varchar | 事件类型 | info、warning、error |  |  |
| project | varchar | 项目名 | 项目名由团队负责人跟相关人员统筹分配，切勿自行添加，参考：<span class="colour" style="color:rgb(85, 85, 85)">joysdk-pay afsdk-pay joysdk afsdk</span> |  |  |