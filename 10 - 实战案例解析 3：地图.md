# 实战案例解析 3：地图

小程序通过 map 组件实现地图相关功能，本文将结合常见的地图使用场景介绍 map 组件的常用属性和相关地图组件控制。

> 本章案例相关代码已提交至 github ，内容持续更新，可以自行下载进行本地调试，[点此访问](https://github.com/mobikeFE/xiaoce-demo)

## map 组件

map 组件算是小程序最复杂的组件之一，组件相关属性可查阅[官方文档](https://developers.weixin.qq.com/miniprogram/dev/component/map.html)，下面我们将结合具体使用场景来介绍 map 组件的相关属性。

以摩拜单车地图的使用场景为例：

<div style="text-align: center;margin-bottom: 20px;">
    <img style="width: 350px;" src="https://user-gold-cdn.xitu.io/2018/7/2/16458447bf123b08?w=735&h=1125&f=png&s=593453"></img>
</div>

- 进入地图时，显示用户位置、周边单车和禁停区位置
- 拖动地图时，更新地图中心点、周边单车和禁停区位置
- 城市禁停与运营区围栏
- 地图归位时回到初始用户位置
- 地图上提供活动banner、用户中心等功能入口
- 地图多业务tab（单车、电单车）

map 组件的使用示例如下所示: 
```html
<map id="map" show-location longitude="{{userPos.longitude}}" 
latitude="{{userPos.latitude}}" markers="{{markers}}" controls="{{controls}}" 
polyline="{{polyline}}" bindtap="mapTap" bindmarkertap="markertap" 
bindcontroltap="controltap" bindregionchange="regionchange"></map>
```

## 组件属性

### markers

> 定义地图上标记点，需要指定**标记点经纬度**。

markers 通过指定经纬度定位在地图组件上相应的点，能够跟随地图拖动，符合上述单车位置显示的使用场景。实际项目中即通过设置 markers 属性实现当前位置周边单车位置显示，配合组件 `bindmarkertap` 可处理地图 markers 的点击事件（事先需定义对应 id ），实现用户与周边显示单车的相关交互功能。
```js
Page({
  data: {
    markers: []
  }
  markertap (e) {
    // 获取点击marker的id
    const markerId = e.markerId
    // 获取点击marker的属性
    const marker = this.data.markers[markerId]
    ...
  }
})
```

markers 显示层级

业务场景中涉及多个单车 marker 显示，单车密度较大时会在地图组件上堆叠在一起显示，那么是否可以控制它们的显示层级呢？解决方案是调整 markers 数组中的顺序，地图按照数组顺序渲染 markers，将层级最高的 marker 放在数组最后即可实现显示层级最高。

```js
// 封装地图 markers 处理方法
function renderNearBy (data) {
  let bikes = []
  let originalBikes = data.originalBikes
  originalBikes && originalBikes.forEach((item, key) => {
    item.latitude = item.distY
    item.longitude = item.distX
    item.width = 50
    item.height = 50
    item.id = originalBikes.length - 1 - key
    // 最近的单车marker设置callout属性
    if (key == 0) {
      item.callout = {
        content: '离我最近',
        color: '#FFFFFF',
        fontSize: 10,
        padding: 8,
        borderRadius: 12,
        bgColor: '#000000',
        display: 'ALWAYS'
      }
    }
    bikes.push(item)
  })
  // 翻转数组顺序，实现最近的车渲染层级最高
  data.callback(bikes.reverse())
}
```

### controls

定义相对地图的**固定位置**，不随地图移动。

适用于定义覆盖在地图上的固定入口，如上述用户中心入口。与 markers 组件类似，配合 `bindcontroltap` 可处理地图 controls 组件的点击事件（事先需定义对应 id ），实现相关交互功能。
```js
Page({
  data: {
    controls: []
  }
  controltap (e) {
    // 获取点击control的id
    const markerId = e.controlId
    // 根据id进行后续处理
    ...
  }
})
```

controls 功能也可通过 cover-view 组件实现。且官方文档也有说明，controls 即将废弃，建议直接使用 [cover-view](https://developers.weixin.qq.com/miniprogram/dev/component/cover-view.html)。

### cover-view

覆盖在原生组件之上的文本视图，只支持嵌套 cover-view、cover-image。

前面已经介绍过，cover-view 可实现地图 controls 的功能，但 cover-view 不止支持覆盖 map 组件，实际上，cover-view 可覆盖的原生组件包括 map、video、canvas、camera 等。

使用 cover-view 支持多业务 tab 切换:
```html
<map>
  <cover-view>
    <cover-view class="tabs">
      <cover-view data-idx="{{index}}" wx:for="{{tabs}}" wx:key class="tab-item {{index == active ? 'active' : ''}}" bindtap="tap">
        {{item}}
      </cover-view>
    </cover-view>
  </cover-view>
</map>
```
```js
Page({
  data: {
    active: 0
  },
  tap(e) {
    let idx = e.currentTarget.dataset.idx
    if (idx !== this.data.active) {
      this.setData({
        active: idx
      })
      // 针对具体 tab 的业务处理
      ...
    }
  }
})
```

项目中可以封装 tab 组件来使用，组件的具体使用方法可参考[组件化](https://juejin.im/book/5b30c3b351882574957a788f/section/5b30c446e51d4558d35fbd80)。

### polyline

> 指定一系列坐标点，从数组第一项连线至最后一项。

定义地图上画线，支持定义线条虚线，颜色，宽度，粗细，厚度属性，不支持填充颜色。另外，`circle` 属性支持定义填充颜色，但顾名思义，只支持在地图上画圆。那么如何在地图定义多条 polyline ？

显然，polyline 满足地图禁停区和运营区围栏画线的业务需求，但实际应用中，围栏肯定不止一个，正确的使用方法是将 polyline 定义为数组，数组中每项为一条独立画线，若需定义封闭画线，只要指定 `points`数组的首项与末项为同一经纬度即可。
```js
// 定义地图polyline
Page({
  onShow() {
    const polyline = []
    // 经纬度数组
    const points = [
      {
        latitude: "39.907211",
        longitude: "116.399559"
      },
      {
        latitude: "39.907211",
        longitude: "116.399559"
      },
      ...
    ]
    polyline.push({
      points,
      color: '#FF0000DD',
      width: 8
    })
    this.setData({
      polyline
    })
  }  
})
```

### bindregionchange

> 地图视野发生变化时触发

地图拖动会多次触发 `bindregionchange` 事件，在回调中可以在地图视野变化时更新地图渲染。

```html
<map id="map" bindregionchange="regionchange"></map>
```
```js
Page({
  data: {},
  regionchange(e) {
  }
})
```

## 地图组件控制

小程序提供了 [wx.createMapContext](https://developers.weixin.qq.com/miniprogram/dev/api/api-map.html#wxcreatemapcontextmapid) 创建并返回 mapContext 对象，我们根据官方文档中 mapContext 支持的方法列表依次展开讨论它们在摩拜单车小程序中的应用场景。

### getCenterLocation

> 获取当前地图中心的经纬度

在拖动地图时，需要获取地图中心点周边的单位位置信息，聪明的你肯定已经想到地图中心点的经纬度就是通过 `getCenterLocation` 来获取。

### moveToLocation

> 将地图中心移动到当前定位点，需配合 map 组件的 `show-location` 使用

拖动地图查看单车信息后，地图归位的功能就是通过 `moveToLocation` 来实现的。

注意：`moveToLocation` 引起地图视野变化时同样会触发 `regionchange` 事件。

### includePoints

> 定义显示在可视区域内的坐标点列表

通过指定 `points` 数组定义显示在可视区域内的坐标点列表，使单车全部显示在地图可见区域时可调用，使用方法比较简单，需要注意的一点是在定义 `padding` 属性时，iOS 支持定义[上, 右, 下, 左]的边距，而安卓只能识别数组第一项，上下左右的 `padding` 一致，因此 `padding` 第一项不能超过屏幕宽度的一半，否则容易引起地图组件崩溃。

## 优化

### 地图拖动

```html
<map id="map" bindregionchange="regionchange"></map>
```

地图视野变化开始和结束时分别会触发 bindregionchange 回调，可以在回调中判断 `e.type === 'end'` 时，再进行地图渲染更新。

```js
Page({
  data: {},
  regionchange(e) {
    if(e.type === 'end') {
      // 更新地图渲染数据
    }
  }
})
```

此外，moveToLocation、 includePoints 也会触发 bindregionchange 回调。

### 地图渲染更新

includePoints 引起地图视野变化时同样会触发 bindregionchange 回调，但在完成地图渲染数据更新后，有时并不希望 includePoints 时再次更新渲染数据。因无法通过 e.type 判断，所以可设置一个 flag，includePoints 触发 bindregionchange 回调时不更新渲染数据。


## 小结

本小节通过介绍 map 组件的相关属性和地图组件控制，让读者对 map 组件的使用有比较清晰的了解。结合摩拜单车的使用场景，读者可以了解到组件的支持功能和相关实现细节，以及功能实现中可能遇到的问题。
