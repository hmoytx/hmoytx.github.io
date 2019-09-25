---
layout:     post
title:    利用ningx+cloudflare突破GFW 
subtitle:   科学上网
date:       2019-09-25。
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - 上网
---
# 利用ningx+cloudflare突破GFW  


## 前言  
因为特殊情况，最近被ban的力度又大了，自己的好几个机器被ban，机场也不稳，发现一个有意思的现象，能ping通主机，但是没办法kx上网，用tcping测了下端口，emmmm，是被阻断了，这玩意从全协议封锁改成仅tcp阻断了，悲伤。   

## 简单解释  
当你的外面的机器 IP 被TCP封锁(阻断)后，你依然可以正常的向外面的代理发送数据（客户端连接服务端），但是外面在向你返回数据的时候，还是要经过一堵墙的，而墙发现发送者IP(代理服务器)在黑名单中，于是就会阻断、拦截，这样你的客户端就收不到来自服务端的返回数据了（酸酸乳上表现为：超时或空连）。  
而现在的代理工具基本都是走tcp，要三次握手，在这个握手过程中被拦了，就莫得办法了。  

## 解决方案  
- 有些能ping通的机器能不能用icmp协议来做代理？  
可以啊，但是速度捉鸡，你用个锤子？  
- 能不能用UDP呢？  
也可以，但是实际中，运营商会对UDP QOS限速，你又得通过伪装UDP成TCP才能绕过，又是死循环。你上你m的网。  
- 目前简单的解决方法：  
1.发现不行了，不能用了，但是能ping通，试试换个端口，重新开一次服务端，又几率能解决这个问题，或者换个代理工具（wireguard啥的）。  
2.再不济就等吧，推测是会定期解封一些ip的，不然v4地址哪里够哦。  
3.利用WS+CDN的模式绕过，但是免费的CDN在国内的速度不是很理想，可以当成一种备用的方式。  
这里我着重讲下第三种，呼应下标题。 
## 准备工作
1.整个域名，namesilo上买一个便宜的，或者用freenom申请免费的。  
2.注册cloudflare
3.你的微皮爱死      

## 部署
### 环境部署
在你的机器上直接装个宝塔就完事了，把nginx装了，数据库，ftp，php那些统统不要。  
```
Centos安装命令
yum install -y wget && wget -O install.sh http://download.bt.cn/install/install.sh && sh install.sh

Debian安装命令
wget -O install.sh http://download.bt.cn/install/install-ubuntu.sh && bash install.sh
```
然后根据你的域名创建个站点  
![bt_website](/img/bt_website.png)  
点击设置-> SSL -> Let's Encrypt 生成证书，稍等一会。  
![bt_ssl](/img/bt_ssl.png)  
### 配置域名以及解析  
我这里用的namesilo的，现在注册好的cloudflare上进行配置。点击Add Site。    
![cloud_add](/img/cloud_add.png)  
然后选择免费那个。  
![cloud_0](/img/cloud_0.png)  
然后在DNS中删除默认的，添加解析记录A。并且把域名那边的默认3个DNS解析服务器删除且换成红框中的服务器。  
![cloud_res](/img/cloud_res.png)    
![namesilo_cgdns](/img/namesilo_changedns.png)  
然后修改下域名解析。  
![namesilo_cgip](/img/namesilo_changeip.png)  
然后访问下如果可以就OK了会要点时间。  
为了部署测试方便建议关了cloudflareDNS界面中的那朵云，直接解析。  
### 部署V2ray
一键安装。  
```
bash <(curl -L -s https://install.direct/go.sh)
``` 
然后修改配置文件
```
vi /etc/v2ray/config.json
```
改成
```
{
    "log": {
        "access": "/var/log/v2ray/access.log",
        "error": "/var/log/v2ray/error.log",
        "loglevel": "warning"
    },
    "inbound": {
        "port": 4333, #你的端口
        "protocol": "vmess",
        "settings": {
            "udp": true,
            "clients": [
                {
                    "id": "24d1b51a-1fce-******", #你的id
                    "level": 1,
                    "alterId": 64
                }
            ]
        },
        "streamSettings": {
            "network": "ws", #用websocket
          "wsSettings": {
          "path": "/cobra" #自定义
        }
        }
    },
    "outbound": {
        "protocol": "freedom",
        "settings": {}
    },
    "outboundDetour": [
        {
            "protocol": "blackhole",
            "settings": {},
            "tag": "blocked"
        }
    ],
    "routing": {
        "strategy": "rules",
        "settings": {
            "rules": [
                {
                    "type": "field",
                    "ip": [
                        "0.0.0.0/8",
                        "10.0.0.0/8",
                        "100.64.0.0/10",
                        "127.0.0.0/8",
                        "169.254.0.0/16",
                        "172.16.0.0/12",
                        "192.0.0.0/24",
                        "192.0.2.0/24",
                        "192.168.0.0/16",
                        "198.18.0.0/15",
                        "198.51.100.0/24",
                        "203.0.113.0/24",
                        "::1/128",
                        "fc00::/7",
                        "fe80::/10"
                    ],
                    "outboundTag": "blocked"
                }
            ]
        }
    }
}
```

### 修改nginx配置文件
用宝塔修改就行。  
在server中添加内容。  
```
location /cobra {
          proxy_redirect off;
          proxy_http_version 1.1;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection "upgrade";
          proxy_set_header Host $http_host;
          proxy_intercept_errors on;
          if ($http_upgrade = "websocket" ){
             proxy_pass http://127.0.0.1:10000; #你的端口
          }
        }
```
然后重启ngixn。   
###  客户端v2ray配置
windows：（盗个图）  
![v2ray_win](/img/v2r_win.png)  
mac：
![v2ray_mac1](/img/v2r_mac1.png)  
![v2ray_mac2](/img/v2r_mac2.png)  
记得把TLS钩上。  
移动端：
安卓的用BifrostV，配置简单的。  
