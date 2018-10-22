# 小程序开发环境优化

微信小程序在开发测试中，会有如下的不便：

- 本地开发的时候，需要 mock 数据来提升开发效率
- 小程序最终运行在手机端，需要更全面的显示调试信息
- 测试环境较多，经常变更环境，就需要经常发布开发版或者体验版

本节将详细介绍这些问题的解决方案，这些方案也可以轻松移植到其他小程序中。

## mock 数据

这里介绍了两种 mock 数据方式，大家可权衡成本与收益自行选择。

### 方式一：项目内构建

这个方式相对比较主流和低成本一些，缺点也比较明显：需要一定的技能、可移植性差。

如何存储文件？

1. 我们在根目录下新建文件夹 mock

目录结构：

```
├─mock
  ├─data
  ├─app.js
  ├─route.js
```

定义路由：

```js
'GET /wx/**/getTotal***': 'data/**/getTotal.json',
```

定义数据文件：

```js
{
  "code": 0,
  "message": "*****",
  "object": {}
}
```

创建服务：

整体我们采用 express 来搭建本地服务，同时循环 mock 文件的目录，生成对应的路由匹配函数，最终读取文件，返回对应的 JSON 内容。

```js
var app = require('express')()
var route = require('./route')
var fs = require('fs')
var path = require('path')

app.get('/', function (req, res) {
  res.send('Hello World')
})

for (var name in route) {
  var s = name.split(' ')
  var method = s[0].toLowerCase()
  var url = s[1]
  var mockFile = route[name]
  app[method](url, (function (mockFile) {
    return function (req, res) {
      res.setHeader('Content-Type', 'application/json')
      res.send(fs.readFileSync(path.join(__dirname + '/' + mockFile)))
    }
  })(mockFile))
}

app.listen(3000)
```

### 方式二：Mock 平台接入

