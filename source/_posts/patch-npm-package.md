---
title: 给 npm package 打补丁
date: 2017-10-14 14:50:32
tags: node, js, npm, react-native
categories: 技术相关
---

~~用英文写了好久，实在是写不下去全删了，太蹩脚了哈哈哈哈~~

大家都知道 React Native 更新频率很快，这么久了也没有出稳定版，经常会有一些API的变动导致三方库不兼容新版本，这都是很常见的。比如0.47去除了Android的`createJSModules`方法，相对应的所有的三方库都得把这方法前面的`@Override`给去掉，不然就会编译失败了。

<!-- more -->

一般也就这么几个选项：

1. 提issue，等开发者更新代码

    如果碰上年久失修的三方库，无人维护怎么办呢，等更新要等到猴年马月，或者压根就再也不更新了。

2. fork项目，改一行，提交PR

    为了一行`@Override`，有点杀鸡用牛刀，项目依赖要先改成自己fork的项目，然后过n个礼拜，等开发者合并代码、提交npm之后，再把依赖改回来。还得把这事儿一直记着，没准就忘了呢。

3. 直接改`npm_modules`目录下的代码

    这就更不靠谱了，一般都是在`.gitignore`里面忽略掉的，多人合作怎么办，几个月之后忘记了这事儿跑不起来了又怎么办，很麻烦。

4. 不升级

    没准新版本解决了我的一个bug，我非升不可呢，多矛盾哈。

那怎么办呢？

下面有请本次嘉宾 patch-package 出场 ：）


# 使用 patch-package

## 安装

`yarn add --dev patch-package` （不用说了）

然后在`package.json`加入如下内容：

```
"scripts": {
  "prepare": "patch-package"
}
```

这样`patch-package`就会在每次npm库新增、升级、删除等操作之后，自动加上你打的补丁。

## 使用

1. 在`node_modules`目录下，对你想打补丁的三方库进行修改，比如将`node_modules/react-native-xxx/android/src/main/path/to/xxxPackage.java`中的`createJSModules`方法的`@Override`前缀删除。
2. 运行`yarn patch-package react-native-xxx`，补丁会保存在`patches/react-native-xxx+a.b.c.patch`，每次`yarn install/add/remove/...`等操作后，`patch-package`会自动为你打上补丁，直到某一天`react-native-xxx`升级以后，补丁冲突了（很大可能就是fix了），那么它的使命也就完成了

## 小结

适合一些比较小的地方的修改，改个一两行之类的，碰上react-native不想merge的分支，可能用这个不太合适。。说不定更新没几个版本就冲突了，那还是老老实实fork吧。。


# References

[[Android] Add file input to WebView](https://github.com/facebook/react-native/pull/12807)
[[Android] Add onShouldStartLoadWithRequest callback to WebView](https://github.com/facebook/react-native/pull/6478)
[Custom WebView](https://facebook.github.io/react-native/releases/next/docs/custom-webview-android.html)
[ds300/pack-package#15](https://github.com/ds300/patch-package/issues/15)
[Easier React Native upgrade with patch-package](http://blog.novanet.no/easier-react-native-upgrade-with-patch-package/)
[Building React Native from source](https://facebook.github.io/react-native/docs/android-building-from-source.html)
