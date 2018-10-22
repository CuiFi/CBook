# 主流解决方案对比 - WePY 和 mpvue

WePY 和 mpvue 这两个解决方案的出现，意味着小程序开发框架本身存在一些问题，让不同类型的开发者有一定进入成本。

我们来回顾一下它们的历史：

![](https://user-gold-cdn.xitu.io/2018/7/6/1646fa3e20ae7530)

目前 WePY 在 npm 可下载的还是 **1.7.2** 版本，官方 repo 目前 **2.0.x** 分支的 package.json 也还在 **1.1.8** 版本。

下面我们会主要从体积和能力两方面来给大家介绍一下 WePY 和 mpvue 不一样的地方，最后给出大家两个解决方案选择的建议。

## 体积

相比使用小程序框架开发，WePY 和 mpvue 两种方案都需要加载自己的**依赖**代码，所以导致最终打包的  dist 目录文件中会包含依赖文件。

### mpvue 部分：

在项目根目录 package.json 中依赖定义：

```json
"dependencies": {
  "mpvue": "^1.0.11"
}
```

在 src/main.js 入口文件中导入 vue：

```js
import Vue from 'vue'
// 后面是实例化等操作
```

在 `webpack.base.conf.js` 配置文件中，我们通过 `resolve.alias` 转换了一下：

```js
module.exports = {
  resolve: {
    alias: {
      'vue': 'mpvue'
    }
  }
}
```

我们来看一下最后编译生成的目录 `dist/static`，里面有 3 个 js 文件，目录结构如下：

```
├─dist
  ├─static
     ├─js
        ├─app.js
        ├─vendor.js
        ├─manifest.js
```

* app.js 包含了业务代码

* manifest.js 包含了 webpack runtime（webpackBootstrap - install a JSONP callback for chunk loading），但是注意一下，这里是注册在 global 的：

```js
var parentJsonpFunction = global["webpackJsonp"];
global["webpackJsonp"] = function webpackJsonpCallback
(chunkIds, moreModules, executeModules) {
  // ...
}
```

* vendor.js 包含了 mpvue 等依赖的第三方包。

webpackJsonp 函数的调用如下：

```js
global.webpackJsonp()
```

一些 getApp 和 Page 定义如下：

```js
try {
  if (!global) global = {};
  global.process = global.process || {};
  global.process.env = global.process.env || {};
  global.App = global.App || App;
  global.Page = global.Page || Page;
  global.Component = global.Component || Component;
  global.getApp = global.getApp || getApp;
} catch (e) {}
```

Vue 的加载方式如下：

```js
(function (global, factory) {
   true ? module.exports = factory() :
  typeof define === 'function' && define.amd ? define(factory) :
  (global.Vue = factory());
}(this, (function () {
  // ...
})
```

### WePY 部分：

在项目根目录 package.json 中依赖定义

```json
"dependencies": {
  "wepy": "^1.6.0"
}
```

在 src/app.wpy 入口文件中导入 wepy：

```js
import wepy from 'wepy'
```

我们看一下最后编译之后的目录 `dist/npm`，目录结构如下，并详细说明了每个文件的作用。

```
├─dist
  ├─npm
     ├─wepy
        ├─lib
          ├─app.js         全局app逻辑，请求优化、promisify API、拦截器功能等 
          ├─base.js        定义了 $createApp 和 $createPage 等方法
          ├─component.js   组件逻辑，脏值检查、组件通信等
          ├─event.js       事件方法
          ├─native.js      空，代码里用于app.js中重新定义wx接口
          ├─page.js        继承component，page的一些优化
          ├─util.js        工具方法
          ├─wepy.js        入口文件
```

因为每个项目使用的方案不同，所以不能定量的去比较这两个框架体积的大小，但是，引入这些框架的确是会增加小程序代码体积的。

## 能力

我们会从 3 个维度来阐述：数据流管理、组件化和工程化。

### 数据流管理

相比传统的小程序框架，这个一直是我们作为资深开发者比较期望去解决的，在 web 开发中，随着 flux、redux、vuex 等多个数据流工具出现，我们也期望在业务复杂的小程序中使用。

#### WePY 部分

WePY 默认支持 Redux，在脚手架生成项目的时候可以内置，使用方式如下：

在脚手架示例代码中有一个 store 文件夹，目录结构如下：

```
├─src
  ├─pages
     ├─index.wpy
  ├─store
     ├─actions
        ├─counter.js
        ├─index.js
     ├─reducers
        ├─counter.js
        ├─index.js
     ├─types
        ├─counter.js
        ├─index.js
     ├─index.js
  ├─app.wpy
```

WePY 中使用了 wepy-redux，示例代码中的版本为 **1.5.9**，[传送门](https://github.com/Tencent/wepy/tree/wepy-redux%401.5.9/packages/wepy-redux)

具体方式如下：

##### 在 src/app.wpy 文件中

第一步：导入 wepy

```js
import wepy from 'wepy'
```

第二步：调用 wepy-redux 的 setStore 方法

```js
import { setStore } from 'wepy-redux'
```

第三步：调用 store/index.js 的 configStore 方法

```js
import configStore from './store'

// 执行 configStore 创建的 Store
const store = configStore()

// 调用 setStore
setStore(store)
```

我们看一下 configStore 函数的细节：调用 redux 的 createStore 函数，生成 Store，这里用到了 redux-promise，并调用了 redux 的 applyMiddleware 来扩展 redux。

```js
import { createStore, applyMiddleware } from 'redux'
import promiseMiddleware from 'redux-promise'
import rootReducer from './reducers'

export default function configStore () {
  const store = createStore(rootReducer, applyMiddleware(promiseMiddleware))
  return store
}
```

注意，我们还导入了 reducers/index.js，我们看一下具体内容：

导入了 reducers/counter.js

```js
export * from './counter'
```

具体内容如下：

导入了 redux-actions 的 handleActions 方法

```js
import { handleActions } from 'redux-actions'
```

导入了 types/counter.js，最终导出 handleActions 方法

```js
import { INCREMENT } from '../types/counter'

export default handleActions({
  [INCREMENT] (state) {
    return {
      ...state,
      num: state.num + 1
    }
  }
}, {
  num: 0,
  asyncNum: 0
})
```

我们看一下 types/counter.js 文件：

```js
export const INCREMENT = 'INCREMENT'
```

##### 在 src/pages/index.wpy 文件：

调用了 wepy-redux 的 connect，这里其他不做细节描述

```js
import { connect } from 'wepy-redux'
@connect({
  // ...
})
```

#### mpvue 部分

mpvue 作为 vue 的移植版本，当然支持 vuex，同样在脚手架生成项目的时候可以内置，使用方式如下：

在脚手架示例代码中有一个 counter 文件夹，里面包含了 store.js 文件，目录结构如下：

```
├─src
  ├─pages
     ├─counter
         ├─index.vue
         ├─main.js
         ├─store.js
```

内容如下：

1、导入 vuex：


```js
import Vue from 'vue'
import Vuex from 'vuex'
```

2、调用 Vue.use：

```js
Vue.use(Vuex)
```

3、实例化：

我们通过 new Vuex.Store 创建实例，定义了 state 和 mutations

```js
const store = new Vuex.Store({
  state: {
    count: 0
  },
  mutations: {
    increment: (state) => {
      const obj = state
      obj.count += 1
    }
  }
})
```

4、导出

```js
export default store
```

在 index.vue 文件中，我们在 template 中给 button 定义了一个事件：

```html
<button @click="increment">+</button>
```

事件中我们通过 store.commit 触发 vuex 的 increment

```js
import store from './store'
export default {
  methods: {
    increment () {
      store.commit('increment')
    }
  }
}
```

### 组件化

如果你和我们一样，经历了从无到有的小程序业务开发，建议阅读【小程序的组件化开发】章节，进行官方语法的组件库开发（从基础库 **1.6.3** 开始，官方提供了组件化解决方案）。

WePY 类似 vue 实现了单文件组件，最大的差别是文件后缀 .wpy，只是写法上会有差异，具体可以查看【主流框架使用案例 1：WePY】章节，学习起来有一定成本，不过也会很快适应：

```js
export default class Index extends wepy.page {}
```

mpvue 作为 vue 的移植版本，支持单文件组件，template、script 和 style 都在一个 .vue 文件中，和 vue 的写法类似，所以对 vue 开发熟悉的同学会比较适应。

### 工程化

我们在第一节中提到过：所有的小程序开发依赖官方提供的开发者工具。开发者工具简单直观，对调试小程序很有帮助，现在也支持腾讯云（目前我们还没有使用，但是对新的一些开发者还是有帮助的），可以申请测试报告查看小程序在真实的移动设备上运行性能和运行效果，但是它本身没有类似前端工程化中的概念和工具。

WePY 内置了构建，通过 `wepy init` 命令初始化项目，大致流程如下：

1. wepy-cli 会判断模版是在远程仓库还是在本地，如果在本地则会立即跳到第 3 步，反之继续进行。
2. 会从远程仓库下载模版，并保存到本地。
3. 询问开发者 `Project name` 等问题，依据开发者的回答，创建项目。

<!--
<img src="https://static.mobike.com/odin/static/img/mobike_vs_wepy.jpeg">
-->

mpvue 沿用了 vue 中推崇的 webpack 作为构建工具，但同时提供了一些自己的插件以及配置文件的一些修改，比如：

- 不再需要 `html-webpack-plugin`
- 基于 webpack-dev-middleware 修改成 webpack-dev-middleware-hard-disk
- 最大的变化是基于 webpack-loader 修改成 mpvue-loader

但是配置方式还是类似，分环境配置文件，最终都会编译成小程序支持的目录结构和文件后缀。

## 小结

本节主要从体积和能力两方面对比了 WePY 和 mpvue。但是在框架选择上，其实很多时候考核的点类似：社区、主创的贡献频度、开发的易用性等等，我们自己也经历了一些选择，以下给出一些建议：

1. 有 Vue.js 学习和开发经验的人  
2. 想把 Vue.js 项目改造成小程序项目的同学  
3. 对 Vue.js 特别钟爱的人

建议直接选择 mpvue。

如果你坚信原生的才是最高性能的，那建议你选择原生写法。

其他就建议使用 WePY。


## 扩展链接

* [wepy-redux](https://github.com/Tencent/wepy/tree/wepy-redux%401.5.9/packages/wepy-redux)
* [redux 之 createStore](https://cn.redux.js.org/docs/api/createStore.html)
* [redux 之 applyMiddleware](https://cn.redux.js.org/docs/api/applyMiddleware.html)
* [webpack resole 官方文档](https://webpack.js.org/configuration/resolve/)
* [awesome-mpvue](https://github.com/mpvue/awesome-mpvue)
* [awesome-wepy](https://github.com/aben1188/awesome-wepy)