摩拜有一个统一的在线 [Mock 平台](https://mock.mobike.com/)，我们基于 [easy-mock](https://github.com/easy-mock/easy-mock) 做了深度的二次封装。

#### 新建接口
![mock](https://user-gold-cdn.xitu.io/2018/6/29/1644bcf5c69c7c2f?w=1761&h=1404&f=jpeg&s=97779)

可以添加返回的 JSON 数据，设置请求 Method、URL 以及添加接口描述等。


#### 接口列表

![mock](https://user-gold-cdn.xitu.io/2018/6/29/1644bcf5c2a78320?w=2058&h=1438&f=jpeg&s=207761)

每个接口的含义一目了然，操作区域可以进行预览、编辑等操作。

当开启 Mock 后，所有的业务请求都会指向 Mock 平台，我们在请求方法里统一进行了如下处理：

```js
let prefix = 'https://XXXX.mobike.com/api/' // 线上或测试域名
if (MOCK) {
  prefix = 'https://mock.mobike.com/mock/XXXXXX/' // 修改为 Mock 平台统一域名
}
```

> 注意：需要在开发者工具中开启“不校验合法域名”。

<div style="text-align: center">
  <img style="width: 300px;" src="https://user-gold-cdn.xitu.io/2018/6/29/1644bcf727013fdf?w=400&h=211&f=jpeg&s=16942"></img>
</div>


#### 代码中 Mock 配置

在一些场景中，为了方便 Mock 请求之后的返回，我们在全局管理变量：

```js
module.exports = {
 MOCK: false
}
```

比如支付场景中，开启 Mock 模拟支付成功，而不用真正去调用 `wx.requestPayment`，避免因为支付环境问题拖慢其他开发速度：

```js
if (MOCK) {
  wx.showModal({
    content: '支付成功',
    success (res) {
      if (res.confirm) {
        success()
      }
    }
  })
} else {
  wx.requestPayment({
    timeStamp: `${timestamp}`,
    nonceStr: noncestr,
    package: `prepay_id=${prepayid}`,
    signType: 'MD5',
    paySign: sign,
    success,
    fail (res) {}
  })
}
```

还比如对扫码二维码文本格式的校验，如果开启 Mock ，就可以绕过校验，也可以加快开发速度：

```js
function isValidQR (text) {
  return MOCK || /XXXXXXX/.test(text)
}
```

## 调试 vConsole

小程序有自带的 vConsole 功能，在开发版或体验版小程序右上角“打开调试”即可开启，面板信息如下图所示。

<div style="text-align: center">
  <img style="width: 300px;" src="https://user-gold-cdn.xitu.io/2018/6/29/1644bcf5c4d7462c?w=1080&h=1856&f=jpeg&s=425284"></img>
</div>

Log 面板下显示已打印的 log 信息，支持 log、info、warn、error 四种格式。System 面板下可查看小程序的系统信息。此外，微信提供了 API [`wx.setEnableDebug(OBJECT)`](https://developers.weixin.qq.com/miniprogram/dev/api/setEnableDebug.html) 设置是否打开调试开关（可对正式版生效）。或者可先在开发版或体验版打开调试，再切到正式版即能查看 vConsole。

> 注意：vConsole 默认会打印小程序及页面的生命周期，因此我们能够看到小程序启动后先后触发 `onLaunch` - `onShow` - `onLoad` - `onShow` - `onReady`。

因开发调试的需要，项目通常需要在多处打印 log，而线上正式版并不需要打印这些 log 信息，可通过 npm 包 `gulp-uglify` 去除项目中的 console 代码。

```js
plugins.uglify({
  compress: {
    // 去除console的代码
    drop_console: true,
    ...
  }
})
```
## 测试报告

小程序支持免费的云真机测试环境以及一整套测试方案在上线前检测小程序缺陷。云测试可以模拟用户使用的方式进行测试，执行完毕后自动生成测试报告，可以提供异常发现、性能数据分析、机型覆盖等报告数据。

最新版开发者工具提供了提交测试的入口，如下图所示。

<div style="text-align: center;">
  <img style="width: 700px;" src="https://user-gold-cdn.xitu.io/2018/7/4/16465829a721bc40?w=1652&h=804&f=jpeg&s=113110"></img>
</div>

需要注意的是，开发工具对于代码包体积做了限制，上传代码包体积不得超过 2MB：

<div style="text-align: center;">
  <img style="width: 400px;" src="https://user-gold-cdn.xitu.io/2018/7/5/1646a814082d90b2?w=842&h=314&f=png&s=69188"></img>
</div>


具体细节可以参考[官方文档](https://developers.weixin.qq.com/miniprogram/dev/devtools/monkey-test.html)。


## 摇一摇切换测试环境

为了方便测试同学在不同测试环境下测试，我们开发了通过摇一摇切换测试环境的功能。所谓的“摇一摇”其实是利用微信提供的监听手机加速度的 API [wx.onAccelerometerChange(CALLBACK)](https://developers.weixin.qq.com/miniprogram/dev/api/accelerometer.html#wxonaccelerometerchangecallback)，在监听回调里处理跳转切换环境页面。

```js
// 定义初始值
var lastTime = 0
var x = 0
var y = 0
var z = 0
var lastX = 0
var lastY = 0
var lastZ = 0
var shakeSpeed = 120 // 摇一摇阈值
// 定义加速度监听回调
function shake () {
  var nowTime = new Date().getTime() // 记录当前时间
  // 如果这次摇的时间距离上次摇的时间有一定间隔 才执行
  if (nowTime - lastTime > 100) {
    var diffTime = nowTime - lastTime // 记录时间段
    lastTime = nowTime // 记录本次摇动时间，为下次计算摇动时间做准备
    x = acceleration.x // 获取x轴数值，x轴为垂直于北轴，向东为正
    y = acceleration.y // 获取y轴数值，y轴向正北为正
    z = acceleration.z // 获取z轴数值，z轴垂直于地面，向上为正
    // 计算 公式的意思是 单位时间内运动的路程，即为我们想要的速度
    var speed = Math.abs(x + y + z - lastX - lastY - lastZ) / diffTime * 10000
    var flag = true
    // 如果计算出来的速度超过了阈值，那么就算作用户成功摇一摇
    if (speed > shakeSpeed) {
      // 触发间隔周期内再次检测到摇一摇，直接返回
      if (+(new Date()) - lastTriggerTime < triggerDuration) {
        flag = false
      }
      // 避免多次跳转环境切换页面
      getCurrentPages().forEach((item) => {
        if (item.__route__ == 'pages/console/index') {
          flag = false
        }
      })
      if (flag) {
        // 跳转环境切换页面
        wx.navigateTo(...)
        lastTriggerTime = +(new Date())
      }
    }
    lastX = x // 赋值，为下一次计算做准备
    lastY = y // 赋值，为下一次计算做准备
    lastZ = z // 赋值，为下一次计算做准备
  }
}
// 监听加速度数据
wx.onAccelerometerChange(shake)
```

> 注意：上线时需去除摇一摇功能的代码，可使用代码块预处理工具 **jdists** 实现，具体使用规则可查看[文档](https://github.com/zswang/jdists)。

## 小结

开发环境优化是一个持续的过程，没有一个固定的标准。比如，Mock 平台的引入是结合其他 Web 项目一起完成的，而“摇一摇切换环境”功能 则是由摩拜单车 App 提供灵感实现的。
