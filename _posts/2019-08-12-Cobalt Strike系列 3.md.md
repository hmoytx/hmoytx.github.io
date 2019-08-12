---
layout:     post
title:    Cobalt Strike系列 3
subtitle:   Cobalt Strike Beacon
date:       2019-08-12
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - 渗透测试
    - Cobalt Strike
    - 红队
---

# Cobalt Strike系列 3 beacon基本操作




# 简介

- 基本命令
- 常用功能
- 浏览器劫持
- 代理有关

## 基本命令
如图所示，常用的一些命令，基本是一些文件操作以及beacon自身的一些命令。
![基本命令](/img/command.png)

## 常用功能
### 键盘记录模块
命令行中输入keylogger，然后在视图中查看键盘记录的内容。
![keylogger](/img/keylogger.png)
如果要终止这个任务可以用上面的jobkill命令。
![jobkill](/img/jobkill.png)
### 截屏模块
命令行中输入screenshot或者右键beacon选择目标里面有屏幕截图，然后在视图中查看截屏的内容。可能会导致靶机的卡顿。
![screenshoot](/img/screenshot.png)
### 加载powershell
使用命令powershell-import加载powershell脚本，注意给上绝对路径。导入成功后使用命令powershell xxx执行命令
![powershell-import](/img/powershellimport.png)
也可以用powerpick执行一些基本的信息收集。
![powerpick](/img/powerpick.png)
### 加载mimikatz 转储hash
直接用命令也可以，一般直接右键beacon执行Run mimikatz。
![mimikatz](/img/mimikatz.png)
转储hash也类似，转储后可以在视图中的凭据信息查看。
![hashdump](/img/hashdump.png)
### 还有很多这里就不一一介绍了，可以自己去看手册。
## 浏览器劫持
这是一个比较牛逼的功能，但是也很不稳定，我自己在测试的时候因为网络问题，证书问题是没有成功的。而且有限制。
主要是利用web代理，劫持转发目标浏览器进程数据到指定的端口上，然后我们再从该端口访问，相当于拿着目标浏览器中的数据进行访问。会话cookie劫持，登录他人帐号都是可以的
但是暂时只是在IE上好使，也不稳定，而且会导致浏览器卡顿。
使用时要先确定开启了IE进程。然后ps查看pid。执行browserpivot PID x86
![browser](/img/broser.png)
配置好浏览器代理后访问网站即可。
![browser1](/img/broser1.png)
## 代理有关
端口转发，内置的可以用，但是实际中会用别的类似ew之类的转发工具。这里简单讲一下。rportfwd 389 192.168.1.181 3389，转发到服务器的389端口。
停止的话 就是rportfwd stop 389 。
图没有，不配了，这是很基本的用法，后面详细将内网的时候会用其他方式来做讲解。
算了随便盗个图吧。
![rdpfwd](/img/rdpfwd.png)