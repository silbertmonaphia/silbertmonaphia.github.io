---
title: 当Kali Rolling作为笔记本唯一一个系统
author: smona
published: true
date: '2016-10-08 13:53:21'
layout: post
---
# **前言**
  最近原来的Win7一个dll文件损坏，然后自己去找了一个dll文件替换，结果替换了两下开不了机了，用/sfc命令企图让win7自己修复，然后正中了那句话：win7的修复功能10次有9次不成功的，windows一直给我一种底层一直都很神秘，用户自己不容易维护，每次出问题想修复问题，但是最后都只能重装系统（错误信息不够精准和详细），所以一气之下，就把SSD的数据备份出来，直接格掉原来的win7，上了个Kali rolling～
  
   不过这么帅气的冲动的举动往往带来严重的后果，因为我对linux系统还没有完全了解，之前到最近才接触了不到一个多月的linux，现在直接整机换linux，有些现实的问题直接摆在我的面前，现在我就来分享一下换了Kali以后的一些故事。
   
   众所周知，Kali，Ubuntu,Mint都是Debian一派的，所以一派之内除了linux那套以外，有更加多的相似的地方，所以我从Mint到Kali不算是特别陌生，但是Kali给我的感觉更加依赖命令的输入执行，对鼠标操作会更加少一点。

   首先配置源的问题，我先给出我自己用的源：

```
#kali官方源
deb http://http.kali.org/kali kali-rolling main non-free contrib
#中科大的源
deb http://mirrors.ustc.edu.cn/kali kali-rolling main non-free contrib
   
```
官方和中科大的源都很好，我自己是教育网，感觉中科大隐隐约约在update的时候和upgrade的时候会比官方kali源快一点，实际你可以自己测试一下。

   接下来不是直接开始干活，你还要配置你的工作环境，这个问题你在搬家之前完全不会想到搬家过来各种软件细节排错问题会让你忙上一个星期，加上合乎笔记本硬件以及自身工作学习需求的搭配设定可以吃掉你一头半个月的时间，让你有种突然回到刚开始用windows的时候的感觉——不知道用什么软件更好满足自己的需求。
 
   经过我半个多月的实践，试验，对比，体验，期间还在linux上写了一个科技制作的登录注册界面和完成了将近1万字的形式政策论文，我选出了一些开源软件能基本满足工作学习娱乐需求，以下是推荐：

**0.中文输入法**
----

刚刚装好linux,作为中文使用者，最应该首先要做的就是中文输入法，那么fcitx就是不二选择，中文名叫小企鹅输入法还有记得安装相应的中文字体。
fcitx在源里面就有，所以直接apt-get install fcitx (安装输入法框架)

16.6.8补充：从安装kali直到现在都是用fcitx，当初也想装搜狗拼音输入法，但是恰好要写超长的论文什么的，就直接用fcitx自带的拼音输入法顶上了，但是最近闲下来还是希望能用上搜狗拼音输入法，一方面是为了它的云词库，另一方面是智能推荐什么的（fcitx拼音打字还是不怎么流畅）。多亏了搜狗和Ubuntu Kylin有合作，所以官方有deb包，dpkg系的都很简单就可以安装了。因为linux搜狗拼音输入法是基于fcitx框架的，所以搜狗拼音输入法之前要装fcitx，我之前装了fcitx，但是在装的时候还是没有满足依赖关系，用sudo apt-get -f install修复依赖关系就好了，然后再安装下载的.deb包就好了，重启输入法，就可以愉快地使用搜狗拼音输入法了，体验了几天，虽然没有办法登陆，而且皮肤也没有多少，但是输入体验很好。

