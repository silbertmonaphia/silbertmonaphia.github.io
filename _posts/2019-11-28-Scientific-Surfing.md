---
published: true
category: Network
title: 科学上网年代记
author: smona
date: '2019-12-04 14:35:47'
layout: post
---
- [科学上网](#科学上网)
- [VPN时代](#vpn时代)
- [Shadowsocks时代](#shadowsocks时代)
- [V2ray时代](#v2ray时代)
- [机场时代](#机场时代)
- [未来](#未来)
- [线路](#线路)

# 科学上网  

  科学上网，我相信很多程序猿都不陌生，可以说几乎所有中国程序猿必备的技能。  
  
  我国程序猿和国外程序猿相比在搜索引擎上天生就是个劣势，百度搜索质量堪忧，而且计算机技术还是国外的比较厉害的情况下，就得不停地访问谷歌搜索资源。  
  
  其实不止是谷歌服务，像我这种一开始只是谷歌，后面逐步对油管，IG等都产生了重度依赖，导致在某段时间我的梯子全挂掉的情况下，竟然不知道怎么去上网了，国内优质资源虽然有但是太少了，而且不好搜索。  
  
  当然翻出去了用谷歌，你还得具备相当强的英文搜索和快速阅读能力，虽然借助于谷歌的搜索算法，就算是搜索中文，质量也比百度好，但是中文只是占据全部搜索内容的一小部分，很大一部分都是英文，会英文搜索才算真的用好谷歌。  

  言归正传，想想自己都玩了4年多的梯子，也是时候总结一下之前科学上网的种种姿势，这里不讨论具体技术，只是讨论近3-4年的演变。  

# VPN时代  

  第一次翻墙出去是大学三年级的时候，那时候第一次接触Linux敲开编程大门，然后在不断搜索中，知道了墙的存在，也知道了Google这种和百度一样的搜索引擎存在，所以当时就非常想翻出去看看。
  
  第一次用的是免费的GreenVPN节点，当时能打开Google就已经很开心了，不管它是不是总是连不上或者访问像蜗牛一样，但是时间过了2个月左右，就不满足了，所以就按照月的方式购买，当时的技术是经典的PPTP和传说中穿透力很强的L2TP，以及以tls加密为特点OpenVPN，而这些技术后面我才知道，本身不是用来翻墙的，而是用来企业的员工访问企业服务的专用通道，由于这几种底层技术特征很明显，所以GreenVPN为代表的这种VPN供应商服务一个月不如一个月，大概1年多以后彻底不行  

# Shadowsocks时代  

  GreenVPN用不了以后，就得找到新的梯子，因为有了一定计算机技术能力，所以就琢磨自己来。  
  
  在搜索中知道了Shadowsocks(后简称SS，基于Python的版本是第一版)这种划时代的存在，就算后面作者clownwindy被请去喝茶，但是SS仍然被开源社区的人们延续了下来，直到现在都还有不断针对墙来做的开发升级。
  
  如果你搜索SS，你也会知道还有一个和SS很像的SSR。SSR是当SS作者被请喝茶后，某女大学生破娃酱Fork出去继续开发的一条分支，后面成了自己独特的一个品牌，本身架构和思路和SS是一致的，只是在SS上加了很多功能特性，比如混淆，当然后面的SS-libv(C版本)也实现了这些功能。
  
  在我认为SSR被当成SS后的最佳解决方案的时候，破娃酱却因为自己被人肉，压力大，停止更新SSR，并且删除源码库，但是会给大家Fork的时间。当时电报群第一时间知道消息的我赶紧Fork了一份SSR代码，尽管我知道，我就算Fork了一份也不一定有机会开发，但是众人的智慧和血汗不能就折在这里了啊。  

  SS/SSR的速度和稳定性都比旧的VPN时代的好，以前VPN时代每天都出现连不上的情况，在SS/SSR时代可能一周才会出现一次延迟高的情况，当时选择的是vultr tokyo的VPS，每个月5刀。我只买了一台VPS，没有考虑过单点故障挂了怎么办，因为当时时间多，而且SS/SSR似乎发展很好，所以没有多虑，SS/SSR时期的梯子平均在200-300ms左右的延时我就满足了，
  虽然现在看来这种延时还是太慢了，但是相对于VPN时代，我总算对稳定性和速度有了要求，这是好事，有要求意味着进步。  
  
  BTW，我也是在SS/SSR时代才开始大量逛油管，玩IG，在网上时间基本都访问国外的网站，可以说快速稳定的梯子是这一切基础和开端。  

# V2ray时代  

  在SSR宣布停止更新代码后，我就知道要开始找下家了，因为墙一直升级，虽然当下SSR还挺好，但是万一有一篇论文说能正确识别SSR流量，那么SSR的节点就可能一夜之间大规模覆灭，有时候不得不承认为墙工作的人也是很聪明的。  
  
  然后我就遇到了V2ray，作者在国外，天然不怕被喝茶，只是更新很慢而已。V2ray发展出了一个更强大的概念——平台化，它不单只提出了自己Vmess协议，而且支持协议可配置，支持SS,SSR多协议接入，并且吸取了SS/SSR发展至今的一些优秀的设计。
  
  如获至宝的我赶紧切换到Vmess上，测试以后发现果然没有让我失望，无论是稳定性和速度都很好，折腾V2ray这个时期大概是我刚大学毕业半年的时候吧，然后这种方案一直用了大概2年后的今天，中途上了BBR，也优化了VPS的一些TCP参数，一番优化后访问油管4K视频完全没有压力，后面因为遭遇墙的升级影响，被迫上了websocket伪装和tls加密，也专门买了个域名做tls加密。  

   当能优化都优化的时候，还是会出现无法出去的情况，我就考虑到可能是节点的问题反而成为主要的影响了，因为最新的V2ray毫无疑问是当前最好的FQ技术。那么怎么解决节点的问题？节点的测试和调试都得需要我花费大量的时间精力和金钱，如果还是学生时代的我，可能还消耗的起，但是毕业工作两年后的我就没有这么多时间精力耗在这件事情上了，不然浪费的时间精力金钱都够我肉翻了，所以在只是增加了一台搬瓦工的LA节点作为备胎后，拥有两个节点的我开始思考起有没有高ROI的方案了。  

# 机场时代  

   今年的一场同学聚会上，我从同学A了解到他一直在用机场FQ，虽然之前我也有搜索的时候搜到过机场的概念，但是没有当回事，直到最近两个节点频繁连不上，我才开始用了前后大概一天的时间研究机场要怎么玩。

   其实机场的概念不是很复杂，是我现在这种双节点的增强版，底层技术也是SS/SSR/Vmess这三种，只是机场主有很多节点(是我两个节点数目的10倍，20倍)，通过共享VPS，分摊费用和成本，这种东西个人做风险很大，一般得有一定资源和背景的人才敢做  

   最终我读了几篇机场测速文章后，就包月用上了机场，随后我就发现原本我养了两个节点VPS的缺点在哪里:  

     - 需求上有浪费:我只是要梯子出去而已，并不需要计算和存储，所以VPS的计算资源和硬盘空间我是浪费的了，而且流量每个月只是用到1%不到，更加浪费;  
     - 技术上费时费力:墙在升级，梯子也得升级，而且梯子载体VPS也得维护，我越来越没有这么多时间投入到这个技术的军备竞赛中了;  
     - 节点上不灵活:机场通过分布各地的节点，平衡了墙无脑封杀某片区域的几率，用户更灵活地切换线路;  
     - 费用上性价比低:因为合理按照自身使用率选择套餐，所以减少了资源浪费，所以因为资源浪费的那部分费用就省了下来;  

   现在机场的延迟在100ms以内，甚至20ms内，真是快到不相信自己眼睛，不过托机场的福，我得把我原本macOS客户端的V2rayX换成V2rayU以支持订阅功能(试过ClashX虽然有订阅，但是本身不好控制就Pass了)，手机iOS的小火箭shadowrocket本身支持订阅，也就不用折腾着换成Surge  

   因此，我又从买服务FQ到自己搭梯时代再回到买服务上，但是以防万一，我还是留了一台VPS作为备用，毕竟机场的目标有点大，就怕它哪天突然挂了，我还能有出去购买其他机场的手段  

# 未来  

  其实一路走来，技术不断迭代，墙和梯子一直都在军备竞赛，只要保持更新就不会出不去，因为出去的需求一直都在，只是当中的成本问题。
  
  本质上，个人的使用涉及的其实是一个钱的问题，所以与其注意力集中在梯子技术升级上，不如花费到自己真正感兴趣的方向上，然后把FQ交给专业的人去做就好了，毕竟我也是经历过几个FQ技术迭代周期的人，已经清楚自己想折腾的不是这个，也清楚自己的需求究竟是什么，配置合理的成本。  

  如果想从机场时代再升级，那么下一个目标就是肉翻了，2年前我就有这个念头，只是肉翻不是说说而已，成本巨大到保守估计得消耗自己15年到20年的青春才能肉翻成功。但是我相信，只要一直往这个方向走就能成功，经验也是这么印证着，美好的时间不应当花在和GFW的斗争上，我自己的在FQ这件事上的技术热情已经没有了，后面可能只是维护一台备胎VPS，有空有心情才会更新一下FQ动态了最近技术，至于像之前拼命买节点，测试节点，BBR，TCP优化，上ws，买域名上tls是不会有了。  

# 线路
  20200801更新

  这里再补充一下节点问题吧,因为突然连接不上自己的机场和某个友人聊到了节点。上面都是讲技术了，如果说技术影响的是能不能翻出去，那么速度就是靠节点的网络环境好不好了，比如到现在2020年，CN2 GIA线路就是个不错的选择  

  0.供应商:GCP/AWS，Linode/Vultr/Bandwagon/GigsGigsCloud,千万别用阿里云/腾讯云这种国内容易审查的云，做内网传输的伪IPLC也不行  

  1.普通163(ChinaNet-AS4134): 就是电信用户最经常遇到的电信线路，等级最低，省级/出国/国际骨干节点都以202.97开头，全程没有59.43开头的CN2节点。在出国线路上表现为拥堵，丢包率高  

  2.CN2-GA(英文Chinatelecom Next Carrier Network，缩写为CNCN，进一步缩写为CN2,AS4809):CN2里属于Global Transit的产品(又名GIS-Global Internet Service)，在CN2里等级低，省级/出国节点为202.97开头，国际骨干节点有2～4个59.43开头的CN2节点。在出国线路上拥堵程度一般，相对于163骨干网的稍强，相比CN2-GIA，性价比也较高。CN2-GT中国国际出口拥有自己的单独线路，但是在国内的链路还是使用的163骨干网络  

  3.CN2-GIA线路: CN2里属于Global Internet Access的产品，等级最高，省级/出国/国际骨干节点都以59.43开头，全程没有202.97开头的节点。在出国线路上表现最好，很少拥堵，理论上速度最快最稳定，当然，价格也相对CN2-GT偏高。整个GIA的出口带宽较小，在较大流量攻击的时候更容易导致整GIA下的网络波动  

  4.单双向CN2:我们使用mtr或者traceroute，通过跟踪网络包的路由节点，来判断具体的网络承载类型  

  5.IPLC线路(市面上常见的是使用阿里云内网传输的伪iplc，或是采用iplc线路的NAT产品，产品特征是不过墙，延时低，速度快)  

  6.NAT-VPS:NAT-VPS是共享同一个公网ipv4)地址，通过端口映射方式与外界通信和提供服务的VPS。NAT VPS的主要缺点是能使用的端口有限制，一般十个左右，并且大多数商家不允许选择端口号。但NAT VPS有它盖不住的优点：便宜。NAT-VPS省去了ip费用，带宽也是共享的，价格一般比普通vps要便宜不少  

  6.BGP AR宣告:BGP(Border Gateway Protocol)边界网关协议是联通，电信和移动之间能够互通的基础  

  CN2决定了网络质量会优于普通163承载网络，但也不一定，除了承载网络之外，机房的地理位置也很重要。比如Google Cloud Platform (GCP) 的台湾机房，用过的同学都知道，虽然没有走CN2网络，但是由于地理位置很近，速度还真是快得一逼!  

  中国电信骨干网的三大国际出口分别是，北京，上海，广州，全国的出口网络最后都会汇集到这三个出口点  