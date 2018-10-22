# 主流框架使用案例1：WePY

[WePY](https://github.com/Tencent/wepy) 是一款让小程序支持组件化开发的框架，通过预编译的手段让开发者可以选择自己喜欢的开发风格去开发小程序。比较显著的特点是，类 Vue 开发风格 和 小程序细节优化，如请求列队，事件优化等。

## 准备工作

官方提供了命令行脚手架来创建 WePY 的项目，这种方式极大的方便了项目构建与开发，本案例中我们将使用最新的脚手架来创建项目：

1、安装脚手架

```
sudo npm install wepy-cli -g
```

我们本地安装的版本是 v1.7.3。
<!-- 
<img src="https://user-gold-cdn.xitu.io/2018/7/3/1646013e4ccbaafe?w=180&h=46&f=jpeg&s=9874"></img> -->

2、新建项目 wepy-xiaoce-example-1

```
wepy init standard wepy-xiaoce-example-1
```

类似 vue-cli ，它支持一些模板创建，这里我们选择 standard 模版。除此之外，还选择了 ESLint 和 Redux，如下图：

![](https://user-gold-cdn.xitu.io/2018/7/5/16469bb063600d41?w=1146&h=280&f=jpeg&s=84522)


它还支持其他类型的项目模板创建，使用 `wepy list` 命令可以查看 wepy 支持的项目模板类型。

![](https://user-gold-cdn.xitu.io/2018/6/28/16446aa0248c2452?w=655&h=263&f=jpeg&s=42278)

在安装依赖之前，我们先看一下目录：

对比官方推荐的小程序目录结构：

```
├─src                    
  ├─pages
    ├─index
      ├─ index.js
      ├─ index.json
      ├─ index.wxss
      ├─ index.wxml
  ├─ app.js
  ├─ app.json
  ├─ app.wxss          
```

WePY 的开发目录结构如下所示：

```
├─src                    
  ├─components         
    ├─ list.wpy      
  ├─pages              
    ├─index.wpy
  ├─app.wpy
```

可以看到，目录下文件后缀都是 .wpy (类似于 vue)，pages 文件夹优化为类似 vue 的单文件目录，将样式、脚本和模板放在一个文件。

```
<style lang="less">
</style>

<template>
</template>

<script>
</script>
```

script 内写法如以下示例：

```
<script>
import wepy from 'wepy';

export default class Index extends wepy.page {
    config = {}
    data = {}
    components = {}
    methods = {}

    events = {}
    onLoad() {}
}
</script>
```


| 属性 | 说明 |
| ----- | ----- | ----- |
| config | 页面配置对象，对应于原生的 page.json 文件，类似于 app.wpy 中的config |
| data | 页面渲染数据对象，存放可用于页面模板绑定的渲染数据 |
| components | 页面组件列表对象，声明页面所引入的组件列表 |
| methods | wxml 事件处理函数对象，存放响应 wxml 中所捕获到的事件的函数，如 bindtap、bindchange |
| events | 存放响应组件之间通过 `$broadcast`、`$emit`、`$invoke` 所传递的事件的函数 |
| 生命周期 | 支持小程序生命周期函数 |

**注意：** methods 属性只能声明页面 wxml 标签的 bind、catch 事件，不能声明自定义方法，这与 Vue 中的用法是不一致的。

index.wpy 内通过继承 wepy.page 的类创建一个 Page 页面的实例

```
export default class Index extends wepy.page {
}
```

再来关注下 src/app.wpy 的内容：

```
<style lang="less">
</style>
<script>
import wepy from 'wepy'

export default class extends wepy.app {
  config = {
    pages: [
      'pages/index'
    ],
    window: {
      backgroundTextStyle: 'light',
      navigationBarBackgroundColor: '#fff',
      navigationBarTitleText: 'juejin xiaoce',
      navigationBarTextStyle: 'black'
    }
  }

  globalData = {}

  onLaunch() {}
}
</script>
```


可以看到，app.wpy 中所声明的小程序实例继承自 wepy.app 类，主要包含以下内容：

| 属性 | 说明 |
| ----- | ----- | ----- |
| config | 页面配置对象，对应于原生的 app.json 文件 |
| globalData | 全局数据对象 |
| 生命周期 | 小程序的生命周期函数都支持 |

## 如何使用

### 组件

src/components 用于存放组件文件，需通过继承 wepy.component 的类创建组件实例

```
<script>
import wepy from 'wepy';

export default class List extends wepy.component {
    data = {}
    methods = {}

    events = {}
    onLoad() {}
}
</script>
```

下面来看，如何引入组件，以 pages/index.wpy 中引入 list.wpy 为例：

```
import List from '../components/list'


export default class Page extends wepy.page {
    components = {list: List}
}
```

### 加载外部 npm 包

众所周知，小程序内不支持直接调用 npm 包，这给开发带来了极大的不便。而 WePY 增加了引入外部 npm 包的功能，我们可在项目中使用 `import '**'` 引入外部 npm 包

在 WePY 编译过程中，我们能够发现

<div>
  <img style="width: 400px;" src="https://user-gold-cdn.xitu.io/2018/7/2/1645b36acc9978eb?w=358&h=45&f=png&s=13612"></img>
</div>

可以看到， WePY 将对应的依赖文件从 node_modules 拷贝至 dist 目录下的 npm

而在编译后的项目文件中，require 路径被替换为相对路径，实现引入 npm 包的功能。

```js
var ** = require('./../npm/**/..');
```

### 支持 Promise

小程序原生代码中需要将成功回调作为参数传入，WePY 将一些 wx.xxx 的 API 的返回封装为 Promise，从而支持 Promise 写法：

以 wx.request 为例：
```js
// 原生写法
wx.request({
  url: '***',
  success: function (res) {
    // 请求成功回调
    callback && callback(res)
  }
});
// WePY 写法
wepy.request('***').then(res =>{
  console.log(res)
});
```

## 小结

按照上面的介绍，就可以轻松创建一个 WePY 项目。想了解更多 WePY 的设计细节，可以阅读【高级篇 - WePY 设计细节】。
