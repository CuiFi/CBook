# 小程序项目构建

为了帮助开发者简单且高效地开发和调试小程序，微信推出了 [微信开发者工具](https://developers.weixin.qq.com/miniprogram/dev/devtools/download.html)。官方推荐使用该工具进行代码编辑，但是相比于其他成熟的编辑器，它还是有些不完美的地方。所以，现在一般采用的开发模式是：编辑器写代码，微信开发者工具做预览功能。

在第一节中，我们介绍了小程序开发的基础知识，不难发现，开发中会有如下的不便：
- 不支持 Sass/Less 这类预处理器；
- 自定义了 rpx 单位，而传统的 Web 开发者更习惯使用 px 单位；
- WXSS 中不支持相对路径的静态资源引用，只能是 https 协议开头的绝对路径。

本节介绍的项目构建将逐一解决这些开发痛点。

## 开发选型

在开始介绍项目构建之前，我们先来看一下小程序的开发流程。

<img src="https://user-gold-cdn.xitu.io/2018/6/27/164410f4a3eaeac2?w=826&h=129&f=png&s=8611">

基于上述流程，项目构建需要综合考虑两方面：一个是要遵循小程序代码体系，另一个是开发效率。

### 小程序代码体系

#### 目录结构

微信小程序的基础文件结构分为两部分，一个是主体部分，放在根目录下，有如下 3 个文件：

| 文件 | 必填 | 作用 |
| ---- | ---- | ---- |
| app.js | 是 | 小程序逻辑 |
| app.json | 是 | 小程序公共配置 |
| app.wxss | 否 | 小程序公共样式表 |

另一个是小程序页面，由 4 个文件组成，这 4 个文件必须具有相同的路径与文件名：

| 文件 | 必填 | 作用 |
| ---- | ---- | ---- |
| js | 是 | 页面逻辑 |
| wxml | 是 | 页面结构 |
| wxss | 否 | 页面样式表 |
| json | 否 | 页面配置 |

#### 包体积

最后打包上传的代码包，微信限制了包体积的大小：

- 整个小程序所有分包大小不超过 **8MB**
- 单个分包/主包大小不能超过 **2MB**

### 设计方案

基于上述的分析，最终采用如下的技术方案：

将目录分为开发目录和构建目录，构建目录分两部分：一部分是需要上传的小程序代码，另一部分是上传到 CDN 的资源。使用 gulp 编译工具处理开发代码，最终生成构建目录。

## 如何实现

先来看看开发目录结构。

### 开发目录结构

```
├─client                 开发目录
  ├─consts               常量
  ├─images               图片资源（体积较小）
  ├─qcloud               需要上传至 CDN 的资源文件
  ├─libs                 第三方 JS 库 
  ├─pages                小程序页面
    ├─components         组件
      ├─dialog           弹窗组件
    ├─index              首页
      ├─index.wxml
      ├─index.js
      ├─index.json
      ├─index.less 
    ├─login              登录页
    ├─....
  ├─utils                工具方法集
    ├─index.js
    ├─....
  ├─app.js               入口文件
  ├─app.json             配置文件
  ├─app.less             主 CSS 文件
```

### 构建目录

上传目录结构：

```
├─dist                 
  ├─consts               常量
  ├─images               图片资源（体积较小）
  ├─libs                 第三方 JS 库 
  ├─pages                小程序页面
    ├─index              首页
      ├─index.wxml
      ├─index.js
      ├─index.json
      ├─index.wxss 
    ├─....
  ├─utils                工具方法集
  ├─app.js               入口文件
  ├─app.json             配置文件
  ├─app.wxss             主 CSS 文件
```

上传目录与开发目录不同的地方在于，去除了 `qcloud` 目录，并且 `.less` 转换为 `.wxss` 文件。

上传至 CDN 的目录如下，具体生成的文件，将在本节结尾处展开。

```
├─.tmp                   
  ├─qcloud               资源文件
    ├─404-b5f7c77b8c.png
    ├─...
  ├─rev
    ├─rev-manifest.json  映射表
```

### gulp 编译处理

使用 [gulp v4.0.0](https://www.npmjs.com/package/gulp/v/4.0.0) 版本来进行构建，v4 版本比 v3 版本的优点在于，增加了两个函数 `gulp.series()` 和 `gulp.parallel()`，用于更好地处理任务之间的同步和异步关系。

#### 样式文件处理

结合摩拜前端的技术体系，我们采用 **Less** 预编译语言，结合 **PostCSS** 来处理样式文件。

1. 编译 .less 文件
2. PostCSS 处理，将 px 单位转换为 rpx 单位，可以使用我们封装的插件 [postcss-px2units](https://github.com/yingye/postcss-px2units)
3. 修改文件后缀名为 .wxss 输出

> 因为 **微信开发者工具** 提供了自动样式补全功能，所以编译时并不需要样式补全。

#### JS 文件处理

使用 ES6 语法，配合 ESLint，来检查语法错误。因为微信小程序提供了一些全局的变量和方法，所以需要在 `.eslintrc` 进行声明。

```
{
  "globals": {
    "Page": false,
    "Component": false,
    "wx": false,
    "App": false,
    "getApp": false,
    "getCurrentPages": false
  },
}
```

因为 **微信开发者工具** 提供了 **ES6 转 ES5** 功能，所以也可以不进行 Babel 编译。

#### CDN 文件处理

受上传包体积大小的限制，我们需要将一些体积较大的资源上传到 CDN。这些文件的处理分为以下三步：

1. 压缩处理，并加 md5 签名

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

[gulp-rev](https://www.npmjs.com/package/gulp-rev) 插件主要是给文件添加 md5 签名，`rev.manifest()` 方法会生成一个原文件名和加了 md5 签名的映射文件，示例如下：

```json
{
  "404.png": "404-b5f7c77b8c.png",
  "bluetooth.png": "bluetooth-fe52e58d3a.png",
  ...
}
```

2. 上传至 CDN

我们将资源文件会推送至腾讯云，基于此，我们封装了 [gulp-upload-qcloud](https://github.com/yingye/gulp-upload-qcloud)。

```js
function uploadTask () {
  return gulp.src(PATHS.tmp.qcloudFiles)
    .pipe(qcloudUpload({
      AppId: '***',
      SecretId: '***',
      SecretKey: '***',
      Bucket: '***',
      Region: '***',
      Prefix: '***/***',
      OverWrite: false
    }))
}
```

3. 替换引用路径

在开发时，通过 `/qcloud/` 来标识需要上传到 CDN 的资源。

```html
<image src="/qcloud/bluetooth.png"></image>
```

在编译时，根据 `/qcloud/` 关键字匹配内容，再根据上一步生成的映射文件替换内容，主要是利用 `gulp-rev-collector` 插件实现的。

```js
const revCollector = require('gulp-rev-collector')

function replaceQcloudTask () {
  return gulp.src([PATHS.tmp.revFiles, PATHS.dist.basicFiles])
    .pipe(revCollector({
      replaceReved: true,
      dirReplacements: {
        '/qcloud/': function (manifest_value) {
          return `${CONFIG.cdnPath}/${manifest_value}`
        }
      }
    }))
    .pipe(gulp.dest(PATHS.dist.baseDir))
}
```

## 分包加载

微信限制了小程序包体积的大小，对于超过微信 2MB 限制的小程序来说，就需要采用分包加载的方案了。同时，分包加载的方案也优化了**内容到达时间**（time-to-content，或称为首屏渲染时长）。

在小程序启动时，默认会下载主包并启动主包内页面，如果用户需要打开分包内某个页面，客户端会把对应分包下载下来，下载完成后再进行展示。所以，进入分包页面就需要一定的加载时间，并且显示的 loading 无法定制。基于这一特性，需要合理地划分主包和分包页面。

### 实现

先在 app.json 的 `subPackages` 字段中声明项目`分包结构`，示例如下：

```json
{
  "pages": [
    "pages/index",
    "pages/logs"
  ],
  "subPackages": [
    {
      "root": "subPackages/packageA",
      "pages": [
        "pages/mvp_info",
        "pages/mvp_member"
      ]
    }, {
      "root": "subPackages/packageB",
      "pages": [
        "pages/red_packet_bind",
        "pages/red_packet_capturing"
      ]
    }
  ]
}
```

这里，需要注意小程序的引用原则：

- packageA 无法使用 packageB 的资源，但可以使用 app（主包）、自己 package 内的资源

并且小程序是根据文件目录打包，只要不在同一个目录下面，都不会被打进分包。所以需要对一些类库、公共文件做好处理，以降低主包体积。

#### 历史入口兼容

一个页面放入子包中，路径会发生变化，例如 logs 页面由 `/pages/logs` 变为 `/subPackages/packageA/pages/logs`。如果用户访问了以前的页面路径（例如：分享出去的小程序页面、二维码、公众号推送消息等），就无法展示页面内容。如何兼容历史入口呢？

> 在主包中保留历史页面，添加跳转到分包的逻辑。

同时，可以在微信公众平台中统计这些页面的访问次数，对于一些`访问次数很低的页面，可以在主包中移除`。

> 分包加载案例相关代码已提交至 GitHub ，内容持续更新，可以自行下载进行本地调试，[点此访问](https://github.com/mobikeFE/xiaoce-demo)。


## 小结

本节介绍的基于 gulp 工具的项目构建方案，是摩拜单车小程序经过多次迭代的最终解决方案，适用于绝大多数使用原生小程序开发的情况。它不光解决了本节开篇提到的开发痛点，还使得开发效率大幅提升。如果采用 WePY 或者 mpvue 开发，可以阅读【主流框架使用案例】章节。
