---
title: iOS逆向工程实践
date: 2016-08-24 23:27:47
tags: iOS
categories: 技术相关
---


借一次帮同学破解某奇葩的app的机会，初步了解并掌握了iOS逆向相关知识。

<!-- more -->

## 目的

解出某个类似挖矿的app的通信协议，然后写个脚本自动挖矿

## 了解

开始一头雾水，app能挖矿？用了一段时间才大致了解他的原理。。

这个app需要通过蓝牙连接一个特定穿戴设备，保持设备运行，采集所谓的脑电波。。采集的时间越长，脑电波图形越好，最终数据上传到服务端，得分就越高。

置于奇葩的app做这事儿有什么意义。。分数可以换成虚拟货币，而这货币最终是买单的呢，其实里面还有个规则是发展下线，可以从下线的得分得到提成，然后每个人都需要花钱买这个穿戴设备才能采集图形，换取积分，所以钱最终是从下家买的设备这里来的，类似于。。。。你懂得

## 逆向前的分析

先抓包看了一下网络请求，发现发送和接收的数据都经过base64编码了，解出来是一堆乱码，看来是加密过啦。（不然也不用写这篇文章了- -ll）

然后我先反编译安卓版本看了一下（这个有现成工具比较好搞），貌似是用爱加密加固过了，很多关键的函数都反编译失败了看不到，不过还是看到了一些线索：找到两个和AES、RSA相关的类名，以及一个getPubKey()函数，里面应该是RSA公钥。于是猜想数据应该是经过AES和RSA加密了。

于是碰到一个坑：python没法用公钥解密数据。。。搜了很久，解决方案都是一本正经地说，你不应该用私钥加密，公钥解密，这样做是不对的

(╯‵□′)╯︵┻━┻

对不对关我pis啊！！！@#￥%……&*

一度怀疑我是不是做错了，是不是app里还藏了私钥（然而并没有）

花了大半天还是解决不了，最后用java写了个用公钥解密的类。。。

分析不下去了，那么开始逆向吧- -ll

## 脱壳

AppStore下载下来的ipa都是经过加密的，解压出.app，用`otool`可以看到二进制文件的信息里，有一个`cryptid`字段，1代表被加密，0就是没加密，解密的过程被称作脱壳

获取脱壳后的应用有两种途径：

1. 从PP助手等越狱市场下载越狱应用，比较省事，缺点是冷门app你不一定找得到（或是Surge这种特殊的app，应该不算冷门吧，但是越狱市场都是找不到的）

