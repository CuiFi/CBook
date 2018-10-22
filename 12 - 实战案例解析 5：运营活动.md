# 实战案例解析 5：运营活动

从摩拜单车小程序运营至今，已经有1年半的时间了，期间我们支持了很多的运营活动，本节将介绍其中几项有代表意义且适用性高的案例。

## 在地图上绘制 banner

地图上的资源位（banner）如下图：

![banner](https://user-gold-cdn.xitu.io/2018/7/5/164696404a506210?w=400&h=474&f=jpeg&s=55171)

```
<cover-image class="banner" wx:if="{{bannerImg}}" bindtap="onBannerTap" src="{{bannerImg}}"></cover-image>
```

因为覆盖在地图（map）组件，需要使用`cover-view`（[介绍](https://developers.weixin.qq.com/miniprogram/dev/component/cover-view.html)）。

### 多种事件类型

`onBannerTap`是点击回调，目标行为（action）是通过后台配置返回，支持多种事件类型：

- 跳转到小程序页面
- 跳转到H5页面
- 跳转到其他小程序
- 调用js回调函数

我们规定`action`需要按照指定格式定义，以便于区分这几种事件类型：

- 以`pages`开头表示跳转到小程序页面，如`pages/login/index`
- 以`https`开头表示跳转到H5页面，如`https://www.mobike.com`（注意：需要在mp后台配置业务域名白名单）
- 以`wx`开头表示跳转到其他小程序，如`wx99999999999`，表示其他小程序的`appid`，微信规定`appid`统一以wx开头
- 以`event:`开头表示，如`event:getPhoneNumber`，表示调用`getPhoneNumber`回调函数

### 封装`getActionType`

封装一个获取不同banner事件类型的公用函数：

```
function getActionType(action) {
  let actionType
  if (/^pages/.test(action)) {
    actionType = 0
  }
  if (/^http/.test(action)) {
    actionType = 1
  }
  if (/^wx/.test(action)) {
    actionType = 2
  }
  if (/^event/.test(action)) {
    actionType = 3
  }
  return actionType
}
```


### 触发banner点击事件

根据不同的`actionType`触发不同的banner点击事件：

```
Page({
	...
  onBannerTap() {
    let {
      actionType,
      link
    }
    switch (this.data.actionType) {
      case 0:
        wx.navigateTo({
          url: link
        });
        break
      case 1: gotoH5(link); break
      case 2: goToMiniProgram(link); break
      case 3: this[link](); break
    }
  }
})
```

注意：需要根据不同的`actionType`和`action`解析出目标`link`，可能是个H5页面甚至是个函数名。`gotoH5`以及`goToMiniProgram`是封装后的方法。

## 变换地图上的车标

地图车辆 markers 显示图标，支持运营活动自定义图片 icon，如下图所示：

<div style="text-align: center;">
  <img style="width: 300px;" src="https://user-gold-cdn.xitu.io/2018/7/2/1645b438c035003d"></img>
</div>

之前在地图组件我们已经介绍过，地图组件的 markers 配置了单车的位置及显示图标的路径，通过替换图标路径可以实现运营活动配置。

**注意：** `markers` 的图标需配置项目目录下的图片路径或**临时路径**。

### 获取每种单车类型的配置图片url

```js
const bikeTypes = config.bikeTypes
const bikeIcons = []
bikeTypes.forEach((item) => {
  bikeIcons[item.type] = item.iconNormal
})
```

### 下载图片资源到本地，获取图片的临时路径

利用 `wx.downloadFile(OBJECT)` 方法，来下载资源。
```js
function downloadFile(url, callback) {
  wx.downloadFile({
    url,
    success: function(res) {
      callback && callback(res)
    }
  })
}

for (let type in bikeIcons) {
  ((type) => {
    downloadFile(bikeIcons[type], (res) => {
      // 图片下载失败设置项目目录下的默认图片
      bikeIcons[type] = res.statusCode === 200 ? item.filePath : defaultBikeIcons[type]
    })
    if (++downloadTime >= bikeTypes.length) {
      // 保存到全局变量
      getApp().globalData.bikeIcons = bikeIcons
    }
  })(type)
}
```

### 替换 markers 图标路径

```
const markers = [{
  latitude: '**'
  longitude: '**'
  width: '**'
  height: '**'
  id: '**',
  iconPath: bikeIcons[biketype]
}, {
  ...
}]
this.setData({markers})
```

## 绘制二维码

产品提了一个离线生成二维码的需求，当时想到了一些优秀的前端开源库 [jquery-qrcode](https://github.com/jeromeetienne/jquery-qrcode) 和 [node-qrcode](https://github.com/soldair/node-qrcode)，由于小程序中没有 DOM 的概念，这些库在小程序中并不适用。

所以，针对微信小程序的特点，封装了 [weapp.qrcode.js](https://github.com/yingye/weapp-qrcode) ，用于在小程序中快速生成二维码。

`weapp.qrcode.js` 主要利用小程序 canvas 组件 API 来绘制二维码，使用方法可参考 [README.md](https://github.com/yingye/weapp-qrcode/blob/master/README.md)。

## 自定义导航栏

微信小程序的默认导航栏只能设置导航栏背景色值(不能使用渐变、透明度等)、标题的色值，这不能满足部分人的开发需求，所以微信官方提出了 navigationStyle，用来设置自定义导航栏。在 app.json 中设置 navigationStyle 值为 custom，即可自定义导航栏，只保留右上角胶囊状的按钮。

```json
{
  "pages": [
    "pages/index/index"
  ],
  "window": {
    "navigationStyle": "custom"
  }
}
```

由于使用了自定义导航栏，页面自带的返回按钮也会消失，所以建议在页面左上角添加跳转到主页或者返回上一级页面按钮。

![](https://user-gold-cdn.xitu.io/2018/7/31/164eecebf039cf95?w=750&h=587&f=jpeg&s=8155)

> 本章案例相关代码已提交至 github ，内容持续更新，可以自行下载进行本地调试，[点此访问](https://github.com/mobikeFE/xiaoce-demo/tree/master/demo2)

## 小结

本节重点介绍了摩拜小程序几个典型的运营案例，以及每个案例涉及的技术细节，特别是配置的运营素材的下载及保存功能。
