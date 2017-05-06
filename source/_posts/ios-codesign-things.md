---
title: iOS代码签名与重签名
date: 2016-09-25 22:59:56
tags: iOS
categories: 技术相关
---

作为一个iOS开发者，与代码签名或设备配置文件打交道是不可避免的。

一个完好的iOS系统会验证App的签名、设备配置文件（Provisioning）、授权机制（Entitlements），来决定这个应用是否允许被安装、运行。

## 证书和密钥

在我们的开发机器上保存着私钥和对应的证书，证书是苹果认证部门进行签名过的、确保信息准确无误的。经过私钥签名后，iOS系统在安装的时候可以验证这个应用是否被第三方篡改过。
Mach-O二进制文件的签名保存在文件内部，其它资源文件的签名则保存在`Example.ipa => Payload/Example.app/_CodeSignatue/CodeResources`文件中，以plist的形式。

将证书导入到Keychain中，然后就可以用`codesign`命令行工具签名了

```
# 签名
$ codesign --force --sign 'iPhone Distribution: Hangzhou XXX Technology Co., LTD' Example.app

# 验证
$ codesign --verify --verbose Example.app
```

<!-- more -->

## 授权机制（Entitlements）

Entitlements 算是一个配置列表，列出了这个应用一些基本信息（Bundle Id，Team Id, 开发/生产环境等）和申请的权限（推送，iCloud，调试等）。与 Xcode 的 Capabilities 选项卡对应。

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
        <key>application-identifier</key>
        <string>7TPNXN7G6K.ch.kollba.example</string>
        <key>aps-environment</key>
        <string>development</string>
        <key>com.apple.developer.team-identifier</key>
        <string>7TPNXN7G6K</string>
        <key>com.apple.developer.ubiquity-container-identifiers</key>
        <array>
                <string>7TPNXN7G6K.ch.kollba.example</string>
        </array>
        <key>com.apple.developer.ubiquity-kvstore-identifier</key>
        <string>7TPNXN7G6K.ch.kollba.example</string>
        <key>com.apple.security.application-groups</key>
        <array>
                <string>group.ch.kollba.example</string>
        </array>
        <key>get-task-allow</key>
        <true/>
</dict>
</plist>
```

在配置文件（Provisioning）中也包含一个 Entitlements，两者要对应上才行，否则安装不了。配置文件是苹果开发者后台下载的，也是经过签名无法篡改的。这样的话，开发者就只能申请部分权限，不能越权操作了:)

## 配置文件（Provisioning）

配置文件是经过苹果签名的、在苹果开发者平台生成的、经过CMS(Cryptographic Message Syntax) 加密的plist文件。它把签名、授权和沙盒联系到了一起。

配置文件是`.mobileprovision`格式的，被Xcode统一保存在`~/Library/MobileDevice/Provisioning Profiles/`目录下，文件名为Provisioning Profiles 的 UUID。

可以用OpenSSL或者macOS下的`security`工具进行解码：

```
# macOS
$ security cms -D -i example.mobileprovision

$ openssl smime -inform der -verify -noverify -in example.mobileprovision
```

输出内容如下：

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>AppIDName</key>
	<string>XC Wildcard</string>
	<key>ApplicationIdentifierPrefix</key>
	<array>
	<string>P4PX4Q7QS3</string>
	</array>
	<key>CreationDate</key>
	<date>2016-09-21T02:57:17Z</date>
	<key>Platform</key>
	<array>
		<string>iOS</string>
	</array>
	<key>DeveloperCertificates</key>
	<array>
		<data>.....(base64 format)</data>
	</array>
	<key>Entitlements</key>
	<dict>
		<key>keychain-access-groups</key>
		<array>
			<string>P4PX4Q7QS3.*</string>
		</array>
		<key>get-task-allow</key>
		<false/>
		<key>application-identifier</key>
		<string>P4PX4Q7QS3.*</string>
		<key>com.apple.developer.team-identifier</key>
		<string>P4PX4Q7QS3</string>
	</dict>
	<key>ExpirationDate</key>
	<date>2017-09-21T02:57:17Z</date>
	<key>Name</key>
	<string>universal_production</string>
	<key>ProvisionsAllDevices</key>
	<true/>
	<key>TeamIdentifier</key>
	<array>
		<string>P4PX4Q7QS3</string>
	</array>
	<key>TeamName</key>
	<string>Hangzhou XXX Technology Co., LTD</string>
	<key>TimeToLive</key>
	<integer>365</integer>
	<key>UUID</key>
	<string>8822a242-ea6f-4392-a00a-e8669cb90f43</string>
	<key>Version</key>
	<integer>1</integer>
</dict>
</plist>
```

里面包含了很多信息，交给iOS逐项确认后，应用才会被安装、运行。这也是为什么未越狱的iOS系统如此安全的原因。从开发者账号、证书、权限一步步下来，都需要经过苹果逐项授权，弄不得半点假。

**相关链接：**

