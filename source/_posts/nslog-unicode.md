---
title: Xcode控制台输出的编码问题
date: 2017-01-11 21:54:01
tags: iOS
categories: 技术相关
---

`NSLog`在打印集合对象的时候（`NSDictionary`/`NSArray`/`NSSet`），如果集合中的元素有中文字符，直接打印集合对象会被转义。例如：

```
NSDictionary *dict = @{@"chinese": @"中文"};
NSLog(@"%@", dict);
```

打印结果为：

```
2017-01-11 22:05:42.701095 xctest[4258:8003944] {
    chinese = "\U4e2d\U6587";
}
```

StackOverflow上说，这是因为Foundation框架默认用的是`UTF-16`格式，导致`NSLog`打印C字符串的时候，非ASCII字符会被转义。（[NSLog incorrect encoding](http://stackoverflow.com/questions/720052/nslog-incorrect-encoding)）

原本在Xcode8之前可以用插件解决这个问题，插件会对输出的日志进行反转义，丝毫不会影响到项目代码。然而Xcode8禁止了自定义插件的使用🙄，现在插件只能对文本区做一些操作，局限性很高。。。

<!-- more -->

## 分析

`NSDictionary`与`description`相关的方法总共有三个：

- `- [NSDictionary description]`
- `- [NSDictionary descriptionWithLocale:]`
- `- [NSDictionary descriptionWithLocale:indent:]`

前面两个的实现很好猜：

```
- (NSString *)description {
    return [self descriptionWithLocale:nil];
}

- (NSString *)descriptionWithLocale:(NSLocale *)locale {
    return [self descriptionWithLocale:locale indent:0];
}

- (NSString *)descriptionWithLocale:(NSLocale *)locale indent:(NSUInteger)level {
	// ......
}
```

尝试用Hopper打开`CoreFoundation.framework`看了一下，第三个方法是调用了CFString函数进行的组装，问题应该出在这里。

## 解决方案

1、Xcode插件（Xcode 8之前可以）
2、重写集合对象的`descriptionWithLocale:indent:`方法（参考：[swift-corelibs-foundation](https://github.com/apple/swift-corelibs-foundation/)）
3、找到NSLog最终调用的打印函数，用fishhook替换c函数，在打印之前把`\Uxxxx`转换回来，（脑洞大开一下）
4、避免集合对象的直接打印，使用自定义方法，打印前转成JSON字符串（对现有项目，改动较大）

最后我选择了方案2，参考苹果开源的NSDictionary.swift重写了`descriptionWithLocale:indent:`方法，顺便把NSDictionary打印的其它问题一并解决了。[NSObject+NSLog](http://cocoapods.org/pods/NSObject+NSLog)

----

## NSLog打印NSDictionary存在的问题

- unicode编码（就前面说的）
- 嵌套缩进

```
2017-01-14 10:11:28.448512 xctest[19360:9004384] {
    dict =     {
        dict2 =         {
            key = value;
        };
    };
}
```

多层嵌套的字典，`=`和`{`越离越远

- 不同类型打印的格式不统一，有歧义

```
// NSDictionary *dict = @{@"string": @"0", @"int": @(0), @"double": @(0.1)};
// NSLog(@"%@", dict);

2017-01-14 10:19:15.231084 xctest[19483:9007695] {
    double = "0.1";
    int = 0;
    string = 0;
}
```

字符串、整数、小数很难区分清楚
