---
title: iOS动态库的按需加载
date: 2017-03-20 20:44:43
tags: iOS
categories: 技术相关
---

## 起因是这样的

App引入了两个摄像头厂商的SDK，由于他们都用到了ffmpeg，大致是注册解码器的时候产生了冲突，导致A.framework无法使用，Crash了。。。

A.framework的工程师看完错误日志以后认为，是B.framework没有编译进xxx音频解码器，并且注册时机过早，导致A库注册失败，产生闪退。他给的解决方案是：
1、给他一些时间修改ffmpeg源码，调整注册解码器的时机。
2、让B库编译ffmpeg的时候加入xxx音频解码器。

第一个方案听起来就很不放心，并且我也不太明白为什么注册非要改源码才可以？？
考虑到传统厂商的反应速度与沟通成本，在方案二的实施过程中，尝试了一下动态库的按需加载。

<!-- more -->

## 尝试

今年我们放弃了iOS7的支持，才有机会尝试动态库的集成。

首先我新建了一个子项目，对B静态库重新做了一层封装，并且引入所需的系统库、三方静态库依赖，把他重新包装成一个动态库。
这样会清晰很多，接近二十多个依赖，直接链接到App工程里的话，都搞不清楚是什么地方用到了，合并分支也很不方便。

### 使用静态库

将framework链接到项目中：
`Targets-->Build Phases-->Link Binary With Libraries`

### 使用动态库

将framework链接到项目中：
`Targets-->Build Phases-->Link Binary With Libraries`

并且作为资源文件拷贝到Bundle：
`Targets-->Build Phases-->Copy Bundle Resources`

### 动态库按需加载

上一种方法是在应用启动时自动加载动态库的，如果要做按需加载、释放，需要把动态库从`Link Binary With Libraries`列表中移除，在需要的时候自行加载。主要用到了这两个方法：

`-[NSBundle loadAndReturnError:]`
`-[NSBundle unload]`

示例代码：

```objc
// 加载动态库
- (BOOL)loadFramework:(NSString *)frameworkName {
    NSString *path = [NSString stringWithFormat:@"%@/%@", [NSBundle mainBundle].privateFrameworksPath, frameworkName];
    NSBundle *bundle = [NSBundle bundleWithPath:path];
    if (!bundle) {
        NSLog(@"%@ not found", frameworkName);
        return NO;
    }
    
    NSError *error;
    if (![bundle loadAndReturnError:&error]) {
        NSLog(@"Load %@ failed: %@", frameworkName, error);
        return NO;
    } else {
        NSLog(@"Load %@ success", frameworkName);
    }
    
    return YES;
}

// 调用B.framework库中的Person类
- (void)test {
    [self loadFramework:@"B.framework"];
    Class Person = NSClassFromString(@"Person");
    if (!Person) {
        return nil;
    }
    
    Person *person = [[Person alloc] init];
    NSLog(@"%@", [person doSomeThing]);
}
```

`dlopen()`也可以加载动态库，但是应该是过不了应用审核的，所以没有尝试。

需要注意的地方是，因为编译期没有链接动态库，只是把动态库作为资源文件放入应用Bundle目录中，所以`Person`类在主程序中是不存在的，直接`[[Person alloc] init]`会报编译错误：`Undefined symbols for architecture xxx`。需要先加载动态库对应的Bundle，然后在运行时通过`NSClassFromString`获取对应的类，再进行初始化。

## 结果

成功解决了两个库的冲突问题，并且动态库的按需加载，使得我有能力对最终的ipa文件做裁剪。同一个ipa安装包，提供给C客户的需要使用A、B两类摄像头，而D客户并不需要摄像头功能，那我就可以把ipa里的动态库删掉，然后重签名，不光节省几十m的体积，也节约了频繁编译花费的大量时间。

## 参考资料

- [https://www.gitbook.com/book/leon_lizi/-framework-/details](https://www.gitbook.com/book/leon_lizi/-framework-/details)
- [https://skyline75489.github.io/post/2015-8-14_ios_static_dynamic_framework_learning.html](https://skyline75489.github.io/post/2015-8-14_ios_static_dynamic_framework_learning.html)
