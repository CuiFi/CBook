# 实战案例解析 4：分享

> 本章案例相关代码已提交至 GitHub ，内容持续更新，可以自行下载进行本地调试，[点此访问](https://github.com/mobikeFE/xiaoce-demo)

分享功能是产品设计中重要的一环，旨在吸引更多的潜在用户。微信小程序运行在微信环境中，具有天然的优势。

因为分享功能涉及的知识点比较零散，所以本节以摩拜单车小程序的一个运营活动为例，来全面介绍分享功能。该活动的功能如下图所示：

<img src="https://user-gold-cdn.xitu.io/2018/7/4/16463f0123413d06?w=1868&h=1324&f=jpeg&s=230628">

## 分享到会话

在页面中，定义 `onShareAppMessage(options)` 方法，来设置该页面的转发信息。如果想要在页面内发起转发，需要通过给 `button` 组件设置属性 `open-type="share"`，需要注意的是只能是 `button` 组件，更多API可以参考 [转发](https://developers.weixin.qq.com/miniprogram/dev/api/share.html)。

示例：

```js
Page({
  onShareAppMessage: function (res) {
    // 区分页面右上角分享和页面内 button 点击
    if (res.from === 'button') {
      // 来自页面内 button 转发按钮
      console.log(res.target)
    }
    return {
      title: '送你一个现金大礼包！扫码骑摩拜就能领现金！',
      path: '/page/***/index',
      imageUrl: 'https://***.mobike.com/***.png'
    }
  }
})
```

参数说明：

- title: 分享文案
- path: 分享页面路径
- imageUrl: 分享图片，可以是本地文件路径或者网络图片路径，建议使用网络图片。当不传入 `imageUrl` 时，会默认使用截图，显示图片的长宽比是 5:4。

### 获取群标识

这个需求要求统计分享到不同群的个数，那我们如何拿到群标识呢？主要是通过 `wx.getShareInfo(OBJECT)` 方法获取，需要传入 shareTicket。

1. 定义携带 shareTicket 的转发

2. 在分享时，获取 shareTicket

3. 根据 shareTicket 获取群标识

```js
Page({
  onLoad: function (options) {
    // 定义携带 shareTicket 的转发
    wx.showShareMenu({ withShareTicket: true })
    ...
  },
  onShareAppMessage: function (res) {
    let that = this
    return {
      title: '送你一个现金大礼包！扫码骑摩拜就能领现金！',
      path: `/pages/***/index`,
      imageUrl: '***',
      success: function (res) {
        // 获取 shareTicket
        var shareTickets = res.shareTickets
        if (shareTickets && shareTickets.length !== 0) {
          wx.getShareInfo({
            shareTicket: shareTickets[0],
            success: function(res) {
              console.log('加密之后的群标识：', res.iv)
            })
          }
        }
      }
    }
  }
})
```


### 模版消息

该活动有这样一个功能点：当用户收到待领取的红包时，会在微信服务通知 中，收到一条模版消息。下面来看一下如何发送一条模版消息。

1、 登录 [微信公众平台|小程序](https://mp.weixin.qq.com) 获取或者新定义模版，参考下图定义字段。

<img width="600" src="https://user-gold-cdn.xitu.io/2018/7/3/1646006c8fbe8bfa?w=1602&h=688&f=jpeg&s=88871">

2、 用户行为，触发模版消息

模版消息的触发条件有两个，一个是**支付**，另一个是**提交表单**。我们的触发场景不是用户支付，那只能把分享按钮做成表单。

```
<form report-submit="true" bindsubmit="shareBtnTap">
  <button formType="submit" open-type="share">发红包 赚赏金</button>
</form>

Page({
  shareBtnTap (e) {
    // 获取formId
    console.log('用于发送模版消息的formId: ', e.detail.formId)
  },
})
```

其中，`report-submit` 属性，表示是否返回 formId。最后在 `shareBtnTap` 方法中，获取到 formId。

3、 调用接口下发模版消息

在该需求中，模版消息的发送，是在某个特定时机触发，这部分由后台来执行。调用微信接口，传递 模版id、formId、模版内容等。

```js
{
  "touser": "OPENID",  // 接收用户的openid
  "template_id": "TEMPLATE_ID",  // 模版id
  "page": "index",  // 用户点击后的跳转页面
  "form_id": "FORMID",
  "data": {
      "keyword1": {
          "value": "339208499"
      },
      "keyword2": {
          "value": "2015年01月05日 12:30"
      },
      "keyword3": {
          "value": "粤海喜来登酒店"
      } ,
      "keyword4": {
          "value": "广州市天河区天河路208号"
      }
  },
  "emphasis_keyword": "keyword1.DATA"  // 模板需要放大的关键词，不填则默认无放大
}
```
说明：

- `keyword` ：配置的文案，如上图所示
- `emphasis_keyword` ： 模板需要放大的关键词，上图中的“一份现金奖励”文案就是通过此方法放大显示的。

## 分享到朋友圈

看到这个标题有没有很困惑，微信小程序并没有提供分享到朋友圈的 API，那是怎么做到的？

这里是引导用户将带有小程序码的图片保存到本地，再自行分享到朋友圈。其他用户长按图片识别小程序码时，跳转到指定页面。

那如何生成小程序码呢？微信目前支持两种二维码，小程序码和小程序二维码，如下图所示。

<img width="600" src="https://user-gold-cdn.xitu.io/2018/7/4/16463f01235abba2?w=1078&h=576&f=jpeg&s=106814">

微信为满足不同的需求和场景，提供了三种接口生成二维码。

| 接口类型 | 码类型 | 生成个数是否受限制 |
| ----- | ----- | ----- |
| A接口 | 小程序码 | 是 |
| B接口 | 小程序码 | 否 |
| C接口 | 二维码 | 是 |

A接口和C接口生成二维码数量有限，所以B接口更适合携带不同 query 的运营活动场景。

B接口生成的小程序码，永久有效，数量暂无限制。在请求B接口时，页面放在 `page` 字段中，不能携带参数。参数应放在 `scene` 字段中，最大是32个可见字符。

请求B接口参数示例：

```js
{
  scene: 'user_123:token_123',  // 参数
  page: 'pages/redPacket/index',  // 跳转页面
  width: 430  // 二维码宽度
}
```

这样就可以生成小程序码了，当用户扫码后，小程序跳转至 `pages/redPacket/index` 页面。

在 `pages/redPacket/index` 页面中，再处理参数 scene。

```js
Page({
  onLoad (option = {}) {
    console.log('小程序码上携带参数scene ', option.scene)
    // 处理scene
    this.sceneControl(option.scene)
  }
})
```

这样就完成了一个二维码的生成和处理，想了解更多可以参考 [获取二维码](https://developers.weixin.qq.com/miniprogram/dev/api/qrcode.html)。

回到分享到朋友圈的图片，我们发现，小程序码只是这张图片的一部分。受活动策略的限制，该图片地址是后台合成返回到前端的。那如何把这幅图片下载到手机相册呢，主要分两步，下载和保存。

1、 下载图片资源到本地，获取图片的临时路径

利用 `wx.downloadFile(OBJECT)` 方法，来下载资源。

```js
wx.downloadFile({
  url: 'https://***.mobike.com/***/***.png', // 一定要有文件后缀
  success: function(res) {
    if (res.statusCode === 200) {
      console.log('临时图片路径: ', res.tempFilePath)
    }
  }
})
```

2、 根据图片的临时路径，将其保存到手机相册

利用 `wx.saveImageToPhotosAlbum()` 方法，将上步获取的临时路径保存到相册。调用这个方法后，微信会弹出是否授权该小程序访问手机相册，如果用户选择不允许的话，是无法下载图片的。

```js
wx.saveImageToPhotosAlbum({
  filePath: this.data.filePath,
  success: function (res) {
    wx.showModal({
      title: '已保存到相册',
      content: '请自行分享到朋友圈，好友领红包，你赚赏金',
      showCancel: false,
      confirmText: '确定',
      success: function(res) {
        if (res.confirm) {
          // 返回上一页
          wx.navigateBack()
        }
      }
    })
  },
  fail: function (res) {
    wx.showModal({
      title: '下载失败',
      content: '未获得你的授权，请尝试截取屏幕分享至朋友圈。',
      showCancel: false,
      confirmText: '确定',
      success: function(res) {
        if (res.confirm) {
          // 返回上一页
          wx.navigateBack()
        }
      }
    })
  }
})
```
## 小结

本节以摩拜单车运营活动为例，详细的介绍了微信小程序的分享功能。为了尽可能覆盖全部的使用场景，还介绍了发送模版消息、生成小程序二维码等内容。了解了这些知识，可以让你轻松应对分享需求。

注：本节内容于2018年6月25日截稿，如若发现后续分享策略有变化，请以微信官方文档为准。
