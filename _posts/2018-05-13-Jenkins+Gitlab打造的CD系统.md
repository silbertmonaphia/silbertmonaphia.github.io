---
published: true
category: OS
---
* 目录
{:toc}

# 前言  
  终于有机会填坑了，Jekins这东西，从之前学习DevOps开始，一直就围绕这它打转，但是就是没有机会去自己亲手搭建一套，最近有机会自己去搭建一套Jenkins + Gitlab的自动发布流，花了好几天时间，踩了一些坑才把Jenkins有个稍微深入的认识  
  
#　Gitlab  
  在写Jekins之前，先聊聊Gitlab，Gitlab是类似Github的东西，不过虽然Github可以付费成为私有项目，但是对于公司而言，当然还是能把代码放在公司服务器就放在公司自己的服务器更好，因此就有了Gitlab，Gitlab的定位是私建的git repo,以前公司的选择是svn，但是现在越来越多的公司都转向git，那么就希望拥有svn一样私密的特性以及权限管理，所以就有了gitlab。其实我个人的理解很简单，如果是开源的，个人的项目，可以选择Github，如果是个人的而且又想私密的，可以Github付费或者国内的开源中国gitee可以免费建立私密项目，而自建的，那么就是Gitlab没跑了
  
#　Jenkins
  Gitlab的搭建我就不详细叙述了，根据官方的Tutorial一步步上就好了。而无论是gitlab,github,gitee，对于jenkins而言，就只是想要一个repo地址和Webhook，这三者都能提供。

## 安装  
  Jenkins的安装方法有很多，但是我最推荐的是直接jenkins.war来装，因为这个依赖最少，而且最简洁(深受"less is more"影响啊!)，启动直接`java -war jenkins -http-post=8080`即可，如果是想后台的，那么就`nohup java -war jenkins -http-post=8080 &`，对于本身是java的项目，还可以把jenkins.war包对劲Tomcat启动，但是由于自己现在的项目还是Python，服务器上面没有Tomcat之类的东西，所以就多一事不如少一事，保持简单最重要。  
  P.s.官方下载的jenkins的jenkins.war包装好以后竟然是中文的，有点小惊喜，我还想着怎么汉化，真不知道它是怎么检测系统环境的...  
  
##　插件  
  安装好以后就可以访问服务器的8080端口了，比如`http://you_servier_ip:8080`，开始初始化配置，一开始要输入一个管理员密码的，你按照提示到服务器上找就好了，然后还要选择默认插件安装，因为简洁哲学思想影响，默认的插件我插件都没有安装，因为它默认的安装的插件其实对于Jenkins + Gitlab完成自动部署任务来说，有一些不必要的～  
  那么我现在就告诉你，如果想达到" 能够git push到gitlab后，gitlab主动推送到Jenkins部署 "(**这就是我们的目的!**)所必须的插件:　　

```
Requred:  
1.Git plugin
2.GitLab Plugin
3.Gitlab Hook Plugin
4.Email Extension Plugin(为了能够更好地发送邮件，当然jenkins本身也有邮件功能)
5.SSH Slaves plugin(为了能够管理多个节点)

Optional:
1.Subversion Plug-in(如果托管在svn)
2.TODO Plugin(如果托管在gitee)
3.GitHub plugin(如果托管在github)
```

