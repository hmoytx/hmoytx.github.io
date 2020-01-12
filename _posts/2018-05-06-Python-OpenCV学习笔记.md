---
layout:     post
title:    Python-OpenCV学习笔记
subtitle:   cv对图像的基本操作-2
date:       2018-05-06
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - python 
    - OpenCV
---

# 前言
在[上篇文章](https://hmoytx.github.io/2018/05/05/Python-OpenCV%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/)中介绍了opencv中图像文件的基本属性，以及部分基本操作。

# 基本操作-2
## 仿射变换
在仿射变换中，原图中所有的平行线在结果图像中同样平行，就是先经过线性变化，再平移，这里需要在原图像中找到三个点以及在输出图像中的位置，然后创建偏移矩阵，通过getAffineTransform。最后将矩阵传递给warpAffine。
```python
import cv2
import numpy as np


img = cv2.imread("image/gakki.jpg")
rows,cols,channel=img.shape
pts1=np.float32([[50,50],[200,50],[50,200]])
pts2=np.float32([[10,100],[200,50],[100,250]])
M=cv2.getAffineTransform(pts1,pts2)
dst=cv2.warpAffine(img,M,(cols,rows))
cv2.namedWindow("Image")  #窗口
cv2.imshow("Image", dst) #show image

cv2.waitKey(0)
```
效果如图
![fangshe](/img/fangshe.png)

## 区域图像覆盖
```python
import cv2
import numpy as np

img = cv2.imread("image/gakki.jpg")
roi = img[0:100, 0:100]
img[100:200, 100:200] = roi
cv2.namedWindow("Image")  #窗口
cv2.imshow("Image", img) #show image

cv2.waitKey(0)
```
![fugai](/img/fugai.png)

## 通道颜色修改
```python
import cv2
import numpy as np

img = cv2.imread("image/gakki.jpg")

img[:, :, 0] = 0 #对RGB全通道修改
cv2.namedWindow("Image")  #窗口
cv2.imshow("Image", img) #show image

cv2.waitKey(0)
```
![RGB](/img/RGB.png)


# 结语
暂时就先介绍这些操作，其他还有很多，用到的时候再一一介绍，本身参考的资料和博客比较多，所以顺序上有点乱。
