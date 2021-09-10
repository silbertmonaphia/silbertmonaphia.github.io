---
published: true
category: OS
title: 编辑器与IDE
author: smona
date: '2019-05-30 13:01:47'
layout: post
---

配置好编辑器能让你的工作效率翻倍

我常用的编辑器除了服务器上面无可奈何的vim以外(说无可奈何是过了，vim其实是一个挺好的编辑器，个人而言不太喜欢all-in-one的emacs)，就是SublimeText了，现在多了一个新宠vscode，不过vscode还没有上手，现在只是用来写这篇文章和维护这个静态博客。我不太喜欢用比如PyCharm，Eclips这些IDE，可能是刚开始用windows学习编程的时候被这些IDE的笨重搞怕了，真正让我喜欢上写代码的是SublimeText2，当然现在已经升级到了SublimeText3，编辑器加上插件能够追上甚至超过那些IDE了

# Vim  

# Sublime  
Sublime真是除了notepad++以外的小白福星

1.Anaconda
python在sublime上编写代码媲美IDE全靠Anaconda这个插件~  
python_interpreter设置对应的python版本就会按照那个版本的python校验语法
如果你python_interpreter没有设置为3.5以上的版本，那么对于python3.5新增的type hint写法，anaconda会说你这种type hint写法是语法错误的，这个时候只要改python_interpreter为python3.5以上的版本就可以了，不需要另外装SublimeLinter和SublimeLinter-contrib-mypy  

写了type hint以后我觉得最好的好处是可以不用再在docstring里面编写参数类型和参数返回类型了  

"anaconda_linting_behaviour": "load-save"
- "always" - Linting works always even while you are writing (in the background)
- "load-save" - Linting works in file load and save only
- "save-only" - Linting works in file save only

"anaconda_linter_phantoms":true
anaconda will show errors inline

"mypy": false

# VSCode  
这几年非常流行的编辑器，微软开发的，风头甚至盖过了github开发的Atom