---
layout:     post
title:    Cobalt Strike系列 1
subtitle:   Cobalt Strike 部署
date:       2019-08-08
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - 渗透测试
    - Cobalt Strike
    - 红队
---

# Cobalt Strike系列 1 部署


>Cobaltstrike简介
Cobalt Strike是一款美国Red Team开发的渗透测试神器，简称CS。菜鸡我自己正在学习，因此写一下自己学习的历程与在此过程中碰到的问题，希望给各位一点帮助。
最近这个工具很火，其拥有多种协议主机上线方式，集成了提权，凭据导出，端口转发，socket代理，office攻击，文件捆绑，钓鱼等功能。同时，Cobalt Strike还可以调用Mimikatz等其他知名工具。一句话，牛逼，好用，就完事了。

# 部署过程

- 服务器选购
- 服务器环境配置
- 运行服务
- 客户端连接

## 服务器选购
这里不多介绍，如果是专门的搞这个的，一般不用自己去选购，组织上会照顾你的→_→
自己使用的话建议用搬瓦工，vultr这种便宜的为主，或者白嫖的VPS也行。
## 服务器环境配置
Cobalt Strike现在最新的版本应该是3.14，不过据说不稳定，我倒是没觉得，反过来自带生成payload倒能过好多杀软，建议学习的话还是用3.12或3.13。

操作系统的话我用的是Centos7，这个看个人喜好吧。

因为工具是java开发的，所以需要配置JAVA环境，版本的话建议是1.8。
```shell
yum update //更新
yum install -y java-1.8.0-openjdk* //安装，默认路径为/usr/lib/jvm
---------------------------------------------------------------
vi /etc/profile //设置jdk环境变量，在文件中添加下面的内容，保存退出
    #set java environment
    JAVA_HOME=/usr/lib/jvm/jre-1.8.0-openjdk.x86_64
    PATH=$PATH:$JAVA_HOME/bin
    CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
    export JAVA_HOME CLASSPATH PATH
---------------------------------------------------------------
source /etc/profile  //使profile生效
---------------------------------------------------------------
echo $JAVA_HOME  //检查是否生效，看是否输出
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.181-3.b13.el7_5.x86_64
---------------------------------------------------------------
java -version  //查看版本是否正确
openjdk version "1.8.0_222"
OpenJDK Runtime Environment (build 1.8.0_222-b10)
OpenJDK 64-Bit Server VM (build 25.222-b10, mixed mode)
```

## 运行服务
将有关文件上传到服务器，然后启动teamserver。
```shell
chmod +x teamserver
./teamserver host  password  //启动服务
nohup ./teamserver host  password  &  //或者用nohup挂起运行
```
![启动服务](/img/star_service.png)

## 客户端连接
启动客户端。
```shell
cobalt strke.bat
```
![客户端连接](/img/client.png)
登陆后的界面如图所示
![客户端界面](/img/client_ui.png)
