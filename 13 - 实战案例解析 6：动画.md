# 实战案例解析 6：动画

小程序支持两种动画实现方式：CSS3 animation 属性及官方提供的 API，我们将分别示例两种动画实现，并重点介绍官方创建动画实例的 API 及相关的使用技巧。

> 本章案例相关代码已提交至 github ，内容持续更新，可以自行下载进行本地调试，[点此访问](https://github.com/mobikeFE/xiaoce-demo)

## CSS3 动画

![效果图](https://user-gold-cdn.xitu.io/2018/7/3/164600daa88c7201?w=599&h=529&f=gif&s=90800)

小程序支持使用 `CSS3 animation` 定义动画。
```css
// 设置动画元素的 animation 属性
view {
  animation: circle 1.4s linear infinite;
}
// 通过 @keyframes 创建动画
@keyframes circle {
  0% {
    transform: rotate(0deg);
  }
  60%, 100% {
    transform: rotate(50deg);
  }
}
```

**注意：** cover-view 从基础库 **1.6.0** 起支持css transition动画，且 transition-property 只支持 transform (translateX, translateY) 与 opacity

## 官方提供 API

动画效果图如下:

![英雄](https://user-gold-cdn.xitu.io/2018/6/25/1643692e7599f7c8?w=422&h=720&f=gif&s=13554479)

如何在小程序中实现上述动画效果呢？

小程序提供了创建动画实例的 API **wx.createAnimation**（[官方文档](https://developers.weixin.qq.com/miniprogram/dev/api/api-animation.html)）。

一个完整的动画实例的创建过程：
```js
// 创建动画实例
var animation = wx.createAnimation()
// 实例调用动画方法描述动画，最后调用 step() 表示动画完成
animation.opacity(0).step({delay: 7500, during: 500})
// 最后调用 export 方法导出动画数据
this.setData({
    animation: animation.export()
})
```
上述动画案例是由多个图片按一定的时间线依次展示组成，使用 [ image 标签](https://developers.weixin.qq.com/miniprogram/dev/component/image.html)加载动画图片资源，其中每张图片作为一条单独的时间线，多个动画实例同时播放，实现上述效果图。其中动画实例定义在 `image` 组件的 `animation` 属性。

动画功能实现：

```html
<image animation="{{animation}}" class="boy" src="/images/gr_boy.png"></image>
```
```js
Page({
  data: {},
  onShow() {
    // 页面加载时触发动画播放事件
    this.animationStart()
  }
  animationStart() {
    const animation = wx.createAnimation({
      duration: 1000,
      timingFunction: 'linear'
    })
    animation.opacity(0).step({delay: 7500, during: 500})
    this.setData({
      animation: animation.export()
    })
  }
  ...
})
```

可以看到每个动画实例都是使用 `wx.createAnimation` 创建动画实例，调用实例的方法来描述动画，`step()` 表示一组动画完成，最后通过动画实例的 `export` 方法导出动画数据传递给组件的 `animation` 属性。

## 小技巧：

### tip 1: 如何选择动画开始播放的时间点？

动画所需的图片资源较多，页面加载完成后图片资源未必全部加载完成，若播放动画可能出现图片加载不全。

使用 image 的 **bindload** 属性监听图片加载完毕并计数，加载完成的图片个数满足条件时触发动画播放事件。

```html
<image animation="{{animation}}" src="/qcloud/gr_boy.png" bindload="loaded"></image>
```
```js
Page({
  data: {},
  animationStart() {
    ...
  },
  loaded(e) {
    let loadedNum = this.data.loadedNum || 0
    this.setData({loadedNum: ++loadedNum})
    if (this.data.loadedNum >= 10) {
      // 动画开始
      ...
    }
  },
    ...
})

```

### tip 2: 如何实现动画跳过功能？

> 小程序 API 不支持暂停或跳过动画播放，实现方式为直接展示动画结束状态。

小程序动画 API 不支持暂停或停止功能。此处实现动画跳过功能的方式为使用 `this.setData({ended: true})`切换动画结束展示图片 class 属性，通过设置 `z-index: 990; opacity: 1` 使动画结束图片堆叠层级最高，实际动画播放仍在继续但已不可见。

```html
<image class="endedImg {{ended ? 'end' : ''}}" src='/qcloud/gr_end.jpg' bindload="loaded"></image>
```
```css
.end {
  z-index: 999
  opacity: 1;
}
```

### tip 3: 背景音乐播放与静音功能

> 小程序提供了播放背景音乐的 API **wx.playBackgroundAudio**（[官方文档](https://developers.weixin.qq.com/miniprogram/dev/api/media-background-audio.html#wxplaybackgroundaudioobject)）。

动画开始时即调用 **wx.playBackgroundAudio** 开始背景音乐播放，关闭背景音乐时调用 **wx.stopBackgroundAudio** 关闭。因背景音乐需随动画即时播放，因此此处静音功能实际为关闭背景音乐，不再支持暂停与重新播放功能。

### tip 4: 重新进入小程序动画重复播放

动画播放过程中将小程序切至后台，小程序重新从后台进入前台显示，出现动画重复播放且时间线不一致的问题。

设置 `exist` 状态标记动画播放状态，重新从后台唤醒小程序时不再重复执行播放动画。  

```js
Page({
  data: {
    exist: false
  },
  animationStart() {
    this.setData({
      exist: true
    })
    ...
  },
  loaded(e) {
    let loadedNum = this.data.loadedNum || 0
    this.setData({loadedNum: ++loadedNum})
    if (this.data.loadedNum >= 10 && !this.data.exsit) {
      // 动画开始
      ...
    }
  },
  ...
})
```

### tip 5: 限定动画播放时间

由于动画实例创建时间并非严格同步，多个动画实例同时播放时动画播放时间有可能大于设定时间。

设置定时器，限定时间后直接展示动画结束状态。

```js
Page({
  data: {},
  animationStart() {
    ...
    this.timer = setTimeout(() => {
      that.setData({
        ended: true,
        exsit: false
      })
    }, 20000)
  },
  ...
})
```

## 小结

本节介绍了小程序内动画的两种实现方式，重点介绍了小程序提供的动画实例 API，动画使用方法并不复杂，我们结合使用案例介绍了一些结合小程序生命周期使用的小技巧。相信会对大家在小程序实现动画有所启发。
