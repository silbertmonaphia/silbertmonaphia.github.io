---
published: false
category: DB
title: 行为分析:用户画像USER
author: smona
date: '2022-07-07 20:49:00'
layout: post
---

- [行业调研](#行业调研)
  - [数数科技](#数数科技)
  - [神策数据](#神策数据)
- [Profile设计](#profile设计)
  - [神策版本](#神策版本)
  - [历史版本](#历史版本)
  - [最新设计](#最新设计)
- [问题](#问题)

在用户侧写数据(Profile)基础上，配合针对用户侧写数据做二次计算统计后的「用户标签」就是针对单用户的用户画像了。而用户分群其实也是用户画像的一种，单用户画像体现的是个人的特点，而用户群的画像则体现的是群体共同表现出来的共性特点。

# 行业调研
## 数数科技
Ref:[官方文档-User设计](https://docs.thinkingdata.cn/ta-manual/latest/getting_started/set_properties.html#%E4%B8%80%E3%80%81%E4%BA%8B%E4%BB%B6%E5%B1%9E%E6%80%A7%E8%BF%98%E6%98%AF%E7%94%A8%E6%88%B7%E5%B1%9E%E6%80%A7)

用户属性表示的是用户的不变的属性以及最新状态，建议将下列三种属性设置为用户属性：
(1)固定属性：固定属性指的是用户不会变更的属性，这些属性往往是注册时的信息或首次产生某行为时的信息，比如注册时间、注册来源渠道、性别、用户名、首次付费时间等等。对于这样的属性，在埋点时请调用user_setOnce，并建议在首次属性名前加上“first”
(2)最新状态`latest`：最新状态指的是用户的当前状态，往往是用户最后产生某行为的信息，比如最后上线时间、最后付费时间等等。对于这样的属性，在埋点时请使用user_set，并建议属性名前加上“latest”
(3)累计值：累计值实质上是「最新状态」的一种特殊形式。累计值的数据类型为数值型，主要是产生某重要行为的次数或者数值型的最新状态，比如累计付费次数、累计付费金额、累计登录次数等等，在埋点时请调用user_add，每次调用都会在原先的数值上进行累加操作

数数通过以下几个方法去做用户属性值设定 "user_setOnce"、"user_unset"、"user_add"、"user_del"、"user_append"

属性分类：
1.系统属性：事件表、用户表中固有的系统字段，如事件时间、账户 ID 等，一定以“#”号开头
2.预置属性：在数据接入过程中，由 SDK 自行生成的字段，如 IP、国家等，一定以“#”号开头
3.自定义属性：根据业务需求定制上传的属性，如付费金额等，不以“#”号开头
4.「虚拟属性」：通过 SQL 规则而新生成的新属性
5.「维度表属性」：通过基于已有属性上传维度表而新生成的属性，这类属性可能因覆盖而消失

## 神策数据
Ref:[官方文档-User设计](https://manual.sensorsdata.cn/sa/latest/tech_knowledge_model-1573771.html#id-.%E6%95%B0%E6%8D%AE%E6%A8%A1%E5%9E%8Bv1.13-User%E5%AE%9E%E4%BD%93)

(1)记录收集 User Profile
每个 User 实体对应一个真实的用户，用 distinct_id 进行标识，描述用户的长期属性(即 Profile)，并且该用户可与其所从事的行为，也即 Event 进行关联。
应该收集哪些字段作为 User Profile，也完全取决于产品形态以及分析需求，简单来说，就是在能够拿到的那些用户属性中，哪些对于分析有帮助，则作为 Profile 进行收集。
在数据收集上，一般记录 User Profile 的场所，是用户进行注册、完善个人资料、修改个人资料等几种有限的场合，与 Event 类似，我们也强烈建议在「后端记录和收集」 User Profile，而非前端或者客户端。

(2)字段记录在 Profile 还是 Event 的取舍
有些时候，我们可能会纠结，某个与用户相关的字段是应该记录在Profile 还是记录在 Event，一个基本的原则就是:
- Profile 记录的是用户特征的属性，例如：出生地、性别、注册地、首次广告来源类型等。
- Event 的字段，记录的是事件发生时的特征，字段的取值具有「场景性」，例如 省份、城市 、设备型号、是否登录状态等。

e.g. 拿 “地址”为例，Event 记录的是本次下单时使用的地址，Profile 记录的往往是 “常用地址”。

Profile 字段取值常常是根据用户自己录入或者基于某些条件得到的用户规律特征，字段值随用户更新信息而发生取值变更。一般来说为了降低维护成本，我们更偏向于在Profile用一些「固定不变字段」，比如 “年龄” 和 “出生日期”，因为“年龄”字段的值需要定时更新且年龄可以使用出生日期计算得到，具有可替代性，那么就没有必要使用需要频繁更新的 “年龄”字段，我们会更建议使用 “出生日期”。

(3) User Profile 注意的问题
1.单边/双边用户
单双边是针对产品有多个身份使用用户时才会进行区分
- 单边用户，即仅有一类用户的产品，如健身产品Keep ，聊天工具QQ等
- 双边用户如 O2O 产品，用户可能是普通消费者，也可能是商家用户。需要根据产品的不同，提前对用户识别和相应属性进行设计

2.缓慢变化维
如果遇到一些会发生变化的属性，比如用户的VIP等级，不能只作为用户属性传进用户表中，还需在事件表中，记录一个 “当前发生事件 VIP 等级” 这个属性。因为当前会员等级的统计，和发生事件时用户的会员等级统计是两种情况。

# Profile设计
## 神策版本
根据神策[预设字段](https://manual.sensorsdata.cn/sa/2.3/tech_knowledge_layout-22249967.html#id-.%E6%95%B0%E6%8D%AE%E6%A0%BC%E5%BC%8Fv2.1-%E9%A2%84%E7%BD%AE%E5%B1%9E%E6%80%A7)可得如下DDL

```sql
CREATE TABLE `user_profile` (
  name varchar comment '用户名',
  signup_time Datetime comment '注册时间',
  city varchar comment '用户所在的城市',
  province varchar comment '用户所在的省份',
  utm_matching_type varchar comment '渠道追踪匹配模式',

  first_visit_time Datetime comment '首次访问时间',
  first_referrer varchar comment '首次前向地址',
  first_referrer_host varchar comment '首次前向域名',
  first_browser_language varchar comment '首次使用的浏览器语言',
  first_browser_charset varchar comment '首次浏览器字符类型',
  first_search_keyword varchar comment '首次搜索引擎关键词',
  first_traffic_source_type varchar comment '首次流量来源类型',

  utm_source varchar comment '首次广告系列来源',
  utm_medium varchar comment '首次广告系列媒介',
  utm_term  varchar comment '首次广告系列字词',
  utm_content  varchar comment '首次广告系列内容',
  utm_campaign varchar comment '首次广告系列名称',

  first_id varchar comment 'device_uid未登录状态',
  second_id  varchar comment '游戏的account_id或sdk的channel_user_id/username登录状态',
  is_login_id  Bool comment '是否登录，登录了有second_id，没登录则只有first_id',
  PRIMARY KEY ()
)
PARTITION BY HASH KEY () PARTITION NUM 100
CLUSTERED BY ()
TABLEGROUP user
OPTIONS (UPDATETYPE='realtime')
COMMENT '用户侧写';
```

## 历史版本
我们的之前注册表的设计，缺点就是还是针对事件来设计的，没有面向用户，且一些变化比较频繁的字段也带进来了

```
CREATE TABLE `channel_register_all` (
  `source_host` varchar COMMENT '日志来源host',
  `fingerprint` varchar COMMENT '指纹',
  `log_uuid` varchar COMMENT '日志uuid',
  `longitude` varchar COMMENT '经度',
  `latitude` varchar COMMENT '纬度',
  `province` varchar COMMENT '省份',
  `city_name` varchar COMMENT '城市名',
  `log_time` bigint COMMENT '日志时间戳(13位毫秒级)',
  `ymd` int COMMENT '日志年月日',
  `ym` bigint COMMENT '日志年月',
  `hour` tinyint COMMENT '日志小时',
  `minute` tinyint COMMENT '日志分',
  `event_time` bigint COMMENT '事件时间戳(13位毫秒级)',
  `ip` varchar COMMENT 'IP地址',
  `username` bigint COMMENT '融合渠道用户ID',
  `account_id_old` varchar COMMENT '游戏账号(channelId_channelUserId)',
  `account_id` varchar COMMENT '游戏账号(channelUserId.channelName)',
  `device_uid` varchar COMMENT '设备唯一标识符',
  `imei` varchar COMMENT 'Android设备标识',
  `idfa` varchar COMMENT 'IOS设备标识',
  `manufacturer` varchar COMMENT '设备制造商',
  `device` varchar COMMENT '设备型号',
  `screen_height` smallint COMMENT '屏幕高度',
  `screen_width` smallint COMMENT '屏幕宽度',
  `os` tinyint COMMENT '操作系统ID(1-IOS,2-Android,3-WinPhone,4-macOS,5-Windows)',
  `os_version` varchar COMMENT '操作系统版本',
  `network_type` varchar COMMENT '网络类型',
  `operators` varchar COMMENT '网络运营商',
  `sdk_version` varchar COMMENT 'SDK客户端版本号',
  `package_version` varchar COMMENT '游戏包版本',
  `browser` varchar COMMENT '浏览器',
  `browser_version` varchar COMMENT '浏览器版本',
  `user_agent` varchar COMMENT 'UserAgent',
  `url` varchar COMMENT '当前页面',
  `referer` varchar COMMENT '页面来源',
  `lib` varchar COMMENT '日志SDK类型',
  `lib_version` tinyint COMMENT '日志SDK版本',
  `event` varchar COMMENT '事件',
  `event_type` varchar COMMENT '事件类型',
  `project` varchar COMMENT '项目',
  `app_id` bigint COMMENT '游戏ID',
  `game_name` varchar COMMENT '游戏名',
  `channel_group_id` int COMMENT '渠道类型(1-IOS,2-IOS越狱,3-安卓,4-安卓内测)',
  `channel_id` int COMMENT '渠道ID',
  `sub_channel_id` int COMMENT '子渠道ID',
  `account_type` varchar COMMENT '账号类型(phone-手机,email-邮箱,ordinary-普通,internal-内部)',
  `phone` varchar COMMENT '注册手机',
  `email` varchar COMMENT '注册邮件',
  `idcard` varchar COMMENT '身份证号',
  `realname` varchar COMMENT '真实姓名',
  `birth_provice` varchar COMMENT '出生省份',
  `birth_city` varchar COMMENT '出生城市',
  `birthday` int COMMENT '诞生日 e.g.20210831',
  `birth_year` smallint COMMENT '出生年',
  `birth_month` int COMMENT '出生月',
  `birth_day` int COMMENT '出生日',
  `gender` varchar COMMENT '性别',
  `verified_at` bigint COMMENT '防沉迷实名认证时间戳-13位毫秒',
  `pi` varchar COMMENT '防沉迷账号ID-PI',
  PRIMARY KEY (`username`,`ym`)
)
PARTITION BY HASH KEY (`username`) PARTITION NUM 100
CLUSTERED BY (`ymd`,`phone`,`email`,`birth_year`,`gender`,`idcard`,`username`)
TABLEGROUP channel
OPTIONS (UPDATETYPE='realtime')
COMMENT '渠道(融合)注册表(全量)';
```

## 最新设计
(202207版本)结合上面神策、数数的用户User Profile设计，以及我们历史已有的注册表设计，立足于当前奥飞、融合、游戏三个维度的用户，可得如下DDL，系一张大宽表(一般随着游戏发展会逐渐发展到有[300~2000](https://manual.sensorsdata.cn/sa/2.3/%E6%95%B0%E6%8D%AE%E6%A0%BC%E5%BC%8F-1573774.html#id-.%E6%95%B0%E6%8D%AE%E6%A0%BC%E5%BC%8Fv1.13-%E5%B1%9E%E6%80%A7%E6%95%B0%E4%B8%8A%E9%99%90)用户属性)

```sql
CREATE TABLE `game_mid_player_profile` (
-- 固定值
  `distinct_id/user_id?/player_id?` varchar comment '用户标识UUID 由下面7个用户识别号计算得出',
  `device_uid`  varchar comemnt 'SDK设备号',
  `udid`  varchar comemnt '游戏设备号(理论上和device_uid一致)',
  `channel_user_id` varchar comemnt '奥飞SDK账号',
  `username` varchar comemnt '融合SDK账号',
  `account_id_old` varchar comemnt '游戏账号(channelId_channelUserId)(取奥飞账号或者融合账号取决接的是哪个SDK)',
  `account_id` varchar comemnt '游戏账号(channelUserId.channelName)(取奥飞账号或者融合账号取决接的是哪个SDK)',
  `server` int COMMENT '游戏服ID',
  `role_id` bigint COMMENT '角色唯一标识符',
-- 固定值.现实身份信息
  `idcard` varchar comemnt '身份证',
  `realname` varchar COMMENT '真实姓名',
  `birth_provice` varchar COMMENT '出生省份',
  `birth_city` varchar COMMENT '出生城市',
  `birthday` int COMMENT '诞生日 e.g.20210831',
  `birth_year` smallint COMMENT '出生年',
  `birth_month` int COMMENT '出生月',
  `birth_day` int COMMENT '出生日',
  `gender` varchar COMMENT '性别',
  `phone` varchar COMMENT '手机',
  `email` varchar COMMENT '邮件',
-- 固定值.首次信息first
  `fisrt_signup_time` datetime comment '首次注册时间',
  `first_ip` varchar COMMENT '首次IP地址',
  `first_longitude` varchar COMMENT '首次经度',
  `first_latitude` varchar COMMENT '首次纬度',
  `first_province` varchar COMMENT '首次省份',
  `first_city_name` varchar COMMENT '首次城市名',
  `first_channel_id` int COMMENT '首次渠道ID',
  `first_sub_channel_id` int COMMENT '首次子渠道ID',
  `first_os` tinyint COMMENT '首次操作系统ID(1-IOS 2-Android 3-WinPhone 4-macOS 5-Windows)',
  `first_os_version` varchar COMMENT '首次操作系统版本',
  `first_network_type` varchar COMMENT '首次网络类型',
  `first_app_id` varchar COMMENT '首次游戏ID',
  `first_game_name` varchar COMMENT '首次游戏名',
-- 最新状态latest
指挥官名 role_name
指挥官等级 role_level
上一次登出时间 latest_logout_time
?登陆时间 login_time
通关关卡列表 round_list
-- 累计值accumulate
主线关卡剧情跳过次数 round_skip_times
付费金额 cash
?平均消耗专注(登录时间内)
)
COMMENT '用户画像';
```

其他字段一些信息没有放进去，可以按需获取，信息越多玩家的画像就越清晰
```sql
-- 自然人属性
  肤色
  瞳色
  脸型
  血型
  身高
  体重
  三围
-- 社会人属性
  家庭关系(父母、伴侣、后代以及3代内亲属)
  常用家庭住址
  教育水平
  语言
  职业
  公司
  年收入
  负债
  征信
  资产状况(存款、车子、不动产、公司以及持有的金融证券等)
  医疗记录(过往病史、过敏药物、手术记录、住院记录等)
  犯罪记录(违法、犯罪、拘禁、牢狱等)
-- 个人偏好
  食物偏好
  颜色偏好
  音乐偏好
  气味偏好
  购物类目偏好
  衣着偏好
```

# 问题
1.根据神策[标识用户](https://manual.sensorsdata.cn/sa/2.3/tech_knowledge_user-7540285.html)文档多次用户标识我们使用一对一还是多对一，一对一就是永远更新为最新的设备号哪怕后面换设备了不记录中间设备变化状态，而多对一则会另外维护一个设备数组(e.g. deviceid_a,deviceid_b,deviceid_c)记录设备变更历史，从左到右为从旧到新。我的建议是采用一对一，因为维护多对一目前(202109)不太有分析场景，而且具备较高维护成本
2.上面最新版的设计其实是已经预定义了用户的属性标签，但是用户的标签并非一成不变，后续如果通过纵向学习计算出用户的一些新的标签，这部分标签就无法预先自定义，尽管第一阶段对于用户画像预先定义的标签已经足够，但是必须要考虑后续机器学习对数据挖掘得出标签的情况如何解决，使得技术方案留下可扩展性