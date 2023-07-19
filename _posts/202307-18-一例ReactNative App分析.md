---
layout:     post
title:      一例ReactNative App分析
subtitle:   一例ReactNative App分析
date:       2023-07-18
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - SRC挖掘
---
#  一例ReactNative App分析

## 前言
最近工作中碰到一个app，发现传参是加密的，需要解密，按照之前常规的做法发现没有hook到对应的加密逻辑，最后才发现是基于ReactNative开发的，借此来浅浅地学习下ReactNative的逆向。  
介绍下ReactNative，RN是Facebook开源的一个跨平台移动应用开发框架，是Facebook开源的JS框架 React 在原生移动应用平台的衍生产物，支持iOS和安卓两大平台。RN使用Javascript语言，类似于HTML的JSX，以及CSS来开发移动应用。  

## 分析App
首先分析它的登录协议，登录界面长这样。  
![1](/img/230718_app.png)     
随便输入手机发送验证码，抓个包发现是加密的。  
![2](/img/230718_burp.png)     
这里就要开始分析它的算法了。  

## Java层分析
直接frida-dexdump一下，拖进工具看看java层代码，直接搜接口找了对应的接口。  
![3](/img/230718_interface.png)      
找到对应的调用方法。  
![4](/img/2307018_sendcode.png)     
我一开始直接对应的方法，没有输出，然后hook HashMap的put方法，发现也没有输出，这里卡了很久，后面记起来他UA是ok3，又hook了一下ok3,终于有输出了，打印了一下堆栈：  
![5](/img/230718_hook.png)  
发现调用栈压根没有走过我之前测试hook的那些方法，打印出了一堆react相关的调用，这里开始意识到这个app是基于ReactNative开发的。  
验证下猜想，解压查看了下对应的lib目录，确实是基于ReactNative开发的。  
![6](/img/230718_reactnative.png)  
确定了那就简单了。  


## 分析JS代码
解压apk,在assets目录下，可以看到index.android.bundle文件。  
这里面包含了应用主要的业务逻辑代码，我改个后缀直接放编辑器里格式化查看。  
![7](/img/230718_js.png)   
里面基本主要业务的接口调用代码都在了。   
![8](/img/230718_smssend.png)   
继续全局搜加解密代码，发现了一处国密调用。  
![9](/img/230718_decryptjs.png)   
这个应用并不复杂，基本都是靠js去实现对应的加解密，里面写的比较清楚，直接把key iv拿过来进行加解密测试，发现已经能看到明文了。  
![10](/img/230718_decrypt.png)   




## 总结
总体来看，安卓ReactNative应用分析没有那么复杂，主要的业务逻辑（index.android.bundle文件）能很容易进行分析，虽然可以混淆，但是还是能有一定的可读性。二来应用运行时不校验JS代码，可以通过hook替换JS文件实现代码注入，具备了可调试能力。  


