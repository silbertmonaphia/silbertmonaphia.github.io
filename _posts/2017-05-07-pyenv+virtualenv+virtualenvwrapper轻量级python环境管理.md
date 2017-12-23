---
title:  "pyenv+virtualenv+virtualenvwrapper轻量级python环境管理"
author: smona
published: True
date: 2017-05-07 20:54:15
layout: post
---

前言
--

今晚帮一个童鞋解决需求，无意中把最近用到virtualenv和virtualenvwrapper串着用了起来，又知道了原来还有pyenv这么一个东西，感觉这样的python环境控制有必要再来一写，因为对比前面写到的一篇[Docker+Git效率工作](http://blog.csdn.net/qq_29245097/article/details/52996911)中提到的docker，我现在感觉就python开发而言，虽然docker能牢牢管住整个软件以来环境，但是后来用过一段时间，我倒是觉得每次都要进去docker做操作测试还是显得太笨重了(累)，现在的这么一套pyenv+virtualenv+virtualenvwrapper管理python开发相关的一套依赖则显得轻量得多。

pyenv
--
  我是今晚才知道这个东西的，今晚我的那个同学的需求是，希望能在ubuntu14.04 用pip安装的时候不会显示下面这个警告，他觉得很脏(嗯，我也觉得)：
  

> /usr/lib/python2.7/site-packages/urllib3/util/ssl_.py:318: SNIMissingWarning: An HTTPS request has been made, but the SNI (Subject Name Indication) extension to TLS is not available on this platform. This may cause the server to present an incorrect TLS certificate, which can cause validation failures. You can upgrade to a newer version of Python to solve this. For more information, see https://urllib3.readthedocs.io/en/latest/security.html#snimissingwarning.

  其实警告并不影响python和python模块的使用，但是每次都出现这个对于我们这些有洁癖的人来说简直不能忍。由于是ubuntu14.04，我apt-get update以后再apt-get upgrade python以后发现，14.04的源里面(清华大学的镜像源)里面最新的python版本也才2.7.6，而根据warning里面提示的"You can upgrade to a newer version of Python to solve this."得知，它推荐的是升级现在的python版本来解决这个warning问题，那么既然没有办法通过apt-get来更新我们的python，那么我们只能自己另外装一个，当然我们可以下载tar包来解压编译安装，不过这个略显麻烦，我们有更加好的方案——pyenv，关于它是干什么的，我觉得它的GitHub官方文档就写的很清楚:"pyenv lets you easily switch between multiple versions of Python."，所以对付ubuntu14.04里面python只有2.7.6老版本来说，要获取新版本的python，用pyenv简直不要太好- -。  

```
1.pyenv install --list
查看可安装的列表
2.pyenv version
查看当前版本
3.pyenv versions
查看已经安装的所有版本
4.pyenv install 2.7.13
下载安装python 2.7.13版本
5.pyenv rehash
每次新下载安装一个版本都要rehash一下
6.pyenv global 2.7.13
设定全局python版本为2.7.13
7.pyenv local 2.7.13
设定局部python版本为2.7.13
8.pyenv uninstall 2.7.13
卸载2.7.13版本的python
更多指令看官方文档：https://github.com/pyenv/pyenv/blob/master/COMMANDS.md
```
P.s.记录一个和python的解释器相关的东西：
export PYTHONPATH=/usr/local/bin/可以往到python解释器sys.path最前面添加'/usr/local/bin/'的路径。
这个动作影响的是python解释器import的时候，当import一个模块的时候，python解释器会逐一从sys.path提供的路径下寻找是否有相应的模块，所以如果用pyenv设定了当前版本不是系统版本而又想要用系统安装的模块(比如ubuntu中apt安装python-requests)，可以通过指定PYTHONPATH系统环境变量的办法，让python解释器能自己找到系统安装的python模块。

virtualenv+virtualenvwrapper
--
  其实以前我用过virtualenv，但是觉得它启动的操作太麻烦了，要先进入用virtualenv虚拟出来的目录下的bin中，然后source activate才能启动，就觉得它不行。后面发现了docker，觉得docker比virtualenv要好很多，而且virtualenv只能管python的包，其他的包啊，模块啊，库啊还有一些软件依赖，它就管不了，而docker都能管，就拼命说docker好。
  直到前些日子知道有virtualenvwrapper这么一个工具以后，才开始觉得其实在单纯的只有python的开发中，对于python模块依赖管理来说，virtualenv+virtualenvwrapper就够了，没有必要上docker(前提是真的只有python代码和python模块)，virtualenv管理环境，而virtualenvwrapper简化原来的操作(其实我觉得就是拿来包装virtualenv的)，使得原来要先deactivate，再切换到目标的虚拟环境目录下的bin/目录再source activate的操作，现在只用一句work on就搞定在不同python环境之间的切换了。

```
#在安装virtualenvwrapper之前请确保安装了virtualenv，因为virtualenvwrapper是依赖于virtualenv的。
1.mkvirtualenv -p python3 --no-site-packages blabla
#指定python版本为系统的python3并且不带入任何系统模块
2.workon blabla
#直接切换到blabla虚拟环境，如果你现在是比如test环境要切换到blabla环境，可以不用deactivate，而直接workon blabla即可
3.deactivate
#退出blabla虚拟环境到系统环境
4.rmvirtualenv blabla
#删除 blabla虚拟环境
更多命令：http://pythonguidecn.readthedocs.io/zh/latest/dev/virtualenvs.html
```  

