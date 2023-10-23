---
layout:     post
title:      NewStarCTF2023公开赛week3安卓wp
subtitle:   NewStarCTF2023公开赛week3安卓wp
date:       2023-10-19
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - 安卓逆向
    - CTF
---
#  NewStarCTF2023公开赛week3安卓wp

## runfaraway
放进jadx，发现是传入字符串和key（固定），通过调用native层的encode方法，判断是否正确。  
![1](/img/231019_1_jadx.png)    
将so放入ida分析，可以看到，这v7是字符串（图没截全），v8是key，经过算法处理后，再用test函数处理v7，得到的值与下面的一串密文比较。   
![2](/img/231019_1_cipher.png)    
先看下test做了什么事，不看函数猜一下应该是做了什么编码，不然v7经过处理应该是没法直接进行明文比较了。   
看了应该是改过码表的base64：  
![3](/img/231019_1_base64.png)      
我在test函数传入时将值修改成密文的值，然后看他解密后的结果，验证下猜想：  
![4](/img/231019_1_change.png)    
然后继续运行，查看输出值：  
![5](/img/231019_1_cipher1.png)    
把这串玩意复出来，用他的码表base64解密，发现还原后的结果正确，猜对了：  
然后继续运行，查看输出值：  
![6](/img/231019_1_base64right.png)    
接下来就简单了，在test函数之前下断点，将密文解密还原，然后通过unidbg按位爆破即可。  
先把密文转化成byte数组，然后跑代码。   
代码如下：   
```java
    public boolean checkcode(String p1) throws FileNotFoundException {
        byte[] flagbytes = {24,64,8,16,-64,105,-39,120,32,-128,40,-47,120,8,-70,-94,-78,8,88,112,-31,40,-72,24,-70,-86,-102,-80,16,0,64,0,32,112,8,96,48,88,-78,-96,32,-39,-96,104,80,-72,-39,112,-80,-39,-104,-86,-30,-56};
        final boolean[] res = {false};
        emulator.getBackend().hook_add_new(new CodeHook() {
            @Override
            public void hook(Backend backend, long address, int size, Object user) {
                if (address == module.base + 0x0D3C) {
                    RegisterContext context = emulator.getContext();
                    byte arg0 = context.getPointerArg(0).getByte(index);
                    byte arg1 =flagbytes[index];
                    if (arg0 == arg1) {
                        index++;
                        res[0] = true;

                    }
                }
            }
            @Override
            public void onAttach(UnHook unHook) {

            }

            @Override
            public void detach() {

            }
        }, module.base + 0x0D38, module.base + 0x0D48, null);

        String methodDec = "encode(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String";
        Object obj = timuclass.callStaticJniMethodObject(emulator, methodDec, p1, "deadbeef");
        return res[0];
    }

    public void crack() throws FileNotFoundException {
        String flag = "";
        for (int i = 0; i < 54; i++) {
            for (int ch = 32; ch < 127; ch++) {
                String str = (flag + (char) ch + "~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~").substring(0, 54);
                boolean success = checkcode(str);
                if (success || i + 1 == index) {
                    flag += (char) ch;
                    System.out.println(flag);
                    break;
                }
            }
        }
    }
```
 最后结果如下：   
 ![7](/img/231019_1_flag.png)    


