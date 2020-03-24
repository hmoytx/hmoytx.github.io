---
layout:     post
title:    Cobalt Strike 4.0-初体验
subtitle:   域内组策略有关学习
date:       2020-03-24
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - 红队
    - 内网渗透
    
    
---
# Cobalt Strike 4.0-初体验

## 简单加固
修改端口，默认证书。  
在teamserver中，找到server_port，将端口修改为其他端口。  
证书中的dname修改成非默认的即可。也可以用文件夹内的keytool工具来修改。    
![200324_1](/img/200324_tamserver.png)   

## Listener
4.0中做了较大改动，进行了整合。对C2的攻击方式进行了优化，增加了代理选项。支持填写多个域名。    
![200324_2](/img/200324_listener.png)  
这里测试了http/https上线。  
填了域名，再来试试cdn上线，再使用重定向器，将CS服务器更好的隐藏。  
这里cdn用的是cloudflare。  
![200324_4](/img/200324_cdn.png)  
这里测试的端口是http:2052，https:2053。默认是clf能将2053映射到对应解析ip的2053端口。  
将这个端口填入HTTPS Port(C2)。  
HTTPS Port(bind)填入自己cs服务器监听端口，这个任意，不占用开放即可。  
Profile这里默认就行，有需要可以修改，会对流量特征进行修改。  
然后保存即可。  

## Redirectors
重定向器，在这个实验中起到流量中转的作用，将上线域名的解析配置成中转服务器的ip。  
根据监听器类型，使用socat对http(s)流量进行转发。  
```
socat TCP4-LISTEN:2052,fork TCP4:cs ip:39666
```
![200324_3](/img/200324_socat.png)  
这样从外面来看只能看到域名 cdn或者是中转服务器，能有效的隐藏c2。暴露了也能直接舍弃，被反制的可能性也小。  

## 测试  
生成一个exe测试上线。  
可以看到连接的是cdn。  
![200324_5](/img/200324_2052.png)  
可以看到上线。  
![200324_6](/img/200324_beacon.png)  
流量。  
![200324_7](/img/200324_wireshark.png)  
同理https也是一样区别在于端口。  
![200324_8](/img/200324_https.png)  

