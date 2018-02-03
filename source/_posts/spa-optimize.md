---
title: SPA性能优化实践
date: 2018-02-03 14:00:00
tags: Front-End
categories: 技术相关
---

# 代码体积优化

## 代码分割（Code Splitting）

代码分割可以拆分业务模块和三方模块，在版本迭代时降低三方模块的更新频率，还可以实现资源文件按需加载，加快应用首屏加载速度。

### 业务代码拆分

在开发以及版本迭代的过程中，三方模块很少有变化，可以拆分出来，通过浏览器缓存，加快新版应用的载入速度。

<!-- more -->

1.在`webpack.config.js`定义多个entry入口：

```js
// webpack.config.js

module.exports = {

    // entry: './index.js',
    entry: {
        main: './index.js',
        vendor: [
            'react',
            'react-router',
            'moment',
            // ...
        ],
    },

    // ...

};
```

vendor可以手动指定，也可以把`package.json`里的`dependencies`全部加入：

```js
vendor: [
    ...Object.keys(path.resolve('./package.json').dependencies),
],
```

设置完以后，打包出来的`main.js`包含业务+公共代码，`vendor.js`包含公共代码。

2.使用`CommonsChunkPlugin`插件，指定vendor为公共模块

```js
// webpack.config.js

module.exports = {

    plugins: {
        // ...
        new webpack.optimize.CommonsChunkPlugin({
            names: ['vendor', 'manifest'],
        }),
    },

    // ...

};
```

manifest是webpack的运行时模块，会随业务代码而改变，需要与公共模块分离。


```
  172.6 KB (-150.45 KB)  build/static/js/main.6b091018.js
  145.54 KB              build/static/js/vendor.5e7317a1.js
  35.39 KB               build/static/css/main.6123c843.css
  835 B                  build/static/js/manifest.24abdf54.js

```

可以看到main文件拆分后，减少了将近一半的体积。`vendor.[hash].js`被浏览器缓存，因为更变不频繁，这部分代码很大概率不需要重复下载了。


### 按需加载

当SPA应用包含了多个页面的时候，首屏页面代码是必须被加载的，而其他页面并不一定会去访问，因此这部分代码并不是必须的，可以按需加载。

webpack提供了`import()`方法在运行时动态加载ES Module，返回一个Promise对象。

配合react-router可以做如下改动：

改动前：

```jsx
import React from 'react';
import { BrowserRouter as Router, Router, Switch } from 'react-router-dom';

import Task from './task';
import Mall from './mall';
import My from './my';
// ...

export default () => (
  <Router>
    <Switch>
      <Route exact path='/task' component={Task} />
      <Route exact path='/mall' component={Mall} />
      <Route exact path='/my' component={My} />
      {/* ... */}
    </Switch>
  </Router>
);

```

改动后：

```jsx
import React from 'react';
import { BrowserRouter as Router, Route, Switch } from 'react-router-dom';

const asyncComponent = (importComponent) => (
  class AsyncComponent extends React.Component {
    state = {
      component: null,
    }

    async componentDidMount() {
      const { default: component } = await importComponent();
      this.setState({ component })
    }

    render() {
      const C = this.state.component
      return C
        ? <C {...this.props} />
        : null
    }
  }
)

export default () => (
  <Router>
    <Switch>
      <Route exact path='/task' component={asyncComponent(() => import(/* webpackChunkName: 'task' */ './task'))} />
      <Route exact path='/mall' component={asyncComponent(() => import(/* webpackChunkName: 'mall' */ './mall'))} />
      <Route exact path='/my' component={asyncComponent(() => import(/* webpackChunkName: 'my' */ './my'))} />
      {/* ... */}
    </Switch>
  </Router>
);

```

将每个页面从`import Page from './path/to/page'`的同步加载改为`() => import('./path/to/page)`异步加载，加载完毕后再渲染到页面上。

webpack会对这种加载方式进行优化，每个页面的代码打包成单独的文件，在需要时按需加载。文件名称通过`import()`参数中写注释指定。例如`/* webpackChunkName: 'xxx' */`

```
  144.68 KB (-873 B)    build/static/js/vendor.dd6678e3.js
  48.2 KB               build/static/js/task.4ef57338.chunk.js
  44.74 KB              build/static/js/mall.3f31508b.chunk.js
  40.02 KB              build/static/js/my.16e54bab.chunk.js
  13.73 KB (-21.67 KB)  build/static/css/main.c8a90ac2.css
  9.05 KB (-163.55 KB)  build/static/js/main.b7f8610d.js
  1.19 KB (+383 B)      build/static/js/manifest.7d1798c7.js
  .....
```

main.js从170k变成了9k，访问`/my`页面时所需的代码文件体积从170k变成了9k+40k，减少了约70%的体积。

# 资源文件优化

图片：

- 图标用css、svg、iconfont代替
- 渐进式图片加载
- 访问oss的图片资源带上参数（比如运营同学上传了几十张超清大图。。。）
- 图片懒加载

# 服务端优化

- 启用gzip压缩（很简单，效果也很明显）

```
    ##
    # Gzip Settings
    ##
    gzip on;
    gzip_vary on;
    gzip_min_length 10k;
    gzip_proxied expired no-cache no-store private auth;
    gzip_types text/plain text/css text/xml text/javascript application/x-javascript application/xml application/javascript;
    gzip_disable "MSIE [1-6]\.";
```

- http2多路复用
- 开启HSTS
- 服务端渲染

# 参考资料

[react-router 4 代码拆分](http://blog.csdn.net/forAlienZHOU/article/details/73437057)
[基于webpack Code Splitting实现react组件的按需加载](https://zhuanlan.zhihu.com/p/26228500)
[使用import()配合webpack动态导入模块时，如何指定chunk name](https://github.com/mrdulin/blog/issues/43)
[react-router 按需加载](https://segmentfault.com/a/1190000007141049)
[webpack v3 结合 react-router v4 做 dynamic import — 按需加载（懒加载）](https://segmentfault.com/a/1190000011128817)
[Lazy Loading with React and Webpack 2](https://medium.com/front-end-hacking/lazy-loading-with-react-and-webpack-2-8e9e586cf442)
