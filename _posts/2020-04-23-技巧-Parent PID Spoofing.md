---
layout:     post
title:    技巧-Parent PID Spoofing
subtitle:   PPID欺骗
date:       2020-04-23
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - 红队
    - 内网渗透
    - 技巧
    
    
---
# 技巧-Parent PID Spoofing	
## 简介
编号-T1502，通过欺骗新进程的父进程标识符（PPID），以逃避进程监视防御或提升特权。一般情况下，除非明确指定，都是从父进程或者是调用的进程中去产生新的进程。  
## 利用cs中自带功能
首先先上线一个beacon。如果是直接启动的会有一个对应的进程。  
![200422_1](/img/200422_beacon.png)  
若果是不落地上线利用powershell等，则beacon是挂在powershell的进程下的，比较容易被监测到。   
一般会先把进程迁移注入到一个相对稳定不容易排查的进程。  
选定对应的进程注入即可。  
![200422_2](/img/200422_inject.png)  
会返回一个新的beacon，然后把之前的beacon退出移除即可。  
然后选择一个进程右键，set as PPID。   
![200422_3](/img/200422_set.png)  
也可以beacon中输入 ppid xxxx，两者是一样的。  
然后spawn一个新的会话。默认是挂在rundll32下，ppid是设置的ppid。  
![200422_4](/img/200422_rundll.png)  
这样虽然替换了ppid，但是下面挂了一个rundll32还是比较奇怪的。  
通过spawnto来指定对应的进程派生会话。  
![200422_5](/img/200422_spawnto.png)  
然后再派生会话。  
![200422_6](/img/200422_iexplore.png)  
这样就可以了。  
## 利用powershell
[https://github.com/countercept/ppid-spoofing/blob/master/PPID-Spoof.ps1](https://github.com/countercept/ppid-spoofing/blob/master/PPID-Spoof.ps1)
直接运行脚本即可：
```
Import-Module .\PPID-Spoof.ps1
PPID-Spoof -ppid 所在进程ppid -spawnto "C:\Windows\System32\notepad.exe" -dllpath "C:\calc.dll"
```
![200422_7](/img/200422_spoof.png)  
执行以后查看下进程，对应父进程下多了一个notepad，且弹出计算器。  
![200422_8](/img/200422_calc.png)  

## 参考
[http://blog.leanote.com/post/snowming/de88219734d1](http://blog.leanote.com/post/snowming/de88219734d1)  
[https://github.com/redcanaryco/atomic-red-team/blob/master/atomics/T1502/T1502.md#atomic-test-1---parent-pid-spoofing-using-powershell](https://github.com/redcanaryco/atomic-red-team/blob/master/atomics/T1502/T1502.md#atomic-test-1---parent-pid-spoofing-using-powershell)  