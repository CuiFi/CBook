# 实战案例解析 8：蓝牙

作为一家科技公司，摩拜有太多你可能不知道的“黑科技”，比如蓝牙方式开锁。

第一代单车，车锁内置的是 GSM 模块，智能车锁与插了 SIM 卡的功能手机类似，可以通过运营商给智能锁发送短信解锁。但网络信号不是全方位的，难免会有信号死角导致开锁不成功。甚至极端情况运营商发生故障，也会导致大面积无法开锁（发生过），所以急需另一种不依赖运营商网络的开锁方式。而蓝牙开锁（低功耗蓝牙技术 BLE）就是其中一种极为理想的方案。

本章将要介绍小程序核心蓝牙 API ，以及我们如何通过蓝牙一步步实现摩拜单车开锁功能。

## 蓝牙开锁步骤

完整的一次开锁流程大致如下图：

![bluetooth_process](https://user-gold-cdn.xitu.io/2018/7/5/1646882439897f36)

其中，通过蓝牙开锁关键分为以下几步：

- 开启蓝牙适配器 (wx.openBluetoothAdapter)
- 搜索指定蓝牙设备 (wx.startBluetoothDevicesDiscovery)
- 连接搜索到设备 (wx.createBLEConnection)
- 发送开锁指令 (wx.writeBLECharacteristicValue)

细心的读者会发现，上面列出了一些关键步骤的小程序 JavaScript 方法名，具体的官方的文档可以在这里看得到：[蓝牙适配器接口](https://developers.weixin.qq.com/miniprogram/dev/api/bluetooth.html)。

### 开启蓝牙适配器

```js
wx.openBluetoothAdapter({
  success: function (res) {
    success(res)
  },
  fail: function (res) {
    fail(res)
  }
})
```

直接使用微信提供的 `wx.openBluetoothAdapter` 方法即可，我们可以加上必要的回调函数。

### 搜索指定蓝牙设备

```js
function startBluetoothDiscovery(success, fail) {
  wx.onBluetoothDeviceFound(function(deviceFoundRes) {
    let device = deviceFoundRes.devices[0]
    if (device) {
      let advertisData = ''
      try {
        advertisData = ab2hex(device.advertisData)
      } catch (e) {}
      console.log(advertisData)
      success(advertisData)
    }
  })
  wx.startBluetoothDevicesDiscovery({
    allowDuplicatesKey: false,
    success: function (discoveryRes) {
      console.warn('start discovery', discoveryRes)
    },
    fail: function (discoveryRes) {
      fail(discoveryRes)
    }
  })
}
```

上面是我们封装的搜索蓝牙设备（wx.startBluetoothDiscovery）的方法，其中最终获取的 `advertisData` 就是我们需要的蓝牙关键数据，要注意的是搜索到的蓝牙设备 device 中 advertisData 字段表示当前蓝牙设备的广播数据段中的 ManufacturerData 数据段，其是一个 ArrayBuffer 类型数据，无法使用 `console.log` 打出，微信也给出了 ArrayBuffer 转十六进制字符串代码示例：

```js
function ab2hex(buffer) {
  var hexArr = Array.prototype.map.call(
    new Uint8Array(buffer),
    function(bit) {
      return ('00' + bit.toString(16)).slice(-2)
    }
  )
  return hexArr.join('');
}
```
我们用 ab2hex 对 advertisData 进行转换即可。

注意：

`allowDuplicatesKey`：是否允许重复上报同一设备， 如果允许重复上报，则 `onDeviceFound` 方法会多次上报同一设备。

### 连接设备
通过搜索到的设备，我们得到`deviceId`，进而和这个设备创建连接：

```js
// 连接低功耗蓝牙设备
wx.createBLEConnection({
  deviceId,
  success() {
    // 获取蓝牙设备所有service服务
    wx.getBLEDeviceServices({
      deviceId,
      success(getServicesRes) {
        let service = getServicesRes.services[0]
        let serviceId = service.serviceId
        // 获取蓝牙设备某个服务中的所有characteristic特征值
        wx.getBLEDeviceCharacteristics({
          deviceId,
          serviceId,
          success(getCharactersRes) {
            let characteristic = getCharactersRes.characteristics[0]
            let characteristicId = characteristic.characteristicId

            // 启用低功耗蓝牙设备特征值变化时的 notify 功能，订阅特征值
            wx.notifyBLECharacteristicValueChange({
              state: true,
              deviceId,
              serviceId,
              characteristicId,
              success() {
                // 监听低功耗蓝牙设备的特征值变化
                wx.onBLECharacteristicValueChange(function(onNotityChangeRes) {
                  // 此处可以开始向蓝牙设备传输开锁指令
                })
              }
            })
          }
        })
      }
    })
  },
  fail() {}
})
```

一顿猛如虎的操作后，我们终于得到 `serviceId`、`characteristicId` 这些关键值，接下来就可以传输开锁指令。

注意：
- serviceId 和 characteristicId 都简单取了返回列表第一个值，如果有多项需要进行枚举判断。
- 省略了 fail 回调，需要增加异常处理。

### 发送开锁指令

我们使用 [wx.writeBLECharacteristicValue](https://developers.weixin.qq.com/miniprogram/dev/api/bluetooth.html#wxwriteblecharacteristicvalueobject) 向蓝牙设备特征值中传输二进制数据。BLE 数据包传输规定了一次只能传输 20 byte，所以如果数据量较大，需要做分包传输处理：

```js
let chunkCount = Math.ceil(command.length / 18)
let subCommands = []

for (let i = 0; i < chunkCount; i++) {
  subCommands.push(`${chunkCount}${i}${command.substr(i * 18, 18)}`)
}
```

以上就是将一个数据包 `command` 拆分的过程，`chunkCount` 表示总共多少包，`i`表示当前是第几个包，这通常是 BLE 设备所需要的必要识别信息。接下来就是分布发送每个包数据：

```js
function sendCommand(i) {
  if (i < chunkCount) {
    let subCommand = subCommads[i]
    wx.writeBLECharacteristicValue({
      deviceId,
      serviceId,
      characteristicId,
      value: str2ab(subCommand),
      success: function (res) {
        setTimeout(function() {
          send(i + 1)
        }, 20)
      },
      fail: function (res) {
        fail(res)
      }
    })
  }
}
sendCommand(0)
```

注意：
- 传送的数据需要是 ArrayBuffer 类型，使用 `str2ab` 方法封装下：

```js
function str2ab(str) {
  var buf = new ArrayBuffer(str.length)
  var bufView = new Uint8Array(buf)
  for (var i = 0, strLen = str.length; i < strLen; i++) {
    bufView[i] = str.charCodeAt(i)
  }
  return buf
}
```
- 需要前一个数据包返回结果后再发送后续数据包，否则一些安卓设备会引起发送异常。
- 每个数据包发送增加相应间隔（setTimeout），减少一些安卓设备发送异常。

## 注意事项

除了上面提到的一些，还有几个关键注意事项。

### 优先从已搜索到设备中寻找设备

`wx.onBluetoothDeviceFound` 的含义是：监听寻找到新设备的事件。如果在搜索蓝牙设备方法里设置了 `allowDuplicatesKey` 为 `false`，则已经搜索到的设备不会在这个方法回调里，可以使用 `wx.getBluetoothDevices` 先从已搜索到设备里寻找设备。

### 注意添加异常回调

很多手机对蓝牙支持不是很好，尤其是安卓设备，会出现各种异常，需要在多处 fail 回调里增加异常捕捉。

### 注意添加超时处理

搜索设备，连接设备都需要增加相应的超时失败处理。

## 小结

蓝牙 API 可能很多小程序用得不多，但通过它可以看出小程序连接端能力的强大之处，并且小程序可以抹平 API 兼容性的差异，使开发者使用起来更为便捷。