16.7.2补充：之前没有详细讲中文字体问题，其实这个问题没有弄清楚也挺严重的，特别是对于滚动更新而言，因为总是要维持最新版本，所以极有可能会出现中文乱码，全是方块的情况，谁叫一般最新版本都是英文的呢。http://www.linuxdiyf.com/linux/20701.html 这是我找到并且在kalirolling上验证可行的中文乱码解决方法：本来如果我一apt-get dist-upgrade就各种方块出来的包括浏览的网页内容也是方块， 所以在你刚装好系统后，用apt-get install locales 确认机子有没有安装locales，如果安装了会有提示的，如果没有安装这个命令就正好可以安装，在输入dpkg-reconfigure locales打开图形界面，向下找到en_US.UTF-8和zh_CN.UTF-8,将en_US.UTF-8选为默认，之后apt-get install xfonts-intl-chinese  xfonts-intl-chinese-big  ttf-wqy-microhei  ttf-wqy-zenhei安装中文字体,安装好以后重启即可。

16.9.1补充：今天作死改了/etc/passwd的内容（作死的我把我的用户的UID改成0～），结果马上就出问题了，那就是终端里面无论你做什么都会说不认识你（我UID明明改成了0，非要说不认识我这个1000UID的用户....瞬间一头黑线），用其他tty登陆root把/etc/passwd内容修改回去，再回来登陆gnome，gnome就变得奇奇怪怪，最严重的是，gnome-termianl都打不开了！

