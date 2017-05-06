---
title: iOS使用Fastlane多渠道自动化打包
date: 2016-04-16 22:09:04
tags: iOS
categories: 技术相关
---

## 需求

由于需求的变化，公司打算将同一份App提供给不同的厂商，名称、图标、配色等由厂商自定义，内部称作公版App，随着客户越来越多，手工修改肯定是不满足要求的，必须得自动化。

之前一直在用python打企业包上传至蒲公英，xcodebuild用着还是有点复杂的。。后来发现了fastlane，好东西呀，还有好多功能有待尝试😊

<!-- more -->

## 实现

### 前期准备

先整理了一下需要自定义的部分，打算整理成一份配置文件，在打包之前自动替换各种资源文件和参数，然后自动打包，再上传到蒲公英：

- 应用名称（多语言）
- 版本号
- Bundle Id
- URL Scheme
- FAQ、隐私政策的地址
- Copyright文案
- 配色方案（UED提供，导航栏、列表、提示、按钮等各种颜色、背景色）
- 平台AppKey、SceretKey
- 友盟AppKey
- 登录方式（手机、邮箱）、是否要支持第三方登录
- 第三方登录的各种Key
- 应用图标
- 启动图
- 其它图标

参数与Android保持统一，方便以后服务端集成两方的自动打包脚本。

因为支持多语言，应用名称保存在`./项目名/*.lproj/InfoPlist.strings`中，`CFBundleDisplayName=xxx`, 可以直接新建文本来替换。

版本号、BundleId、URL Scheme等参数可用`/usr/libexec/PlistBuddy`进行替换。

配色方案我整理在一个plist文件里，也一样替换。

应用图标可以用python的PIL库切成各种尺寸的图直接替换，启动图由于比例有好多种，暂时先让客户提供了。

剩下所有参数以宏定义的方式整理在了一个头文件中，替换之☺️

### 脚本

图片资源、配置参数替换之前用python已经写好了。fastlane主要用来打包签名上传，采用配置文件的形式显得清晰一些，可以稍微少写几行代码。

读取配置文件，转成字典：

```python
if len(sys.argv) != 2:
	print('usage: python package_ios.py [配置文件目录]')
	return

config_dir = sys.argv[1]
with open('./%s/config.json' % config_dir) as data_file:
	#过滤注释。。。。
	data = data_file.read()
	array = data.split('\n')
	data = ''
	for a in array:
		data += a.split('\t//')[0] + '\n'
	config = json.loads(data)
```

替换应用名称：

```python
f = codecs.open('./TuyaSmart/zh-Hans.lproj/InfoPlist.strings','w+b','UTF-8')
f.write('CFBundleDisplayName="%s";' % config['app_name_cn'])
f.close()
//...
```

替换配色方案：

```python
def replace_color(config):
	for (key, value) in config['colors'].items():
		os.system('/usr/libexec/PlistBuddy -c "set %s %s" ./TuyaSmart/customColor.plist' % (key, value))
```

生成各个尺寸应用图标、替换启动图：

```python
def replace_icon(config_dir):
	icon_path = './%s/res/Icon.png' % config_dir
	if not os.path.isfile(icon_path):
		return

	img_orig = Image.open(icon_path).convert('RGBA')
	
	dic = {
		'Icon-29@2x.png': (58, 58),
		'Icon-29@3x.png': (87, 87),
		'Icon-40@2x.png': (80, 80),
		'Icon-40@3x.png': (120, 120),
		'Icon-60@2x.png': (120, 120),
		'Icon-60@3x.png': (180, 180),
	}

	for (name, size) in dic.items():
		img = img_orig.resize(size, Image.ANTIALIAS)
		img.save('./TuyaSmart/Images.xcassets/AppIcon.appiconset/%s' % name)


	img_path = './TuyaSmart/Images.xcassets/AppIcon.appiconset/Icon-60@2x.png'
	if os.path.isfile(img_path):
		shutil.copy(img_path, './TuyaSmart/业务/Base/Resource/')


def replace_launch_img(config_dir):
	arr = [
		'01-login-preview-4.png',
		'01-login-preview-5s.png',
		'01-login-preview-6.png',
		'01-login-preview-6plus.png',
	]
	for name in arr:
		img_path = './%s/res/%s' % (config_dir, name)
		if os.path.isfile(img_path):
			shutil.copy(img_path, './TuyaSmart/Images.xcassets/LaunchImage.launchimage/')
```

