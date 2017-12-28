---
title: '高频访问IP限制 --Openresty(nginx + lua) [反爬虫之旅]'
author: smona
published: true
date: '2017-08-25 22:45:47'
layout: post
---

#  前言
  嗯....本人是从写爬虫开始编程的，不过后面做web写网站去了，好了，最近web要搞反爬虫了，哈哈哈，总算有机会把之以前做爬虫时候见识过的反爬一点点给现在的网站用上了~ 做爬虫的同志，有怪莫怪喽~还有求别打死 > <
  
  首先要提一下AJAX，现在普天下网页几乎都是往特定的数据接口请求数据了，除了什么首屏渲染这种服务端渲染好html以外，几乎没有什么静态网页了。我看了有一些帖子说AJAX让爬虫难做，可是我觉得结合一些工具(比如chrome的开发者工具)，找到AJAX所请求的后端数据接口一点也不难，而且现在自己也写过一段时间的web后端数据接口，发现接口的设计往往都是往简单易懂的方向做，外加从2000年出现REST风格，更是让接口设计越来越简明了。所以其实如果一个web站点没有察觉到有爬虫的存在，或者察觉到了，但是没有想要做一点数据保护措施，它是不会再AJAX上做文章的，那么如果单纯的AJAX，其实并没有任何反爬的作用，所以别再说AJAX反爬什么的了，何况AJAX生出来就不是为了反爬的
    
  然而在现在的前后端分离的时代，前端反爬还是有的搞的，基于我不太懂JavaScript，就不展开来说，我只是听说过什么参数加密啊，数据混淆什么的，但其实概括起来都是一种对数据接口的隐藏，这让一些不太懂js的人，也跟着懵逼了(比如说我 : <)，但是你要知道，前端代码最终还是要请求一个url的，无论它把这个过程拆开成多散，弄得多复杂都好，只要是需要数据，就必然需要请求一个后端接口(这个接口可以是SOAP，不过21世纪恐怕更多的是RESTful的)，所以对于数据保护而言，更加需要重点关注的是后端数据接口的保护。
  
  本反爬虫之旅系列将会一点点从各个方面垒高数据保护墙，但是请记住，因为网站数据的公开性，所以，只是延缓被盗库的时间而已，想自己在网站上公开的数据完全不被爬走是不可能的。那么我们的目标就是：让盗库耗时被延缓到一个比较长的时间里面，那么对于爬取数据方而言，这些数据的价值将会随着时间的增加而降低，数据的价值=利用价值 - (爬取成本+数据贬值速度) * 爬取时间(不用纠结来源了，我说的)
  
   这一篇就讲最基础的“给过频IP弹验证码”这种入门级防护实现，虽然花钱买点代理IP就可以搞定这种实现，但是至少也让他们增加了成本，但是我们相对地并没有花费多少成本，而且过频IP弹验证码除了能反爬，也能抵御一部分的CC攻击(短时间大量的爬虫请求堪比CC攻击啊)，虽然没有多大的作用，但是起码比裸奔强！这也算是功能上的复用吧

  反爬虫之旅预告:
 1. 过频IP弹验证码[应用外]
 2. 数据接口的url设计(uuid)和内容横向范围限制(参考angel.co)[应用内]
 3. 用户可见(参考微博)以及内容纵向切割(盈利点思考)[应用内]

----------

# 统览

