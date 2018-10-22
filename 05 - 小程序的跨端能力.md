# 小程序的跨端能力

<div style="text-align: center;">
  <img width="480" src="https://user-gold-cdn.xitu.io/2018/7/5/16469564dc5675d8?w=678&h=622&f=png&s=29704">
</div>

随着小程序生态的不断扩大，小程序现在具备了载入 H5、唤起 App、跳转至其他小程序等能力，本节将向大家详细介绍这些跨端能力。

## 载入 H5

### web-view 组件

web-view 组件是一个用来承载网页的容器，其加载的 URL 需要配置到域名白名单中，在微信公众平台【设置】->【开发设置】->【业务域名】下进行设置。

<img src="https://user-gold-cdn.xitu.io/2018/7/4/16463dbc7e0aa44d?w=2356&h=824&f=jpeg&s=86046">

一个小程序中可能会载入很多 H5 页面，通常我们会创建一个通用的 page 来承载这些 H5 页面。下面，就来创建这个通用的 page，给它起名为 web。

第一步：在 index.wxml 中引入 web-view 组件

```html
<!-- web/index.wxml -->
<view>
  <web-view src="{{u}}"></web-view>
</view>
```

第二步：在 index.js 中读取数据

```js
Page({
  name: 'web',
  data: {
    u: ''
  },
  onLoad(option) {
    const u = option.u
    this.setData({
      u
    })
    console.log('[web-view url]', u)
  }
})
```

> 一般情况下，我们需要对 url 进行转义。

```js
decodeURIComponent(option.u)
```

第三步：在我们的业务中，需要追加一些用户信息和渠道等参数，因此我们增加了一个 `fixWebLinkURL` 函数来做统一的转换。

```js
const u = fixWebLinkURL(decodeURIComponent(option.u))

function fixWebLinkURL(url) {
  // 把 hash 放在最后面
  let qs = `********`
  let match = url.split('#')
  if (match[1]) {
    qs += '#' + match[1]
    url = match[0]
  }
  return ~url.indexOf('?') ? `${url}&${qs}` : `${url}?${qs}`
}
```

按照以上三步，我们就完成了 web page 的开发。这时从小程序页面跳转到某 H5 页时，可以用如下代码简洁实现：

```js
wx.navigateTo({
  url: `/pages/web/index?u=${url}` 
})
```

### H5 调用小程序方法