调用fastlane打包：

```python
os.system('fastlane Enterprise bundleId:%s' % config['packageName'])
```

### Fastfile配置

```ruby
ENTERPRISE_IDENTIFIER = 'com.xxx.xxx.Hoc'
ENTERPRISE_CODESIGNING_IDENTITY = 'iPhone Distribution: xxxx'

PLIST_FILE_PATH = 'xxx/Info.plist'

uKey = 'xxxx'
_api_key = 'xxxx'

before_all do
  # ENV["SLACK_URL"] = "https://hooks.slack.com/services/..."
  cocoapods
  
end

desc "使用方法 `fastlane Enterprise bundleId:<Bundle Id>`"
lane :Enterprise do |options|

  time = Time.new
  date = "#{time.year}.#{time.month}.#{time.day}"
  badge(shield: "build-#{date}-blue")

  bundleId = options[:bundleId]
  # version = get_info_plist_value(path: PLIST_FILE_PATH, key: "CFBundleShortVersionString")
  gym(
    scheme: "TuyaSmartPublic",
    clean: true,
    silent: true,
    archive_path: "./Build/#{bundleId}/",
    output_directory: "./Build/#{bundleId}/",
    output_name: "#{bundleId}-#{date}.ipa",
    export_method: "enterprise",
    configuration: "Release",
    xcargs: "PRODUCT_BUNDLE_IDENTIFIER=\"#{ENTERPRISE_IDENTIFIER}\"",
    codesigning_identity: "#{ENTERPRISE_CODESIGNING_IDENTITY}",
  )

  sh "echo '上传至蒲公英...' curl -F 'file=@../Build/#{bundleId}/#{bundleId}-#{date}.ipa' -F 'uKey=#{uKey}' -F '_api_key=#{_api_key}' -F 'updateDescription=#{bundleId}' http://www.pgyer.com/apiv1/app/upload"

end

# You can define as many lanes as you want

after_all do |lane|
  # This block is called, only if the executed lane was successful

  # slack(
  #   message: "Successfully deployed new App Update."
  # )
end

error do |lane, exception|
  # slack(
  #   message: exception.message,
  #   success: false
  # )
end

```

用`badge`可以给企业包的应用图标加标签，方便区分，很实用

## 遇到的问题

- "鼓励一下我们"的功能，需要跳转至App Store对应的地址`https://itunes.apple.com/app/id%@`，需要客户创建应用后从苹果后台获取App Id，很麻烦。后来找到一个接口可以根据Bundle Id查询App Id `https://itunes.apple.com/lookup?bundleId=%@`，解决了这个问题。
- 蒲公英文件的分块上传，当时对python和分块上传都不熟悉，找了好多代码，后来蒲公英开发文档里建议用curl分块上传。。
- Bundle Id需要去苹果后台手动创建，目前打的都是企业包，给客户演示用的，如果公用一个Bundle Id，上传至蒲公英、fir.im都会被归到同一个地址去，没法做区分。这个暂时没时间解了，等服务端同学做出后台来吧😊

## Todo

- codesigning、provisioning做成可配置（参照APICloud）
- 用jenkins持续集成

## 参考

[iOS 持续集成之 fastlane + jenkins (持续更新)](http://www.devlizy.com/ios-da-bao-quan-cheng-pei-zhi/)

[使用fastlane实现iOS持续集成](http://everettjf.github.io/2015/09/08/ios-ci-with-fastlane)

[如何在运行时改变App的图标](http://ios.jobbole.com/82532/)
