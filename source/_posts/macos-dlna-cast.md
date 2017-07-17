---
title: macOS 实现 DLNA 投屏
date: 2017-07-17 23:43:21
tags: macOS, swift, dlna
categories: 技术相关
---

月初做的demo，实现了简单的dlna投屏功能。是我有点天真了，最初的设想是实现类似airplay的屏幕镜像功能，把电脑屏幕作为一个视频源在电视机上播放，在了解了dlna协议、视频直播技术之后发现，好像不行。看来这东西不存在也是有原因的。。。

<!-- more -->

## DLNA

DLNA投屏的过程主要通过UPnP协议来完成，过程与许多智能设备（蓝牙、）的通讯大抵相似，总共三步：

1. 设备的发现（与连接）
2. 查询设备支持哪些服务
3. 对特定服务发送指令，接收事件回调

对应到DLNA的UPnP协议：

1. SSDP发现设备
2. HTTP获取设备信息
3. SOAP控制设备

具体在 [Eliyar's Blog - iOS 实现基于 DLNA 的本机图片，视频投屏](https://eliyar.biz/iOS_DLNA_with_local_image_and_video/) 有详细的过程。

然而，DLNA标准只要求实现HTTP GET，因此设备到底支持哪些视频格式，是否支持RTSP、HLS，这是不知道的。很不幸，我家的电视貌似是不支持，屏幕镜像计划泡汤。

## 视频直播

// 回来写

[关于直播，所有的技术细节都在这里了（二）](http://blog.ucloud.cn/?p=699)

## 参考资料

[iOS 实现基于 DLNA 的本机图片，视频投屏](https://eliyar.biz/iOS_DLNA_with_local_image_and_video/)

[OSX下面用ffmpeg抓取桌面以及摄像头推流进行直播](http://www.cnblogs.com/damiao/p/5233431.html)

[dlnap](https://github.com/cherezov/dlnap)

[DLNA_UPnP](https://github.com/ClaudeLi/DLNA_UPnP)

[AVCaptureScreenInput - AVFoundation | Apple Developer Documentation](https://developer.apple.com/documentation/avfoundation/avcapturescreeninput)

[iOS如何实现TCP、UDP抓包](https://www.coder4.com/archives/5273)

[关于直播，所有的技术细节都在这里了（二）](http://blog.ucloud.cn/?p=699)