2. 自行砸壳/脱壳，使用[dumpdecrypted](https://github.com/stefanesser/dumpdecrypted)、[Clutch](https://github.com/KJCracks/Clutch)工具（需要一台越狱的设备）砸壳后再用ssh、scp之类的拷贝到mac上，略为繁琐

## 重签名

按理越狱手机是没有这个问题的吧？。。可能9.3.3是不完美越狱的缘故

替换成有效的bundle_id, codesigning_identity, privisioning_profile，再对app重新打包，就可以在手机上安装了（包括非越狱设备）

这个过程可以用[iOS App Signer](https://github.com/DanTheMan827/ios-app-signer)实现，还算方便

## 安装越狱相关工具

一大堆东西，有的没的反正能装的都装上吧。。

- iOSOpenDev（越狱开发环境）
- theos（同上）
- [cycript](http://www.cycript.org/)
- [Clutch](https://github.com/KJCracks/Clutch)
- [class-dump](https://github.com/nygard/class-dump)（从Mach-O文件中，导出所有OC类的头文件）
- mobiledevice（直接安装ipa到手机上）
- ideviceinstaller（同上）
- [yololib](https://github.com/KJCracks/yololib)（将dylib注入到二进制文件，Hook App）
- Hopper（Mac反编译工具）

iOS版本太新还是蛮尴尬的，越狱相关的工具一般不会很快的更新，甚至有些都不维护了，再加上9.3.3是不完美越狱，导致部分工具不可用（cycript），部分教程不可参考，遇上了很多困难。

再由于我要破解的app是swift写的，class-dump不可用，iOSOpenDev新建的模板也没法直接用，亚历山大啊╮(╯▽╰)╭

## 寻找要Hook的目标

先列出app所有类名和方法名。class-dump、flex3、hopper这些都能用，其中class-dump可以直接导出所有头文件，不过暂不支持swift。

由于OC的runtime机制，方法都是动态的，如果像java那样直接混淆类名和方法名，在运行时动态调用方法就会出错吧。所以能从类名里获取到很多信息。

（希望以后能有这方面的加密工具，找到过几个相关的脚本感觉不是太可靠，并且需要修改代码配合混淆，有一定的开发量，不是很方便）

大致确定了几个目标：

- `[SecurityUtil encryptAESData:app_key:]`
- `[SecurityUtil decryptAESData:app_key:]`
- `[MindAsset.RSAUtils addRSAPrivateKey:tagName:]`
- `[MindAsset.RSAUtils addRSAPublicKey:tagName:]`
- `[MindAsset.HttpManager uploadDataToAliyun:accessKeyId:secretKeyId:sucessblock:]`

我尝试的方案是：编写dylib注入到应用中，在应用初始化的时候，swizzle这几个方法，然后在设备日志里找到我打印的日志，分析线索

为什么不直接调试app呢(╯‵□′)╯︵┻━┻

我也不知道，试了一下调试不了。。有空再琢磨一下为什么

感觉直接调试会更方便

## 开始Hook

新建Xcode项目，选择`CaptainHook Tweak`模板，`*.xm`是一个模板，可以用`%hook`、`%log`、`%orig`方便的对某个类的方法进行hook。

xm文件头部定义了一个error：

```
#error iOSOpenDev post-project creation from template requirements (remove these lines after completed) -- \
	Link to libsubstrate.dylib: \
	(1) go to TARGETS > Build Phases > Link Binary With Libraries and add /opt/iOSOpenDev/lib/libsubstrate.dylib \
	(2) remove these lines from *.xm files (not *.mm files as they're automatically generated from *.xm files)
```

需要从你的`iOSOpenDev`目录里引入`libsubstrate.dylib`，然后把这段error注释掉就好了。在build的时候项目会跑一个脚本对这些标记进行替换，再用宏定义引入至`*.mm`文件中，编译成dylib。

xm文件用着不太习惯，没有语法高亮了。。直接像平时开发的时候那样加个category，自己写swizzle也可以。。

[xm文件写法-Logos](http://iphonedevwiki.net/index.php/Logos)

```
%hook SecurityUtil

+ (id)encryptAESData:(id)data app_key:(id)key {
    
    id result = %orig(data, key);
    
    NSLog(@"encryptAESData '%@' '%@' '%@'", data, key, result);
    
    return result;
}

+ (id)decryptAESData:(id)data app_key:(id)key {
    
    id result = %orig(data, key);
    
    NSLog(@"decryptAESData '%@' '%@' '%@'", data, key, result);
    
    return result;
}

%end

```

编译后在Products找到编译好的dylib，然后用yololib工具把它注入到Mach-O文件中。

```
→ ./yololib MindAsset hook.dylib
2016-08-23 22:24:50.338 yololib[71513:1121551] dylib path @executable_path/hook.dylib
2016-08-23 22:24:50.351 yololib[71513:1121551] dylib path @executable_path/hook.dylib
Reading binary: MindAsset

2016-08-23 22:24:50.351 yololib[71513:1121551] Thin 64bit binary!
2016-08-23 22:24:50.351 yololib[71513:1121551] dylib size wow 56
2016-08-23 22:24:50.352 yololib[71513:1121551] mach.ncmds 60
2016-08-23 22:24:50.352 yololib[71513:1121551] mach.ncmds 61
2016-08-23 22:24:50.352 yololib[71513:1121551] Patching mach_header..
2016-08-23 22:24:50.352 yololib[71513:1121551] Attaching dylib..

2016-08-23 22:24:50.352 yololib[71513:1121551] size 51
2016-08-23 22:24:50.352 yololib[71513:1121551] complete!
```

再把Mach-O和dylib一同复制回.app目录下，重新打包签名，装上手机运行，用Xcode或者命令行工具`idevicesyslog`查看手机运行日志：

```
Aug 29 00:15:48 iPhone7s MindAsset[11330] <Warning>: encryptAESData '{"password":"123456","version":"1.0.22","mobile":"18000000000","platform":"iOS"}' 'fucgqsdlefath' 'YV4F2uYUK3BmM90at/5leyfsz7/OJ+V/k8WTyC0WYrP4M1zbSZdIP47XxuPaXXCsRRdCf0YeldRtMBwCbRNc9rTgNgTz+BGWQ/iaYubbVv71w8yMJ/gCDu8nL8ijoeKl'
Aug 29 00:15:48 iPhone7s MindAsset[11330] <Warning>: decryptAESData '<1c4e3425 bff4c6ec 525dd10b b3afc339 944337ff 9d5a2ef3 066d7e07 0d05bf22 8e623abf a80be7c1 555cd64c ba436103 61830f09 b34e99a1 a29f3d4a 0800ca50 effb627e baf387bf 00b8a175 7f3b3b97 93fcbb47 68984e88 8828f665 775af738 2620e5b8 53127e7b 4ba016bb c77b5ad6 e5ec0e20 640f6608 625fb4da 9078e1ed 5257e9da 192d55bc 1f3c4ffa 3679f358 5dd79f05 3b7652e4 de0f2e07 10f44396 bd8a1ec0 3a181590 6dccbee6 ddb96f8b bf9a22b5 4878833d ff624f0a 41377208 5532679e 5b5d6145 914d61ae 0fa42ca5 67807fa2 98775bd8 8badae71 be95363b ea22806c 49c7e0fc 11eb86fb 58e5a0af efc902e8 407bd0e0 019cc2bb 8a841c65>' '5Mvk20CB' '{"status":0,"data":{"uid":xxx,"invitor_name":"xxx","huanxin_id":"xxx","huanxin_pwd":"xxx","in_use_huanxin_customer":1,"waiter_id":"xxx","waiter_name":"xxx"}}'
```

Hook成功了，发现确实调用了AES方法，但是每次加密解密的aesKey都是会变的，看着像随机生成的，那么联想到apk文件里找到的RSA公钥，再对RSA的加解密方法hook一下看看

## C函数Hook

之前查看了一下应用所有用到的类，没有第三方的加密库，那么我猜用的是系统自带的`SecKeyEncrypt`、`SecKeyDecrypt`方法

C函数也是可以hook滴，用facebook的[fishhook](https://github.com/facebook/fishhook)吧

```
#import <fishhook/fishhook.h>
#import <dlfcn.h>
#import <CommonCrypto/CommonCrypto.h>


//SecKeyEncrypt
static OSStatus (*orig_SecKeyEncrypt)(SecKeyRef, SecPadding, const uint8_t *, size_t, uint8_t *, size_t *);
static OSStatus my_SecKeyEncrypt(
                                 SecKeyRef key,
                                 SecPadding padding,
                                 const uint8_t *plainText,
                                 size_t plainTextLen,
                                 uint8_t *cipherText,
                                 size_t *cipherTextLen) {

    OSStatus result = orig_SecKeyEncrypt(key, padding, plainText, plainTextLen, cipherText, cipherTextLen);
    
    NSData *cipherData = nil;
    NSData *plainData = nil;
//    NSData *cipherData = [NSData dataWithBytesNoCopy:cipherText length:cipherTextLen];
//    NSData *plainData = [NSData dataWithBytesNoCopy:(void *)plainText length:plainTextLen];
    NSLog(@"SecKeyEncrypt SecPadding: %d, plainData: %@, cipherData: %@", padding, plainData, cipherData);
    
    return result;
}

//SecKeyDecrypt
static OSStatus (*orig_SecKeyDecrypt)(SecKeyRef, SecPadding, const uint8_t *, size_t, uint8_t *, size_t *);
static OSStatus my_SecKeyDecrypt(
                                 SecKeyRef key,
                                 SecPadding padding,
                                 const uint8_t *cipherText,
                                 size_t cipherTextLen,
                                 uint8_t *plainText,
                                 size_t *plainTextLen) {

    OSStatus result = orig_SecKeyDecrypt(key, padding, cipherText, cipherTextLen, plainText, plainTextLen);
    
    NSData *cipherData = nil;
    NSData *plainData = nil;
//    NSData *cipherData = [NSData dataWithBytesNoCopy:(void *)cipherText length:cipherTextLen];
//    NSData *plainData = [NSData dataWithBytesNoCopy:plainText length:plainTextLen];
    NSLog(@"SecKeyDecrypt SecPadding: %d, cipherData: %@, plainData: %@", padding, cipherData, plainData);
    
    return result;
}

//CCCryptorCreate
static CCCryptorStatus (*orig_CCCryptorCreate)(CCOperation, CCAlgorithm, CCOptions, const void *, size_t, const void *, CCCryptorRef *);
static CCCryptorStatus my_CCCryptorCreate(
                                          CCOperation op,
                                          CCAlgorithm alg,
                                          CCOptions options,
                                          const void *key,
                                          size_t keyLength,
                                          const void *iv,
                                          CCCryptorRef *cryptorRef) {

    CCCryptorStatus result = my_CCCryptorCreate(op, alg, options, key, keyLength, iv, cryptorRef);
    
    NSData *keyData = [NSData dataWithBytesNoCopy:(void *)key length:keyLength];
    NSData *ivData = [NSData dataWithBytesNoCopy:(void *)iv length:keyLength];
    NSLog(@"CCCryptorCreate CCOperation: %d, CCAlgorithm: %d, CCOptions: %d, key: %@, iv: %@", op, alg, options, keyData, ivData);
    
    return result;
}

@implementation NSObject (FishHook)

+ (void)load {
    
    NSLog(@"fishhook start");
    
    orig_SecKeyEncrypt = dlsym(RTLD_DEFAULT, "SecKeyEncrypt");
    orig_SecKeyDecrypt = dlsym(RTLD_DEFAULT, "SecKeyDecrypt");
    orig_CCCryptorCreate = dlsym(RTLD_DEFAULT, "CCCryptorCreate");
    
    rebind_symbols((struct rebinding[3]){
        {"SecKeyEncrypt", my_SecKeyEncrypt},
        {"SecKeyDecrypt", my_SecKeyDecrypt},
        {"CCCryptorCreate", my_CCCryptorCreate},
//        {"SecKeyEncrypt", my_SecKeyEncrypt, (void *)orig_SecKeyEncrypt},
//        {"SecKeyDecrypt", my_SecKeyDecrypt, (void *)orig_SecKeyDecrypt},
    }, 3);
    
    NSLog(@"fishhook end");
    
}

@end

```

反正就大概这么个意思。。。最后没有从这儿获得啥信息😷

拿手机日志和网络请求对比，就搞清楚每个接口的请求格式了。至于前面随机生成的aesKey，经RSA公钥加密，随网络请求藏在了HTTP Header中，Response也是一样。

## 小结

iOS的安全性被很多人给忽视了，没想到也是挺脆弱的。。

从这次尝试中发现还是有很多运气的成分的╮(╯▽╰)╭

首先要猜的到大概的关键词，找到加密解密相关类，然后hook看日志，再和网络请求的数据进行对比，最后搞清楚数据结构，写出伪造请求的脚本。

遇到的困难主要是：
- 刚出的9.3.3不完美越狱，部分工具不可用
- 基本是纯swift写的app，没hook成功，用hopper看具体实现很困难，有点无从下手。。

感觉应该有更好的方法，我的这个流程（编码，编译dylib，注入，打包，重签名，安装，看日志）太长了😷

有空继续研究