到其他tty察看到/var/log/syslog，找到事发时间段，发现错误提示 org.gnoem.Terminal:/org/gnome/Terminal/Factory0:Error calling StarServiceByName....（还有一大段的见下图）
![这里写图片描述](http://img.blog.csdn.net/20160901231637607)

果断谷歌之！找到这个——https://bbs.archlinux.org/viewtopic.php?id=180103，虽然不是完全相同的情况，但是尝试用里面的建议做了locale的设置 ——localectl set-locale LANG="en_US.UTF-8"，reboot后可以启动gnome-terminal了！喜大普奔～我完全没有想到一个语系问题竟然直接可以导致gnome-terminal无法唤出....然后再重新设置了一边语系：终端输入dpkg-reconfigure locales打开图形界面，向下找到en_US.UTF-8和zh_CN.UTF-8,将en_US.UTF-8选为默认，由于原本我们的字体就装好了，装中文字体就省略了，接着重启基本上问题就可以解决了，如果出现弄不出搜狗拼音输入法的情况，请直接到系统设置里面改区域语言里面的语言选项就可以了～

感悟：用的起linux就别怕折腾，也别一出问题就重装！如果一出问题就重装，那和windows有什么区别，linux还是很好fix的，而且google神器在手，要相信神器的力量，相信开源的力量～还有注意备份，我已经三四次后悔没有备份了！

**1.浏览器**
----

最好用google-chrome-stable,chromium虽然从源安装很方便但是界面的确很丑，chrome访问网页体验好，稳定，简洁，省心.

Firefox比chrome好在扩展好，而且和chrome内核不一样，两个互相补充，基本能应对绝大多数网页.


**2.VPN/SS**
----

首先把OpenVPN,和PPTP装上,你要愿意还可以装上network-manager-ssh
说到VPN，我这里推荐一下国内的：
我一般都是用GREENVPN，虽然技术可能有点更新慢，而且vip还偶尔断线，但是作为老牌，它不会某天就人间蒸发，携款潜逃，还是挺好用的;我也试过超级VPN和绿豆VPN（最好的加速兔已经被绿豆合并了，而且加速兔没有linux客户端要wine解决，我就放弃了），不过单独在chrome的插件还是有，不过它没有月收费，一下就要年费才能用，这对新用户一点都不友好，听说有免费措施，但是相对于green来说对新用户还是差一点，不过听说速度挺快而且不断线，像我这种只是google,youtube等上上网的用户就算了，土豪和要玩外服游戏的朋友可以试一下

16.7.3更新：更加进阶的是自己通过例如班瓦工的海外VPS搭建一个VPN，这个需要技术更加深一点，用到诸如shadowsocks的技术，但是速度有保障，而且除了当VPN还可以同时干一些其他的活儿，毕竟是一台云主机～

17.12.28更新:当前比较好用的是v2ray,SS现在波动太大,本来想写一篇梯子教程的，但是思量一番觉得意义不大，因为梯子技术层出不穷，一开始以为能坚挺的SS,结果被喝茶,另一个版本的SSR，也逃不过停更，而且检查越来越厉害，当下v2ray也只是因为比较新所以才得以幸免。因此在如此更新频繁下去写一些教程，我个人认为没有多大意义，教程应该更新到项目本身，项目本身文档说明足够详细，也就没有我们在博客里面写什么梯子教程了吧

**3.Office软件**
----

个人感觉用OpenOffice比wps好，wps还要特地找缺少的字体，折腾的累了还是OO吧。
P.s.关于在windows的兼容问题，建议你在linux上用OO编辑好的文档比如说word文档（其他excel和ppt我还没有试过）保存为.odt，不然如果你保存为.doc文件在windows系统上打开原来一些样式全部不见了，包括一些缩进和排版，混乱不堪，而你保存为.odt的话，现在word2007支持.odt的文档，你用word2007打开还是能保留样式的
如果你有其他linux的OO到win下的excel或ppt的乱码或者格式解决的好方法，欢迎下方评论。

16.7.3更新：最近各种大作业汇报都要用ppt展示，因为课室的电脑清一色的windows，所以兼容性问题凸显，OO不能完美解决，特别是样式问题，也想过用云端解决方案，比如Ipresst,但是用过你才发现，就算是弄一个简单的展示，也做起来特别痛苦，至少比原来在PPT上做痛苦多了，特别是处理位置关系，也不容易有全局掌握逻辑，所以像之前说的，如果不需要多平台的展示就OO，用OO看个内容就好了，如果要兼容性，linux下就试下WPS～这样兼容性有提高。

OpenOffice在kali rolling的源里面没有，你还是到官网下载个deb包装吧：https://www.openoffice.org/download/index.html

16.7.8补充：WPS的安装，首先到官网下载一个wps for linux 的deb包，dpkg -i 安装之，你会遇到两个问题：1.缺少libpng12-0，你可以自己百度谷歌找找，我这里用的是debian自己的deb包：https://packages.debian.org/search?keywords=libpng12-0；2.成功安装wps以后，打开wps时候报错“系统缺失字体symbol、wingdings、wingdings 2、wingdings 3、webding”，一种解决办法就是你到windows里面找相应的字体，还有一种就是到http://download.csdn.net/detail/tao_627/7221953 这里下载这个deb包安装之（这个deb包原来是在wps社区流出来的，现在社区里面不知道为什么找不到了）。

这里再附上一篇LibreOffice-OpenOffice.org之间的历史以及中文稳定版安装教程：http://www.hfu.edu.tw/~kwc/CourseBulletin/SoftwareApplication/OOo.htm
吐槽一句：自古编码和字符兼容问题就是最麻烦的存在.

**综上所述：如果你不想像上面我这样，各种辗转反复，那我推荐你直接用WPS，最开始我是因为怕去找字体，所以才去弄WPS，不过上面的16.7.8补充中，由于LibreOffice和OpenOffice这俩货的兼容问题在中国来说是非常严重的，所以我就辛苦点还是去找到了字体，然后把WPS装了起来。在中国，可能除了Windows自家的Office办公软件以外，最多人用的，最多人电脑上有的应该就是WPS了吧，这就解决了文件传播问题，其实为什么不像国外一样，尽可能用pdf文档呢，又干净又舒服，而且还统一.....**

**4.思维导图**
----

freemind和xmind在Linux界面都很丑，VYM我也用过，也没有觉得多好，还是直接用云端的百度脑图，它还支持导出到本地，双重保险。


**5.音乐**
----

我用的是Github里面的一个项目，CLI界面的网易云，叫musicbox，可以自己到github里面搜索，体验还不错，不过挂上国外的代理配置我还不会，所以我一接上VPN就听不了;还有它的下载（缓存）还不太会，它是用shift+c缓存，然而我并不知道它缓存到哪里，缓存成啥样了...貌似最近网易官方又出了for linux的GUI的网易云，有空让我的Gnome也去试试.

16.7.2更新：昨天到现在下载了网易自家官方的网易云音乐for linux的deb包，各种依赖问题，包括libfontconfig1的版本过旧问题（源里面那个不符合官方的网易云音乐的安装要求），还有freetype2缺少问题等等，觉得非常麻烦，而且还需要了夹层依赖问题，就是往前依赖其他软件，往后依赖其他软件的问题，非常麻烦，建议要用图形界面的网易云音乐的话得非常能折腾,或者直接上Ubuntu也可以，Ubuntu装官方的网易云，非常简单～不然用CLI的网易云足以。


**6.视频**
----

MPlayer,虽然系统自带那个VLCplayer已经很不错了，还是多配一个吧.


**7.代码编辑**
----

用的是Sublime-text-3，下面简称ST3（不建议用2代的，因为现在开发的都到3代了，有些问题解决不好），因为它扩展性很好，特别是还有印象笔记扩展，一举两得，满足我的需求，就没有用"notepad++"了，不过这款编辑器还得再琢磨琢磨，至于gedit什么的玩玩就好了，扩展性能并没有ST3好，最重要是不知道怎么用gedit写evernote.[我是evernote患者]
当然字符界面当然用vim/vi了，这里就不多说了。

16.7.4补充：上面没有展开说，觉得现在有必要补充一下，首先ST3为什么好用，是因为他有强大的Package Control，有了这个插件，就可以有很多很多好用的插件便捷安装，ST3对于你而言，要解决的问题，是中文输入的问题，你会发现在ST3内你是无法直接输入中文的，可以参考这个方法：
http://www.jianshu.com/p/bf05fb3a4709 ，然后插件我就不推荐了，特别说的一个就是，ST3的Package Control里面就有evernote(印象笔记插件)，在ST3里面用evernote是我目前为止在linux里面见过最好的evernote替代方案，因为官方并没有for linux的evernote！不过还是要各种配置，ST3才能跑的飞起来。其实你可以发现，好用的都是轻量可扩展性强的，但是要用户自己折腾定制的软件，既然你用了linux，那就注定你是为折腾而活。


ST3官方地址：https://www.sublimetext.com/3
Package Control地址：https://packagecontrol.io/installation
如何在ST3里面使用Evernote：http://blog.saymagic.cn/2015/06/20/write-blog-by-sublime.html
Evernote for Sublime Text的github地址：https://github.com/bordaigorl/sublime-evernote


而对于vim，说它好用，前提是你要折腾它好多遍，原生没有配置的vim并不好用，而好用的，那些人说的六到飞起来的vim是经过配置的适合自己的vim，所以你不要刚用了会儿原生的vim，就说不好用，你学会配置后就不会这样了，配置时候比较痛苦吧，不过也是一劳永逸的事情，因为它的配置文件可以带走，你还可以把配置丢到你的博客，在新的需要配置的vim，你可以直接拷贝过去，所以说vim神器就是这么来的  

16.10.4更新:  
最近要经常用到vim了，所以就稍微做了点常用的vim设置，这里贴出来备忘:
```
" /etc/vim/vimrc
syntax on
 set nu             "设置行号
 set cursorline     "高亮当前行
 set expandtab      "全文tab替换成空格
 set softtabstop=4  "tab应替换为4个空格
 set ts=4	        "设置tab输入长度为4空格
 set autoindent     "自动缩进
 set shiftwidth=4   "换行时自动缩进的宽度

```
其实上面我还故意省略很多带有英文双引号（"） 的语句，这些双引号是注释的意思，那么注释不起作用的语句我就省地方不贴出来了
然后还有很多更加好用的配置和插件什么的，以后有机会再补充吧，其实讲真如果不经常用vim来大规模编辑程序，上面这些常用的设置应该就能满足你了  

16.10.8更新：vim中是通过:set fileencoding:utf-8来修改文件编码类型的，而shell中用enca命令来查看编码格式。

**8.网页上的视频**
----

因为现在网页上的视频大多数还在用Flash播放（可怜的我不装Flash连B站视频都看不了，B站何时升级成HTML5啊！），HTML5还没有普及，所以安装一个Flash是有必要的，虽然对于Flash，linux一直支持不好，各种消耗资源，只有期待HTML5快点普及，那时候就可以把Flash卸载了.
从源安装flash即可，apt-get install flashplugin-nonfree

17.2.20更新：
"Adobe Flash 无法正常使用"的问题  
嗯，2017鸡年春节已经过了，chrome的也更新了，但貌似在更新中也把浏览器的flash插件禁用了，所以有些网页上有些flash会看不了，之前flashplugin-nonfree的方法不太可以了，然后稍微谷歌了一下，发现chrome的帮助里面就有["Adobe Flash 无法正常使用"](https://support.google.com/chrome/answer/6258784?hl=zh-Hans),跟着做就可以fix这个问题，提个醒，第二步在chrome://components中更新中需要走代理更新，不然的 Adobe Flash Player显示的还是0.0.0.0,然后其实我这会儿已经是 24.0.0.221这个版本了，至于其他浏览器，应该就有其他浏览器的办法，我只是以最常用也最爱用的chrome做例子，不过网页flash的问题不大，因为以后都是html5的天下，随着各种页面的升级改造，flash就快成为过去式了，而H5页面的视频播放才会成为主流



**9.发热问题**
----

现在很多笔记本都带双显卡，Nvidia公司使用optimus（擎天柱）智能切换集成显卡和独立显卡，linux则用的是bumblebee（大黄蜂）。

如果不是特别需要独立显卡的话强烈建议把独立显卡直接在bios关掉，省的去配置bumblebee,而且还可能遇到就算bumblebee正确配置，可是独立显卡还是呼呼在跑的奇葩问题，劳心劳力。
最后，偷偷告诉你，看1080p的视频Intel自家的集成显卡完全够用，当然你跑游戏那我没话说，这你就自己折腾去吧。

在linux里面解决了独立显卡的发热大户，基本发热就有显著降低，剩下就是CPU这个发热问题，因为在windows下是有智能调节CPU等资源的管理软件的，但是linux不找就没有，CPU就呼呼地一直全速前进，不管你是否在工作还是休息.

那么这里推荐两款软件：

【监控】一个是显示系统情况的软件conky,conky是对系统（甚至是一些非系统的）信息的集成，特别是它强大到可以运行一些脚本程序扩展它的功能（conky的execi语句等），在打造精美桌面的同时，也作为信息汇集中心.

【调节】另外一个是用户制定策略调用系统cpufreq来控制CPU频率的软件cpufreqd，不过你得学学linux系统内核管理CPU频率的相关知识以及怎么按照自己需求定制cpufreqd配置文件.

**【监控】Conky** 

<https://github.com/brndnmtthws/conky/wiki/User-Configs>

2016.6.11补充：光是上面的github的东西还不够，那里面的东西看的一头雾水，感觉并没有讲清楚conky的配置，里面值得一用的只是那些大神给出来的超赞的配置方案.而对于第一次用conky的人，如果你英文好，建议先用man仔细看下conky的说明，然后打开默认的/etc/conky.conf，对着这份配置文件先一条条看看，再对照manpage的说明，了解每一句都代表什么，主要关注点在conky.config和conky.text后面两个大括号里面的内容

manpage不方便就这个
<http://conky.sourceforge.net/config_settings.html>

如果你看不懂英文，没问题，有人做了部分翻译，对比着看吧
<http://www.mikewootc.com/wiki/linux/usage/conky.html>
<http://zoomq.qiniudn.com/ZQScrapBook/ZqFLOSS/data/20091213015339/>
<http://www.coctec.com/docs/linux/show-post-177305.html>

最后给出我自己的配置，先上图：
![这里写图片描述](http://img.blog.csdn.net/20160611145325108)

对应的再贴上我的/etc/conky.conf 里面的配置内容，不建议你直接复制，最好自己对着每一个条目，看怎么设置出相应的内容

```
conky.config = {
	minimum_size =280, 5,
    alignment = 'top_right',
    background = true,

    gap_x = 10,
    gap_y = 35,

    own_window = true,
    own_window_colour ='black',
    own_window_type ='normal',
    own_window_class = 'Conky',
    own_window_type = 'desktop',
    own_window_hints ='undecorated,below,sticky,skip_taskbar,skip_pager',
    

    own_window_argb_visual =true,
	own_window_argb_value =128,
	own_window_transparent =true,


	no_buffers = true,
    double_buffer = true,

    use_xft = true,
    text_buffer_size =2048,
    override_utf8_locale =true,
    xftfont = 'Monospace:size=10',
    xftalpha = 1,

    default_color = 'white',


    draw_outline = false,
    default_outline_color = 'white',

    draw_shades = true,
    default_shade_color = 'black',

    draw_borders = false,
    border_width = 1,
    stippled_borders = 0,

 	    
    use_spacer = 'none',
    uppercase = false,

	net_avg_samples = 2,
    cpu_avg_samples = 2,
	update_interval = 4,

    draw_graph_borders = true,
    out_to_console = false,
    out_to_stderr = false,
    extra_newline = false,
    show_graph_scale = false,
    show_graph_range = false
}

conky.text = [[
${color}$alignc $nodename
${color}$alignc $sysname $kernel on $machine
$hr
${color}$alignc ${time %Y %m %d %A} 
${alignc}${time %k:%M}
${color}$alignc Uptime:$uptime
$hr
$freq Mhz  ${alignc}Temp:$acpitemp°C
Core1:${execi 6 /usr/bin/sensors | grep Core\ 0| paste -s | cut -c15-18,19-21}°C
Core2:${execi 6 /usr/bin/sensors | grep Core\ 1| paste -s | cut -c15-18,19-21}°C
${color}CPU1:${color white} ${cpu cpu1}%       ${color}CPU2:${color white} ${cpu cpu2}% 
${color}CPU3:${color white} ${cpu cpu3}%        ${color}CPU4:${color white} ${cpu cpu4}% 
${color}RAM:${color white} $memperc%  $mem/$memmax 
/ $alignc${fs_used_perc /}%  ${fs_used /}/${fs_size /} 
/home $alignc${fs_used_perc /home}% ${fs_used /home}/${fs_size /home} 
$hr
Processes:$running_processes/$processes
Proc-Name            PID   CPU%   MEM%
${top name 1} ${top pid 1} ${top cpu 1} ${top mem 1}
${top name 2} ${top pid 2} ${top cpu 2} ${top mem 2}
${top name 3} ${top pid 3} ${top cpu 3} ${top mem 3}
$stippled_hr
Mem-Name
${top_mem name 1} ${top_mem pid 1} ${top_mem cpu 1} ${top_mem mem 1}
${top_mem name 2} ${top_mem pid 2} ${top_mem cpu 2} ${top_mem mem 2}
${top_mem name 3} ${top_mem pid 3} ${top_mem cpu 3} ${top_mem mem 3}
$hr
WLAN:${addr wlan0}  
ETH:${addr eth0}
PPP:${addr ppp0}
Up:${upspeedf wlan0} k/s                   Down: ${downspeedf wlan0} k/s
Tot: ${color}${totalup wlan0}             Tot: ${totaldown wlan0}
Bat $battery_percent%$battery_bar
]]

```
注意上面代码中conky.text后字段的/usr/bin/sensors说明要用到sensors，用apt-get install lm-sensors即可，不然显示不出来。
16.10.4更新：真的不建议你复制粘贴，因为我今天把上面这个配置文件替换掉我在Ubuntu的配置文件就位置错乱，而且不一会儿就自己消失了，呵呵....


**【调节】Cpufreqd** 
----

<http://skyao.github.io/2015/08/08/linux-cpufreqd/>


P.s.什么？你说还有硬盘发热？不好意思我用SSD的，SSD基本最高也就45度，根本不成问题


**10.通讯**
----

QQ和微信都是现在社交甚至工作都不可少的东西（这两样还都是大企鹅的...）,而且现在都喜欢直接用扣扣或者微信传文件，都不用邮箱了，其实我倒是挺喜欢用邮箱的.

这里推荐扣扣用wine qq2013 32位的，如果是64位系统的童鞋记得装32位的版本的wine，因为默认系统如果有装wine那只有64位版本的

清风网络空间里面给的方法wineqq2013地址：

http://phpcj.org/wineqq/


16.7.3更新：使用了一段时间，wineqq真的用的很难受，透着浓浓的2000年的设计气息，而且输入密码稍微有个大写的，你得弄半天，体验也不好，如果能不用QQ传输文件的话，就别用这个wineqq。

16.7.9再次提醒一句，这个wineqq得wine32位的才能跑，所以你要知道自己机子的wine究竟是不是wine32，wine64是没反应的，wine32装起来也简单，直接源里面就有～而且清风这里的wineqq貌似是持续更新的，感觉越来越好用，所以收回7.3号不好用的发言，个人表示非常支持～
（持续更新就意味着越来越好～）

而微信在github已经有好人写了一个，你跟着弄就好了

wechat的github项目地址：

https://github.com/geeeeeeeeek/electronic-wechat/blob/master/README_zh.md

16.7.3更新：注意一下nodejs直接apt-get install nodejs就可以了,然后如果发生node:not found 的错误，别担心，这里http://stackoverflow.com/questions/21168141/cannot-install-packages-using-node-package-manager-in-ubuntu 说了，"In summer 2012 Debian maintainers decided to rename Node.js executable to prevent some kind of namespace collision with another package",改名以后就会有新旧名称兼容问题，因为虽然人知道nodejs就是node但是程序很死板，nodejs你不说它就不知道等于node，这里单纯用ln -s /usr/bin/nodejs /usr/bin/node，再npm install 还是会报错，所以还是从源里面再下载一个nodejs-legacy ，sudo apt-get install nodejs-legacy,亲测后面这种下载nodejs-legacy以后再运行npm install以后可以成功安装完不报错，然后npm start也能成功启动～  

16.8.19更新：如果在npm install过程中中断的话，如果这时候node_modules下面的模块没有下载完的话，会出现"npm warn unmet dependency ..which is version x.x.x"的错误，而且没有办法跳过，解决办法是用npm uninstall把原来已经部分安装的模块先卸载干净，再npm install.  

16.8.20更新：本版本今天崩了，请看[issue 318](https://github.com/geeeeeeeeek/electronic-wechat/issues/318)
可以用着CLI界面的顶着，如果你能接受的话，这是link：https://github.com/sjdy521/Mojo-Weixin


**11.PPA的问题**
----

有些新手可能不了解PPA，其实PPA在linux世界里面挺普遍的，如果说/etc/apt/sources.list记录的是是官方源，那么ppa记录的就是私人源，你可以当做第三方软件之类的。
先检查自己有没有安装python-software-properties这个工具包，没安装用不了add-apt-repository这个命令。
然后就sudo add-apt-repository ppa:user/ppa-name
你问我比较大的第三方源软件在哪里找，我推荐lauchpad.net,墙内也能上～

16.7.2更新：源里面的python-software-properties 被software-properties-common代替了
16.10.4.更新：Ubuntu16.04里面竟然已经有了software-properties-common，就欺负我们Kali维护人少....


**12.触摸板的模拟单击问题**
----

这个问题其实也挺冷门的，kali rolling在之前的版本里面的设置里面的鼠标和触摸板设置里面就有“单触触摸板作为点击”的复选框，但是在更新后面不知道哪个版本更新后，我就神奇地发现，当我单触笔记本触摸板并没有反应，我习惯不是用触摸板左下方的左物理按键作为左键点击的，而是直接单触触摸板作为鼠标左键点击的，我找了挺久，也看过试过很多方法，最后找到这个方法可行：     http://blog.csdn.net/lonelygo/article/details/7392744，直接看第三点就好了，就是说找到/usr/share/X11/xorg.conf.d/50-synaptics.conf 这个文件（注意也许/usr/share/X11/xorg.conf.d是一样的，但是50-synaptics.conf有可能是不一样的，我的就是叫做70-synaptics.conf，具体你得自己看看你系统的相应目录下是什么名字 ），用vim打开文件，找到类似于下面这段代码的部分改成下面这代码的样子，然后重启就行。

```
Section "InputClass"
      Identifier "touchpad catchall"
      Driver "synaptics"
      MatchIsTouchpad "on"
      MatchDevicePath "/dev/input/event*"
            Option "TapButton1" "1"
            Option "TapButton2" "2"
            Option "TapButton3" "3"
EndSection 
```



**13.修改开机grub等待时间**
----
16.10.4更新：
  默认的grub默认等待时间是很长的（我指的是在选系统启动那里），一般少则10s，多则30s，如果你习惯自己按还好，可是强迫症如我就想着去改系统配置，找到/boot/grub/grub.cfg这个文件，如果权限不够就用管理员权限给他一个可写的权限，然后vim，找到10/30的地方（我这里是Ubuntu/Kali是10/30,其他发行版有待实验），改成1或者其他你喜欢的数字就可以了，这样你就可以在开机时候，不用因为忍受不了默认的几十秒而用手去按回车，当然你要说你喜欢按回车或者你喜欢按个开机就走开做其他东西，那我也无话可说，此处是提供给强迫症患者，比如我.


**14.kali终端快捷键**
----
17.2.5更新：
    这个纠结我很久，之前也找过一些"在kali中如何用Ctrl+Alt+T组合键打开终端的办法"，但是貌似找到的方法在kali里面试过都不太可以，知道今天看到的[这篇](https://g2ex.github.io/2016/03/16/Linux-Tips/)，里面说的办法才成功
    设置-键盘-快捷键-自定义快捷键(最下面的"+")，命令(command)中输入x-terminal-emulator，下面的修改(edit)按钮，再按下自定义快捷键，比如Ctrl + Alt + T
    
后记：
刚开始学习系统的话最好的熟悉方式当然是折腾各种软件，有一种说法就是不装上个20次Linux，都好不意思说你懂装linux，当然这里是夸张的说法，但是也说明了不断重装软件以增加对linux的熟悉程度，也更加能欣赏到linux 的好的地方，但是熟悉到一定程度就不要再重重复复装软件了，重装系统了，那就是浪费时间了，所以熟悉到一定程度就得学会备份系统！这样就避免出现问题的时候又要重装，先不论有没有重要数据，光是重新配置就令人发毛，要知道配置一个适合自己的linux环境是多么不容易，所以一定要记住备份系统！！！  

16.10.4补充：折腾了这么多，终于对Kali的环境搭建有感觉了，接下来最好的避免重复工作的办法就是【**封装**】 自己的系统，然后做成系统安装盘，然后就不用从零开始搭建环境了，相当于有了属于自己的系统盘～重新封装我原来以为就dd/dump那样容易，不过谷歌了一下都讲的好像挺复杂的，然后好像我又把封装和备份的概念弄混淆了....

  

**15.CAJ文件**
----
  最近在看准备毕业设计的论文，可是有些文献是caj格式,linux中打开CAJ文件的确麻烦先apt-get install装playonlinux(python写的wine前端)，然后打开playonlinux，搜索cajviewer,如果没有就得自己下一个cajviewer的exe文件，至于怎么安装你可以上网查一下“怎么用playonlinux去安装本地的exe文件”，安装好以后，在playonlinux中打开cajviewer，再打开你要打开的caj文献文件，选择文件＝>打印，如果提示说找不到打印机，就sudo at-get install cups-pdf装一个打印PDF的printer，重启cajviewer，再打印即可转为pdf格式的文件
