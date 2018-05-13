---
published: false
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
  
##　插件  
  安装好以后就可以访问服务器的8080端口了，比如`http://you_servier_ip:8080`，开始初始化配置，一开始要输入一个管理员密码的，你按照提示到服务器上找就好了，然后还要选择默认插件安装，因为简洁哲学思想影响，默认的插件我插件都没有安装，因为它默认的安装的插件其实对于Jenkins + Gitlab完成自动部署任务来说，有一些不必要的～  
  那么我现在就告诉你，如果想达到" 能够git push到gitlab后，gitlab主动推送到Jenkins部署 "所必须的插件:　　

```
Requred:  
1.Git plugin
2.GitLab Plugin
3.Gitlab Hook Plugin
4.Email Extension Plugin(为了能够发送邮件)
5.SSH Slaves plugin(为了能够管理多个节点)

Optional:
1.Subversion Plug-in(如果托管在svn)
2.TODO Plugin(如果托管在gitee)
3.GitHub plugin(如果托管在github)
```

P.s.因为公司的服务器没有直接访问国外节点的能力，而我自己多次尝试更改为国内的比如清华开源镜像和搜狐开源镜像无果后(可以到[这里](http://mirrors.jenkins-ci.org/status.html)看一些jenkins插件的开源镜像站点)，果断转向自己上[官方插件站](https://plugins.jenkins.io/)去搜索相应的hpi，通过Archive跳转，在找到自己想要的版本来下载，


  
