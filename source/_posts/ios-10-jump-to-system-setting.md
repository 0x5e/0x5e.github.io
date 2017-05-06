---
title: iOS 10 跳转系统设置
date: 2017-03-04 23:32:37
tags: iOS
categories: 技术相关
---

在iOS 10之前，跳转到系统设置的地址是`prefs:root=XXXX`，这个地址在iOS 10之后失效了。去年查了很多资料，都说是苹果不让用了。
今年发现米家的更新日志里有一条：优化Wi-Fi类设备连接跳转功能
其实就是改了下规则，变为`App-Prefs:root=XXXX`。iOS 8~10 系统均可用:)

还有：`-[UIApplication openURL]`已Deprecated，记得用`-[UIApplication openURL:options:completionHandler:]`代替

<!-- more -->

### Sample

```objc
#define iOS10 ([[UIDevice currentDevice].systemVersion doubleValue] >= 10.0)

NSURL * url = [NSURL URLWithString:@"App-Prefs:root=WIFI"];
if ([[UIApplication sharedApplication] canOpenURL:url]) {
    if (iOS10) {
        [[UIApplication sharedApplication] openURL:url options:@{} completionHandler:nil];
    } else {
        [[UIApplication sharedApplication] openURL:url];
    }
}
```

### References

[Private URL Scheme](http://iphonedevwiki.net/index.php/Preferences.app)
