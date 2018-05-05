---
layout:     post
title:    Python-OpenCV学习笔记
subtitle:   cv对图像的基本操作-1
date:       2018-05-05
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - pythoon 
    - OpenCV
---

# 前言
## OpenCV简介
OpenCV是一个跨平台的计算机视觉库，它轻量级而且高效，由一系列的C函数和部分C++类构成，同时提供了Python，Ruby等语言的接口，实现了图像处理和计算机视觉方面很多的算法。

## 安装
这里用的是python3.6，直接去下载opencv_python包(注意系统位数和版本)，下载后用pip install安装。在命令行进入python尝试import cv2，如果没有报错就可以了。

# 基本属性
cv2.imread(filename, 属性) 第二个参数指定图像用哪种方式读取文件
    cv2.IMREAD_COLOR：读入彩色图像，默认参数，Opencv 读取彩色图像为BGR模式！
    cv2.IMREAD_GRAYSCALE：读入灰度图像

cv2.imwrite(保存图像名，需要保存的图像)

cv2.imshow(窗口名， 图像文件) 显示图像，可以多个创建

cv2.waitKey() 等待键入，等待特定的时间，检测是否有键盘键入

cv2.namedWindow(窗口名，属性) 创建一个窗口，属性指定窗口大小模式
    cv2.WINDOW_AUTOSIZE：根据图像大小自动创建大小
    cv2.WINDOW_NORMAL：窗口大小可调整

cv2.destoryAllWindow(窗口名) 删除建立的窗口

## 具体代码
```python
import cv2

#选取几个常用的函数演示一下
img = cv2.imread("image/gakki.jpg") #路径
cv2.namedWindow("Image")  #窗口
cv2.imshow("Image", img) #show image
cv2.waitKey(0)

```
运行结果
![gakki](/img/gakki.png)

# 基本操作

## 获取属性
    img.shape 输出一个元组，(x, y, 通道)
    img.size 输出上面元组的乘积

## 输出文本
处理图片时，将一些信息以文字的形式展现在图片上
cv2.putText(图片名, 文字, 坐标, 文字颜色)

## 缩放
实现缩放图片并保存，在使用OpenCV时常用的操作。cv2.resize()支持多种插值算法，默认使用cv2.INTER_LINEAR，缩小最适合使用：cv2.INTER_AREA，放大最适合使用：cv2.INTER_CUBIC或cv2.INTER_LINEAR。 
res=cv2.resize(image,(2*width,2*height),interpolation=cv2.INTER_CUBIC) 
或
res=cv2.resize(image,None,fx=2,fy=2,interpolation=cv2.INTER_CUBIC) 
此处None本应该是输出图像的尺寸，因为后边设置了缩放因子

## 平移
cv2.warpAffine(src, M, dsize[, dst[, flags[, borderMode[, borderValue]]]]) 
平移就是将图像换个位置，如果要沿(x,y)方向移动，移动距离为(tx,ty),则需要构建偏移矩阵M。
M = [[1, 0, x],[0, 1, y]]
```python
import cv2
import numpy as np

img = cv2.imread("image/gakki.jpg")
rows,cols,channel=img.shape
M=np.float32([[1,0,100],[0,1,50]])
dst=cv2.warpAffine(img,M,(cols,rows))  #第三个参数是输出图像大小
cv2.namedWindow("Image")  #窗口
cv2.imshow("Image", dst) #show image
cv2.waitKey(0)
```
![move](/img/move.png)

## 旋转
首先需要构造一个旋转矩阵，可通过cv2.getRotationMatrix2D获得。
```python
import cv2
import numpy as np

img = cv2.imread("image/gakki.jpg")
rows,cols,channel=img.shape
M=cv2.getRotationMatrix2D((cols/2,rows/2),45,0.6)#旋转中心， 旋转角度， 缩放因子
#第三个参数为图像的尺寸中心
dst=cv2.warpAffine(img,M,(2*cols,2*rows))
cv2.namedWindow("Image")  #窗口
cv2.imshow("Image", dst) #show image

```
![xuanzhuan](/img/xuanzhuan.png)