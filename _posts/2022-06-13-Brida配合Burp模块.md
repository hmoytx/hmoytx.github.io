---
layout:     post
title:    Brida配合Burp模块
subtitle:  Brida配合Burp模块
date:       2022-06-13
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - SRC挖掘
    - 安卓逆向
---
# Brida配合Burp模块

## 前言
前一篇文章介绍了brida的安装及其基本的用法，这篇文章来讲一下brida如何更流畅地和burp联动起来。    

## js编写
拿了一个app做测试，先简单定位到对应的加密函数。不放心可以先用frida测试一下，能否hook到数据，这里不演示了。    
![1](/img/220613_cryptofun.png)    
根据上篇文章的格式，写一个hook的脚本。  
![2](/img/220613_cryptofun.png)    

## 配置brida自定义plugin
到brida中的Custom plugin面板，简单配置下。   
主要是在起作用的范围，以及hook的函数，处理的时机。  
![3](/img/220613_plugin1.png)    
配置完后添加然后启用。  
![4](/img/220613_plugin2.png)       

## repeater以及intruder测试
简单抓了一个涉及加密参数的包。  
![5](/img/220613_package.png)       
把他放到repeater中，修改对应的加密参数为明文。然后发送。打开brida插件的调试界面。  
![6](/img/220613_parambefore.png)      
可以看到原始数据包中的明文，被加密函数处理后替换成密文发送了。  
![7](/img/220613_debug.png)       
然后再在intruder中添加一些测试数据。  
![8](/img/220613_intruder.png)      
开始爆破就行了，日志中都是有的，这里不继续演示了。  


## 总结
对brida的自定义插件功能进行了简单的研究与尝试，极大的简化了我们测试app工作，也更好的把hook和网络数据包联动了起来。  
赶紧上手去挖自己的高危严重吧！   


