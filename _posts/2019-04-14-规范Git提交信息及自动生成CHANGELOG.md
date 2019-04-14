---
published: true
category: OS
title: 规范Git提交信息及自动生成CHANGELOG
author: smona
date: '2019-04-14 12:52:47'
layout: post
---

<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [前言](#前言)
	* [commitizen](#commitizen)
	* [commitlint](#commitlint)
	* [standard-version](#standard-version)
	* [简述语义化版本](#简述语义化版本)
* [参考](#参考)

<!-- /code_chunk_output -->

# 前言

大学时候就用git管理自己的代码，无论是分支还是工作流都会有规范，唯独到git-commit的message就随心所欲了，没有找到章法。还有以前一直都是自己手写`CHANGELOG.md`，每次都是自己一点点地编写，可是当开发完到要写`CHANGELOG.md`的时候就忘记之前自己都做了什么改动了，还得一点点翻看之前的commit记录，然而自己的提交记录简直惨不忍睹，都看不出自己修改了什么东西。  

规范git提交信息的好处:

1. 自动生成CHANGELOGs;
2. 自动生成语义化的版本号;
3. 更容易和你的团队，和公众，以及利益相关者沟通项目的变化;
4. 触发构建和发布的进程;
5. 更容易让其他人为项目贡献代码，因为规范的提交允许他们能知道更加结构化的提交历史;

Ref: [Why Use Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0-beta.2/#why-use-conventional-commits)  

最近友人推荐了一套规范化提交的工具链给我: commitizen => commitlint =>  standard-version，尝试了一下，还不错推荐给大家。  

## commitizen  

[commitizen](https://github.com/commitizen/cz-cli)是git提交信息框架，相当于生成一个模板，你照着填写提交信息就可以了，`commitizen`可以指定不同的Adapter以满足不同的提交规范。  

[cz-conventional-changelog](https://github.com/commitizen/cz-conventional-changelog)是`commitizen`指定的一个Adapter，符合AngularJS团队提交规范。  

如果不想用AngularJS规范，想自定义规范，可以用`cz-customizable`Adapter来自定义规范，把`cz-conventional-changelog`换成[cz-customizable](https://github.com/leonardoanalista/cz-customizable)，本文就不展开了。  

个人推荐npm全局安装，反正每个git管理的项目提交的时候都受用  

```shell
npm install -g commitizen cz-conventional-changelog
echo '{ "path": "cz-conventional-changelog" }' > ~/.czrc
```

## commitlint  

[commitlint](https://github.com/conventional-changelog/commitlint)可以帮助我们校验commit messages是否符合规范，不符合的直接拒绝提交。  

一般是项目捆绑以防止项目开发成员出现不规范的提交信息，所以这个`commitlint`一般都会是团队的要求，在开发者本地校验一次，以及在CI集成部署的时候校验一次会是一种比较好的做法。  

而个人而言，如果你习惯每次提交都走`commitizen`，就没有必要安装这个`commitlint`，因为你的提交肯定是合乎规范的。  

开发者本地在安装`commitlint`之后还可以npm装个[husky](https://github.com/typicode/husky)绑定到git-hook上，每次git-commit就触发git-hook执行`commitlint`检查，就不用每次都手动执行`commitlint`，只管commit就好了  

安装和使用`commitlint`和`husky`本文就不多阐述了，根据下面的Ref官方教程实践即可。  

Ref: [commitlint guides local setup](https://conventional-changelog.github.io/commitlint/#/guides-local-setup)  

## standard-version  

[standard-version](https://github.com/conventional-changelog/standard-version)能为有了上面规范提交的项目自动生成CHANGELOG和语义化的版本号  

和commitizen一样推荐全局安装，毕竟全部项目都是走`standard-version`  

```shell
npm install -g standard-version

standard-version # 生成或者更新CHANGELOG.md并打上符合语义化版本号的git-tag
git push --follow-tags origin master && npm publish # 或者docker push, gem push, 等等
```

## 简述语义化版本  

版本号格式：X.Y.Z (主版本号.次版本号.修订号):  

1. 进行不向下兼容的修改时，递增主版本号;  
2. API 保持向下兼容的新增及修改时，递增次版本号;  
3. 修复问题但不影响API 时，递增修订号;  

Ref: [语义化版本 2.0.0](https://semver.org/lang/zh-CN/)  

# 参考  

[优雅的提交你的 Git Commit Message](https://juejin.im/post/5afc5242f265da0b7f44bee4)  
[Angular 团队的规范](https://github.com/angular/angular.js/blob/master/DEVELOPERS.md#-git-commit-guidelines)  
[Conventional Commits specification](https://www.conventionalcommits.org/en/v1.0.0-beta.3/)  