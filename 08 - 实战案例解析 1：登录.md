# 实战案例解析 1：登录

本节会逐一介绍手机号验证码登录、微信一键授权登录和 SSO 自动登录等方式，以及如何结合小程序特色进行自动登录。

在前面两个最常用的登录方式中，我们加上了整个前后台调用的流程，期望给读者一些指引。

> 本节案例相关代码已提交至 GitHub ，内容持续更新，可以自行下载进行本地调试，[点此访问](https://github.com/mobikeFE/xiaoce-demo)。

## 手机号验证码登录
![login](https://user-gold-cdn.xitu.io/2018/7/2/1645b105b05da298?w=400&h=430&f=jpeg&s=41502)

这是比较传统和通用的登录方式，通过发送短信验证码验证用户，为了`防止短信服务商可能出问题`，我们也增加了`语音播报验证码`功能。

```html
<view class="login">
  <form report-submit="true" bindsubmit="onLoginSubmitTap">
    <view class="input-area">
      <text>手机号码</text>
      <input maxlength="11" bindinput="checkPhoneNumber" bindblur="checkPhoneNumber" type="number" placeholder-style="color:#b2b2b2" placeholder="填写手机号"/>
      <text class="get-code" bindtap="sendCode">获取验证码</text>
    </view>
    <view class="input-area">
      <text>验证码</text>
      <input bindinput="checkVerifyCode" bindblur="checkVerifyCode" type="number" placeholder-style="color:#b2b2b2" placeholder="输入验证码"/>
    </view>
    <view class="voice-code" bindtap="getVoiceCode">收不到短信，试试语音验证码</view>
    <button id="loginBtn" type="primary" formType="submit" bindtap="login" >开始</button>
  </form>
</view>
```

这是大致的 WXML 结构，我们使用了 `form` 表单处理提交请求，同时也使用了 `bindinput`、`bindblur`等来处理输入校验，具体用法可参考官方文档[表单组件 input](https://developers.weixin.qq.com/miniprogram/dev/component/input.html)。

类似于上述的 WXML 表单结构中：

- `onLoginSubmitTap`：点击登录的回调
- button 需要指明 `formType="submit"`

下面我们就按照流程给大家介绍一下具体的交互。

### 第一步：验证用户输入

注意到输入框都加入了 `bindblur` 事件，可以在用户输入完（失去表单焦点）后即时验证输入值，例如手机号可以做如下验证：

```js
Page({
  checkPhoneNumber(e) {
    let phoneNo = e.detail.value
    // 正则验证
    let isValid = /^1[3456789]\d{9}$/.test(phoneNo)
    if (isValid) {
      this.setData({
        disabled: false
      })
      ...
    } else {
      ...
    }
  }
})
```

### 第二步：发送验证码

点击之后，调用接口，给输入的用户手机号发送短信：

```js
Page({
  sendCode() {
    wx.request({
      url: 'https://path/to/sendcode',
      success(res) {
        ...
      }
    })
  }
})
```

### 第三步：调用登录接口

传入 3 个参数，分别是：

* `jscode`：调用 `wx.login` 获取 jscode 传给服务端，服务器以 jscode 换取**用户唯一标识 openid** 和**会话密钥 session_key**，之后可以绑定 openid 到用户账号中，以便做自动登录。
* `smscode`：验证码
* `phoneNo`：手机号

```js
Page({
  onLoginSubmitTap() {
    let self = this
    wx.login({
      success(res) {
        let jscode = res.code
        wx.request({
          url: 'https://path/to/smslogin',
          data: {
            jscode,
            smscode: self.data.smscode,
            phoneNo: self.data.phoneNo
          },
          success(res) {
            // 登录成功
            let {
              userid,
              accesstoken
            } = res
            ...
          }
        })
      }
    })
  }
})
```

登录成功后，可以将接口返回的用户信息存储到本地，其中：

* `userid`: 用户唯一标识
* `accesstoken`: 用户加密信息等



## 微信一键授权登录
![login](https://user-gold-cdn.xitu.io/2018/7/2/1645b105b0600829?w=400&h=430&f=jpeg&s=45516)

### 获取 openid

上文提到，通过调用`wx.login()`获取 code 传回服务端，不仅仅可以得到会话秘钥 session_key ，还能得到用户唯一标识 openid。

> 注意：临时登录凭证 code 只能使用一次。


### 获取 session_key

小程序调用`wx.login()`获取临时登录凭证 code ，并回传到服务器，服务器以 code 换取用户唯一标识 openid 和会话密钥 session_key，详细见[官方文档](https://developers.weixin.qq.com/miniprogram/dev/api/api-login.html)。

注意：`wx.login()`不能直接在 `bindgetphonenumber` 回调中调用，因为它会刷新登录状态，此时传回给服务端使用的 code 换取的 sessionKey 不是加密时使用的 `sessionKey`，导致解密失败。可以进入页面时候就调用`wx.login()`获取 code ，或者在回调中先使用 `wx.checkSession()` 进行登录态检查，避免 login 刷新登录态。

```js
Page({
  onShow() {
    wx.login({
      success: (res) => {
        this.setData({
          code: res.code
        })
      }
    })
  },
  onGetPhoneNumber(e) {
    let data = e.detail
    let params = {
      encryptedData: data.encryptedData,
      iv: data.iv,
      code: this.data.code
    }
    // 传给服务端
    ...
  }
})
```

### `getPhoneNumber`按钮

通过小程序提供的[按钮组件](https://developers.weixin.qq.com/miniprogram/dev/component/button.html)可以直接授权获得用户手机号，这比传统手机号+验证码方式要便捷很多：

```html
<button open-type="getPhoneNumber" bindgetphonenumber="onGetPhoneNumber"></button>
```

`open-type`的值设为`getPhoneNumber`，指定按钮功能是获取手机号，并且可以从 `bindgetphonenumber` 回调中获取到用户信息。

注意：此时获取到的用户信息是微信服务器返回的加密数据，服务端需要结合 session_key 以及 app_id 进行解密获取手机号。

说明：

- `open-type`：微信开放能力
- `bindgetphonenumber`：手机号授权成功的回调

### 调用后端登录接口

```js
Page({
  ...
  getPhoneNumber(e) {
    let data = e.detail

    wx.request({
      url: 'https://path/to/login',
      data: {
        encryptedData: data.encryptedData
        iv: data.iv
      },
      success(res) {
        // 登录成功
        let {
          userid,
          accesstoken
        } = res
        ...
      }
    })
  }
})
```

说明：

- `encryptedData`：包括敏感数据在内的完整用户信息的加密数据
- `iv`：加密算法的初始向量

### 后台解密数据

通过 `encryptedData` 和 `iv` ，解密到明文用户数据（[解密算法](https://developers.weixin.qq.com/miniprogram/dev/api/signature.html#wxchecksessionobject)），结构如下：

```json
{
  "phoneNumber": "18600000000",
  "purePhoneNumber": "18600000000",
  "countryCode": "86",
  "watermark": {
    "appid": "APPID",
    "timestamp": "TIMESTAMP"
  }
}
```

> 整个加解密流程的 demo 我们放到了 GitHub 上面，[点此访问](https://github.com/mobikeFE/wechat-nodejs-sdk)。

需要替换几个值，如下：

```js
// 小程序的 appId
var appId = '**'

// sessionKey
var sessionKey = '**'
```

关于 `sessionKey` 的获取方式，在官方文档中有[登录凭证校验](https://developers.weixin.qq.com/miniprogram/dev/api/api-login.html#wxloginobject)。

然后调用官方给出的加密文件包：


```js
var WXBizDataCrypt = require('./WXBizDataCrypt')

var pc = new WXBizDataCrypt(appId, sessionKey)

// 调用 decryptData 函数
var data = pc.decryptData(encryptedData , iv)
```




## SSO 自动登录

SSO（Single Sign On）即单点登录，用户只要登录了微信，就可以直接登录摩拜单车小程序。

用户访问摩拜单车小程序，都会生成一个唯一标识 `openid` ，我们只要将这个唯一标识和摩拜账号进行绑定，就可以做到用户进入小程序自动登录账号。



### 自动登录

有了基础 SSO 自动登录服务，前端还需要基于此基础服务进行一系列自动登录操作：

- 初始化小程序时，用户未登录，直接调用 SSO 登录服务自动登录；
- 初始化小程序时，用户登录失效，调用 SSO 登录服务恢复登录；
- 小程序运行过程中，用户登录失效（比如未操作时间过长，或者在其他地方重新生成登录信息导致这个登录信息失效），调用 SSO 自动登录服务恢复登录。

可以参考如下的流程图：

![sso process](https://user-gold-cdn.xitu.io/2018/7/2/1645b105b0a41bdb?w=732&h=423&f=png&s=15072)

可以将 SSO 登录封装为一个基础的 service 服务：

```js
function ssoLogin(onSuccess, onFail, onComplete) {
  wx.login({
    success(res) {
      wx.request({
        path: 'https://demo.com/sso'
        data: {
          code: res.code
        },
        success(data) {
          onSuccess(data)
        },
        fail(res) {
          onFail(res)
        },
        complete(res) {
          onComplete(res)
        }
      })
    }
  })
}
```

## 多端登录

摩拜单车支持多端登录，比如 iOS 客户端、Android 客户端、H5 网页以及小程序，几个端之间不互斥，支持同时登录。为此，我们为每个端分配了`platform` 值，每次请求都会传给服务端各自的 `platform` 值，服务端可以据此做不同的逻辑处理。

虽然支持多端登录，但不同设备的同一端的登录信息是互斥的，因为每个`platform`只会维持唯一一个有效的登录信息（token）。

## 退出登录

SSO 自动登录固然方便，但需要给用户提供退出登录的功能，或者说，提供用户主动更新 openid 与账号的绑定关系。比如摩拜单车的“切换账号”功能，新的账号登录后，原有的绑定关系在服务端自动解绑，并更新为新的绑定关系。

## 常见问题汇总

#### 小程序传 code 获取 session_key 和 openid 报下面的错

> string(78) "{"errcode":40163,"errmsg":"code been used, hints: [ req_id: 929Xta02322065 ]"}"

code 被用过了，只能用一次。

#### 当用户从分享页面进入，在完成登录之后如何跳转回该页面

在登录成功之后，调用 `wx.navigateBack()` 方法，实现跳转回上级页面。


## 小结

登录是小程序产品非常普遍的功能，通过本节的介绍，我们可以知道如何在小程序里设计登录，以及利用微信环境的优势去优化登录体验。

