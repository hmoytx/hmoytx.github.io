---
layout:     post
title:    CobaltStrike with CloudFlare
subtitle:   CobaltStrike 利用 CloudFlare 进行隐藏
date:       2020-08-21
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - 渗透测试
    - 红队
    - 工具
    - 内网渗透
    
    
    
---
# CobaltStrike with CloudFlare

## 起因
这几天在某威胁情报平台上发现之前已经失效的CS标签又被重新标记了，也许是因为那啥？强迫症起来了，迁移了下C2重新换了ip，再重新加固下。  

## 端口
证书和端口修改的方式这里不再重新讲，修改对应端口，生成证书。  
这里可以再狠点，墙上设置下规则，或者iptables设置下，只允许跳板代理访问cs服务端的端口。  
![200821_1](/img/200821_firewall.jpg)  
我这里是通过跳板访问，没有设置白名单访问跳板。  
问题也不大，就算标记了也只是跳板，除非CS被爆菊了。  
![200821_2](/img/200821_login.jpg)  


## 重定向
重点放在前端的重定向。  
讲在前面，这里https使用的是2053端口，原因之前讲过，不再重复。   
### 证书
之前使用socat直接转发重定向器流量到CS的监听器，但是不是很稳定，参考了文章换成了nginx做转发。  
结合了一下cloudflare的cdn和证书。  
clf的SSL设置如图所示，由于开启了CDN，所以SSL需要分为两部分，一部分是浏览器到clf CDN服务器的传输加密，一部分是clf CDN服务器到你网站服务器之间的数据传输。  
![200821_3](/img/200821_ssl.jpg)  
之前的架构采用的是Flexible模式，clf CDN与重定向器之间流量没有进行加密。   
这里插入个题外话，https上线的证书问题，这个证书跟前面在teamserver中修改的证书是两码事，这个证书要修改需要配置C2 profile。  
回来，这里改成Full模式，然后建议使用clf的Origin CA，证书免费，在SSL/TLS的Origin Server菜单下，选择”Create Certificate“，创建证书下载。 
![200821_4](/img/200821_pem.jpg)  
然后在nginx中配置(假定证书目录如下):  
```
ssl_certificate    /usr/local/nginx/conf/ssl/cert.pem;
ssl_certificate_key    /usr/local/nginx/conf/ssl/key.pem;
```
重载以后就是全加密了。  
### 限制访问
开启clf的Authenticated Origin Pulls功能。这样客户端在访问你网站服务器的内容时，需要提交客户端证书验证，否则不允许访问，防止服务器数据被非clf CDN服务器访问。   
1.下载证书:[https://support.cloudflare.com/hc/zh-cn/article_attachments/360044928032/origin-pull-ca.pem ](https://support.cloudflare.com/hc/zh-cn/article_attachments/360044928032/origin-pull-ca.pem )，保存为crt格式。在SSL/TLS的Origin Server菜单下开启Authenticated Origin Pulls功能。  
![200821_5](/img/200821_originpull.jpg)  
2.在Nginx中配置:  
```
ssl_client_certificate /usr/local/nginx/conf/ssl/origin-pull-ca.crt;
ssl_verify_client on;
```
3.重载nginx，直接尝试ip访问，可以发现为400，提示No required SSL certificate was sent。  
![200821_6](/img/200821_400.jpg)  
### ip策略
当然可以进一步限制，可以考虑在Nginx上，屏蔽一切非clf CDN服务器的访问，我们可以在[https://www.cloudflare.com/ips/](https://www.cloudflare.com/ips/)看到clf使用的IP，可以新建一个conf文件，allow上面的ip，然后在nginx.conf中包含该配置文件即可。  
![200821_7](/img/200821_iprange.jpg)  


## 测试
域名访问下页面，没有问题。  
![200821_8](/img/200821_page.jpg)  
测试下转发是否启用，访问C2 profile中的URI。  
![200821_9](/img/200821_js.jpg)  
生成一个exe测试下。  
![200821_10](/img/200821_cscs.jpg)  


## 注意
配置的时候记得 X-Forwarded-For 头配置，不然存在上线ip不正确的问题，可以根据参考文章中设置进行配置。  

## 参考文章  
[https://mp.weixin.qq.com/s/OK0m9lln5-XjHHkWLwMxHg](https://mp.weixin.qq.com/s/OK0m9lln5-XjHHkWLwMxHg)  
[https://jayshao.com/cloudflare-nginx-ssl/](https://jayshao.com/cloudflare-nginx-ssl/)  



