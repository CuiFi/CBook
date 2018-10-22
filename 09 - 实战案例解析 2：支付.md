# 实战案例解析 2：支付

支付是除了**登录**之外的第二个常见功能。在介绍微信支付功能之前，还是需要提醒开发者：

先要在微信后台申请微信认证，如果你有微信支付商户号，就可以直接绑定已有的；如果没有，可以申请一个新的微信支付商户号。如下图所示：

![申请](https://user-gold-cdn.xitu.io/2018/7/2/1645b236fcd73927?w=1126&h=758&f=png&s=146759)

本节将从 **微信支付** 和 **免密支付** 两方面展开。

## 微信支付

充值、购买虚拟卡片等业务都会用到微信支付，我们先介绍一下最核心的 [API](https://developers.weixin.qq.com/miniprogram/dev/api/api-pay.html#wxrequestpaymentobject)：

```js
wx.requestPayment(OBJECT)
```

我们一般会在支付按钮点击事件中来触发请求，再调用这个 API：

```js
wx.requestPayment({
  timeStamp: `${timestamp}`,
  nonceStr: noncestr,
  package: `prepay_id=${prepayid}`,
  signType: 'MD5',
  paySign: sign,
  success,
  fail (res) {
    console.log('onFail')
    // 不是用户取消弹出提示
    if (res.errMsg != 'requestPayment:fail cancel') {
      onFail() && onFail(res)
    }
  },
  complete (res) {
    wx.hideLoading()
  }
})
```

支付的参数：

| 参数 | 含义 |
| ---- | ---- |
| signType | 签名算法，值是固定的目前： MD5 |
| timeStamp | 当前的时间 |
| nonceStr | 随机字符串 |
| package | 接口返回的 prepay_id 参数值，提交格式如：prepay_id=* |
| paySign | 签名 |

注：最后四项是业务下单接口返回的。

## 免密支付

免密支付在 App 端比较常见，用户可以进行小额快捷支付，不需要过多的操作。

具体流程如下图所示：

![pay_mianmi](https://user-gold-cdn.xitu.io/2018/7/2/1645b12d3cd81cba?w=692&h=413&f=jpeg&s=42536)

那我们如何在小程序中实现它呢？在开始之前，你需要做一些准备：

* 要开通小程序委托代扣的权限
* 要开通商户小程序跳转微信签约小程序的权限

那在开发过程中，我们需要做哪些呢？

* 用户从摩拜小程序发起签约请求
* 摩拜将签约请求参数按照规则拼接之后，通过小程序跳转，向签约小程序发起签约请求
* 用户在微信签约小程序选择支付方式完成签约
* 微信将签约结果返回给摩拜

具体流程如下图所示：

![pay_mianmi_process](https://user-gold-cdn.xitu.io/2018/7/2/1645b12d3ce09e6e?w=738&h=515&f=jpeg&s=62134)

1、版本判断

依赖基础库版本 **1.3.0**，客户端的版本：iOS 的微信版本 6.5.9、安卓的微信版本 6.5.10

```js
checkNoAuthPayVersion () {
  let APP = getApp()
	// 检查是否支持开通免密
	// - 基础库 1.3.0 - ios微信6.5.9 - Android微信6.5.10
	let isAndroid = APP.globalData.isAndroid
	let sdkversion = APP.globalData.sdkversion
	let version = APP.globalData.systemInfo.version // 微信版本
	return versionCompare(sdkversion, '1.3.0') && ((isAndroid && versionCompare(version, '6.5.10')) || (!isAndroid && versionCompare(version, '6.5.9')))
}
```

2、跳转到微信免密的小程序

我们使用到了官方的 API：

`wx.navigateToMiniProgram(OBJECT)` -- 打开同一公众号下关联的另一个小程序

使用方式如下：

```js
wx.navigateToMiniProgram({
  appId: '*****',
  path: 'pages/index/index',
  extraData: {
    //...
  },
  success (res) {
    //...
  },
  fail (res) {
    //...
  }
})
```

参数说明：

- appId 是必须的参数，值是固定的：wxbd687630cd02ce1d
- path 的值固定的：pages/index/index
- extraData 比较重要，是传的数据

详细介绍一下 extraData：

| 参数 | 含义 |
| ---- | ---- |
| mch_id | 商户号 - 微信支付分配的商户号 |
| appid | 公众账号 id - 发起签约的小程序 appid |
| plan_id | 模板 id - 协议模板 id |
| contract_code | 签约协议号 |
| request_serial | 请求序列号 |
| contract_display_account | 用户账户展示名称 |
| notify_url | 回调通知 url |
| sign | 签名 |
| timestamp | 时间戳 |
| mobile | 用户手机号 |

## 小结

通过本节的描述，我们可以知道如何在小程序里实现支付，以及详细的 API 和对应的参数。
