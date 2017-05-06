---
title: 震惊！Xcode原来还有这种功能！
date: 2017-03-26 11:54:41
tags: iOS
categories: 技术相关
---

![](imgs/zhenjing.jpg)

这么多年才发现这个功能。。感觉一大把时光被浪费了- -ll

## Quick Look Debugging

macOS的Quick Look是我很喜欢的功能之一，其实Xcode调试器也有这样的能力，可以快速查看一些实例对象的可视化内容，节省很多调试UI的时间。

![](http://nshipster.s3.amazonaws.com/quicklook-color.gif)

<!-- more -->

### 内置类型

>
- 图片： UIImage，NSImage，UIImageView，NSImageView，CIImage，和 NSBitmapImageRep 都可以快速查看。
- 颜色： UIColor 和 NSColor。 （抱歉，CGColor。）
- 字符串： NSString 和 NSAttributedString。
- 几何： UIBezierPath 和 NSBezierPath，以及 CGPoint，CGRect，和 CGSize。
- 地区： CLLocation 将显示一个很大的，互动的映射位置，并显示高度和精度的细节。
- URLs： NSURL 将显示 URL 所指的本地或远程的内容。
- 光标： NSCursor，为我们中间的光标指示。
- SpriteKit： SKSpriteNode，SKShapeNode，SKTexture，和 SKTextureAtlas 都会被显示。
- 数据： NSData 将漂亮的显示出偏移的十六进制和 ASCII 值。
- 视图： 最后但并非最不重要的，任何 UIView 子类都将在快速查看弹出框中显示其内容，方便极了。

### 自定义类型

实现`debugQuickLookObject`方法，返回任意一个内置类型即可，例如：

```objc
@import Foundation;

@implementation NSDictionary (QuickLook)

- (id)debugQuickLookObject {
    return self.description;
}

@end
```

## 用途

方便调试咯。。。

- 可以用在NSDictionary、NSArray、NSSet上，方便调试时复制内容
- Mantle等其它JSON转Model库
- lottie直接查看动图（不知道好不好实现）

不知道是不是我火星了，应该有挺多人不知道的吧。有空再想想再试试，为提高生产力做点贡献。


## 参考资料

- [http://nshipster.cn/quick-look-debugging/](http://nshipster.cn/quick-look-debugging/)