P.s.因为公司的服务器没有直接访问国外节点的能力，而我自己多次尝试更改为国内的比如清华开源镜像和搜狐开源镜像无果后(可以到[这里](http://mirrors.jenkins-ci.org/status.html)看一些jenkins插件的开源镜像站点)，果断转向自己上[官方插件站](https://plugins.jenkins.io/)去搜索相应的hpi，通过Archive跳转，在找到自己想要的版本来下载(其实这里应该贴图更好理解的，毕竟官方的指南也有点不清不楚...，但是原谅本人没有找到时间做博客的图床，所以一直就在尽力用文字描述清楚...)，下载下来后再上传到jenkins上即可

## 配置工程
   我看到有很多教程都要我们去系统配置里面去配置好再去配置工程，但是试验过后发现，就算没有去配置系统配置，但是在工程配置中配好就可以了，提一点，在没有安装插件的情况下，你新建工程，然后到工程的配置中发现，`源码管理`一栏中就只有"无"这个选项，而`构建触发器`这一栏则只有"触发远程构建 (例如,使用脚本)","其他工程构建后触发","定时构建"和"轮询 SCM"这四项，根本没有什么gitlab钩子什么的，所以所谓Jenkins的牛逼能力，从这点就能看出，都是出于丰富的插件生态，对各种玩法的支持上。  
   我稍微说说钩子(web hook，想详细了解可以看[svn最开始关于hook的定义](http://svnbook.red-bean.com/en/1.5/svn.reposadmin.create.html#svn.reposadmin.create.hooks)，hook在jenkins之前，早在svn时代就已经出现了)和"定时构建"以及"轮询 SCM"的区别，这几个对于刚接触jenkins的人而言是比较混淆的存在:  
   
```
1.定时构建:类似于linux的cron任务，一到时间无论代码库有无更新，都会从代码库拉取代码回来构建;
2.轮询SCM:写法也类似crontab，不过行为上和定时构建不一样，它是时间一到就会检查代码库的代码有没有更新，如果没有更新是不是拉取代码构建的，只有有更新了，才会拉取更新代码回来构建;
3.钩子:钩子可以说是触发器这里最佳选择了，上面的轮询的问题是一方面对代码库压力比较大，而且频率最快也只有1min一次，如果在1min的interval里面代码库有提交，那么还是得等完这一分钟才会由轮询方式来监测到代码更新从而触发构建行为，而钩子则改变了主动方，由jenkins主动定期检查代码库，变成代码库一旦监测到有更新，则主动告知jenkins去更新代码，这样就能解决刚才提到的两个问题了
```

  配置工程我就按部就班了，因为网上许多教程都有覆盖基本的配置，我就补充几点，那些教程鲜有提及的一些配置点，因为我在试着配置工程的时候，根据那些教程，有些地方会卡壳，所以我觉得我应该在这里去做补充:  
  
1.general => 使用自定义的工作空间:这个就是确定你的代码要拉到哪里，即当前工程的操作的初始位置;   
2.general => 限制项目的运行节点:这里后面做多个节点要用到，限制每个工程只在填写的节点部署;    
3.构建触发器 => Build when a change is pushed to GitLab. GitLab webhook URL => Allowed branches:在多节点部署的时候遇到一个问题，就是demo分支更新的时候，应该是对应的demo服务器节点部署，但是master对应了那个线上服务器也部署了，后面才知道这个选项的Allow all branches to trigger this job导致的，换成 Filter branches by name或者Filter branches by regex去做隔离即可  

### gitlab hook配置
  上面我们在工程中源码管理这栏，勾选git并配置好了我们的gitlab上的项目地址，但是这样并不能达到我们的目的，我们还需要在gitlab上配置钩子，先在jenins工程=>构建触发器栏=>Build when a change is pushed to GitLab. GitLab webhook URL=>Secret token去generate一个密码(后面gitlab上的安全令牌就是这个)，我们可以在gitlab上项目页=>设置=>集成里面，把刚才的密码填上，再把推送的url填上(一般在"Build when a change is pushed to GitLab. GitLab webhook URL"这个后面就会有这个url)，再勾选推送事件，去掉开启SSL证书验证(因为我们没有https...)，最后生成钩子即可，生成后推荐test一下能否触发钩子，返回200说明是OK的，这个test并不会真的让jenkins构建，所以安心test。
  
### 构建  
  主要是执行一些你本来拉完代码后的系统命令，比如解压，某些服务的重启(e.g. service nginx restart)，比如依赖包的下载(e.g. pip install -r requirements.txt)等等，让原本每次更新代码到服务器后的一些重复的命令操作都交给jenkins自己自动完成  

### 构建后操作  
　　这一步可以在构建后做一些任务，我现在用到的只有把构建结构通过邮件发送到开发者，告知本次构建成功，不稳定还是失败了，这些都得依靠Email Extension Plugin这个插件...(详细配置待补充)  
  
## 多节点隔离部署  
  官方推荐的是通过ssh去远程操作其他Slave节点服务器完成构建行为，这种方式其他Slave无需像Master主节点一样去安装jenkins，干净不污染Slave节点的环境，但是因为我没有安装默认插件，所以默认没有ssh这种方式，所以得要SSH Slaves plugin这个插件安装好后，然后在系统管理=>管理节点=>新建节点=>启动方式中出现"Launch slave agents via SSH"这个选项
  P.s.配置Slave节点时候，远程工作目录相当于上面配置工程提到的"general => 使用自定义的工作空间"，是你初始的操作目录
  
# 扩展-svn钩子
　　上面只是提到gitlab的钩子配置，这个很方便，这是因为jenkins有专门的对gitlab web hook的插件，而且gitlab本身就写好了有web hook的配置页面，但是svn不同，首先他没有gitlab这么棒的web页面，其次，jenkins也没有什么svn web hook插件，jenkins能够有Subversion Plug-in支持svn源码更新就已经很不错的了，所以如果想做到和Jenkins+Gitlab这样通过钩子的自动部署，svn就得自己写钩子，命名为`post-commit`(post-commit的意思是commit后，而pre-commit意思是commit前，在python的marshmallow校验库中也有类似的pre/post概念)，并放在`$PEPOSITORY/hooks/`下，详细可以参考[Subversion Plugin的jenkins wiki页面](https://wiki.jenkins.io/display/JENKINS/Subversion+Plugin#SubversionPlugin-Postcommithook)以及[svnbook关于post hook编写指南](http://svnbook.red-bean.com/en/1.5/svn.reposadmin.create.html#svn.reposadmin.create.hooks)，这一部分我还没有完成，svn hook这块先留个坑，不定期补上，因为现在jenkins+gitlab用得挺爽的～
