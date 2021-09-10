---
published: true
category: Others
title: SOLID设计原则和23种设计模式
author: smona
date: '2019-05-14 13:01:47'
layout: post
---

# SOLID设计原则 
(SRP,OCP,LSP+LoD,ISP,DIP)
SRP:一个类只承担一个职责  
OCP:对软件实体的改动，最好用扩展而非修改的方式  
LSP:在继承类是，务必重写(override)父类中所有的方法，尤其需要注意父类的protected方法(它们往往是让你重写的)，子类尽量不要暴露自己的public方法供外界调用[子类是父类超集]  
LoD:低耦合,高内聚  
ISP:不要对外暴露没有实际意义的接口，能少则少  
DIP:面向接口编程，提取出事务的本质和共性(抽象)  

23中设计模式也是建立与SOLID设计原则上的  

实际的设计/开发过程应该是:  
1.先做领域模型i(DDD领域驱动设计)，理解业务上要做什么事情，期望拿到什么价值(这个阶段切忌思考
如何抽象接口/类，如何设计类);  
2.考虑设计层面有哪些需要抽象，有哪些依赖，具体方法接口如何设计;  
3.最后才考虑用哪种模式，那个包来实现;  