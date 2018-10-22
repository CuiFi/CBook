# 小程序的组件化开发

在大型软件中，组件化是一种共识，它一方面提高了开发效率，另一方面降低了维护成本。在小程序基础库版本 **1.6.3** 之前，小程序没有自定义组件的概念。那让我们一同来看看在 **1.6.3** 版本前后，在小程序内是如何实现组件化的。

## `template` 方案

最初，我们是利用 `WXML` 提供的 `template` 模板和文件引用方式 `import` 来进行组件化的，为此还封装了 [wx-component](https://github.com/binnng/wx-component)。目前主流的小程序框架 WePY 和 mpvue 也是利用该方式实现的组件化。

**`template` 使用示例**

item.wxml 定义：
```html
<template name="item">
  <view>
    <text>{{text}}</text>
  </view>
</template>
```

在 index.wxml 中，使用：

```html
<import src="item.wxml"/>
<template is="item" data="{{...itemData}}"/>
```

在 index.js 中，定义传递的 data：

```js
Page({
  data: {
    itemData: {
      text: 'template 示例'
    }
  }
})
```

`<template>` 的 `name` 属性是模板的唯一标识，在使用时，通过 `is` 指定使用哪个模板，`data` 来传递参数。

这种方式微信只提供了插入模板和传参的能力，所以 `wx-component` 还增加了私有变量、事件的处理等。用 `wx-component` 构建的组件库有一个明显的问题：可移植性不强。其他开发者使用时，必须引入 `wx-component`，而且还要熟悉 `wx-component` 的 API。

所以，后来在微信官方提出**自定义组件**之后，我们开始积极尝试新的自定义组件。

## 组件基础

### 组件定义

组件需要定义在 `pages/` 目录下，类似于页面，一个自定义组件由 `json`、 `wxml`、 `wxss` 和 `js` 4 个文件组成。

要编写一个自定义组件，首先需要在 `json` 文件中进行自定义组件声明：

```json
{
  "component": true
}
```

### 组件引用

在页面中引入组件，需要在 `json` 文件中进行定义：

```json
{
  "navigationBarTextStyle": "black",
  "usingComponents": {
    "component-dialog": "../components/dialog/index"
  }
}
```

定义后，就可以在页面中使用标签了，例如：

```html
<component-dialog></component-dialog>
```
我们在实践中得出如下的注意事项：

1. 组件的名字**不能包含数字**，比如：

```json
"component-404": "../components/404/index"
```

2. 在 `<map>` 上使用组件，引用组件时要嵌套一层 `cover-view`：

```html
<map>
  <cover-view>
    <!-- 引入自定义组件 -->
    <component-map-dialog></component-map-dialog>
  </cover-view>
</map>
```

### 父子通信

这也是组件化中比较重要的一环，微信小程序的组件类似 `Vue.js` 的组件，强调数据的单向性，但是语法略有不同。

#### 父组件向子组件传递信息

父组件向子组件通信，是通过传入 `properties` 的形式。子组件中定义 `properties` 类型、默认值等，父组件传入，示例如下：

```js
// 子组件:
properties: {
  visible: {
    type: Boolean,
    value: false // 默认值
  },
  content: String
},
  
// 父组件:
<component-dialog visible="{{visible}}" content="text"></component-dialog>
```

其中，`visible` 传入的是变量，`content` 传入的是 `text` 的字符串。

#### 子组件向父组件传递信息

子组件通过触发事件的形式，调用父组件的方法，示例如下：

```js
// 子组件:
that.triggerEvent('onLoginSuccess', res)
  
// 父组件:
<component-login bind:onLoginSuccess="onLoginSuccess"></component-login>
  
// js
onLoginSuccess (data) {
  // 传入的参数会被包装在 detail 下面
  let res = data.detail
}
```

这里需要注意的是，子组件传回的数据是放在 `data.detail` 里面的。

### slot

和 `Vue.js` 一样，小程序组件也支持 `slot`。

```html
<component-dialog>
  <view slot="header">
    <view>提示</view>
  </view>
  <view slot="content">
    <view>{{dialogText}}</view>
  </view>
</component-dialog>
```

但是，默认只有一个 `slot`，需要使用多 `slot` 时，可以在组件 `js` 中声明启用。

```js
Component({
  options: {
    multipleSlots: true // 在组件定义时的选项中启用多slot支持
  },
  properties: { /* ... */ },
  methods: { /* ... */ }
})
```

## 组件示例

<div style="text-align: center">
  <img width="300" src="https://user-gold-cdn.xitu.io/2018/6/27/1644120f33001674?w=750&h=1334&f=jpeg&s=72390">
</div>

下面，我们一起来看下，如何封装一个弹窗组件并使用。

在这个组件中，红包图片、提示文案和按钮文案都是可变的，`wxml` 结构如下：

```html
<view class="mbk-dialog-box">
  <view class="mbk-dialog-header">
    <slot name="header"></slot>
  </view>
  <view class="mbk-dialog-content">
    <view class="mbk-dialog-text">
      <slot name="content"></slot>
    </view>
  </view>
  <view class="mbk-dialog-btns">
    <form wx:if="{{cancelText}}" report-submit="true" bindsubmit="cancelAction">
      <button formType="submit">{{ cancelText }}</button>
    </form>
    <form wx:if="{{confirmText}}" report-submit="true" bindsubmit="confirmAction">
      <button formType="submit">{{ confirmText }}</button>
    </form>
  </view>
</view>
```

将 `header` 和 `content` 定义成 `slot`，是为了满足大部分的定制化需求。 

```js
Component({
  options: {
    multipleSlots: true // 在组件定义时的选项中启用多slot支持
  },
  properties: {
    confirmText: String,
    cancelText: String
  },

  methods: {
    confirmAction: function (e) {
      const formId = e.detail.formId || ''
      this.triggerEvent('confirm', { formId })
    },
    cancelAction: function () {
      const formId = e.detail.formId || ''
      this.triggerEvent('cancel', { formId })
    }
  }
})
```

这样就完成了组件的定义，下面看下如何在页面中使用：

```html
<component-dialog confirmText="扫摩拜拆红包" cancelText="发红包赚赏金" bind:confirm="dialogConfirm" bind:cancel="dialogCancel">
  <view slot="header">
    <image style="width: 146rpx; height: 146rpx;" src="/images/header.png" ></image>
  </view>
  <view slot="content">
    <!-- 折行解决方案 -->
    <view>你已经领取了 {{name}} 的红包</view>
    <view>不能重复领取哦</view>
    <view>快去给好友发红包赚赏金吧</view>
  </view>
</component-dialog>
```

`wxml` 语法暂不支持 `\n` 折行，这里通过使用多个 `<view>` 标签这种 hack 的方式来解决。

```js
Page({
  dialogConfirm: function (data) {
    console.log('用户点击了确认按钮，formId:', data.detail.formId)
  },
  dialogCancel: function (data) {
    console.log('用户点击了取消按钮，formId:', data.detail.formId)
  }
})
```

这样就完成了弹窗组件的创建和调用。

## 组件库

随着在组件方面的积累越来越多，可以把这些组件抽离出来，封装成组件库。

<div style="text-align: center">
  <img width="300" src="https://user-gold-cdn.xitu.io/2018/6/29/1644a90361f979fe?w=750&h=1334&f=jpeg&s=59911">
</div>

上图是摩拜单车封装的组件库。该组件库从功能上分为两部分：基础组件和业务组件。基础组件包含 button、弹窗、分享等，业务组件包含登录、月卡、banner 等。

摩拜单车组件库是一个独立的小程序，因为目前还没有开源，所以在后台设置了搜索不可见。不过我们正在逐步开展开源工作，比如添加定制皮肤、降低和业务的耦合关系等。

## 小结

本节最开始介绍了组件化的临时解决方案，因为其**可移植性不强**，所以推荐大家使用微信后来提出的自定义组件。后面以弹窗组件为示例，向大家详细讲述了如何开发组件。同时也建议大家封装组件库，可以做到多个小程序复用。在后续【主流框架使用案例】章节中，我们也会介绍 WePY 和 mpvue 框架中组件化的用法。
