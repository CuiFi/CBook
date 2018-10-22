# 高级篇 2：mpvue 设计细节

mpvue 是基于 Vue 的 `2.4.1` 版本进行修改的，具体参考 [commit记录](https://github.com/Meituan-Dianping/mpvue/commit/2a947d0baff4b7b1c1dfb72ad66ebcce048f6f1f) -- 确实做了很多对小程序的适配修改，本节我们会从多个方面逐一深入地探究它的实现。

其实我们可以试想几个问题：
* 入口如何获取
* 如何进行不同类似的标签转换比如 a 和 div
* 如何实现 style scoped
* moduleId 的生成依据是什么
* 如何支持 `pages` 里面带有 `^` 符号的，会自动编译成首页
* 如何把 js 和 .vue 文件编译成 json、wxml、wxss 
* 等等

> 本节内容会带着大家比较深入地研究 mpvue、mpvue-loader 以及其他插件的内部实现，我们也把源码中用到的一些知识点以扩展阅读的方式放置到文末，阅读内容较多，建议多次阅读。

## 入口文件

因为也算是多页应用，我们先看一下如何生成 `webpack` 配置中的 `entry`

首先 `src` 下的 `pages` 的目录结构如下：

```
├─pages
  ├─index
      ├─index.vue
      ├─main.js
  ├─logs
      ├─index.vue
      ├─main.js
```

整体实现设计思路如下：

通过 `glob` 包，[官网地址](https://www.npmjs.com/package/glob) 查找文件

```js
// 引入 glob 工具包
var glob = require('glob')
```

调用 `glob.sync` 去目录 `src` 里面匹配 `pages/**/main.js` 规则的文件

注意：

> 每个页面的入口文件命名是有要求的：必须是 `main.js`

```js
// 依赖核心模块 path
var path = require('path')

// 定义函数 getEntry
function getEntry (rootSrc, pattern) {
  // 调用 glob.sync
  var files = glob.sync(path.resolve(rootSrc, pattern))
  return files.reduce((res, file) => {
    // 调用 path.parse
    var info = path.parse(file)
    var key = info.dir.slice(rootSrc.length + 1) + '/' + info.name
    res[key] = path.resolve(file)
    return res
  }, {})
}

// 调用函数
const pagesEntry = getEntry(resolve('./src'), 'pages/**/main.js')
```

解析之后：

```
{ 
  entry: { 
     'pages/index/main': '/Users/zhangyaochun/***/my-project/src/pages/index/main.js',
     'pages/logs/main': '/Users/zhangyaochun/***/my-project/src/pages/logs/main.js' 
   }
}
```


### mpvue-entry

相比上面每一个页面**固定**入口文件名的方案，社区里面也有人提供了另一个方案：

> 集中式页面配置，自动生成各页面的入口文件

工具包的 git 地址：[F-loat/mpvue-entry](https://github.com/F-loat/mpvue-entry)

它也提供[脚手架](https://github.com/F-loat/mpvue-quickstart)，方便快捷地生成对应的目录，对应结构如下：

```
├─pages
  ├─index.vue
  ├─logs.vue
├─pages.js
```

我们会发现几点变化：

1、pages 下面**少了**一层各自的文件夹目录

我们看一下脚手架的模板文件结构，具体如下图所示：

![如图](https://user-gold-cdn.xitu.io/2018/7/1/16453d1e84653323?w=588&h=274&f=jpeg&s=44275)

2、重点是 pages.js 文件

我们看一下它的内容，具体如下图所示：

![如图](https://user-gold-cdn.xitu.io/2018/7/1/16453d1e85bf5f21?w=771&h=225&f=jpeg&s=39698)

随之而来的就是 `entry` 的获取方式也稍微有一些不一样：

在 `webpack.base.conf.js` 中，我们需要引入依赖的封装好的 `mpvue-entry` 工具包

```js
const MpvueEntry = require('mpvue-entry')
```

然后直接调用 `getEntry` 方法，传入 `src/pages.js` 参数

```js
module.exports = {
  entry: MpvueEntry.getEntry('src/pages.js')
}
```


## Dev 环境

一般本地开发我们都使用官方的开发者工具，mpvue 的项目和 vue 项目类似，都是本地命令行执行：`npm run dev`

唯一的区别是：我们发现会变编译出一个 `dist` 目录，结构如下：

![dist目录](https://user-gold-cdn.xitu.io/2018/7/1/16454341553b3324?w=206&h=363&f=jpeg&s=24871)

请注意：

> 本地调试项目时，用开发者工具选择**项目目录**的时候选择的是**整个项目目录**，而不是 `dist` 目录

> 同时，如果你新增了页面文件，需要**重新**执行 `npm run dev` 来进行编译。

其实在 webpack 配置文件中有一些和之前 vue.js 项目配置文件不太一样的地方：

### 生成 wxss 文件，而不是 css 文件

之前我们都是用 `extract-text-webpack-plugin` 来生成对应的 css 文件，但是小程序需要适配生成 `wxss` 文件。
配置起来差异比较小。

```js
var ExtractTextPlugin = require('extract-text-webpack-plugin')

plugins: [
  new ExtractTextPlugin({
    // filename: utils.assetsPath('css/[name].[contenthash].css')
    filename: utils.assetsPath('css/[name].wxss')
  })
]
```

### 不需要生成 html

因为在小程序里面不需要 html 文件，所以我们在移动端常用的 `html-webpack-plugin` 插件就没有用武之地了。


### 需要生成根目录的配置文件

这里的配置文件主要包含 `3` 个文件：

#### app.js

我们看一下编译之后的内容：

```js
require('./static/js/manifest')
require('./static/js/vendor')
require('./static/js/app')
```

#### app.json

我们看一下编译之后的内容：

```js
{
  "pages": [
    "pages/index/main",
    "pages/logs/main"
  ],
  "window": {
    "backgroundTextStyle": "light",
    "navigationBarBackgroundColor": "#fff",
    "navigationBarTitleText": "WeChat",
    "navigationBarTextStyle": "black"
  }
}
```

#### app.wxss

我们看一下编译之后的内容：

```css
@import "./static/css/app.wxss";
```

### 生成 wxml 文件

dist 中有一个目录 `pages`，同时还有一个 `components`， 结构如下：

```
├─components
  ├─index$9694f3d4.wxml
├─pages
  ├─index
     ├─main.js
     ├─main.wxml
     ├─main.wxss
```

main.wxml 的内容如下：

```html
<import src="../../components/index$9694f3d4" />
<template is="index$9694f3d4" data="{{ ...$root['0'], $root }}"/>
```

index$9694f3d4.wxml 的内容如下：

```html
<template name="index$9694f3d4">
  <view class="_div data-v-5eca2e54 container">
    <navigator url="/pages/counter/main" class="_a data-v-5eca2e54 counter">去往Vuex示例页面</navigator>
  </view>
</template>
```

对比一下之前 `src/pages/index/index.vue`

```html
<template>
  <div class="container">
    <a href="/pages/counter/main" class="counter">去往Vuex示例页面</a>
  </div>
</template>
```

### webpack-dev-middleware-hard-disk 插件

webpack-dev-middleware 的 fork 版本，基于 `1.12.0` 版本开始进行的修改

插件的作用和名字符合：可在 dev 时，将编译结果，保存到配置的 output path 路径中。





## mpvue-loader

基于 `vue-loader` 的 `13.0.4` 版本进行的修改，具体参考 [commit记录](https://github.com/mpvue/mpvue-loader/commit/11e672234f26e672039c7f6257683e1ae64f6385#diff-b9cfc7f2cdf78a7f4b91a753d10865a2)

内部也把之前依赖的 `vue-template-compiler` 换成了 `mpvue-template-compiler`  

为了最终按照小程序的目录结构生成对应的资源文件，需要额外处理和生成一些文件。

### dist 目录文件生成原理解析

在 `webpack.base.conf.js` 增加了如下配置，`module.rules` 允许你在 webpack 配置中指定多个 `loader`：

1. test 属性，用于标识出被 loader 进行转换的某个或某些文件。
2. use 属性，表示进行转换时，应该使用哪个 loader。

```js
module: {
  rules: [
   {
      test: /\.js$/,
      include: [resolve('src'), resolve('test')],
      use: [
        'babel-loader',
        {
          loader: 'mpvue-loader',
          options: {
            checkMPEntry: true
          }
        },
      ]
    }
  ]
}
```

下面我们会对比看一下 `src/main.js` 和  `src/pages/index/main.js` 的不同处理

在 `mpvue-loader/lib/loader.js` 中调用了 `compileMP` （在 mpvue-loader/lib/mp-compiler/index.js 文件中提供）

我们预先看一下下面涉及到的全景流程图：

![mpvue-loader](https://user-gold-cdn.xitu.io/2018/7/18/164accffac6531d2?w=784&h=861&f=jpeg&s=86315)



#### resolveTarget

我们先看一下它内部实现的流程图：

![resolveTarget](https://user-gold-cdn.xitu.io/2018/7/18/164aca2eb2bcf291?w=574&h=653&f=jpeg&s=42447)

一开始我们会调用 `resolveTarget` 来生成一个 `fileInfo`：

```js
const fileInfo = resolveTarget(resourcePath, options.entry)
```

这里的 `fileInfo` 提前打印了一下，这里的每一个 `key` 后面都会有用。

* pageType 目前共 3 种类型: component、app、page

具体如图所示:
![pageType](https://user-gold-cdn.xitu.io/2018/7/18/164aca349da4bd20?w=449&h=188&f=jpeg&s=16677)


* src 后面会被用来写入文件用到的
* isApp 是否是 name 等于 app
* isPage 是否是 pageType 等于 page

1、`src/main.js`

```js
{ 
  pageType: 'app',
  src: 'app',
  name: 'app',
  isApp: true,
  isPage: false 
}
```

2、`src/pages/index/main.js`：

```js
{ 
  pageType: 'page',
  src: 'pages/index/main',
  name: 'pages/index/main',
  isApp: false,
  isPage: true 
}
```

解析过程的源码如下：

```js
const fileInfo = resolveTarget(resourcePath, options.entry)
cacheFileInfo(resourcePath, fileInfo)
```

这里的 resourcePath

```js
**/mpvue-example/my-project/src/pages/index/main.js
```

options.entry

```js
{ 
  app: '/****/mpvue-example/my-project/src/main.js',
  'pages/index/main': '/****/mpvue-example/my-project/src/pages/index/main.js' 
}
```


resolveTarget 的源码实现如下：

```js
function resolveTarget (dir, entry) {
  const originName = getKeyFromObjByVal(entry, dir)
  const name = originName || getNameByFile(dir)
  const isApp = name === 'app'
  const pageType = isApp ? 'app' : (originName ? 'page' : 'component')
  const isPage = pageType === 'page'

  let src = 'app'
  if (isPage) {
    src = getPageSrc(name)
  }

  return { pageType, src, name, isApp, isPage }
}
```

getKeyFromObjByVal 的源码实现如下：从一个指定对象里面找对应 val 相同的那个 key 进行返回

```js
function getKeyFromObjByVal (obj, val) {
  for (const i in obj) {
    if (obj[i] === val) {
      return i
    }
  }
}
```

getNameByFile 的源码实现如下：

```js
function getNameByFile (dir) {
  // const arr = dir.match(/[pages?/components?]\/(.*?)(\/)/)
  const arr = dir.match(/pages\/(.*?)\//)
  if (arr && arr[1]) {
    return arr[1]
  }
  return path.parse(dir).name
}
```


#### cacheFileInfo 缓存

先看一下 cacheFileInfo 的源码实现：

```js
// 通过 Object.create 创建了一个对象
const pagesNameMap = Object.create(null)

function cacheFileInfo (resourcePath, ...arg) {
  // 调用 Object.assign
  pagesNameMap[resourcePath] = Object.assign({}, pagesNameMap[resourcePath], ...arg)
}
```

`src/main.js` 需要调用 clearGlobalComponents

```js
const { src, name, isApp, isPage } = fileInfo
// src/main.js 解析之后 isApp 为 true
if (isApp) {
  // 解析前将可能存在的全局组件清空
  clearGlobalComponents()
}
```

clearGlobalComponents 的源码实现：其实内部维护了一个 `globalComponents` 对象

```js
let globalComponents = {}
function clearGlobalComponents () {
  globalComponents = {}
}
```

所有 js 文件的内容 content 最终会调用 `babel.transform` 来处理。

第一步：引入 babel-core

```js
const babel = require('babel-core')
```

第二步：获取 .babelrc 的配置

```js
const babelrc = getBabelrc(mpOptioins.globalBabelrc)
```

`getBabelrc` 的源码实现如下：
如果有参数 src 而且文件存在，返回 src；其次去找 .babelrc 文件，如果存在，就返回；其他就是返回空字符串。

```js
// 引入核心模块 path
const path = require('path')
// 引入核心模块 fs
const fs = require('fs')

function getBabelrc (src) {
  // src 存在否
  if (src && fs.existsSync(src)) {
    return src
  }
  // .babelrc 存在否
  const curBabelRc = path.resolve('./.babelrc')
  if (fs.existsSync(curBabelRc)) {
    return curBabelRc
  }
  // 返回 ''
  return ''
}
```

第三步：通过 babel.transform 解析内容，生成 metadata 对象

```js
// app 入口进行全局component解析
const { metadata } = babel.transform(content, { extends: babelrc, plugins: isApp ? [parseConfig, parseGlobalComponents] : [parseConfig] })
```

我们看一下 metadata 在 2 个文件中的不同值

1、`src/main.js` 对应的：

```js
{ 
  usedHelpers: [],
  marked: [],
  modules: 
   { imports: [ [Object], [Object] ],
     exports: { exported: [], specifiers: [] } },
  importsMap: { Vue: 'vue', App: './App' },
  rootComponent: './App',
  globalComponents: {},
  config: { 
    code: '{\n  // 页面前带有 ^ 符号的，会被编译成首页，其他页面可以选填，我们会自动把 webpack entry 里面的入口页面加进去\n  pages: [\'pages/logs/main\', \'^pages/index/main\'],\n  window: {\n    backgroundTextStyle: \'light\',\n    navigationBarBackgroundColor: \'#fff\',\n    navigationBarTitleText: \'WeChat\',\n    navigationBarTextStyle: \'black\'\n  }\n}',
     node: 
      Node {
        type: 'ObjectExpression',
        start: 186,
        end: 490,
        loc: [Object],
        properties: [Array] },
      value: { pages: [Array], window: [Object] } 
  } 
}
```

2、 `src/pages/index/main.js` 对应的：

```js
{ usedHelpers: [],
  marked: [],
  modules: 
   { imports: [ [Object], [Object] ],
     exports: { exported: [], specifiers: [] } },
  importsMap: { Vue: 'vue', App: './index' },
  rootComponent: './index' }
```

处理过程：这里最核心的就是匹配 `pages` 里面带有 `^` 符号的，会被编译成首页

我们先看一下 `configObj.pages` 对应的数据，这个是在 `src/main.js` 配置的 `config.pages`

```js
['pages/logs/main', '^pages/index/main']
```

我们先全貌地看一下源码部分，后面会一步一步分析 pages 的变化，最终写入 `app.json`:


```js
const startPageReg = /^\^/

if (config) {
  const configObj = config.value
  if (isApp) {
    const pages = Object.keys(options.entry).concat(configObj.pages).filter(v => v && v !== 'app').map(getPageSrc)

    // 数组的 findIndex 方法
    const startPageIndex = pages.findIndex(v => startPageReg.test(v))
    // 带有 ^ 符号
    if (startPageIndex !== -1) {
      const startPage = pages[startPageIndex].slice(1)
      pages.splice(startPageIndex, 1)
      pages.unshift(startPage)
    }
    configObj.pages = [...new Set(pages)]
  }
  emitFile(`${src}.json`, JSON.stringify(configObj, null, '  '))
}
```

第一层处理 pages：会把入口文件和上面的 `configObj.pages`做数组合并，然后过滤掉之前入口 key 为 app 的数据：

```js
const pages = Object.keys(options.entry).concat(configObj.pages).filter(v => v && v !== 'app').map(getPageSrc)
```

这个 getPageSrc 的源码实现如下：

```js
const path = require('path')
function getPageSrc (pageName) {
  return path.parse(pageName).dir ? pageName : `pages/${pageName}/${pageName}`
}
```

变化之后的：

```js
[ 'pages/index/main', 'pages/logs/main', '^pages/index/main' ]
```

第二层处理 pages：用数组的 findIndex 方法，匹配出以 ^ 符号开头的放置到第一个

```js
const startPageReg = /^\^/
// 这里我们是数组第三个，所以 startPageIndex 为 2（从 0 开始）
const startPageIndex = pages.findIndex(v => startPageReg.test(v))
// 带有 ^ 符号
if (startPageIndex !== -1) {
  // pages[startPageIndex] ==> 取出最后一个 '^pages/index/main'
  // 然后 slice(1) 就是字符串从第二个开始，返回 'pages/index/main'
  const startPage = pages[startPageIndex].slice(1)
  // 调用 splice，第二个参数表示删除 1 个，删除的位置为 2
  // 此时的返回的 pages 是 [ 'pages/index/main', 'pages/logs/main' ]
  pages.splice(startPageIndex, 1)
  // 调用 unshift 将 startPage 放置到 pages 的开头
  pages.unshift(startPage)
}
```

变化之后的：

```js
[ 'pages/index/main', 'pages/index/main', 'pages/logs/main' ]
```

最后再调用 `new Set` 去重一次：

```js
configObj.pages = [...new Set(pages)]
```

变化之后的：

```js
[ 'pages/index/main', 'pages/logs/main' ]
```

最后调用 emitFile 写入 `app.json`，内容是 configObj


```js
emitFile(`${src}.json`, JSON.stringify(configObj, null, '  '))
```

emitFile 是 webpack loader 的一个方法，用来输出一个文件

源码内容在 webpack 3.12.0 版本 `webpack/lib/NormalModule.js` 中：

```js
emitFile: (name, content, sourceMap) => {
  this.assets[name] = this.createSourceForAsset(name, content, sourceMap);
}
```


#### 生成 入口 wxss

```js
emitFile(`${src}.wxss`, genStyle(name, isPage, src))
```

在 lib/mp-compiler/templates.js 中提供了 genStyle 方法，源码如下：

```js
function genStyle (name, isPage, src) {
  const prefix = isPage ? getPathPrefix(src) : './'
  return `@import "${prefix}static/css/${name}.wxss";`
}
```

在 lib/mp-compiler/utils.js 中提供了 getPathPrefix 方法，源码如下：

```js
function getPathPrefix (src) {
  const length = src.split('/').length - 1
  // 调用 repeat 方法
  return `${'../'.repeat(length)}`
}
```

1、src/main.js

最终写入了一个 app.wxss，里面内容就和之前讲的符合预期：

```js
@import "./static/css/app.wxss";
```

2、src/pages/index/main.js

最终写入了一个 pages/index/main.wxss，里面内容就和之前讲的符合预期：

```js
@import "../../static/css/pages/index/main.wxss";
```

#### 生成 入口 js

```js
emitFile(`${src}.js`, genScript(name, isPage, src))
```

在 lib/mp-compiler/templates.js 中提供了 genScript 方法，源码如下：

```js

function genScript (name, isPage, src) {
  const prefix = isPage ? getPathPrefix(src) : './'

  return `
require('${prefix}static/js/manifest')
require('${prefix}static/js/vendor')
require('${prefix}static/js/${name}')
`
}
```

1、src/main.js

最终写入了一个 app.js，里面内容就和之前讲的符合预期：

```js

require('./static/js/manifest')
require('./static/js/vendor')
require('./static/js/app')

```


2、src/pages/index/main.js

最终写入了一个 pages/index/main.js，里面内容就和之前讲的符合预期：

```js

require('../../static/js/manifest')
require('../../static/js/vendor')
require('../../static/js/pages/index/main')

```

### 标签转换

template 中编写的标签是浏览器端支持的标签，小程序中使用需要转换。
下面我们会以示例中的 a 标签和 div 标签的转换作为示例，具体分析如下：

#### a 链接转换

在上面的案例中，我们使用了 `a` 链接做跳转，代码如下：

```html
<a href="/pages/counter/main" class="counter">去往Vuex示例页面</a>
```

编译之后的代码如下：

```html
<navigator url="/pages/counter/main" class="_a data-v-5eca2e54 counter">
  去往Vuex示例页面
</navigator>
```

内部实现中，我们在 `mpvue-template-compiler` 中维护了一个 `tagMap` -- 标签转换的白名单对象：

```js
var tagMap = {
  //...
  'a': 'navigator'
}
```

其中 `a` 属于链接元素，同样的 `link` 也会被转换成 `navigator`

会解析成：

```
{ 
  type: 1,
  tag: 'a',
  attrsList: [ [Object] ],
  attrsMap: { href: '/pages/counter/main', class: 'counter' },
  parent: 
  { type: 1,
    tag: 'div',
    attrsList: [],
    attrsMap: [Object],
    parent: undefined,
    children: [Array],
    plain: false,
    staticClass: '"container"',
    static: true,
    staticInFor: false,
    staticRoot: true,
    staticProcessed: true },
  children: [ [Object] ],
  plain: false,
  staticClass: '"counter"',
  attrs: [ [Object] ],
  static: true 
}
```

中途通过 `mpvue-template-compiler` 提供的 `tag` 方法进行转换，关键代码如下：

```js
ast.tag = tagMap[tag] || tag;
```

对应的数据结构：

```
{ type: 1,
  tag: 'navigator',
  attrsList: [ { name: 'href', value: '/pages/counter/main' } ],
  attrsMap: { href: '/pages/counter/main', class: 'counter' },
  parent: 
   { type: 1,
     tag: 'div',
     attrsList: [],
     attrsMap: { class: 'container' },
     parent: undefined,
     children: [ [Object] ],
     plain: false,
     staticClass: '"container"',
     static: true,
     staticInFor: false,
     staticRoot: true,
     staticProcessed: true },
  children: [ { type: 3, text: '去往Vuex示例页面', static: true } ],
  plain: false,
  staticClass: '_a data-v-5eca2e54 counter',
  attrs: [ { name: 'href', value: '"/pages/counter/main"' } ],
  static: true,
  slots: {} 
}
```


#### div 转换

上面的示例中，我们 `a` 标签的父元素为 `div`，代码如下：

```html
<div class="container">
</div>
```

编译之后：转换成了小程序支持的 `view` 标签，同时 `class` 的值也变化了，代码如下：

```html
<view class="_div data-v-5eca2e54 container">
</view>
```


刚才提到的，在 `mpvue-template-compiler` 中，它维护了一个 `tag` 转换的白名单，我们看看 div 对应的：

```js
var tagMap = {
  //...
  'div': 'view'
}
```

类似 div，有很多标签都会转换成  `view`，比如：

* p
* h1-h6
* nav
* ul
* ol 
* table 

#### class 替换原理

我们发现：编译之后的元素上面的 class 多了一些内容

上例中本来的 `container` 变成了 `_div data-v-5eca2e54 container`，那如何实现的呢？

我们在 `mpvue-template-compiler` 的 `convertAst` 方法中发现了如下代码：

1、convertAst 的结构如下，接受 3 个参数：

function convertAst (node, options, util) {
  //...
}

第一个参数： `node` 的对应值如下：

```js
{ 
  type: 1,
  tag: 'div',
  attrsList: [],
  attrsMap: { class: 'container' },
  parent: undefined,
  children: [ [Object] ],
  plain: false,
  staticClass: '"container"',
  static: true,
  staticInFor: false,
  staticRoot: true,
  staticProcessed: true 
}
```

第二个参数 `options`：

```js
{ 
  components: { 
    isCompleted: true, 
    slots: [Object] 
  },
  pageType: 'component',
  name: 'index$9694f3d4',
  moduleId: 'data-v-5eca2e54' 
}
```

我们发现执行之后，如图：

![convertAst class](https://user-gold-cdn.xitu.io/2018/7/1/16455a4272c72a86?w=340&h=112&f=jpeg&s=26160)


```js
// 这里获取的 node.staticClass 就是 "container"
var staticClass = node.staticClass; 
if ( staticClass === void 0 ) staticClass = '';
```

下面是追加 moduleId 的过程：

```js
// 调用 Object.assign 复制一份 node 
var wxmlAst = Object.assign({}, node);

// moduleId ==> data-v-5eca2e54
var moduleId = options.moduleId;

// 判断
if (moduleId && !currentIsComponent && tagConfig.virtualTag.indexOf(tagName) < 0) {
  // 拼接一下变成 data-v-5eca2e54 container
  wxmlAst.staticClass = staticClass ? (moduleId + " " + staticClass).replace(/\"/g, '') : moduleId;
} else {
  wxmlAst.staticClass = staticClass.replace(/\"/g, '');
}
```

## css 方面


### px2rpx-loader

在 mpvue 的开发模式中，我们还是采用 `px` 的写法，然后通过 `px2rpx-loader` 这个 `webpack` 插件来转成 `rpx`。核心内部还是依赖 `px2rpx`，作者参考了我们在移动端常用的 `px2rem`。

前面提到过：`rpx`（responsive pixel）: 可以根据屏幕宽度进行自适应。规定屏幕宽为750rpx。
如在 iPhone6 上，屏幕宽度为375px，共有750个物理像素，则750rpx = 375px = 750物理像素，1rpx = 0.5px = 1物理像素。

那它和 `px2rem` 到底有哪些区别？

大致看了一下源码部分，主要基本都是改了一下 px 相关的函数名称变成 rpx，参考主要 [commit](https://github.com/aOrz/px2rpx/commit/c3f68e26e2605711028ce59e45a29fa9ac96f6b3)


### postcss-mpvue-wxss

### style scoped 

熟悉 vue 的同学大概了解在其单文件组件中，支持 **style scoped**，使用方式如下：

模板示例如下：

```html
<template>
  <div class="mobike-wrap">
    <div class="mobike-wrap-banner"></div>
  </div>
</template>

样式示例如下，我们在 style 标签上添加了 `scoped` 特性：

```css
<style scoped>
.mobike-wrap {
  color: #f05b48;
}
</style>
```

编译之后的模板如下所示：

```html
<div data-v-58a96838 class="mobike-wrap">
  <div data-v-58a96838 class="mobike-wrap-banner"></div>
</div>
```

对应的样式如下所示：

```css
.mobike-wrap[data-v-58a96838] {
  overflow: hidden;
}
```

原理也很直观了：

> 通过`属性选择器`，给 html 节点增加了`自定义属性`，同时也给样式增加了 `[data-v-moduleId]`  


但是，我们看一下官网中提到的小程序支持的 css 选择器，如图所示：

![小程序支持的](https://user-gold-cdn.xitu.io/2018/7/1/16453d1e8569d462)


我们发现，在小程序中不支持**属性选择器**，所以 `mpvue-loader` 做了额外的处理：

编译后的结果如下:

```html
<div  class="mobike-wrap data-v-58a96838">
  <div class="mobike-wrap-banner data-v-58a96838"></div>
</div>
```

对应的样式代码如下：

```css
.mobike-wrap.data-v-58a96838 {
  overflow: hidden;
}
```


我们深入对比一下：

mpvue-loader 的处理：`class`

```js
selector.insertAfter(node, selectorParser.className({
  value: opts.id
}))
```

vue-loader 的处理：`attribute`

```js
selector.insertAfter(node, selectorParser.attribute({
  attribute: id
}))
```

具体如图所示：

![mpvue-loader](https://user-gold-cdn.xitu.io/2018/7/1/16453d1e896c0d20)







### moduleId 生成原理

我们看一下这个 data-v-moduleId 是如何生成的

```js
// mpvue-loader/lib/loader.js
var moduleId = 'data-v-' + genId(filePath, context, options.hashKey)
```

依赖的函数，内部实现如下：

```js
// mpvue-loader/lib/utils/gen-id.js
var path = require('path')
var hash = require('hash-sum')
var cache = Object.create(null)
var sepRE = new RegExp(path.sep.replace('\\', '\\\\'), 'g')

module.exports = function genId (file, context, key) {
  var contextPath = context.split(path.sep)
  var rootId = contextPath[contextPath.length - 1]
  file = rootId + '/' + path.relative(context, file).replace(sepRE, '/') + (key || '')
  return cache[file] || (cache[file] = hash(file))
}
```




## mpvue-webpack-target 插件

它是为 webpack 专门增加的小程序的 `target`。

修改自 `webpack/lib/JsonpTemplatePlugin.js` 中关于 `target` 为 `web` 部分的源码。

1、主要兼容微信小程序中的全局变量。例如把 window 修改为 global 。
2、不支持 DOM 和 DOM 方法，不能进行热更替，注释部分功能。
3、触发 webpack 打包后的启动器。


我们看一下源码部分：

### JsonpChunkTemplatePlugin.js 文件：

如图:

![global 问题](https://user-gold-cdn.xitu.io/2018/6/29/1644aefa8d64cb56?w=957&h=618&f=jpeg&s=157343)

webpack 源码是：

```js
source.add(`${jsonpFunction}(${JSON.stringify(chunk.ids)},`);
```

mpvue-webpack-target 修改后：

```js
source.add(`global.${jsonpFunction}(${JSON.stringify(chunk.ids)},`);
```


### JsonpMainTemplate.runtime.js 文件：

这里主要去掉了 document 相关的调用，增加了 global 的支持

![global 问题](https://user-gold-cdn.xitu.io/2018/6/29/1644afb5760ad02f?w=1068&h=473&f=jpeg&s=139450)


webpack 源码是：

```js
if(parentHotUpdateCallback) parentHotUpdateCallback(chunkId, moreModules);
```

mpvue-webpack-target 修改后：

```js
if(global.parentHotUpdateCallback) global.parentHotUpdateCallback(chunkId, moreModules);
```

其他删除了：

document 也是浏览器端支持的

```js
function hotDownloadUpdateChunk(chunkId) { // eslint-disable-line no-unused-vars
  var head = document.getElementsByTagName("head")[0];
  var script = document.createElement("script");
  script.type = "text/javascript";
  script.charset = "utf-8";
  script.src = $require$.p + $hotChunkFilename$;
  head.appendChild(script);
}
```

XMLHttpRequest 是浏览器端的对象


```js
function hotDownloadManifest(requestTimeout) { // eslint-disable-line no-unused-vars
  requestTimeout = requestTimeout || 10000;
  return new Promise(function(resolve, reject) {
    if(typeof XMLHttpRequest === "undefined")
      return reject(new Error("No browser support"));
    try {
      var request = new XMLHttpRequest();
      var requestPath = $require$.p + $hotMainFilename$;
      request.open("GET", requestPath, true);
      request.timeout = requestTimeout;
      request.send(null);
    } catch(err) {
      return reject(err);
    }
    request.onreadystatechange = function() {
      if(request.readyState !== 4) return;
      if(request.status === 0) {
        // timeout
        reject(new Error("Manifest request to " + requestPath + " timed out."));
      } else if(request.status === 404) {
        // no update available
        resolve();
      } else if(request.status !== 200 && request.status !== 304) {
        // other failure
        reject(new Error("Manifest request to " + requestPath + " failed."));
      } else {
        // success
        try {
          var update = JSON.parse(request.responseText);
        } catch(e) {
          reject(e);
          return;
        }
        resolve(update);
      }
    };
  });
}
```


### JsonpMainTemplatePlugin.js 文件：

大部分还是替换了 window 对象

webpack 源码是：

```js
`var parentJsonpFunction = window[${JSON.stringify(jsonpFunction)}];`,
`window[${JSON.stringify(jsonpFunction)}] = function webpackJsonpCallback(chunkIds, moreModules, executeModules) {`,
```


mpvue-webpack-target 修改后：

```js
`var parentJsonpFunction = global[${JSON.stringify(jsonpFunction)}];`,
`global[${JSON.stringify(jsonpFunction)}] = function 
webpackJsonpCallback(chunkIds, moreModules, executeModules) {`,
```


webpack 源码是：

```js
var parentHotUpdateCallback = this[${JSON.stringify(hotUpdateFunction)}];
```


mpvue-webpack-target 修改后：

```js
var parentHotUpdateCallback = global.parentHotUpdateCallback 
= this[${JSON.stringify(hotUpdateFunction)}];
```


### webpack-mpvue-asset-plugin

#### 使用

1、加载插件

```js
var MpvuePlugin = require('webpack-mpvue-asset-plugin')
```

2、配置插件

```js
plugins: [
  new MpvuePlugin()
]
```

#### 源码解析

首先，它是一个 webpack 插件，所以写法如下：


```js
// 定义一个 MpvuePlugin 函数
function MpvuePlugin() {}

// 设置 apply 方法
MpvuePlugin.prototype.apply = function(compiler) {
  // emit 的固有写法，第一个参数 compilation
  compiler.plugin('emit', function(compilation, callback) {
    //...
  }
}
```

前面的 mpvue 脚手架示例中，我们有 2 个 `webpack.optimize.CommonsChunkPlugin` 的插件配置：

```js
plugins: [
  new webpack.optimize.CommonsChunkPlugin({
    name: 'vendor'
  })

  new webpack.optimize.CommonsChunkPlugin({
    name: 'manifest'
  })
]
```

第一步：获取插件列表

```js
const {options: {entry, plugins}} = compiler;
```

我们看一下打印出来的 `plugins`，大致和我们在 webpack config 文件配置的一样：

```js
[ 
  MpvuePlugin {},
  CommonsChunkPlugin {
    chunkNames: [ 'vendor' ]
    ...
  },
  CommonsChunkPlugin {
    chunkNames: [ 'manifest' ]
    ...
  },
  { apply: [Function: apply] },
  ExtractTextPlugin {},
  ...
]
```

第二步：循环插件，把符合条件的放置到 `commonsChunkNames` 数组中

```js
let commonsChunkNames = [];
// 获取所有的 chunk name
plugins.forEach(item => {
  let { chunkNames } = item;
  // 这里主要是配置 CommonsChunkPlugin 的插件
  if (item.constructor.name === 'CommonsChunkPlugin' && chunkNames) {
    commonsChunkNames = commonsChunkNames.concat(chunkNames);
  }
})
```

这里我们取到的 commonsChunkNames 如下：

```
[ 'vendor', 'manifest' ]
```


下面，会循环 chunks，第一步先获取如下数据：
这里重点关注： `files`、`chunks`、`name`


```js
compilation.chunks.forEach(commonChunk => {
  // 第一步
  const { files, chunks: childChunks, name } = commonChunk;
});
```

我们看一下 compilation.chunks 这个数组，由于内容过多，简单抽取了 `manifest` 部分


```
{
  id: 3,
  ids: [ 3 ],
  name: 'manifest',
  chunks: [ [Object] ],
  parents: [],
  files: [ 'static/js/manifest.js', 'static/js/manifest.js.map' ],
  hash: '1df833e7aeedf3de231229601440193b',
  renderedHash: '1df833e7aeedf3de2312' 
} 
```

我们再来看一个复杂的 `app`：

```
{
  id: 2,
  ids: [ 2 ],
  name: 'app',
  chunks: [],
  parents: [ [Object] ],
  files: 
   [ 'static/js/app.js',
     'static/css/app.wxss',
     'static/js/app.js.map' ],
  entryModule: 
   NormalModule {
     dependencies: [Array],
     context: '/****/mpvue-example/my-project/src',
     id: 2,
     portableId: 'node_modules/babel-loader/lib/index.js!node_modules/mpvue-loader/index.js??ref--2-1!node_modules/eslint-loader/index.js??ref--0!src/main.js',
     index: 0,
     index2: 10,
     depth: 0,
     request: '/****/mpvue-example/my-project/node_modules/babel-loader/lib/index.js!/****/mpvue-example/my-project/node_modules/mpvue-loader/index.js??ref--2-1!/****/mpvue-example/my-project/node_modules/eslint-loader/index.js??ref--0!/****/mpvue-example/my-project/src/main.js',
     userRequest: '/****/mpvue-example/my-project/src/main.js',
     rawRequest: '/****/mpvue-example/my-project/src/main.js',
     resource: '/****/mpvue-example/my-project/src/main.js',
     loaders: [Array],
     fileDependencies: [Array],
     _source: [Object],
     assets: [Object],
     _cachedSource: [Object],
     useSourceMap: true,
     buildTimestamp: 1531620289815,
     cacheable: true,
     exportsArgument: '__webpack_exports__' },
  hash: 'cdfeefb81ca5782ac3031e5f75b29cac',
  renderedHash: 'cdfeefb81ca5782ac303' 
}
```

第二步：判断文件后缀是否是 .wxss

```js
let commonWxssFile = files.find(item => item.endsWith('.wxss'));
```

第三步：如果同时在 commonsChunkNames 数组里面，而且文件后缀是 wxss

```js
if (commonsChunkNames.indexOf(name) > -1 && commonWxssFile) {
  // 循环 childChunks
  childChunks.forEach(item => {
    // 是否以 wxss 结尾
    let wxssFile = item.files.find(item => item.endsWith('.wxss'));
    // 过滤 app
    if (item.name === 'app' && wxssFile) {
      return;
    }
  })
}
```

关键操作：会做一次内容替换，追加上 `@import`

```js
try {
  if (compilation.assets[wxssFile]) {
    // 通过 source 获取到源码内容
    let wxss = compilation.assets[wxssFile].source();
    // 拼接上前面的 @import
    wxss = `@import "/${commonWxssFile}";\n${wxss}`;
    // 再覆盖回去
    compilation.assets[wxssFile].source = () => wxss;
  }
} catch (error) {
  console.error(error, wxssFile)
}
```



### 扩展链接：

* [webpack loader](https://www.webpackjs.com/concepts/loaders/)
* [Array.prototype.unshift](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/unshift)
* [Array.prototype.splice](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/splice)
* [Array.prototype.findIndex](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/findIndex)
* [Object.assign](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/assign)
* [extract-text-webpack-plugin](https://www.npmjs.com/package/extract-text-webpack-plugin)
