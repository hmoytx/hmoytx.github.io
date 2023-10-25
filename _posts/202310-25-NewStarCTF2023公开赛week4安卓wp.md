---
layout:     post
title:      NewStarCTF2023公开赛week4安卓wp
subtitle:   NewStarCTF2023公开赛week4安卓wp
date:       2023-10-25
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - 安卓逆向
    - CTF
---
#  NewStarCTF2023公开赛week4安卓wp

## anti
放进jadx，发现增加了调试检测。  
![1](/img/231025_1_anti.png)    
继续看算法，很眼熟，又是rc4，将输入值加密后传入native层encode函数判断。   
![2](/img/231025_1_rc4.png)    
直接打开so，去找函数，这里是通过JNI_Onload动态注册。  
![10](/img/231025_1_jnionload.png)    
找到对应的encode函数。  
![3](/img/231025_1_xxx.png)      
通过获取输入字符串，进行b64（看名字可以知道是base64），然后和密文比较。     
![4](/img/231025_1_cipher.png)    
b64是变过表的base64，找到码表：    
![5](/img/231025_1_table.png)    
直接去网站解密一把梭，最后结果如下：   
![6](/img/231025_1_flag.png)    


## iwannarest
放进jadx，东西很少，直接看so。   
![7](/img/231025_2_jadx.png)     
同样的，也是动态注册，找到对应的encode函数。  
![8](/img/231025_2_encode.png)       
接下来分析算法，这里的算法看起来是和之前tea的有区别，查了下是xxtea，这里只进行32轮运算。
这里有个key是不知道，我用ida动态调试了下so，拿到key的值，最后直接贴出解密代码：   
```c
#include <stdlib.h>
 
int main()
{
    unsigned int v0 = -979885167;
    unsigned int v1 = 861271549;
    int v12 = 33 * (-1640531527);
    //1697034457
    int v13 ;
    // 加密
    int key[27] = {1, 1, 4, 5, 0x7AD7A6C0, 0x73, 0x7AD7A6C7, 0x73, 0x7AD7AB78, 0x73, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0};
    /*
    for (int i=0; i < 32; i++) { 
        v13 = (v12 >> 2) & 3;
        v0 += (((v1<<2) ^ (v1 >> 5)) + ((v1 >> 3) ^ (v1<<4))) ^ ((key[v13] ^ v1) + (v1 ^ v12));
        v1 += (((v0<<2) ^ (v0 >> 5)) + ((v0 >> 3) ^ (v0<<4))) ^ ((key[v13 ^ 1] ^ v0) + (v0 ^ v12));
        v12 -= 1640531527;
    }*/
    
    for (int i=0; i < 32; i++) { 
        v12 += 1640531527;
        v13 = (v12 >> 2) & 3;
        
        v1 -= (((v0<<2) ^ (v0 >> 5)) + ((v0 >> 3) ^ (v0<<4))) ^ ((key[v13 ^ 1] ^ v0) + (v0 ^ v12));
        v0 -= (((v1<<2) ^ (v1 >> 5)) + ((v1 >> 3) ^ (v1<<4))) ^ ((key[v13] ^ v1) + (v1 ^ v12));
        
    }
    

    
    printf("%ld\n",v0);
    printf("%ld",v1);
}
```    
最后运算得出16进制密文：   
```
6761 6C66
7544 477B
306C 4937
5075 3376
336C 7A31
4733 6D74
7564 4072
7D65 7461
```    
这里还有大小端序问题，最后结果是：   
```
666c61677b47447537496c3076337550317a6c33746d3347724064756174657d
```   
最后结果如下：  
![9](/img/231025_2_flag.png)    

