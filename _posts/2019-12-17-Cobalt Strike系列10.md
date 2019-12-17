---
layout:     post
title:    Cobalt Strike系列10
subtitle:   通过CDN上线主机
date:       2019-12-17
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - 渗透测试
    - 红队
    - 内网渗透
---
# Cobalt Strike系列10

## 简介
主要介绍域名上线的两种方式，各有各的优点。DNS Beacon在绕过防火墙上还有比较有效的，想要更深入的了解这个过程，可以仔细看看DNS的解析过程。还有一种就是传统的http上线。两者各有各的优缺点，配置起来很相似。  


## 准备工作
需要一个域名，就各种碰瓷大厂，或者通用的域名，这样不容易被识别为恶意域名。为了更好的隐藏自己的teamserver，需要加上CDN。  
这里我用的cloudflare。  
![191217_1](/img/191217_cloudflare.png)  
然后去替换域名那边的解析服务器地址为途中的两个。  
![191217_2](/img/191217_changedns.png)    
dig +trace一下看看CDN是否已经生效。  
![191217_3](/img/191217_dig.png)  

## DNS Beacon
创建一个listener，这里注意cloudflare中能转发的端口是以下的端口：  
```
HTTP：
http://xxx:80 --> http://源服务器:80
http://xxx:8080 --> http://源服务器:8080
http://xxx:8880 --> http://源服务器:8880
http://xxx:2052 --> http://源服务器:2052
http://xxx:2082 --> http://源服务器:2082
http://xxx:2086 --> http://源服务器:2086
http://xxx:2095 --> http://源服务器:2095
```  
所以在填写端口的时候要注意只能填以上端口，而且要确保服务器上也开启这个端口。  
![191217_4](/img/191217_newlistener.png)  
记得添加解析服务器。  
![191217_7](/img/191217_dns.png)  
同时再创建一个本地的http listener，监听端口为前面的端口，这里我用的是2086。  
然后去生成对应的payload到靶机上运行。  
![191217_5](/img/191217_DNSbeacon.png)  
查看下流量。  
![191217_6](/img/191217_wireshark.png)    
不过这里有个坑，我测试的时候有几次会出现一个奇怪的情况，存在一个线程是连接了我teamserver的端口，有时候是运行payload后出现以下后面消失，不知道是为什么。   
而且这个DNS Beacon实际上还是http的，因为一般的DNS Beacon都是很慢的。而我这个跟正常的http beacon没太大区别。     

## HTTP
这个跟上面的过程基本是一样的，就是监听器和生成的时候改成了http，也是能够上线且隐藏自己的teamserver，这里不再多写了。  


## 总结
这里发现的问题还是有点多，主要的就是DNS的Beacon在加了CDN的情况下还是有问题，http这个就很稳定。而且在测试直接解析到teamserver上的DNS Beacon时，发现比较卡顿，但是流量比较隐蔽，http beacon则相反，各有优缺只能这么说。    
