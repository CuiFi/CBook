# 实战案例解析 7：长连接

长连接使得 client/server 保持持久连接，在会话期间双方在这条连接上进行通信。相比于 HTTP ，它可以做到服务器主动向客户端推送信息。下图是 client/server 建立长连接的设计图。

<img src="https://user-gold-cdn.xitu.io/2018/7/5/164687b3b456d4cb?w=2064&h=1080&f=png&s=298525">

## WebSocket

WebSocket 就是长连接的一种形式，下面介绍一下如何在微信小程序中发起 WebSocket。

1、 建立连接

```js
wx.connectSocket({
  url: 'wss://***.***.com',
  protocols: ['mqtt']
})
```

其中，`protocols` 为子协议数据，本节后半部分将展开描述。

2、 监听 WebSocket 连接打开事件

```js
wx.onSocketOpen(function(res) {
  console.log('WebSocket连接已打开！')
})
```

3、监听 WebSocket 错误

```js
wx.onSocketError(function(res){
  console.log('WebSocket连接打开失败，请检查！')
})
```

4、 通过 WebSocket 连接发送数据

发送数据时，需要先进行 `wx.connectSocket`，并在 `wx.onSocketOpen` 回调之后发送。

```js
wx.sendSocketMessage({
  data: msg
})
```

5、 监听 WebSocket 接收服务器消息事件

```js
wx.onSocketMessage(function(res) {
  console.log('收到服务器内容：', res.data)
})
```

6、 关闭连接

```js
wx.closeSocket()
```

7、 监听连接关闭

```js
wx.onSocketClose(function(res) {
  console.log('WebSocket 已关闭！')
})
```

在建立连接方法中，提到 `protocols` 使用的是 MQTT 协议，下面来介绍该协议及在小程序中的实现。

## MQTT

MQTT 是一个客户端服务端架构的**发布／订阅**模式的消息传输协议，它的设计思想是轻巧、开放、简单、规范，易于实现。这些特点使得它对很多场景来说都是很好的选择，特别是对于受限的环境如机器与机器的通信（M2M）以及物联网环境（IoT）。

MQTT 协议在客户端中有几个比较常用的控制报文：

- CONNECT - 连接服务器
  客户端到服务端的网络连接建立后，客户端发送给服务端的第一个报文必须是 CONNECT 报文。

- SUBSCRIBE - 订阅主题
  客户端向服务端发送 SUBSCRIBE 报文用于创建一个或多个订阅，每个订阅注册客户端关心的一个或多个主题。为了将应用消息转发给与那些订阅匹配的主题，服务端发送 PUBLISH 报文给客户端。

- PINGERQ - 心跳请求
  客户端向服务端发送心跳请求，告知服务端客户端还活着。

- DISCONNECT - 断开链接
  DISCONNECT报文是客户端发给服务端的最后一个控制报文，表示客户端正常断开连接。

### 如何实现

[MQTT.js](https://github.com/mqttjs/MQTT.js) 是一个[MQTT协议](http://mqtt.org/)的客户端库，可以运行在浏览器 、node 和微信小程序环境中，我们主要是利用这个开源库实现的该功能。该开源库实现的本质，是调用 [小程序 WebSocket API](https://developers.weixin.qq.com/miniprogram/dev/api/network-socket.html)。

从小程序基础库版本 2.2.1 开始，小程序支持使用 npm 安装第三方包，更多信息可以阅读[npm 支持](https://developers.weixin.qq.com/miniprogram/dev/devtools/npm.html)。目前可以采取将 MQTT.js 拷贝至项目目录中进行引用。

和大家介绍一下常用的 API，以摩拜单车小程序获取用户开锁信息为例：

1. 与服务器建立连接
2. 订阅开锁信息
3. 处理开锁信息，关闭连接

```js
const client = mqtt.connect('wxs://***.***.com', {
  // 心跳请求，单位s
  keepalive: 30,
  clientId: '***',
  protocolId: 'MQTT'
})
// 与服务器建立连接
client.on('connect', function () {
  // 订阅开锁信息
  client.subscribe(`mobike/unlocked/***`)
})
client.on('message', function (topic, message, packet) {
  if (topic.indexOf('mobike/unlocked') > -1) {
    // 处理开锁信息，关闭连接
    console.log('锁已开')
    client.end()
  }
})
// 错误处理
client.on('error', function (err) {
  console.error('mqtt err, ', err)
  client.end()
})
```

在收到开锁信息之后，开始订阅关锁信息，用户骑行时间不定，而且中间会出现退出小程序、手机锁屏等情况，所以连接中断的可能性非常大。那如何在中断之后，重新连接服务器呢？

MQTT.js 提供了 `close` 方法，用于处理连接中断，但是无法感知是主动断开连接，还是因为异常中断。在这里，我们定义了一个变量，标识中断是否是主动的，从而判断中断后，是否要重新连接。

```js
// 初始化 主动中断标识
client._activeEndFlag = false
// 定义主动中断连接方法
client._activeEnd = function () {
  client._activeEndFlag = true
  client.end()
}

client.on('close', function (res) {
  console.log('mqtt连接中断，是否重新连接：', !client._activeEndFlag)
  if (!client._activeEndFlag) {
    // 重新建立连接
    console.log('mqtt连接重新建立中...')
    client.reconnect()
  }
})
```

## 小结

在摩拜单车小程序中，开关锁信息的快速展示是用户体验中很重要的一部分。在早期的版本中，是通过轮询接口获取开关锁信息的，弊端是，效率低而且信息展示不及时。后来通过使用 MQTT 协议，与服务器建立长链接，来解决该问题，用户体验大幅提升。因此，大家可以考虑使用长连接的形式，来规避轮询接口，从而优化用户体验。
