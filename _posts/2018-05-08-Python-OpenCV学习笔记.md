---
layout:     post
title:    Python-OpenCV学习笔记
subtitle:   cv捕获摄像头帧
date:       2018-05-08
author:     hmoytx
header-img: img/bg_02.jpg
catalog: true
tags:
    - pythoon 
    - OpenCV
---

# 具体代码

```python
import cv2
 
cap = cv2.VideoCapture(0)  #参数0一般默认笔记本自带的摄像头
    while (1):
        ret, frame = cap.read()  #每一帧都有 所以是动态的 
        cv2.imshow("capture",  frame)
        if (cv2.waitKey(1) & 0xFF == ord('q')):   #q键拍摄退出
            filename = str(int(time.time()))+".jpg"
            cv2.imwrite("img_cap/"+filename, frame)
            break
    cap.release()
    cv2.destroyAllWindows()

``` 
运行效果
![caoture](/img/capture.png)


