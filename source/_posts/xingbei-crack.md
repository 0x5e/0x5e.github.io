---
title: 一次停车场蓝牙开闸系统的破解尝试(未完待续)
date: 2016-04-30 13:18:00
categories: 技术相关
---


## 起因

为了相应G20的号召，2016年初开始，每天都有尽职的警察蜀黍贴罚单，路边违停成本大幅上升...只能每天停在地下车库了😔。

收费的小哥看我天天来停，推荐我用"行呗App"，可以用蓝牙开闸，支付宝付钱(真不是广告)。看着挺神奇的，就很想把它的蓝牙开闸原理搞明白。

## 经过

### 抓包

多次折腾以后发现iOS一个抓包神器:`Replica`，是`Surge`作者开发的网络调试工具，超赞☺️

用法和Surge差不多，打开以后日志里会记录Http请求的所有信息和内容(与Surge相比，多了请求和返回的数据内容)

<!-- more -->

发现若干条相关信息:

```
//获取入口钥匙
http://chargerest.park1.cn/BackgroundApp/parkingEntrance/getEntranceKey.do

//Request Body:
{"carParkId": 123456,"userPhoneId": "appUser1234567890"}

//Response Body
{"orderType":1,"errorcode":0,"errorMessage":"","systemTime":"2016-04-28 10:20:15 4","keyData":"7t7p8jaZ31XK+ijUrzRvTeSomXDzRAqlhaCa5xwoxQ="}


//获取出口钥匙
http://chargerest.park1.cn/BackgroundApp/parkingExport/getExportKey.do

//Request Body:
{"carParkId": 123456,"userPhoneId": "appUser1234567890"}

//Response Body
{"errorcode":0,"errorMessage":"出场临停用户发放KEY成功","systemTime":"2016-04-28 18:20:15 4","keyData":"AqlhXDzRoxQaCa5xwXK+ijUrzRvTeSo7t7p8jaZ31m="}


//还有两条入口和出口开启成功的请求,应该是和计费相关的
http://chargerest.park1.cn/BackgroundApp/parkingExport/exportOpenSuccess.do
```

`keyData`应该就是蓝牙开闸的钥匙了。

### 反编译

`keyData`肯定是加密过的，不然也太容易了吧..(恩..想想也是)

再换位思考一下，既然都加密了，那`systemTime`肯定也会用到的..不然传给我做啥= =ll

这里用到了`jadx-gui`进行安卓版本的反编译，勾上`Tools->Deobfuscation`可以起到一定程度的反混淆

用`getExportKey`，`keyData`，`systemTime`，`userPhoneId`，`Bluetooth`等关键字做了大量的搜索与整理,理清了大致开闸流程如下:

1. 进入停车场之前，app里点击开闸按钮，手机会搜索附近的蓝牙设备与广播，并获取广播内容(蓝牙4.0收发数据是可以不用与设备配对的)，解析出停车场名称，闸门名称，停车场id，闸门id，随后在手机上显示各个闸门的名称，供用户选择。这个过程不需要联网。
2. 选定闸门后，会进行网络请求获取`keyData`，然后进行各种解密和移位操作。总结了一下，入参为`keyData`，`systemTime`，`userPhoneNo`(与`userPhoneId`对应,登录时获取)，闸门id(入口为0,出口为非0)，当前时间(年月日)，返回结果是一个byte数组。
3. 将byte数组发送给闸门后，闸门开启，同时手机调用计费接口开始计费。
4. 出场前缴费，后面流程和入场类似。

### 猜想

首先开闸应该是通过手机蓝牙发送数据，而不是通过网络请求。因为有些停车场也是有蓝牙开闸的，用的是一个带有纽扣电池的蓝牙小圆片。并且手机在开闸成功后调用了`xxxOpenSuccess.do`接口。

我猜测停车场和行呗都有计费系统，并且两个系统是相互独立的。(不然不需要手机调用开始计费的接口)

猜测后台流程如下:

1. 闸门获取到手机发出的key，然后分别去停车场后台与行呗后台查询，是否放行。
2. 放行后，行呗后台会计时，停车场后台可能也会计时。
3. 出场前缴费。
4. 出场时闸门取到key，从任意一个后台查到已缴费，即可放行。

## 结果(待验证)

因为计费仍然是通过手机调用接口，如果把该请求拦截了，在出场之前才发送，那么会造成行呗后台计费时间不准确，可以达到近乎免费的效果😳。

五一回来后去试一下..真可以的话...嘻嘻..我什么也不知道☺️

## 解决方法

- 将各个停车场和行呗的计费系统合并(难度有点大)
- 用手机网络请求开闸，然后行呗后台再调用接口让停车场开闸(链路太长，稳定性会有问题，需要改造)
- 改为使用图像识别的开闸系统(这个...)
- 加大app的安全性(好像没什么效果)

----

### 测试结果 

2016/05/06:

几天测试下来,只需要在用app正常入场以后,再伪造出场记录(获取出场key,再调用停止计费接口,直接调用第二个接口会失败),然后在出场之前伪造入场记录,即可免费停车..

调用了一下找车位接口,发现杭州支持行呗蓝牙开闸的停车场还蛮多的,大大小小竟然有100多个🙄🙄🙄
