---
layout:     post
title:      unidbg取巧解ollvm题一例
subtitle:   unidbg取巧解ollvm题一例
date:       2023-08-19
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - 安卓逆向
    - CTF
---
#  unidbg取巧解ollvm题一例

## 前言
是去年的一道题，之前是用模拟执行试图去掉混淆，然后用frida爆破取巧做出来，这次改用unidbg进行分析，针对这种flag按位判断的题，较快定位判断点。后面就是unidbg hook进行爆破。  
之前还要考虑反调试，frida检测，这次使用unidbg就没有那么多顾虑了。  
## 分析App
直接拖进jadx，MainActivity内容如下，代码不多，基本流程就是，获取输入字符，通过一个native层的checkflag判断输入是否正确：  
![1](/img/230819_jadx.png)     
把so直接放进ida，混淆没法看。  
![2](/img/230819_ollvm.png)     
这里开始先不看ida了，先用unidbg执行一遍。   

## Unidbg模拟执行
搭好架子，直接跑，不用补环境一次通。  
这里不知道flag长度，可以先跑一次，把长度爆破出来。  
![3](/img/230819_unidbgrun.png)      
只需要记录下执行的汇编指令数量，就可以根据执行数量来判断flag的长度。这里用unicorn的方式hook记录，  
```
........
    public boolean checkcode(String p1) throws FileNotFoundException {
        emulator.getBackend().hook_add_new(new CodeHook() {
            @Override
            public void hook(Backend backend, long address, int size, Object user) {
                list.add(address);
            }

            @Override
            public void onAttach(UnHook unHook) {

            }

            @Override
            public void detach() {

            }
        }, module.base + 0x93CC, module.base + 0x93CC+0x4844, null);
        String methodDec = "checkflag(Ljava/lang/String;)Z";
        boolean obj = timuclass.callStaticJniMethodBoolean(emulator, methodDec, p1);
        System.out.println(p1.length() + "----" + list.size());
        list.clear();
        return obj;
    }
........
    public static void main(String[] args) throws Exception {
        MainActivity t = new MainActivity();
        int length = 40;
        StringBuilder stringBuilder = new StringBuilder(length);
        String result = "";
        for (int i = 0; i < length; i++) {
            stringBuilder.append("0");
            result = stringBuilder.toString();
            t.checkcode(result);
        }
//        System.out.println(t.checkcode("fla0000000000002"));
    }

}
```
跑了一下，可以明显看到16的时候执行的代码量相比前面的递增1400左右要多了不少，大概就能看出来flag的长度应该是16。    
![4](/img/230819_flaglen.png)     
此时已经知道flag的长度，可以填充，假设flag最后是flag{xxxxx}，则按位尝试，形如f000000000000000,fl00000000000000,fla0000000000000，去获取对应执行的地址，并记录日志，然后对比执行的地址差异：  
![5](/img/230819_tracelogf.png)  
分别执行完0000000000000000，f000000000000000，fl00000000000000，在vscode中对比日志，发现后者多了很多执行地址：  
![6](/img/230819_diff1.png)  
将多执行的地址抠出来，再对比，发现是约840条重复执行，这里其实就能看出来是在按位比较，错一位就跳出，16位flag应该会有16轮比较：  
![7](/img/230819_diff2.png)  
进一步分析循环判断的地址，结合ida，这里我选择从底往上回溯，可以看到一连串连续看了下F5后看不出东西继续往上找，找到不连续的地方应该是B跳转过来。  
![8](/img/230819_lianxu.png)  
找到一个不一样的跳转，但是没有多代码，继续往上   
![9](/img/230819_dc08.png)  
在这一处发现有判断的代码了。   
![10](/img/230819_check.png)  
F5下，可以看到根据判断结果，赋值v429，这时候再往下看看：  
![11](/img/230819_f5.png)  
会跳入0xdc08，然后直接到0x9464，最后赋值给v425，即上面判断的关键变量：  
![12](/img/230819_v429.png)  
此时可以尝试在上面的判断处去hook寄存器值，输入定长flag，按位进行爆破。  



## unidbg爆破
这里so是armv64，我用dobby进行hook。  
```
......
        Dobby dobby = Dobby.getInstance(emulator);
        dobby.instrument(module.base + 0xBDC0, new InstrumentCallback<Arm64RegisterContext>() {
            @Override
            public void dbiCall(Emulator<?> emulator, Arm64RegisterContext ctx, HookEntryInfo info) {
                long w13 = ctx.getXLong(13);
                long w14 = ctx.getXLong(14);
                if (w13 == w14) {
                    index++;
                }
            }
        });
......

    private void crack() throws FileNotFoundException {
        String flag = "";
        for (int i = 0; i < 16; i++) {
            for (int ch = 32; ch < 127; ch++) {
                index = 0;
                String str = (flag + (char) ch + "~~~~~~~~~~~~~~~~").substring(0, 16);
                boolean success = checkcode(str);
                if (success || i + 1 == index) {
                    flag += (char) ch;
                    System.out.println(flag);
                    break;
                }
            }
        }
    }
......
```
先用占位符凑16位长度，按位爆破，很快就能跑出来。  
![13](/img/230819_flag.png)  





## 总结
针对这种ollvm的逆向题，要花时间用trace去还原算法难度确实好大（我很菜），要老老实实的去看tracelog中的寄存器变化跟出结果。但是在这一例中，flag最后是通过按位去比较得出的，这里我取巧是通过了对比不同输入下重复执行的指令地址去定位flag比较的指令地址，不能说是个很好的通用方法，但是确实能在一定程度上简化了分析的流程。  