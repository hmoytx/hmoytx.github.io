---
layout:     post
title:    Cobalt Strike系列 2
subtitle:   Cobalt Strike Listener与Payload
date:       2019-08-09
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - 渗透测试
    - Cobalt Strike
    - 红队
---

# Cobalt Strike系列 2 listener与payload




# 简介

- Listener介绍
- 创建Listener
- 生成Payload
- Beacon介绍

## LIstener介绍
在3.14的版本中一共提供了9种监听器。其中bind_tcp是3.13之后新增的。
> windows/beacon_dns/reverse_dns_txt 
  windows/beacon_dns/reverse_http
  windows/beacon_http/reverse_http 
  windows/beacon_https/reverse_https 
  windows/beacon_smb/bind_pipe 
  windows/beacon_tcp/bind_tcp 
  windows/foreign/reverse_http 
  windows/foreign/reverse_https 
  windows/foreign/reverse_tcp

其中：
- beacon是CS自带的监听器。
- foreign是外部监听器，常用于和msf进行联动。

## 创建Listener
这里用http监听器做演示。
Cobalt Strike -> Listener ,然后下方 add，选择payload为windows/beacondns/reversehttp，端口记得开。
![创建Listener](/img/Listener1.png)
![选择Listener](/img/Listener.png)

## 生成payload
Attack -> 生成后门，这里先不用其他的，用windows Executable。
![选择payload](/img/payload.png)
生成exe。当然是不免杀的，直接就死的那种。
![生成payload](/img/payload1.png)
运行后，会生成一个beacon，在客户端可以看到有主机上线。
![上线](/img/back.png)
## Beacon介绍
反向回来的Beacon，类似于msf中的meterpreter，可以用来执行各种命令，后面会细讲。可以用help命令查看用法。注意：在上面的视图中，上线主机上有爪子的，是高权限主机。默认返回的Beacon的sleep time（也就是心跳时间）是60s，即每隔60s从客户端获取一次命令，实际中会设置更久，这样会更隐蔽，自己做测试的时候可以设置成短一点的5s，可以用命令sleep 10，也可以右键选择设置。
![sleep](/img/sleep.png)
要执行windos命令需要加上shell。如 shell whoami
![whoami](/img/whoami.png)