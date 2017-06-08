---
title: OpenCV 提取拼图轮廓
date: 2017-06-08 21:21:38
tags: opencv, python
categories: 技术相关
---

六二节收到一份拼图礼物，1000片真狗·doge拼图，我有一句话不知当讲不当讲：
>～！@#¥%……&*（）

<!-- more -->

![](IMG_5001.jpg)

我的内心仿佛有1000只doge正在微笑

但是，由于送礼的人比较特殊，这拼图不拼完好像不行：）

![](IMG_5017.jpg)

![](IMG_5018.jpg)

花了一天时间才完成了1/4。。。边拼边想，要是能用图像处理帮我完成就好了。虽然毫无经验，但是拿来入门，尝试一下也是好的，反正很闲嘛（手动滑稽）

### 尝试

opencv提供了很多很多现成方法，但是怎么排列组合，参数调到多少比较合适，这些一时半会儿也学不会。大致按照滤波、增强、检测几个步骤，反复试参数得到了比较明显的轮廓：

原图：

![](IMG_5008.jpg)

轮廓：

![](IMG_5008.out.jpg)

代码：

```python
#!/usr/bin/env python
#-*- coding=utf8 -*-

import cv2
import numpy

img = cv2.imread('IMG_5008.jpg')

###########################
# 1. 滤波

# 高斯滤波
img = cv2.GaussianBlur(img,(3,3),0)

# 中值滤波
# img = cv2.medianBlur(img, 5)

###########################
# 2. 增强

# 灰度直方图均衡
# img = cv2.equalizeHist(img)

# Laplace图像增强
# kernel = numpy.array([[0, -1, 0], [-1, 5, -1], [0, -1, 0]])
kernel = numpy.ones((3,3),numpy.float32)/9
img = cv2.filter2D(img, -1, kernel)

###########################
# 3.检测

# Canny边缘检测
canny = cv2.Canny(img, 50, 150)

# Sobe
# x = cv2.Sobel(img,cv2.CV_16S,1,0)
# y = cv2.Sobel(img,cv2.CV_16S,0,1)
# absX = cv2.convertScaleAbs(x)
# absY = cv2.convertScaleAbs(y)
# sobel = cv2.addWeighted(absX,0.5,absY,0.5,0)

#提取轮廓
_,contours, hierarchy = cv2.findContours(canny, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
print '轮廓集合：%d 个'%len(contours)

# 标记
cv2.drawContours(img,contours,-1,(0,0,255),2)

# # 标记最大区域
# maxArea = 0
# maxAreaIndex = -1
# for i in range(0, len(contours)):
# 	contour = contours[i]
# 	area = cv2.contourArea(contour)
# 	if area > maxArea:
# 		maxArea = area
# 		maxAreaIndex = i
# cv2.drawContours(img,contours,maxAreaIndex,(0,255,0),2)

cv2.namedWindow('img')
cv2.imshow('img', img)
cv2.waitKey(0)
cv2.destroyAllWindows()

cv2.imwrite('out.jpg', canny)

```

反复调整canny等参数的阈值，我得到的要么是由多个片段组成的完整轮廓，要么是单个片段的残缺的轮廓，这样的数据暂时好像没什么用处。

### 参考资料

[Mac上安装python-opencv](https://lizonghang.github.io/2016/07/16/Mac上安装python-opencv/)
[MoreWindows Blog - OpenCV入门指南](http://blog.csdn.net/MoreWindows/article/category/1291764)
[python openCV图像处理之提取轮廓](http://baoxizhao.com/2017/03/18/python%20openCV图像处理之提取轮廓/)
[Opencv图像增强算法实现(直方图均衡化、Laplace、Log、Gamma)](https://loopvoid.github.io/2017/02/19/Opencv图像增强算法实法/)
[OpenCV边缘检测的详细参数调节](http://www.voidcn.com/blog/wishchin/article/p-5716536.html)
