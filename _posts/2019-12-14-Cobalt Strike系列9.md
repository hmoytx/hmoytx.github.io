---
layout:     post
title:    Cobalt Strike系列9
subtitle:   不出网机器上线-1
date:       2019-12-14
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - 渗透测试
    - 红队
    - 内网渗透
---

# Cobalt Strike系列9
## 简介
在内网中会经常碰到目标机器不出网的情况，这时候介绍两种常见的方式进行上线——SMB Beacon与link listener。前者在之前的两篇文章中都有用到，前者都是封装成smb协议比较隐蔽，使用起来更方便，但是必须要用445端口进行通信，而后者是在原有的beacon基础上创建一个监听，定制新的可执行文件，目标回连设置为了内网边界机，要考虑免杀问题。  
## SMB Beacon  
官网对SMB Beacon的介绍是，SMB Beacon是使用命名管道通过父级Beacon进行通讯，当两个Beacons链接后，子Beacon从父Beacon获取到任务并发送。因为链接的Beacons使用Windows命名管道进行通信，此流量封装在SMB协议中，所以SMB Beacon相对隐蔽。  
### 测试
用之前的图了，不再重新做一遍了。  
首先创建一个SMB的Listener。  
![191214_1](/img/191214_smblistener.png)  
然后选择对应的主机派生一个会话。  
![191214_2](/img/191208_spawnsmb.png)  
运行成功后,可以看到 ∞∞ 这个字符，这就是派生的SMB Beacon当前是连接状态你可以主Beacon上用link host链接它 或者unlink host断开它。当用命令断开时 链接符号上面出现disconnected.     
![191214_3](/img/191214_smbbeacon.png)  
然后再通过不停导出hash，登录扩大内网权限。  
![191214_4](/img/191208_DCbeacon.png)  
要注意的是，这些派生出的beacon都是基于命名管道，建立在那个出网的beacon上，一旦那个beacon掉线了，其他的beacon也是没有了的。  
### Link Listener
这个比较好理解，就是在已有的beacon上创建监听，用来作为跳板进行内网穿透。  
首先在已有的beacon上创建一个listener。  
![191214_5](/img/191214_listener.png)  
![191214_6](/img/191214_smb1.png)  
然后生成一个payload，注意要用windows executable(s)生成。  
![191214_7](/img/191214_generate.png)  
然后先上传到原有的beacon，再通过这台上传到内网的不出网的机器上。  
![191214_8](/img/191214_upload.png)  
为了方便，我直接在那台机器上运行了，不用wmic之类的工具远程执行了。  
![191214_9](/img/191214_runbeacon.png)  
可以看到直接上线了，跟smb的形式有点区别，箭头是反向的，这也能让我们更好的理解上线的形式。   
![191214_10](/img/191214_DC.png)  

## 总结
两种方式各有各的特点，cs的功能很强大，这里我只是介绍了内网机器不出网的情况，实际中还有很多的情况下，拿到shell，但是机器也是不出网的，只是端口通过防火墙映射到公网，这里就需要用到CS的External C2了。  