# 摩拜单车小程序技术演变

随着小程序技术体系的更新，我们的小程序也在不断升级。本节将分享几个摩拜单车小程序技术演变的案例，介绍我们如何从最简单的小程序，发展到现在拥有组件化、构建和发布系统的复杂小程序。

## 组件化

复杂小程序中，组件化是一种共识。比如摩拜单车第一个急需组件化功能的需求，是在 2017 年 3 月发布“免费骑行 20 天”运营活动：

![app](https://user-gold-cdn.xitu.io/2018/6/29/1644b88f9bf1b2d7?w=800&h=648&f=jpeg&s=91815)

可以看出，不同的页面有着类似的登录框，使用组件化书写再合适不过了。鉴于当时微信官方还没有组件化解决方案，我们基于 WXML 提供的 [template](https://developers.weixin.qq.com/miniprogram/dev/framework/view/wxml/template.html) 功能，封装了一套简易的“组件工具” —— [wx-component](https://github.com/binnng/wx-component)。原理就是 WXML 使用 `template` 片段，再一步步 merge 组件和 Page 的 `data` 和 `method`，最终实现简易的组件化功能。

有必要提到的一个细节是，wx-component 开发之初，我们咨询了微信正在开发组件化功能的同学，要到了 API 结构，并尽量按照这个结构去实现简易的“组件工具”，以使后期升级的时候改动最小。

从小程序基础库版本 **1.6.3** 开始，小程序支持了[组件化编程](https://developers.weixin.qq.com/miniprogram/dev/framework/custom-component/)，我们很快对原有的“组件”进行了改造升级，重新支持了微信官方组件化方案，原有的方案随即废弃。

更多关于组件化的描述请见【小程序的组件化开发】章节。

## 构建系统

熟悉了小程序开发语言后，慢慢发现其不方便之处，比如不支持 Less 等预编译语言，无法对 JS 压缩进行高级定制，也无法进行 ESLint 等脚本检查，所以我们急需一套类似 Web 开发的构建系统。

综合考虑实现成本和团队成员熟悉程度，最终选择使用 `gulp` 作为构建工具，我们使用 `client` 目录作为开发目录，经过构建工具生成的 `dist` 目录作为小程序发布目录。

构建前：
```
├─client
  ├─pages
    ├─index
      ├─index.wxml
      ├─index.less
      ├─index.js
      ├─index.json
    ├─...
```

构建后：
```
├─dist
  ├─pages
    ├─index
      ├─index.wxml
      ├─index.wxss
      ├─index.js
      ├─index.json
```

更多关于构建系统的描述请见【小程序项目构建】章节。

## 跨页面通信

如何在小程序多个页面间调用方法传递数据？

### 使用 `getApp()`

[getApp()](https://developers.weixin.qq.com/miniprogram/dev/framework/app-service/app.html) 可以用来获取到小程序实例的全局函数。

```js
// page A
Page({
  data: {},
  onLoad() {
    // 设置到 `getApp()` 中
    getApp().showRiding = this.showRiding.bind(this)
  }
  showRiding() {
    ...
  }
})

// page B
Page({
  data: {},
  onUnlockSuccess() {
    ...
    getApp().showRiding()
  }
})
```

最后 `getApp()` 被挂满各个页面的方法，以便被互相调用。

### 使用 `getCurrentPages()` 查找对应页面对象

[getCurrentPages()](https://developers.weixin.qq.com/miniprogram/dev/framework/app-service/route.html) 函数用于获取当前页面栈的实例，以数组形式按栈的顺序给出，第一个元素为首页，最后一个元素为当前页面。

```js
// page A
Page({
  data: {},
  showRiding() {
    ...
  }
})

// page B
Page({
  data: {},
  onUnlockSuccess() {
    ...
    const pageA = getCurrentPages().filter((page) => {
      return page.route == 'pages/A/index'
    })[0]

    if (pageA) {
      pageA.showRiding()
    }
  }
})
```

但你要准确知道每个方法在哪个页面，且每次都需要获取到对应的页面对象，显得十分繁琐。

### broadcast.js

最终我们使用了自行封装的 `broadcast.js`，作为统一的事件广播触发机制：

```js
var broadcast = {
  on: function(name, fn) {
    var data = broadcast.data
    if (data.hasOwnProperty(name)) {
      data[name].push(fn)
    } else {
      data[name] = [fn]
    }
    return this
  },
  fire: function(name, data, thisArg) {
    var fnList = broadcast.data[name] || []
    for (i = 0, len = fnList.length; i < len; i++) {
      fnList[i].apply(thisArg || null, [data, name])
    }
    return this
  },
  data: {}
}
```

上述需求的代码就可以精简为：

```js
// page A
Page({
  data: {},
  onLoad() {
    broadcast.on('showRiding', this.showRiding.bind(this))
  }
  showRiding() {
    ...
  }
})

// page B
Page({
  data: {},
  onUnlockSuccess() {
    broadcast.fire('showRiding')
  }
})
```

既精简了代码，也避免侵入 `getApp()` 对象。

> 适当的时候，注册广播时需要绑定 `this` 值。

## web-view

<div style="text-align: center; margin-top: 20px">
  <img width="300" src="https://user-gold-cdn.xitu.io/2018/7/4/16464c4e235178e1?w=400&h=385&f=jpeg&s=98386">
</div>

通常我们会遇到一些类似用户协议的需求，其特点是经常变更。由于早期小程序还不支持内嵌 H5 页面，只能使用小程序原生页面去实现，内容发生变更时还需要提审发版，周期较长。支持内嵌 H5 页面后，小程序无须发版，H5 修改后即可快速上线。

从基础库 **1.6.4** 开始，小程序支持了 [web-view](https://developers.weixin.qq.com/miniprogram/dev/component/web-view.html) 组件，我们也快速进行了升级，为此封装了跳转到 H5 页面的通用方法：

```js
// 跳转到 H5 页面
function gotoH5(url) {
  wx.navigateTo({
    url: `pages/web/index?url=${url}`
  })
}
```

其中，`pages/web/index` 是通用的 `web` 容器页面：

```html
<view>
  <web-view src="{{url}}"></web-view>
</view>
```

随着业务复杂度增加，H5 页面需要小程序传递的参数值越来越多，我们完善了 `gotoH5` 方法：

```js
function gotoH5(url, params) {
  ...
  const common = `token=${token}&location=${location}&__mbk_xcx__=1`
  if (params) {
    params = `${params}&common`
  } else {
    params = common
  }

  // `url` 中是否已经有参数
  url = url + ( url.indexOf('?') > -1 ? '&' : '?' ) + params
  wx.navigateTo({
    url: `pages/web/index?url=${url}`
  })
}
```

更多关于 `web-view` 的描述请见【小程序的跨端能力】章节。

## CDN

小程序在发布分包加载功能之前，包大小一直被限制在 **2MB** 以内，随着小程序功能越来越丰富，体积也随之变大，眼看就要超过体积限制。除了对资源压缩优化，我们需要将一些图片资源远程化。我们在构建配置中，增加了一项：

```js
const rev = require('gulp-rev')
const gulpLoadPlugins = require('gulp-load-plugins')

const plugins = gulpLoadPlugins()

function compileQcloud () {
  return gulp.src(PATHS.src.qcloudFiles)
    .pipe(plugins.newer(PATHS.tmp.baseDir))
    // 图片压缩
    .pipe(plugins.if(CONFIG.isProduction, plugins.imagemin()))
    // 加md5签名
    .pipe(rev())
    .pipe(gulp.dest(PATHS.tmp.qcloudDir))
    // 生成映射文件
    .pipe(rev.manifest())
    .pipe(gulp.dest(PATHS.tmp.revDir))
}
```

每次发布小程序时，都会将特定路径下的图片文件上传至 `CDN` 中。

在开发过程中，图片资源依旧保持本地化，以便快速更新预览，但一旦到测试或者线上环境，特定的图片资源会被上传到 CDN ，并自动替换源码中的引用路径。自此，摩拜单车小程序一直未突破 **2MB** 体积限制，当然，现在也可以使用[分包加载](https://developers.weixin.qq.com/miniprogram/dev/framework/subpackages.html)方案，整包大小最大支持 **8MB**。

更多关于上传 `CDN` 构建流程的描述请见【小程序项目构建】章节。

## 小结

> 本章案例相关代码已提交至 github ，内容持续更新，可以自行下载进行本地调试，[点此访问](https://github.com/mobikeFE/xiaoce-demo)

本节以摩拜单车小程序为例，介绍了如何对一个复杂小程序进行一步步的技术演变，伴随着微信 API 不断更新优化，又如何将其打造成敏捷高效、不断技术升级的小程序。