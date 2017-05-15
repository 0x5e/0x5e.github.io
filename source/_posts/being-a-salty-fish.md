---
title: Being a salty fish
date: 2017-05-11 00:54:00
tags: vue.js
categories: 技术相关
---

A salty fish is now studying vue.js, so he wanna do sth to practice.
And because he is very very salty, he decide to write article in English(Chinglish).

One day, he is wondering, why not use vue.js to build a blog system?
Although there are lots of wheels implenenting the blog system, he still want to write one by himself. Why? Because very salty, of course.

__**Not finished yet **__

<!-- more -->

![](/imgs/alcohol-salty-fish.png)

## Analyze

Make an analyze first. Which feature should be kept and which feature I'd like to modify, a fully comparison between each blog systems is required.

### Hexo

https://hexo.io

- backend-free
- markdown writing
- plugin
- theme
- i18n

### Vue blog

Here are some vue blog systems above 100 stars. Mostly needs backend.

https://github.com/jcc/blog

https://github.com/Vuedo/vuedo

https://github.com/viko16/vue-ghpages-blog *backend-free*

https://github.com/elva2596/vueBlog

https://github.com/lincenying/mmf-blog-vue2

https://github.com/BUPT-HJM/vue-blog

https://github.com/jiangjiu/vue-leancloud-blog

https://github.com/myst729/Vuelog *backend-free*

### Modules

List By Module：

- Render
	- Markdown
		- Front Matter
		- Style
	- AsciiDoc maybe
- Generate
	- Home Page
	- Archives
	- Tags
	- RSS Feed
- Config
	- Site info
	- Route rules
	- Date/Time format
	- i18n
	- Pagination
	- Others
- Deploy
	- Git
	- Others
- Themes
- Plugins

## Choices

- markdown render
	- webpack loader
		- http://webpack.github.io/docs/using-loaders.html
		- https://github.com/peerigon/markdown-loader
		- https://github.com/QingWei-Li/vue-markdown-loader
	- vue component
		- https://github.com/miaolz123/vue-markdown

- vue-router mode
	- hash (markdown anchor not support)
	- history (backend required)

- vue-router rule
	- path `./posts/{name}`
	- query `./posts?name={name}`

## to learn

- webpack usage
- hexo plugin mechanism
- advantage & disadvantage of spa
- seo
- rss feed
- ssr

## Todos
 
- ~~.md encoding problem~~ `charset=utf-8;`
- img links in .md
- anchor in .md
- first load progres animation
- separate to cli, theme && site project
- plugin implementation mechanism
	- http://kyfxbl.iteye.com/blog/2237538
	- http://nodeonly.com/2015/07/07/npm-postinstall/
	- http://www.ieclipse.cn/en/2016/07/18/Web/Hexo-dev-plugin/
- i18n
- document
- test

many many many todo...
