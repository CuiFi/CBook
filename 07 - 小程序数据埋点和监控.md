# 小程序数据埋点和监控

随着小程序规模越来越大，数据和埋点会显得格外重要。我们不仅仅要去记录 PV、UV、按钮点击等数据，更要从这些数据中分析用户的行为，可以更快速地去发现异常或者优化产品体验。本节主要介绍如何做数据埋点，以及我们在小程序中如何做监控。

## 数据埋点

作为复杂的小程序，数据埋点必不可少。我们使用了两套埋点方案：

- 内部数据统计平台
- 小程序统计后台

摩拜数据统计平台是一个公司的内部数据系统，里面包含 App、H5、小程序、PC 等各种数据。我们将用户登录、开关锁等核心数据录入该平台，方便数据同学分析对比数据。

小程序统计后台也提供了一些基础的数据统计，比如总访问人数、新用户、分享次数等。此外，自定义分析是我们常用的统计之一，本节后续会详细介绍。

### 内部数据统计平台

我们统计了每一位用户**地图加载时长**、**附近单车出现时间**、**开锁耗时**等关键性指标，以便对用户行为进一步分析优化。采用前端向后端发送数据统计请求埋点，log 内容以参数形式发送到 “空 gif” 请求里，这也是通用的日志统计方式：

```js
wx.request({
  url: 'https://log.mobike.com/log/x.gif',
  data: {
    log: '{startTime: 1530179125563, endTime: 1530179138793, ...}'
  }
})
```

初次进入小程序，会有多个统计指标要发送，同时业务上也要发出多个数据请求，考虑到小程序[网络请求并发限制](https://developers.weixin.qq.com/miniprogram/dev/api/api-network.html)，我们对统计请求做了延后处理，优先保证业务请求顺利发出。

```js
let tracks = []

...

// 地图加载完成
if (mapLoaded) {
  tracks.push({
    key: 'MAP_LOADED',
    startTime: ....
  })
}

// 附近单车出现
if (nearbyBikesLoaded) {
  tracks.push({
    key: 'SHOW_NEARBY_BIKES',
    startTime: ....
  })
}
```

选定一个节点表示业务请求发送完成，比如附近的车出现，再统一发送所有埋点请求：

```js
tracks.forEach((log) => {
  wx.request({
    url: 'https://log.mobike.com/log/x.gif',
    data: {
      log
    }
  })
})
```

后期我们也支持了一次请求发送多个埋点统计：

```js
wx.request({
  url: 'https://log.mobike.com/log/x.gif',
  data: {
    log: JSON.stringify(tracks)
  }
})
```

### 小程序统计后台

可以在小程序[管理后台](https://mp.weixin.qq.com/wxopen/home?lang=zh_CN&token=553429190)看到官方统计数据，同时也支持自定义数据分析。

![app](https://user-gold-cdn.xitu.io/2018/7/2/1645afdd3187fee8?w=2024&h=1124&f=jpeg&s=114293)

自定义分析支持两种上报方式：

- 填写配置
- API 上报

填写配置需要在对应 WXML 元素增加 `element` 标识，通常为 `className` 或者 `id`，例如：

```html
<button type="primary" class="login-button">登录</button>
```

在微信后台进行如下配置，就简单完成了一个自定义配置上报：

![app](https://user-gold-cdn.xitu.io/2018/7/2/1645afdd2b507914?w=800&h=415&f=jpeg&s=27252)

API 上报使用 `wx.reportAnalytics` 上报接口，例如：

```js
wx.reportAnalytics('login', {
  citycode: 010
})
```

同样，此类上报也需要在微信后台进行配置。

一切配置完后，我们就可以在 「自定义分析」 -> 「事件分析」中查询统计结果，如下图所示。

![log](https://user-gold-cdn.xitu.io/2018/7/2/1645b007ae2bb8de?w=800&h=688&f=jpeg&s=39407)

详细介绍可参考[官方文档](https://developers.weixin.qq.com/miniprogram/analysis/custom/)。

## 监控

### 业务数据监控

有了埋点数据，我们就可以针对数据做些监控，例如由于摩拜单车业务强烈依赖用户定位，所以我们监控了用户定位失败的信息：

```js
wx.getLocation({
  type: 'gcj02',
  success(res) {
    ...
  },
  fail(res) {
    wx.reportAnalytics('get_position_fail', {
      msg: res.msg
    }
  },
})
```

很多时候，我们发布了新版小程序，但一些用户反馈出问题，经过定位发现用户使用的还是旧版本的小程序。在微信发布强制更新小程序版本功能 [wx.getUpdateManager](https://developers.weixin.qq.com/miniprogram/dev/api/getUpdateManager.html) 之前，这种情况只能让用户删除小程序重新搜索进入。因此，我们统计了用户使用的小程序版本：

```js
wx.reportAnalytics({
  version: '__VERSION__'
})
```

每次在小程序编译发布命令中，添加版本号字段，并对代码中的 `__VERSION__` 进行替换：

```
npm run build 0.9.1
```

其中 `0.9.1` 就是版本号。

通过上述版本号的上报，我们在 mp  数据统计后台就可以查询到结果（选择版本号作为分组），如下图所示。

![](https://user-gold-cdn.xitu.io/2018/7/6/1646f8796e9220d0?w=2390&h=842&f=png&s=51879)

### 程序错误监控

除了业务数据监控，我们还监控了小程序中的代码报错：

![app](https://user-gold-cdn.xitu.io/2018/7/2/1645afdd300c7c21?w=399&h=446&f=jpeg&s=88124)

这些都可以在 mp 后台中的 「运维中心」 -> 「监控告警」中设置，通过这些报警信息可以对代码进行针对性的优化。

## 小结

通过数据埋点和监控，结合真实的线上用户行为和数据去分析，一步步达到我们不断优化产品体验的目的。