在 `web-view` 组件中加载的网页，可以使用内置的一些接口能力，比如最常用的跳转操作、向小程序发送消息，具体可以查看 [web-view · 小程序](https://developers.weixin.qq.com/miniprogram/dev/component/web-view.html)。

#### 1. 判断环境

利用微信提供的 [JSSDK 1.3.2](https://res.wx.qq.com/open/js/jweixin-1.3.2.js) 来判断环境。

```html
<script type="text/javascript" src="https://res.wx.qq.com/open/js/jweixin-1.3.2.js"></script>
```
方式一：

```js
function ready() {
  console.log(window.__wxjs_environment === 'miniprogram') // true
}
if (!window.WeixinJSBridge || !WeixinJSBridge.invoke) {
  document.addEventListener('WeixinJSBridgeReady', ready, false)
} else {
  ready()
}
```

方式二：
```js
wx.miniProgram.getEnv(function(res) {
  console.log(res.miniprogram) // true
})
```

> 目前发现如果使用方式二判断环境，只有在小程序内才能走到回调函数，即在浏览器内运行不会打印出 `false` 。

#### 2. 处理跳转

调用如下的方法，就可以跳转到指定的小程序页面了。

```js
wx.miniProgram.navigateTo({url: '/path/to/page'})
```

#### 3. 向小程序发送消息

小程序端先绑定处理事件：

```html
<web-view src="{{url}}" bindmessage="handleMsg"/>
```

```js
Page({
  handleMsg: function (e) {
    console.log('h5 postMessage is', e.detail.data)
  }
})
```

H5 端触发事件：

```js
wx.miniProgram.postMessage({ data: '这是一条传递给小程序的消息' })
```

> 只会在特定时机（小程序后退、组件销毁、分享）触发并收到消息。

## 小程序之间跳转

小程序和小程序之间也可以相互跳转，例如在开通免密支付时，就是商户小程序和微信签约小程序互跳的过程。`wx.navigateToMiniProgram()` 方法用于处理小程序的跳转。

```js
wx.navigateToMiniProgram({
  appId: '',
  path: '',
  extraData: {
    // 传递的参数
  },
  envVersion: 'develop',
  success(res) {
    // 打开成功
  }
})
```

注意：

- envVersion 是要打开的小程序版本，有 develop（开发版）、trial（体验版）和 release（正式版）。如果当前小程序是正式版，那么跳转到其他小程序也是正式版。
- 在开发者工具上调用此 API 并不会真正地跳转到其他小程序，所以需要真机调试。
- 只有同一公众号下关联的小程序才可以互相跳转。

`wx.navigateBackMiniProgram()` 函数用于返回到上一个小程序，更多信息可以参考 [wx.navigateBackMiniProgram · 小程序](https://developers.weixin.qq.com/miniprogram/dev/api/navigateBackMiniProgram.html)。

> 注：`wx.navigateToMiniProgram` 接口即将废弃，可以使用 [navigator](https://developers.weixin.qq.com/miniprogram/dev/component/navigator.html) 组件来使用此功能。

## 小程序和 App 之间跳转

### App 跳转到小程序

这里，先以 iOS 开发示例来说明 App 跳转到小程序需要的参数：

```objc
WXLaunchMiniProgramReq *launchMiniProgramReq = [WXLaunchMiniProgramReq object];
launchMiniProgramReq.userName = userName;  // 拉起的小程序的username
launchMiniProgramReq.path = path;    // 拉起小程序页面的可带参路径，不填默认拉起小程序首页
launchMiniProgramReq.miniProgramType = miniProgramType; // 拉起小程序的类型
return  [WXApi sendReq:launchMiniProgramReq];
```

其中，`userName` 就是小程序的**原始 ID**，可以在微信公众平台下，【设置页面】->【基本设置】 中找到。

而 Android 端跳转到小程序的代码示例如下：

```java
String appId = "wxd930ea5d5a258f4f"; // 填应用AppId
IWXAPI api = WXAPIFactory.createWXAPI(context, appId);

WXLaunchMiniProgram.Req req = new WXLaunchMiniProgram.Req();
req.userName = "gh_d43f693ca31f"; // 填小程序原始id
req.path = path;                  //拉起小程序页面的可带参路径，不填默认拉起小程序首页
req.miniprogramType = WXLaunchMiniProgram.Req.MINIPTOGRAM_TYPE_RELEASE;// 可选打开 开发版，体验版和正式版
api.sendReq(req);
```

### 小程序跳回 App

这里有几点需要注意：

- 小程序不能打开任意的 App，只能**跳回**分享该小程序卡片的 App
- 小程序只有用户主动触发才能跳回 App，而且不能在任意时机跳回

官网给出了一张图，来展示小程序何时具备**打开 App 的能力**。

<img src="https://user-gold-cdn.xitu.io/2018/7/25/164cfb14f2515bf9?w=715&h=870&f=png&s=37647">

综合上图可以看出：

- 当小程序从 1036（App 分享消息卡片）或 1069（App 打开小程序） 打开时，该状态置为 true。
- 当小程序从 1089（微信聊天主界面下拉）或 1090（长按小程序右上角菜单唤出最近使用历史）的场景打开时，该状态不变，即保持上一次打开小程序时该状态的值。
- 当小程序从非 1036/1069/1089/1090 的场景打开，该状态置为 false。

小程序端使用方法：

```html
<button open-type="launchApp" app-parameter="wechat" binderror="launchAppError">打开App</button>
```

其中，`app-parameter` 为要传递的参数，通过 `binderror` 来监听打开 App 的错误事件。

## 小结

本节介绍了小程序与 H5、小程序与小程序以及小程序与 App 之间的一些交互，可以看出小程序的跨端能力还是受到了很多条件限制，不过相信随着微信小程序越来越开放，跨端的能力也会越来越丰富。

## Q&A

小程序可以直接跳转到公众号文章，并识别二维码关注公众号吗？
> 不能，建议提示用户截屏再用微信扫一扫。