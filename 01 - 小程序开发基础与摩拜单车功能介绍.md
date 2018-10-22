# 小程序开发基础与摩拜单车功能介绍

2017 年 1 月 9 日小程序正式上线，这一年，小程序狂揽 4 亿用户、1.7 亿的日常活跃，上线 58 万个。这是一个巨大的机会，对于企业宣传、拉新用户存在变革性的影响。摩拜单车从 2016 年 10 月拿到小程序平台内测资格，作为第一批上线的微信小程序，活跃度一直保持在前十，本教程主要分享这两年我们在小程序上的开发经验。

首先我们对比小程序开发与 Web 开发的区别，简单介绍下小程序的开发基础。

> 如果你已经是一名熟练的小程序开发者，可以略过本节的小程序开发基础部分；如果你还没有任何小程序开发经验，建议仔细阅读开发基础部分，我们挑选了部分我们认为比较重要的内容，方便你快速入门。

## 小程序开发基础

### 准备工作

注册相关的流程这里就不详细叙述了，可以查看[小程序官网](https://developers.weixin.qq.com/miniprogram/introduction/index.html?t=2018410)，对于开发者，需要先安装开发者工具，下载请点击[此处](https://developers.weixin.qq.com/miniprogram/dev/devtools/download.html)。如果你是 Mac 用户，请直接点击[传送门](https://servicewechat.com/wxa-dev-logic/download_redirect?type=darwin&from=mpwiki)。

开发者工具提供一个配置文件 `project.config.json`，可以在不同的设备中同步开发者工具的配置。官方介绍如下：

> 小程序开发者工具在每个项目的根目录都会生成一个 project.config.json，你在工具上做的任何配置都会写入到这个文件，当你重新安装工具或者换电脑工作时，你只要载入同一个项目的代码包，开发者工具就自动会帮你恢复到当时你开发项目时的个性化配置，其中会包括编辑器的颜色、代码上传时自动压缩等等一系列选项。

更多配置项细节可以参考官方文档 [开发者工具的配置](https://developers.weixin.qq.com/miniprogram/dev/devtools/edit.html#%E9%A1%B9%E7%9B%AE%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6)。

### 开发语言

小程序采用 WXML + WXSS + JS 三种开发语言组合，其和网页编程采用的 HTML + CSS + JS 类似，WXML 用来描述当前这个页面的结构，WXSS 用来描述页面的样式，JS 用来处理这个页面和用户的交互。

#### WXML

WXML（WeXin Markup Language）和 HTML 类似，也有标签和属性，但针对小程序平台做了些优化。

相较 HTML，小程序的标签显得更加简洁，比如 `div`、`section` 、`header`等块级标签统一为 `view`，`p`、`span`、`b` 等文案类标签统一为 `text`，同时也新增了很多实用标签，比如 `picker` 滚动选择器、`map` 地图、`web-view` 网页容器等。

可以简单理解为，小程序所有的标签都是[原生组件](https://developers.weixin.qq.com/miniprogram/dev/component/)。

WXML 支持数据绑定：

```html
<view> {{message}} </view>
```

在 JS 中定义 `message` 变量：

```js
Page({
  data: {
    message: 'Hello MOBIKE!'
  }
})
```

更多关于 WXML 的介绍请见 [WXML · 小程序](https://developers.weixin.qq.com/miniprogram/dev/framework/view/wxml/)。

#### WXSS

WXSS（WeXin Style Sheets）是微信定义的一套样式语言，其具有 CSS 大部分特性，同时为了更适合开发微信小程序，WXSS 对 CSS 进行了扩充和修改。

小程序使用 rpx（responsive pixel）作为尺寸单位。屏幕宽度固定为 750rpx，设置了 rpx 单位的元素可以根据屏幕宽度进行自适应，所以设计稿统一以 750px 输出（iPhone 6 标准）。

小程序没有 `html` 、`body`标签，如果想要设置页面的样式，可以直接使用 `page` 选择器：

```css
page {
  background: #FFFFFF;
}
```

WXSS 对选择器的支持并没有 Web 那么丰富，比如属性选择器 `[attr]`、相邻选择器 `h1 + p` 等都不被支持，只支持以下几种：

![wxss](https://user-gold-cdn.xitu.io/2018/7/1/16453d1e8569d462)

同时，我们也可以在 WXML 中写内联样式：

```html
<view style="color: #FF9900;" />
```

也支持变量：

```html
<view style="color: {{color}};" />
```

更多关于 WXSS 的介绍请见 [WXSS · 小程序](https://developers.weixin.qq.com/miniprogram/dev/framework/view/wxss.html)。

#### JS

小程序 JS 中没有 `window`、`document` 等变量，大部分浏览器中全局方法会被禁用，比如 `alert`。但也有部分被支持，比如 `setTimeout`、`encodeURIComponent`等，具体可以在开发者工具中尝试使用，官方文档并没有详细的介绍。

小程序 JS 添加了“全局”的 `wx` 命名空间，其挂载了很多实用的函数方法，比如 `wx.request` 用来发送网络请求，`wx.setStorage` 用来本地存储，`wx.getLocation` 用来获取用户位置信息等。

更多 `wx` 提供的方法请见 [API · 小程序](https://developers.weixin.qq.com/miniprogram/dev/api/)。

### 生命周期

和大多数前端框架类似，小程序也有生命周期。

#### App 生命周期

整个小程序的生命周期如下图所示：

![app life](https://user-gold-cdn.xitu.io/2018/6/29/1644b72f98d7617a)

说明：

| 属性 | 类型 | 描述 | 触发时机 |
| --- | ----  |---- | -------|
| onLaunch | Function	| 生命周期函数--监听小程序初始化	| 当小程序初始化完成时，会触发 onLaunch（全局只触发一次） |
| onShow | Function	| 生命周期函数--监听小程序显示	| 当小程序启动，或从后台进入前台显示，会触发 onShow |
| onHide | Function	| 生命周期函数--监听小程序隐藏	| 当小程序从前台进入后台，会触发 onHide |

App 的生命周期是在小程序启动后全局监控的，不随着页面切换变化而变化。通过 `App()` 注册小程序，传入对应的生命周期函数，具体文档见[注册程序](https://developers.weixin.qq.com/miniprogram/dev/framework/app-service/app.html)。

#### Page 生命周期

除了整个小程序 App 的生命周期，每个页面（Page）也有自己的生命周期：

![app life](https://user-gold-cdn.xitu.io/2018/6/29/1644b72f98a9b055?w=558&h=430&f=png&s=12894)

说明：

| 属性 |	类型 |	描述 |
| ---- | --- | --- |
| onLoad |	Function |	生命周期函数--监听页面加载 |
| onReady |	Function |	生命周期函数--监听页面初次渲染完成 |
| onShow |	Function |	生命周期函数--监听页面显示 |
| onHide |	Function |	生命周期函数--监听页面隐藏 |
| onUnload |	Function |	生命周期函数--监听页面卸载 |

当页面状态发生变化时，会触发对应的生命周期函数，可以通过 `Page()` 注册页面并传入监听函数。具体文档见[注册程序 · 小程序](https://developers.weixin.qq.com/miniprogram/dev/framework/app-service/page.html)。

## 摩拜单车小程序功能简介

以下是摩拜单车小程序的主要功能介绍，接下来的章节我们将围绕这些功能进行案例解析。

### 开锁用车

每辆摩拜单车车身都贴有唯一的二维码，通过扫码解析出二维码文本判断要开哪辆车，再根据车位置、用户状态等信息判断是否满足开锁条件。

![app](https://user-gold-cdn.xitu.io/2018/6/29/1644b72f98be88b2?w=400&h=777&f=jpeg&s=71348)

可以看到，除了“扫码开锁”按钮外，地图还可以显示附近的单车，并且顶部还配置 banner 资源位，单车的图标可以根据配置变化，banner 支持多种跳转方式。

### 分享红包

![app](https://user-gold-cdn.xitu.io/2018/6/29/1644b72f98cce1ce)

除了开锁用车，摩拜单车小程序还可以给好友发送骑行红包，如何分享红包，分享到个人和群又有什么区别，如何分享到朋友圈？

### 支付功能

![](https://user-gold-cdn.xitu.io/2018/7/4/1646537612849621?w=800&h=648&f=jpeg&s=72238)

充值或者缴纳押金，都离不开支付功能，如何在小程序内支付，如何开启免密支付，支付时有哪些注意事项？

以上都可以在后续章节中找到答案。

