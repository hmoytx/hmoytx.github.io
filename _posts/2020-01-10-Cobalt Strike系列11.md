---
layout:     post
title:    Cobalt Strike系列11
subtitle:   External C2上线不出网机器
date:       2020-01-10
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - 渗透测试
    - 红队
    - 内网渗透
---
# Cobalt Strike系列11 
## External C2简介
Cobalt Strike上线问题主要是两种，一种是存在防护软件，一种是内网机器上线。  
对于存在防护软件的主要是paylaod被杀以及阻断连接，前者可以通过免杀绕过，后者可以通过配置文件，对C2流量进行处理来绕过。   
对于内网机器上线（不出网），一种是纯内网隔离，可以通过smb beacon或者link listener连接跳板机来上线，还有一种是端口映射出网（web映射出网），可以通过External C2来上线。   
ExternalC2 是 Cobalt Strike 引入的一种规范（或者框架），黑客可以利用这个功能拓展C2通信渠道，而不局限于默认提供的 HTTP(S)/DNS/SMB 通道。  
基于这个框架可以开发各种组件：  
- Controller：负责连接 Cobalt Strike TeamServer，并且能够使用自定义的 C2 通道与目标主机上的第三方客户端（Client）通信。  
- Client：使用自定义C2通道与第三 Controller 通信，将命令转发至 SMB Beacon。  
- SMB Beacon：在受害者主机上执行的标准 beacon。  
![200110_1](/img/200110_externalC2.png)  
其中Client与Controller之间的通信可以由用户自己定义，在对上面的最后一种web映射出网的情况，我们可以通过一个web脚本实现这个c2的过程。  

## 测试
这里用到hl0rey的代码：  
[https://github.com/hl0rey/Web_ExternalC2_Demo](https://github.com/hl0rey/Web_ExternalC2_Demo)  
先开启teamserver的external c2，项目里面有给cna脚本，直接load就可以开启了。  
开启成功后会有输出：  
```
[+] External C2 Server up on 0.0.0.0:4444
```
然后先运行一次 external c2.py脚本（管道名为test），生成一个payload.bin文件。上传至受害者服务器。  
然后编译RemoteThreadInject工具，上传以后，先运行一个notepad，再运行RemoteThreadInject，运行成功会回显notepad的pid，此时payload已经被注入了这个进程，开启一个管道test（可以用pipelist查看），也就是上面图中的smb beacon。  
![200110_7](/img/200110_loadpayload.png)  
pipelist查看管道。  
![200110_2](/img/200110_testpipe.png)  
然后编译client，放上去运行。  
![200110_3](/img/200110_startclient.png)  
查看管道可以看到多了两个管道，一个read，一个write。  
![200110_4](/img/200110_readwritepipe.png)  
记得把client的php文件放到web目录下。  
然后再运行external c2.py，输入web端client php的url，我是测试的时候直接放进去了。  
![200110_5](/img/200110_py.png)   
然后很快就可以看到主机上线了。  
![200110_6](/img/200110_beacon.png)  


## 注意  
远程注入的工具可能会在win7下不能运行，原作者测试的时候是在win10测试，我也是在win10测试，需要在win7上运行还要修改一下代码。  
这里用到的是命名管道的方式来完成通信，在尝试修改web脚本的时候发现不是那么好写。  
准备尝试将通信的管道换成一个文件，虽然会有文件落地，但是至少web脚本操作文件会好写很多？（先鸽着）  

## 参考
[https://mp.weixin.qq.com/s/q3QZ41qwFcKaIL7qb6q1fQ](https://mp.weixin.qq.com/s/q3QZ41qwFcKaIL7qb6q1fQ)  
[https://blog.xpnsec.com/exploring-cobalt-strikes-externalc2-framework/](https://blog.xpnsec.com/exploring-cobalt-strikes-externalc2-framework/)  