![这里写图片描述](http://img.blog.csdn.net/20170825212635528?/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMjkyNDUwOTc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
<center>高频访问IP弹验证码架构图</center>
P.s. csdn默认水印real丑，直接去掉图片地址的watermark就可以了

----------

#  OpenResty
我不准备在web应用中做ip的统计和查封，应用就应该只做业务功能，这些基础东西应该由我们应用的前部——专业的Nginx实现  

Nginx本身就有根据ip访问频率的设置，比如[“服务器访问频率限制和IP限制”](http://blog.wuxu92.com/request-rate-limit-and-access-deny/)就有提到。不过Nginx只能强硬地返回个403状态码什么的，但是我们这次ip封禁时间比较久，那么如果误伤到用户，我们仅仅强硬地返回个403，用户将会毫无办法证明自己是人，然后要等很久，那就伤用户就伤得很深了，因此我们需要一种可以让被误伤的用户能及时自行解封的策略，验证码就是一个不错的选择，可是Nginx该怎么接入验证码呢?  

在说明怎么Nginx接入验证码之前，我想先说说验证码本身，其实就基础防护来说，(封IP+验证码)是性价比比较高的一般性基础组合了，比较低廉的成本就能给爬虫制造麻烦，基于这种组合就能筛选掉一部分廉价爬虫。而虽然说至今为止，很多验证码都被破解了，甚至连新型的基于行为的验证码(比如极验的拖条验证和谷大哥的reCaptcha)，都有人提出了破解方案(我今天谷歌一下，居然不止是方案，已经有两三页的教程了- -||| 我得找个时间学习一下了)，但是，这种破解方案却不是谁都可以完美丝滑地应用到自己的爬虫上，这是需要一定功力的，那么换个角度思考，我们在某种程度上已经赢了，毕竟我们只是调用别人一个接口而已，甚至就算我们自己DIY一个汉字的图片验证码也不费多大功夫(汉字字符粘连+带随机噪点+干扰线并不特别难，实在不懂可以参考这篇[“Python 随机生成中文验证码”](https://www.oschina.net/code/snippet_12_325)就有现成代码~大概长这样![这里写图片描述](https://static.oschina.net/uploads/code/201010/14171513_pfze.jpg))，而爬虫要搞定验证码要么自己花钱第三方识别，要么就自己的团队开发识别验证码的工具，总之又提高了他们爬取成本，杀敌一千，自伤只有五百  

虽然有现成的免费的图片验证码生成程序，但是我们在这篇博文里面还是来点新潮的"基于行为"的验证码吧，比如说极验，而关于极验的部署后面还是会提到，个人觉得他们的官方文档后端部署的python那部分讲的不清不楚，后面得自己测试跑一次才知道怎么改...  

那么回归Nginx接入验证码的问题，我们需要Lua，Lua是一个高性能的脚本语言，我感觉和Python很像，但是灵活性比不上Python，而执行速度却比Python快。Lua和C/C++是很亲和的，是补充C/C++灵活性的存在，因为有Lua，只要我们在C/C++中向外引入Lua脚本，那么如若Lua脚本发生了修改，我们也并不需要因此重新编译一次C/C++程序。Nginx本身便是由C/C++编写，所以自然和Lua亲和，而后又有OpenResty项目的存在(捆绑了nginx和lua并自带常用lua模块)，让Lua在扩展Nginx上成为头号选择  
  
P.s.补充一点，其实Lua在Nginx的应用只是Lua应用中很小的一个点而已，它在游戏中才是被广泛地应用，因为：第一，游戏在乎性能体验，所以很多Engine都是用C/C++写的，自然需要Lua做一点粘合性补充; 第二，Lua的性能仅仅次于C/C++，而且还有为了榨干lua性能的LuaJIT的存在，让lua的性能得到进一步地提升,故Lua是C/C++后的第二选择
  
OpenResty本身没有什么好讲的，它最大的功劳就是把Lua比较舒服地捆绑到了Nginx上，其他特性都是Lua本身的东西，所以想把Nginx玩的更加溜，除了彻底玩转Nginx本身以外(Nginx本身的配置就有点像一门小语言了)，Lua会是你不二的选择。

##  下载安装OpenResty
下载安装可以直接参考官网的教程(看安装和新手上路就可以了，以后有空想稍微深入一点的，可以直接看[OpenResty最佳实践](https://www.gitbook.com/book/moonbingbing/openresty-best-practices))

P.s因为我目前工作的本本是MBP，所以是用homebrew安装的，感觉会和linux里面的openresty有点不太一样,osx里面是用openresty这条命令启动才算是openresty，而linux貌似是openresty下的nginx启动的才算是openresty，才能用比如access_by_lua_file或者content_by_lua这种openresty语法
```
我自定义的目录结构如下:
-anti_spider
  -conf/
     -nginx.conf
  -lua/
     -access.lua
  -log/
     -error.log
  -geetest_web/
     -demo/
     -sdk/
     -geetest.py
     -setup.py
     -requirements.txt
```

##  Nginx配置
在openresty下接入Lua脚本就一句话，下面给出nginx.conf示范:
```
worker_processes  1;
error_log logs/error.log;
events {
    worker_connections 1024;
}
http {
    server {
        listen 80;
        location / {
	        access_by_lua_file 'lua/access.lua';
            content_by_lua 'ngx.say("Welcome PENIS!")';
	    }
    }
}
   
```

##  access.lua
```
-- package.path = '/usr/local/openresty/nginx/lua/?.lua;/usr/local/openresty/nginx/lua/lib/?.lua;'
-- package.cpath = '/usr/local/openresty/nginx/lua/?.so;/usr/local//openresty/nginx/lua/lib/?.so;'

-- 连接redis
local redis = require 'resty.redis'
local cache = redis.new()
local ok ,err = cache.connect(cache,'127.0.0.1','6379')
cache:set_timeout(60000)
-- 如果连接失败，跳转到label处
if not ok then
  goto label
end

-- 白名单
is_white ,err = cache:sismember('white_list', ngx.var.remote_addr)
if is_white == 1 then
  goto label
end

-- 黑名单
is_black ,err = cache:sismember('black_list', ngx.var.remote_addr)
if is_black == 1 then
  ngx.exit(ngx.HTTP_FORBIDDEN)
  goto label
end


-- ip访问频率时间段
ip_time_out = 60
-- ip访问频率计数最大值
connect_count = 45
-- 60s内达到45次就ban

-- 封禁ip时间(加入突曲线增长算法)
ip_ban_time, err = cache:get('ip_ban_time:' .. ngx.var.remote_addr)
if ip_ban_time == ngx.null then
  ip_ban_time = 300
  res , err = cache:set('ip_ban_time:' .. ngx.var.remote_addr, ip_ban_time)
  res , err = cache:expire('ip_ban_time:' .. ngx.var.remote_addr, 43200) -- 12h重置
end


-- 查询ip是否在封禁时间段内，若在则跳转到验证码页面
is_ban , err = cache:get('ban:' .. ngx.var.remote_addr) 
if tonumber(is_ban) == 1 then
  -- source携带了之前用户请求的地址信息，方便验证成功后返回原用户请求地址
  local source = ngx.encode_base64(ngx.var.scheme .. '://' ..
    ngx.var.host .. ':' .. ngx.var.server_port .. ngx.var.request_uri)
  local dest = 'http://127.0.0.1:5000/' .. '?continue=' .. source 
  ngx.redirect(dest,302)
  goto label
end

-- ip记录时间key
start_time , err = cache:get('time:' .. ngx.var.remote_addr)
-- ip计数key
ip_count , err = cache:get('count:' .. ngx.var.remote_addr)

-- 如果ip记录时间的key不存在或者当前时间减去ip记录时间大于指定时间间隔，则重置时间key和计数key
-- 如果当前时间减去ip记录时间小于指定时间间隔，则ip计数+1，
-- 并且ip计数大于指定ip访问频率，则设置ip的封禁key为1，同时设置封禁key的过期时间为封禁ip时间

if start_time == ngx.null or os.time() - tonumber(start_time) > ip_time_out then
  res , err = cache:set('time:' .. ngx.var.remote_addr , os.time())
  res , err = cache:set('count:' .. ngx.var.remote_addr , 1)
else
  ip_count = ip_count + 1
  res , err = cache:incr('count:' .. ngx.var.remote_addr)
  -- 统计当日访问ip集合
  res , err = cache:sadd('statistic_total_ip:' .. os.date('%x'), ngx.var.remote_addr)
  if ip_count >= connect_count then
    res , err = cache:set('ban:' .. ngx.var.remote_addr , 1)
    res , err = cache:expire('ban:' .. ngx.var.remote_addr , ip_ban_time)
    res , err = cache:incrby('ip_ban_time:' .. ngx.var.remote_addr, ip_ban_time)
    -- 统计当日屏蔽ip总数
    res , err = cache:sadd('statistic_ban_ip:' .. os.date('%x'), ngx.var.remote_addr)
  end
end

::label::
local ok , err = cache:close()

```
Reference:
1.[nginx和lua](http://blog.csdn.net/yanggd1987/article/details/46679989)
2.[nginx+lua+redis实现验证码防采集](http://blog.csdn.net/yanggd1987/article/details/46914039)
3.[Nginx+Lua+Redis访问频率控制
](http://homeway.me/2015/08/11/nginx-lua-redis-access-control/)

##  启动/重启nginx
```
启动:
 nginx -p `pwd` -c conf/nginx.conf
重载:(修改了lua脚本或者nginx.conf配置每次都要重载生效)
 nginx -p `pwd` -c conf/nginx.conf -s reload
```

##  Redis统计数据持久化
Lua脚本里面有statistic_ban_ip和statistic_total_ip两个统计数据，分别记录了每天的被屏蔽过的ip数量和总共访问的ip数量,那么根据这些数据，我们就可以做分析，比如statistic_ban_ip/statistic_total_ip每日被封禁ip占总ip量的百分比,还有可以结合百度地图的ip地理定位做被封ip的定位，看看哪个地区被封杀最严重~ 甚至还可以以后积累了几个个月甚至几年的redis记录，然后可以做一份 [月被封ip量 - 月份|年份] 的笛卡尔坐标系(Cartesian coordinate system)，然后可以深入分析一下时间分布，根据这种分布，适当地调整一下策略，或者甚至可以做成智能型的  
    
当然现在已经有很多网站前置统计数据的服务了，比如友盟+什么的，但是我们所记录的这些数据是实实在在我们自己一天天"熬"出来的数据，留在本地做数据分析用，或者给其他的什么需求提供数据支持，这个...谁说的准呢?不过数据就是数据，留下来是对的，我们的这些留下来的数据也不是什么垃圾数据，况且，实际工作量也不大(就redis增加两个字段而已)，占用的空间也不大(就一些短字符串而已)  
    
  不过问题是，如果你内存不够,而redis是内存型的数据库，加之也没有必要长年累月都把统计数据堆在redis里面，所以我们得有把这些统计数据，或者可以直接说冷数据持久化到硬盘的定时操作，而至于redis的持久化，这里留个坑，回头再来填
    
    
----------

#  极验 
  现在来讲讲统览图里面的Captcha WebApi的构建,在上面Lua的脚本里面有一句跳转到验证码接口的:
```
local dest = 'http://127.0.0.1:5000/' .. '?continue=' .. source 
ngx.redirect(dest,302)
```
里面的这个http://127.0.0.1:5000/就是统览图里面的Captcha WebApi开放的验证码验证地址，我们在这个地址上部署的是极验的验证码服务(并无广告意思，易盾貌似也不错~)，你可以上他们的官网下载他们的demo，我这里的以Flask demo为例:
```
1.git拉下来
git clone https://github.com/GeeTeam/gt3-python-sdk.git
2.构建geetest
python3 setup.py install 
3.找到启动demo里面的基于flask写的web api
#直接python3 start.py是不行滴！你还需要flask，而且因为还要访问redis，再来个redis
pip3 install Flask
pip3 install redis
python3 start.py
#注意要和start.py以及templates/同一层启动start.py，不然等下找不到templates/下面的login.html和gt.js
#吐槽一下极验的后端部署文档的不完整，我也是自己调试着才知道怎么回事...
```
Refer: [极验文档](http://docs.geetest.com/install/server/python/)

好的，既然能跑了，那么我们得怎么改?要知道他们给的demo是没有redis访问的!
```
1.打开start.py，简单说明一下:
pc_geetest_id和pc_geetest_key你自己申请换上去吧，不详细说明了;
get_pc_captcha()这个就是官方文档那个"嗨复杂的"完整流程图的第一次网站主的客户端对网站主的服务器的请求接口;
pc_ajax_validate()这个是二次验证的，返回的是json格式的;
pc_validate_captcha()和pc_ajax_validate()这个功能一样，只不过这个是返回html;
statichandler()这个估计是前端的脚本需要访问的，不用理;
login()这个就不用解释了;
(login.html的内容其实我们这次完全不是做用户登录，所以用不到提交用户名密码，所以用户名密码那块代码html表格都可以删掉了)

2.新增一个redis的操作函数
def handle_passed_ip(remote_ip):
    # 处理验证通过的ip，注意host,port还有db要和你lua访问的一致!!!
    import redis
    r = redis.Redis(host='127.0.0.1', port=6379, db=0)
    r.delete('ban:' + str(remote_ip))
    r.set('count:' + str(remote_ip), 1)
    return remote_ip


3.改login()

def login():
    import base64
    # 拿到之前lua跳转过来携带的continue参数
    # 即通过base64编码过的记录着访问者访问的原url信息，方便验证通过跳转
    former_url = base64.b64decode(request.args.get('continue'))
    session["former_url"] = former_url
    return render_template('login.html')


4.改pc_ajax_validate()

def pc_ajax_validate():
    gt = GeetestLib(pc_geetest_id, pc_geetest_key)
    challenge = request.form[gt.FN_CHALLENGE]
    validate = request.form[gt.FN_VALIDATE]
    seccode = request.form[gt.FN_SECCODE]
    status = session[gt.GT_STATUS_SESSION_KEY]
    user_id = session["user_id"]
    if status:
        result = gt.success_validate(
            challenge, validate, seccode, user_id, data='', userinfo='')
    else:
        result = gt.failback_validate(challenge, validate, seccode)
    result = {"status": "success"} if result else {"status": "fail"}
	# 从这里开始就是新增的内容
    remote_ip = request.remote_addr  # 获取访问者ip
    remote_ip = handle_passed_ip(remote_ip) #调用我们新增的redis操作函数
    result.update({"former_url": session["former_url"].decode('utf-8')})
    return json.dumps(result)
```
以上后端就改好了，再启动start.py，那么统览图里面的Captcha WebApi的验证码验证服务就起来了~至于前端代码要怎么改?对不起，那得你自己看官方文档研究去，不过我感觉，他们的前端文档写的比后端文档好.......