[代码签名探析](https://objccn.io/issue-17-2/)（原版：[Inside Code Signing](https://www.objc.io/issues/17-security/inside-code-signing/)）

[iOS证书及描述文件制作流程](http://docs.apicloud.com/Dev-Guide/iOS-License-Application-Guidance)


## Xcode的签名管理

Xcode 8 之前，签名是半自动管理的，编译时须指定参数：

```
CODE_SIGN_IDENTITY = "iPhone Distribution: Hangzhou XXX Technology Co., LTD"
PROVISIONING_PROFILE = "8822a242-ea6f-4392-a00a-e8669cb90f43"
```
其中`PROVISIONING_PROFILE`的值为 Provisioning Profile UUID（可选），如果为空，xcodebuild会在`~/Library/MobileDevice/Provisioning Profiles/`目录下自动匹配，找不到的话签名就失败了。


Xcode 8新增了自动签名管理，相应的，之前的半自动管理木有了，变成了全手动管理：

```
# 自动签名管理
CODE_SIGN_IDENTITY = "iPhone Distribution"
DEVELOPMENT_TEAM = P4PX4Q7QS3

# 手动签名管理
CODE_SIGN_IDENTITY = "iPhone Distribution: Hangzhou XXX Technology Co., LTD"
PROVISIONING_PROFILE_SPECIFIER = "P4PX4Q7QS3/universal_production"
```

自动签名管理：`CODE_SIGN_IDENTITY`不再需要指定具体的证书名称；`PROVISIONING_PROFILE`与`PROVISIONING_PROFILE_SPECIFIER`将不起作用。比较坑的一点是，如果 provisioning profile 指定的 bundle id 带有通配符，那么它将无法在自动签名管理下使用。。。

手动签名管理：`PROVISIONING_PROFILE_SPECIFIER`的值为 "Team Id" + "/" + "Provisioning Profile Name"，必选项，不再可选了。。。

**相关链接：**

[Migrating Code Signing Configurations to Xcode 8](https://pewpewthespells.com/blog/migrating_code_signing.html)

[Use xcodebuild (Xcode 8) and automatic signing in CI (Travis/Jenkins) environments - StackOverflow](http://stackoverflow.com/questions/39500634/use-xcodebuild-xcode-8-and-automatic-signing-in-ci-travis-jenkins-environmen)


## 重签名

由于业务需要，我们专门部署了一台内网的mac服务器用于自动打包，重签名。并且提供给产品同事一个后台，用于填写打包的素材，图标，名称，包名，证书之类。
Xcode 8的自动签名管理给我带来些麻烦，自动签名的使用前提是：登录苹果开发者账号。这一步未必能做到自动化，并且客户的账号密码理论上是不会，也不应该提供给我们的，尤其国外客户对这个看的比较重要。所以我们需要客户提供的是`.mobileprovision`和`.p12`两个文件。

之前每次给客户打包都是拉代码，替换完素材然后用xcodebuild重新编译+签名。最近才对代码签名有了大概的了解，其实xcodebuild也是调用codesign进行签名的，那么每次编译其实都是浪费时间，只要每个版本保存一份`.ipa`文件，替换完素材重新签名就可以了。

`.ipa`本质是一个zip压缩包，解压出来大概是这样的目录结构：

- Payload
	- Example.app
		- _CodeSignature
			- CodeResources
		- Info.plist
		- embedded.mobileprovision
		- archived-expanded-entitlements.xcent
		- Example（Mach-O二进制文件）
		- ...
		- 资源文件

`CodeResources`保存了所有文件的签名，`Payload/Example.app/Example`是主程序，Entitlements包含在其中。
`archived-expanded-entitlements.xcent`也是 Entitlements，只是这个文件是方便人们排查问题用的，改它应该是没什么用。
这里的`Info.plist`是压缩后的binary plist，可以用`plutil`或者python 的 biplist 库解析。
`embedded.mobileprovision`就是前面说的配置文件。

需要注意的是，Bundle Id的更改（Info.plist）以及配置文件的替换，需要自己完成，`codesign`不负责这些。。

```
# 重签名
$ codesign --force \
	--identifier com.tuya.example \
	--sign 'iPhone Distribution: Hangzhou XXX Technology Co., LTD' \
	--entitlements entitlements.plist \
	--verbose \
	Payload/Example.app

# 验证
$ codesign --verify \
	--verbose \
	Payload/Example.app

# 打包
$ zip -yr Example.resigned.ipa Payload
```

`codesign`会从 Keychain中获取`--sign`指定的证书和密钥对应用重新签名，同时将Entitlements写入到Mach-O文件中，最后更新`CodeResources`签名。

纯手动签名管理:-)

**相关链接：**

[Digitally resigning IPA](https://sholtz9421.wordpress.com/2012/06/08/digitally-resigning-ipa/)

[Resign an iPhone App, insert new Bundle ID and send to Xcode Organizer for Upload](http://www.ketzler.de/2011/01/resign-an-iphone-app-insert-new-bundle-id-and-send-to-xcode-organizer-for-upload/)

### 改进

因为`codesign`是macOS平台才有的命令行工具，所以只能部署在内网mac服务器上，采用轮询的方式从后台（部署在阿里云）获取打包配置信息，效率比较地下而且很容易出问题，比如网络不好啦，比如电源插头给同事拔去充电啦。。。如果能找到linux下的解决方案就好了。

找到几个不同方案，比较有意思的是，这三种的思路完全不同。第一个是python实现的，跨平台的iOS签名工具，第二个是linux下的darwin模拟层（是这么翻译的不），第三个是在linux下跑一个macOS虚拟机。

并没有太多时间去试，稍微试了一下，好像是不可用，有空再看看:-)

[saucelabs/isign](https://github.com/saucelabs/isign): Code sign iOS applications, without proprietary Apple software or hardware
[darlinghq/darling
](https://github.com/darlinghq/darling): Darwin/macOS emulation layer for Linux http://www.darlinghq.org
[kholia/OSX-KVM](https://github.com/kholia/OSX-KVM): Running Mac OS X El Capitan and macOS Sierra on QEMU/KVM
