---
title: Hexo同时部署到Github和Coding并分流
date: 2017-05-10 18:00:12
tags: blog
---

最近比较闲，可以整理一下博客。。

做分流是因为，平时还是Github用的多，不想把博客的项目单独存放在Coding上，但是呢又想让国内访问快一点（是不是很矫情）

本文主要内容有：

- travis自动部署
	- 生成ssh key
	- 设置deploy key
- 绑定自定义域名
- 域名海内外分流

<!-- more -->

## 用户 Pages & 项目 Pages

每个用户只能创建一个用户Pages，每个项目只能创建一个项目Pages。
用户Pages只能从`master`分支部署，这个坑了我好久，没看仔细。。

[https://coding.net/help/doc/pages/index.html#pages--pages]()

| Pages类型 | Pages默认URL | 允许的部署来源 |
| :------- | :---------- | :----------- |
| Github 用户 Pages | `{user_name}.github.io` | `master` 分支 |
| Github 项目 Pages | `{user_name}.github.io/{project_name}` | `master`分支、`gh-pages`分支、或`master`分支中的`/docs`目录 |
| Coding 用户 Pages | `{user_name}.coding.me` | `master` 分支 |
| Coding 项目 Pages | `{user_name}.coding.me/{project_name}` | `master`分支、`coding-pages`分支、或`master`分支中的`/docs`目录 |

## 部署

因为上面的原因，我把博客源文件存放在`0x5e.github.io`的`blog`分支，编译的网页文件提交到`0x5e.github.io`和`0x5e.coding.me`的`master`分支。（建仓库就不说了）

### 本地部署

安装插件：

```
yarn add hexo-deployer-git
```

在`_config.yml`文件添加配置：

```yml
# Deployment
# Docs: https://hexo.io/docs/deployment.html
deploy:
- type: git
  repo: git@github.com:0x5e/0x5e.github.io.git
  branch: master
- type: git
  repo: git@git.coding.net:0x5e/0x5e.coding.me.git
  branch: master
```

新建一篇文章，然后执行`hexo deploy`，完成。

### Travis自动部署

#### 创建ssh key

```
$ ssh-keygen -t rsa -C "Travis CI"
```

#### 设置为项目deploy key

把刚才创建的ssh公钥加入项目中，注意：需要开启写入权限

Github: `Settings` => `Deploy keys` => `Add deploy key`
Coding: `设置` => `部署公钥` => `新建部署公钥`

#### 加密ssh私钥

```
# Install travis
sudo gem install -n -n /usr/local/bin travis

# Login
travis login --auto

# Encrypt
travis enctypt-file /path/to/id_rsa --add
```

把生成的`id_rsa.enc`保存到`.travis/id_rsa.enc`，待会用到

#### 修改配置文件

修改`.travis.yml`配置文件：

```yml
language: node_js
node_js:
  - "7"
cache:
  yarn: true
  directories:
  - node_modules
before_install:
- openssl aes-256-cbc -K $encrypted_44ad502b10c9_key -iv $encrypted_44ad502b10c9_iv -in .travis/id_rsa.enc -out ~/.ssh/id_rsa -d
- chmod 600 ~/.ssh/id_rsa
- eval $(ssh-agent)
- ssh-add ~/.ssh/id_rsa
- cp .travis/ssh_config ~/.ssh/config
- git config --global user.name '0x5e'
- git config --global user.email '0x5e@sina.cn'
install:
- yarn global add hexo-cli
- yarn
script:
- yarn run deploy  # hexo clean && hexo g -d
addons:
  ssh_known_hosts:
  - github.com
  - git.coding.net
```

`encrypted_xxx_key`，`encrypted_xxx_iv`改为对应的值，还有name & email。

`ssh_known_hosts`也需要添加，不然第一次链接ssh的时候会提示输入`yes/no`，导致无法自动构建而失败。

## 绑定自定义域名

Github: 新增一条`CNAME`记录到`{user_name}.github.io`。
Coding: 新增一条`CNAME`记录到`pages.coding.me`，然后在项目设置里添加自定义域名。

### 海内外分流

其实就是添加两条相同的主机记录，分别到Github和Coding，只是解析线路一条设置为海外，一条设置为国内（默认），如图：

![](dns-resolve.png)

就这样。。

## 参考资料

[在 GITHUB 和 CODING 上同步托管 HEXO 博客](https://munen.cc/tech/coding-pages.html)
