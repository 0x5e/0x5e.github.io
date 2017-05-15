---
title: 使用 Webpack Loader 处理 Markdown 文件
date: 2017-05-15 18:01:41
tags: vue, webpack, markdown
categories: 技术相关
---

Webpack Loader 可以在预编译期间对模块、资源文件进行转换，常见的使用场景：

- CoffeeScript、TypeScript、Vue、ES6 转换为 CommonJS
- 在转换的过程中进行语法检查并提示（或者只检查，不作转换）
- CSS 转换为 JS 代码
- 图片转换为 Data URI(`data:image/png;base64,...`) 直接引用

经过各种loader的预处理，我们能够用`require('xxx-loader!./path/to/file')`的形式，将任意类型的文件引入到代码当中。

## 尝试

作为一个新手，在似懂非懂的情况下，做了各种尝试（反正我比较闲。。），最终还是觉得 Webpack Loader 符合我的需求。

目标：

- 在 Vue 项目中解析 Markdown 文件
- 处理 Markdown 的 Front Matter 头（yaml格式）
- 正确处理 Markdown 中的相对路径

<!-- more -->

### Round 1: vue-resource + front-matter + vue-markdown

1. 把`.md`放在`/static/`目录下，通过`vue-resource`的 ajax 请求获取文件内容
2. 用`front-matter`分别得到头信息（标题、时间、分类、Tag）和正文内容
3. 自行展示头信息的内容
4. 将正文内容交给`vue-markdown`组件进行渲染

```js
// 1.
let url = '/static/posts/demo.md'
this.$http.get(url).then(response => {
  console.log(response.bodyText)
}, response => {
  throw response
}

// 2.
import fm from 'front-matter'
let content = fm(text)
console.log(content.attributes) // {"title": "xxx", "date": "xxx", "tags": "xxx", ...}
console.log(content.body) // content text ...

// 4.
<vue-markdown v-bind:source="content.body" />

```

### Round 2: url-loader + vue-resource + front-matter + vue-markdown

1. `.md`文件被webpack打包js中，用`require`获取`Data URI`地址，然后仍使用`vue-resource`获取文件内容

    ```js
    let url = require('../posts/demo.md')
    this.$http.get(url).then(response => {
    // ...
    ```

2. 其余部分与之前相同


与前者的区别是，`.md`文件交由webpack管理，webpack可以根据文件的大小来决定，到底是打包到js中（节约一次http请求），还是单独保存。

```js
// webpack.base.conf.js
// ...
module.exports = {
  // ...
  module: {
    rules: [
      // ...
      {
        test: /\.md$/,
        loader: 'url-loader',
        options: {
          limit: 8192, // 体积小于8k的文件会被打包
          name: utils.assetsPath('posts/[name].[hash:7].[ext]')
        }
    ]
  }
}
```

这里碰到一个小问题：
用`url-loader`获得的`Data URI`没有设置默认编码，产生乱码。
最后通过字符串替换的方法插入了默认编码`charset=utf-8;`，解决。

```js
// 指定默认编码utf-8
// "data:image/png;base64,..." => "data:image/png;charset=utf-8;base64,..."
var url = url.replace(/;/, ';charset=utf-8;')

```

### Round 3: raw-loader + front-matter + vue-markdown

1. 直接用`require`获取`.md`文件内容，不需ajax请求

    ```js
    let text = require('../posts/demo.md')
    ```

2. 其余部分与之前相同

省去了一个体积略大，并且有点大材小用的`vue-resource`库。（后来才发现`front-matter`、`vue-markdown`大多了。。）

### Final Round 4: json-loader + front-matter-loader + vue-markdown-loader

不小心发现，编译出来的文件已经挺大了，`vendow.*.js`竟然达到了900kb。于是赶紧搜索"webpack体积优化"，找到一串分析各个模块大小的指令：

```sh
# yarn global add webpack
$ webpack --display-modules --sort-modules-by size --config "build/webpack.prod.conf.js"
```

发现`front-matter` => `js-yaml` => `esprima`占了100多kb，其他还有很多50-100kb的库没有仔细看。

再次查阅了一下 Webpack Loader 的资料，发现 loader 有一个很有用的特性：支持链式调用。

>Loader 可以通过管道方式链式调用，每个 loader 可以把资源转换成任意格式并传递给下一个 loader ，但是最后一个 loader 必须返回 JavaScript。

并且`front-matter-loader`的`README.md`中有很好的例子：

```js
// 链式调用
var exampleFrontmatter = require('json-loader!front-matter-loader!./example.md')

// 链式调用+传参
var exampleAttributes = require('json-loader!front-matter-loader?onlyAttributes!./example.md')
var exampleContent = require('raw-loader!front-matter-loader?onlyBody!./example.md')
```

就像函数调用那样，`front-matter-loader!`将`./example.md`转换成了json格式的文件：

```
---
title: 标题
date: 2017-05-15 00:00:00
---

正文内容
```

转换为：

```json
{
  attributes: {
    title: "标题",
    date: "2017-05-15 00:00:00",
  },
  body: "正文内容"
}
```

接着`json-loader!`又将这个json格式的文件，转换成模块（js对象）。于是我就可以按我想要的方式展示内容了。Nice Job！

#### Final Code

核心部分就三句：

```js
// front-matter yaml => json
let frontmatter = require('json-loader!front-matter-loader?onlyAttributes!../posts/demo.md')

// content-body => vue component
let component = require('vue-markdown-loader!front-matter-loader?onlyBody!../posts/demo.md')

// vue dynamic component load
<div :is="component"/>
```

`front-matter-loader?onlyattributes!`后面那段类似于url query，作为参数。每段loader之间用`!`分隔

（如果`vue-markdown-loader`能直接解析front-matter那就更方便了，改天提个PR）


最最最后：一个门外汉的折腾经历，前端大牛们见笑了:-)

## 参考资料

[Loader | Webpack 中文指南](http://zhaoda.net/webpack-handbook/loader.html)

[使用Loader · webpack指南](https://webpack.toobug.net/zh-cn/chapter4/using-loaders.html)

[Data URI 最佳实践](http://madscript.com/html5/datauri-best-practice/)
