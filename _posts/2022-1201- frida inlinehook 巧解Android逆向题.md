---
layout:     post
title:     frida inlinehook 巧解Android逆向题
subtitle:   frida inlinehook 巧解Android逆向题
date:       2022-12-01
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - CTF
    - 安卓逆向
---
#  frida inlinehook 巧解Android逆向题

## 前言
最近又又又是老朋友发来一道Android逆向题，比赛时没做出来，后来参考了别人的思路，还是用frida解出了题目，学到一些思路记录下。  

## Java层静态分析
直接拖进jadx，MainActivity内容如下，代码不多，基本流程就是，获取输入字符，通过一个native层的checkflag判断输入是否正确：       
![1](/img/221201_javacode.png)   


## Native层静态分析
直接解压把libnative-lib.so拖入ida。    
发现是静态注册的，直接能在导出函数里找到，处理伪码后如下，直接是ollvm当时就放弃了：   
![2](/img/221201_ollvm.png)   
尝试用脚本模拟执行so，反混淆一下，emmmmm区别不大，但是好像勉强能看了？  
![3](/img/221201_deollvm.png)   
翻了下看到进来有几处检测，一个很明显判断长度是否16：  
![4](/img/221201_check.png)   
另外两处sub_78B0 sub_74E0，是检测调试：  
![5](/img/221201_antidebug1.png)  
直接patch掉这个检测：  
![6](/img/221201_antidebug.png)   
后面一长串看不懂了，绕晕了，后面看到一个地方很有意思：   
![7](/img/221201_cmp1.png)   
上面这个判断很可疑（其实就是瞎猫撞死耗子），如果这里不行，就没有后文了，显然发了这篇文章这里肯定是可以的。   


## Frida inlinehook 
记录下指令地址，掏出frida开始hook这条指令看看寄存器的值。   
![8](/img/221201_inlinehook.png)   
跑起来，输入测试flag观察结果，我输入个f000000000000000，可以看到出现一次判断成功，后面第二次不相等了就没有后面的判断了：   
![9](/img/221201_hookprint.png)      
再次试试，fl00000000000000，可以看到判断成功两次，同样第三次判断不相等又没有后文了：   
![10](/img/221201_hook1.png)      
根据上面的判断，那思路其实已经有了，逐位爆破应该就是能获取到flag，剩下就是脚本问题，这个好解决，构造一个主动调用，在java层调用checkflag函数，同时inlinehook观察指令判断情况一位一位爆破就行，代码如下：    
![11](/img/221201_code1.png)     
![12](/img/221201_code2.png)   
然后一开始出了个bug，想不通，后来发现是我的flag填充位有问题，不能是0（因为flag中是包含0的，我的代码逻辑没考虑这么多，所以到下一位是0的时候，前一位爆破完，就标志位会出错了）：   
![13](/img/221201_bug.png)     
最后在代码中把填充位换成其他字符，换成z，就愉快的跑出结果了：  
![14](/img/221201_flag.png)    



## 总结
题目设置刚刚好（我看就是不想让人做出来），后面看了别人的wp是用unidbg做的，这个我不怎么会就换成用frida，直接做了出来，主要还是学习一下frida的inlinehook。  