## it2hard2u
放进jadx，发现被BlackObfuscator处理过，各种分支跳转。   
![8](/img/231019_2_jadx.png)     
看了网上说JEB5.1可以去除掉这个控制流混淆，下了个新版本的（JDK17才能跑），一看确实清爽很多。  
![9](/img/231019_2_jeb.png)     
看一下流程，输入以后通过encrypt函数加密生成字符串，传给encode函数在native层，这里encrypt是个标准的rc4算法：  
![10](/img/231019_2_rc4.png)   
用IDA打开分析so，是静态注册，进来可以看到获取字符串以后，依次取出数据进行了一个32轮的运算，最后比较是否与下面的密文相等：  
![11](/img/231019_2_idaso.png)     
看算法的特征，应该是改过的tea算法，这里我选择用unidbg动态调试一下，查看运算输入的值，先还原出一部分。  
unidbg架子搭好直接抠一个rc4算法过来，开始测试：   
这里我下两处断点，第一处下在获取jstring的地方，第二处下在上面开始运算前取数的地方。  
![12](/img/231019_2_bp1.png)  
对应的汇编处的地址如下：  
![13](/img/231019_2_bp2.png)    
在unidbg中加两行调试：   
```java
.........
    public Object checkcode(String p1) throws FileNotFoundException {
        Debugger debugger = emulator.attach();
        debugger.addBreakPoint(module.base+0x15AE4);
        debugger.addBreakPoint(module.base+0x15B28);
        StreamUtils rc4 = new StreamUtils();
        System.out.println(rc4.encryptRC4(p1, "friedchick"));
        String methodDec = "encode(Ljava/lang/String;)Ljava/lang/String";
        Object obj = timuclass.callStaticJniMethodObject(emulator, methodDec, p1, "deadbeef");
        return obj;
    }
.........
```   
开始调试，第一处断点先把传入的数据拿出来：  
![14](/img/231019_2_data1.png)     
然后c执行，到第二处断点，我们直接看12,13两个寄存器，可以发现是4个读取，按先后，13存前一部分，12存后一部分，注意大小端序：  
这里控制台输出为：  x12=0xb4c3bac2 x13=0x9ac25f51   
![15](/img/231019_2_data2.png)     
然后我们把断点打在运算完后存入数据的地方，查看值：  
![16](/img/231019_2_bp3.png)   
得到运算结果：  x12=0x9c1813da x13=0xf244b822    
接下来分析算法，这里的算法看起来是tea，我们照葫芦画瓢先把加密的过程模拟出来，这里稍微优化了下：   
```c
#include <stdio.h>

int main()
{
	unsigned int v0 = 0x9ac25f51;
	unsigned int v1 = 0xb4c3bac2;
	int v12 = -1640531527;
	// 加密
	
	for (int i=0; i < 32; i++) { 
		v0 += ((v1 >> 5) + 2) ^ ((v1 << 4) + 5) ^ (v1 + v12);
		v1 += ((v0 << 4) + 7) ^ (v0 + v12) ^ ((v0 >> 5) + 5);
		v12 -= 1640531527;
	}
	

	
	printf("%ld\n",v0);
	printf("%ld",v1);
}
```   
运行结果如下，是正确的：
```c
2379516734
2077360932

//转16进制
0xF244B822
0x9C1813DA
```  
此时就可以根据下面的密文用解密算法回推数据了：  
```c
#include <stdio.h>

int main()
{
	
	unsigned int v0 = 1709447429;
	unsigned int v1 = 319073545;
	//加密 int v12 = -1640531527;
	int v12 = 1697034457; //解密
	// 加密
	/*
	for (int i=0; i < 32; i++) { 
		v0 += ((v1 >> 5) + 2) ^ ((v1 << 4) + 5) ^ (v1 + v12);
		v1 += ((v0 << 4) + 7) ^ (v0 + v12) ^ ((v0 >> 5) + 5);
		v12 -= 1640531527;
	}
	*/
	// 解密
	
	for (int i=0; i<32; i++) { 
		v12 += 1640531527;
		v1 -= ((v0 << 4) + 7) ^ (v0 + v12) ^ ((v0 >> 5) + 5);
		v0 -= ((v1 >> 5) + 2) ^ ((v1 << 4) + 5) ^ (v1 + v12);
		
	} 
	
	printf("%ld\n",v0);
	printf("%ld",v1);
}
```    
最后运算得出密文：   
```
0601c388c3acc2bdc2b8c2aec2a213c3a2c3bcc3ab0728c38f43c287c294c39d546cc295c385c3a153c2865d78c3bcc2
```   
这里还无法直接解密，因为java层String传过来到native层通过GetStringUTFChars函数会转换成utf8，我们需要转换一下。  
此时这里就碰到问题了，发现密文贴上去直接转换失败：  
![17](/img/231019_2_error.png)    
删除后两位发现能解出部分flag：  
![18](/img/231019_2_flagless.png)    
此时再进行测试，我将"flag{G0ddamn700h@rd_f0ru_n0w!"作为部分解，后面结尾肯定是"}"，我在中间插入一个字符，通过undibg爆破，发现有大量的解正确。。。。   
![19](/img/231019_2_flagmore.png)    
还是再unidbg之前的断点查看，发现只要前面是"flag{G0ddamn700h@rd_f0ru_n0w!"，后面你输多少位都是可以正确，这可能是题目的bug。。。也可能是我的方法不是正确的。  
可以看到后面补再多，只要"0601c388c3acc2bdc2b8c2aec2a213c3a2c3bcc3ab0728c38f43c287c294c39d546cc295c385c3a153c2865d78c3bcc2"存在了，就一定会成功。。。  
![20](/img/231019_2_bug.png)    
最后靠手动试出来是三个感叹号。。。。   

