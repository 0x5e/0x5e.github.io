---
title: iOS运行时更改App语言
date: 2017-01-22 23:25:42
tags: iOS
categories: 技术相关
---

## 需求

主要是方便研发与测试同学的使用，便于验证App在不同语言下的显示效果，希望能够在现有工程不改变大量代码的情况下，尽量满足下面的需求

- 不重启App
- 不改变调用方式（仍然用`NSLocalizedString`）
- 可设置默认语言
- 缺失语言提醒/警告
- 仅在`DEBUG`环境生效，不能影响生产环境
- 对`UIWebView`、`UIBarButtonItem`、`UIPasteboard`等系统组件也能够生效

<!-- more -->

## 分析

多语言文件会被存放在`<App Bundle Path>/<Language Id>.lproj/`下，其中`*.string`在工程中是`"key" = "value";`的文本形式，编译后实际上是二进制的plist文件，可以用`plutil -p /path/to/xxx.strings`查看。

关于语言和地区ID：
[Language and Locale IDs - Apple Developer](https://developer.apple.com/library/content/documentation/MacOSX/Conceptual/BPInternational/LanguageandLocaleIDs/LanguageandLocaleIDs.html)

项目里用到webview的地方很多，访问不同的页面时也需要支持`Accept-Language`的切换。

虽然是个蛮小的需求，但是网上找的些方法暂时还满足不了（也有可能我搜索姿势不太对）

### 现有方案

#### 1、更改 `NSUserDefaults`中`AppleLanguages`字段的值

这是比较被推荐的办法，缺点是重启应用后才生效

```
[[NSUserDefaults standardUserDefaults] setObject:[NSArray arrayWithObjects:@"en", nil] forKey:@"AppleLanguages"];
[[NSUserDefaults standardUserDefaults] synchronize];
```

#### 2、替换`NSLocalizedString`宏

首先实现一个运行时获取多语言文案的方法，比如：

```
@implementation NSBundle (RunTimeLanguage)

+ (NSString *)runTimeLocalizedStringForKey:(NSString *)key {
	// TODO
	NSString *languageId = @"en";
	
	NSString *path = [[NSBundle mainBundle] pathForResource: languageId ofType:@"lproj"];
    NSBundle *languageBundle = [NSBundle bundleWithPath:path];
    return [languageBundle localizedStringForKey:key value:key table:nil];
}

@end
```

然后在`PrefixHeader.pch`中添加：

```
#undef NSLocalizedString
#define NSLocalizedString(key, comment) [NSBundle runTimeLocalizedStringForKey:key]
```

缺点是需要在预编译头文件里添加内容，感觉不太优雅。还有这样对webview不生效。

### TODO

方法一使用的是系统的选取多语言方法，不重启应用切换语言的功能看似无法支持。
方法二的实现会导致webview等系统组件仍然使用了系统语言。

感觉两个方法有点冲突- -ll

争取在假期里完成这个三方库

（有人说todo就是再也不do，希望不是这个样子😊）

不管怎样，去年的指标在年前算是补上了：）@桃小七

----

### Update 2017/1/24

将方案一和方案二做了个合并，仿照`设置 > 通用 > 语言与地区 > iPhone 语言`做了一个高仿的`GSLanguagePickerController`。

[0x5e/GSLanguagePickerController](https://github.com/0x5e/GSLanguagePickerController)

1. 修改语言，保存至`NSUserDefaults`，重启后全局生效
2. 替换`-[NSBundle localizedStringForKey:value:table:]`方法，在重启应用前尽可能的支持多语言

不是很完美，然而有总比没有好。。

现在可以达到的效果是：

| 一个普通标题 | 立刻生效 | 重启应用后生效 |
| ---------- | :-----: | :---------: |
| NSLocalizedString | √ | √ |
| UIBarButtonItem | √(*) | √ |
| UIWebView | × | √ |

其中`UIBarButtonItem`等系统控件的多语言资源是从`UIKit.frameworks`里获取的，而`UIKit.frameworks/*.lproj`命名规则比较旧，有多种格式，会导致部分语言无法立刻生效。

1. `[English Name].lproj`
	
	Examples: `English.lproj`, `French.lproj`, `German.lproj`, `Spanish.lproj`
	
2. `[Language Id]_[Region Id].lproj`
	
	Examples: `zh_CN.lproj`, `zh_HK.lproj`, `zh_TW.lproj`, `en_GB.lproj`
	
3. `[Language Id]-[Script Id].lproj`
	
	Examples: `zh-Hans.lproj`, `zh-Hant.lproj`, `es-ES.lproj`

前两种是已被弃用的规则，但是`UIKit.frameworks`、`InternationalSettings.bundle`等多处仍然存在，第三种是新规则，好像是iOS 9开始的。