# 实战案例解析 9：图表

图表也是小程序经常使用的一个功能，比如微信官方小程序-公众平台助手就是支持图表化展示公众号数据。在传统 web 开发中，基础能力有 svg 标签、canvas 标签，
框架层面常用的有 Highcharts、ECharts、chart.js、d3、raphaeljs 等。

本文会先从最原始的 canvas 标签和 API 使用开始，带着大家完成 3 个简单案例：绘制文本、绘制线和绘制虚线，之后会完成一个稍微复杂的柱状图，后面我们会主要介绍 ECharts 在小程序中的应用，同时也会提到如何在小程序中使用 Highcharts。

> 本节案例相关代码已提交至 GitHub ，内容持续更新，可以自行下载进行本地调试，[点此访问](https://github.com/mobikeFE/xiaoce-demo)。

## 使用内置的 canvas

小程序支持 `canvas`，可以用它来绘制图表。具体使用方式如下：

在 wxml 中：使用 `canvas` 标签，具体可以查看[官网](https://developers.weixin.qq.com/miniprogram/dev/component/canvas.html#canvas)。

这里我们设置了宽高，同时指定了唯一标识 `canvas-id` 为 `mobike`。

```html
<canvas style="width: 300px; height: 400px;" canvas-id="mobike">
</canvas>
```

首先，在 js 中调用 `wx.createContext`（虽然官方提示不建议使用，这里还是可以放心地使用） 创建并获取绘图上下文 `context`。

```js
var context = wx.createContext();
```

中间参照需求可以绘制线、绘制文本以及其他组合等

最后调用 `wx.drawCanvas` （虽然官方提示不建议使用，这里还是可以放心地使用），一般我们在绘制柱状图等场景中会设置 `reserve` 为 `true`。具体可以查看[官网](https://developers.weixin.qq.com/miniprogram/dev/api/canvas/draw-canvas.html)。

代码片段如下所示：

```js
wx.drawCanvas({
  canvasId: 'mobike',  // 对应上了上面的 mobike
  actions: context.getActions()
});
```

### 绘制文本

下面我们带大家实现一个绘制文本的功能，效果如图所示：

![drawText](https://user-gold-cdn.xitu.io/2018/7/22/164c21ef1d28acf8?w=417&h=297&f=jpeg&s=18247)

我们封装了一个函数 `drawText`，内部主要使用了 `fillText`，具体可以查看[官网](https://developers.weixin.qq.com/miniprogram/dev/api/canvas/fill-text.html)。

```js
function drawText(context, text, x, y, size, color) {
  context.beginPath();
  context.setFontSize(size);
  context.setFillStyle(color);
  context.fillText(text, x, y);
  context.closePath();
  context.stroke();
}
```

调用方式如下：

```js
drawText(context, 'mobike', 0, 100, 18, 'red')
```




### 绘制线

这里我们要实现绘制线的功能，效果如图所示：

![drawLine](https://user-gold-cdn.xitu.io/2018/7/22/164c21ef2b73aca0?w=415&h=407&f=jpeg&s=15953)

我们封装了一个函数 `drawText`，内部主要使用 `moveTo` 和 `lineTo`，具体可以查看[官网](https://developers.weixin.qq.com/miniprogram/dev/api/canvas/line-to.html)。

```js
function drawLine(context, x, y, x1, y1, width, color){ 
  context.beginPath(); 
  context.setStrokeStyle(color);
  context.setLineWidth(width);  
  context.moveTo(x, y);  
  context.lineTo(x1, y1);  
  context.stroke(); 
}
```

调用方式如下：

```js
drawLine(context, 0, 18, 100, 18, 2, 'red')
```



### 绘制虚线

这里我们要实现绘制虚线的功能，效果如图所示：

![drawDashLine](https://user-gold-cdn.xitu.io/2018/7/22/164c21ef22cc3352?w=413&h=248&f=jpeg&s=15943)

最核心的还是计算虚线的长度和段数。

```js
function drawDashLine(context, x1, y1, x2, y2, dashLength, width, color){
  context.beginPath();
  context.setStrokeStyle(color);
  context.setLineWidth(width);
  //得到横向的宽度;
  var xpos = x2 - x1;

  //得到纵向的高度;
  var ypos = y2 - y1; 

  // 分几段
  // Math.sqrt() 函数返回一个数的平方根
  // Math.floor() 返回小于或等于一个给定数字的最大整数
  var numDashes = Math.floor(Math.sqrt(xpos * xpos + ypos * ypos) / dashLength); 

  // 利用正切获取斜边的长度除以虚线长度，得到一共可分为多少段
  for(var i = 0; i < numDashes; i++){
    if(i % 2 === 0){
      // 有了横向的宽度和分为多少段，得出每一段的长，起点 + 每段长度 * i，终点 + 每段长度 * i
      context.moveTo(x1 + (xpos/numDashes) * i, y1 + (ypos/numDashes) * i); 
    } else {
      context.lineTo(x1 + (xpos/numDashes) * i, y1 + (ypos/numDashes) * i);
    }
  }
  context.closePath();
  context.stroke();
}
```


调用方式如下：

```js
drawDashLine(context, 0, 80, 80, 80, 10, 2, 'red');
```




### 复杂型 - 柱状图

我们先看一下效果图：

![柱状图](https://user-gold-cdn.xitu.io/2018/7/22/164c21ef1b897604?w=417&h=473&f=jpeg&s=26162)

然后分析一下要做什么：

* 横轴 -- 绘制线
* 纵轴 -- 绘制线
* 横轴文字  -- 绘制文字
* 纵轴文字  -- 绘制文字
* 柱  -- 绘制线

横轴和纵轴的间距设置如下：

1. 横轴间距是固定的，设置为 1
2. 纵轴间距是固定的，设置为 100


先在 data 中定义数据：

```js
Page({
  data:{
    x: [
      {date: '1月', value: 223},
      {date: '2月', value: 278},
      {date: '3月', value: 136},
      {date: '4月', value: 156},
      {date: '5月', value: 69},
      {date: '6月', value: 74},
      {date: '7月', value: 45}
    ]
  }
})
```

然后，封装一个最外层的函数，命名为 `drawCanvas`：

```js
function drawCanvas(context, datas, startX, startY, eachSpacing_x, eachSpacing_y, end_x, points, colors){
  // ...
}
```

这里传入的参数中，我们设置了 2 个初始值：

* startX = 30
* startY = 350

函数内部，我们需要循环传入的数据，获取横轴的 label 值：

```js
var xLabels = [];
var xVals = [];
var yVals = [];
var yRange = 0;
datas.forEach(function(item, index) {
  // 数据对象里面是 date
  xLabels.push(item.date);
  // 数组 push，index 从 0 开始
  // 30 + (index + 1) * 30
  xVals.push(startX + (index +1) * eachSpacing_x);

  // 350 - item.value * 40 / 100
  // 350 - 223 * 40 / 100 ==> 260.8
  // 350 - 278 * 40 / 100 ==> 238.8
  // 350 - 136 * 40 / 100 ==> 295.6
  // ...
  yVals.push(startY - item.value * eachSpacing_y / 100);

  // 如果小于 item.value 就重置一下
  if(yRange < item.value){
    yRange = item.value;
  }
}
```

`xLabels` 最后解析为：

```js
["1月份", "2月份", "3月份", "4月份", "5月份", "6月份", "7月份"]
```

`xVals` 最后解析为：
```js
[60, 90, 120, 150, 180, 210, 240]
```

`yVals` 最后解析为：
```js
[260.8, 238.8, 295.6, 287.6, 322.4, 320.4, 332]
```

yRange 默认从 0 开始，循环的时候最终会返回最大的 `item.value`，这里是 278


```js
// 下舍入
// Math.floor(yRange/100) ==> 2
// yRange 是 300
yRange = (Math.floor(yRange/100) + 1) * 100;

// 350 - (300/100) * 40 ==> 230
var endY = startY - (yRange/100) * eachSpacing_y;
```


我们现在绘制横坐标：


```js
// startX ==> 30
// startY ==> 350
drawLine(context, startX, startY, 280, startY, 2, "#cccccc");
```

然后是横坐标上面的文本：

```js
// startY ==> 350
drawText(context, xLabels, xVals, startY + 15, 12, 'black');
```

注意：

> 这里的 xLabels 和 xVals 和之前不一样，都是`数组`类型，所以需要扩展 drawText

改造方式如下：

```js
if (x instanceof Array) {
  text.forEach(function(item, index) {
    context.fillText(item, x[index], y);
  });
}
```


然后再绘制纵坐标


```js
// startX ==> 30
// startY ==> 350
drawLine(context, startX, startY, startX, endY, 2, "#cccccc");
```

然后是纵坐标上面的文本：


```js
var yVals = [];
for(var i = 0; i < datas.length; i++){
  if(i <= (yRange/100)){
    // eachSpacing_y ==> 40
    yVals.push(startY - eachSpacing_y * i + 4);
  }
}

yVals.forEach(function(item, index) {
  yLabels.push(index * 100);
});
```


```js
drawText(context, yLabels, startX - 26, yVals, 12, 'black');
```

注意：

> 这里的 yLabels 和 yVals 和之前不一样，都是`数组`类型，所以同样需要扩展 drawText


改造方式如下：

```js
if (y instanceof Array) {
  text.forEach(function(item, index) {
    context.fillText(item, x, y[index]);
  });
}
```



最后调用的方式：

```js
drawCanvas(context, this.data.x, 30, 350, 
this.data.eachSpacing_x, this.data.eachSpacing_y, 
this.data.endX, this.data.points1, '#EF5B49');
```


## 使用 ECharts

在 web 开发中，ECharts 可以实现很丰富的图表功能，同时，它也封装了小程序版本，具体包内容可以在此[下载](https://github.com/ecomfe/echarts-for-weixin)。

### 拷贝文件

直接拷贝 `echarts-for-weixin/ec-canvas` 内容至小程序目录，可以放在 `components` 目录，当做组件使用：

![echarts](https://user-gold-cdn.xitu.io/2018/7/22/164c21ef20ad63bb?w=448&h=462&f=png&s=38958)

### 页面中组件化使用

在 pages/echarts/index.json 中声明引用组件：

```json
{
  "usingComponents": {
    "ec-canvas": "../../components/ec-canvas/ec-canvas"
  }
}
```

在 pages/echarts/index.wxml 中引入：

```html
<view class="container">
  <ec-canvas id="echarts-dom" canvas-id="echarts" ec="{{ ec }}"></ec-canvas>
</view>
```

注意：

需要设置 canvas 外层元素的 WXSS 样式，保证 canvas 容器足够，可以使用**绝对定位**：

```css
.container {
  position: absolute;
  top: 0;
  bottom: 0;
  left: 0;
  right: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: space-between;
  box-sizing: border-box;
}
```

`index.js` 结构大致如下：

```js
function init(canvas, width, height) {
  const chart = echarts.init(canvas, null, {
    width: width,
    height: height
  });
  canvas.setChart(chart);

  var option = {
    ...
  };
  chart.setOption(option);
  return chart;
}

Page({
  data: {
    ec: {
      onInit: init
    }
  }
});
```

### 对比 web 中使用

对比下，web 中的实现方法：

```html
<div id="container" style="height: 100%"></div>
<script type="text/javascript" 
src="http://echarts.baidu.com/gallery/vendors/echarts/echarts.min.js">
</script>
```

```js
var dom = document.getElementById("container");
var myChart = echarts.init(dom);
var option = {
  ...
}
myChart.setOption(option, true);
```

因为小程序对 DOM 操作支持不完善，所以在小程序中使用去除了 DOM 操作部分，改用向 data 赋值方式初始化 ECharts。

### 完整示例

看完以上介绍，我们实现一个简单的 “摩拜单车全国车辆分布图”（数据没有任何参考意义）：

```js
import * as echarts from '../../components/ec-canvas/echarts';

let chart = null;

function initChart(canvas, width, height) {
  chart = echarts.init(canvas, null, {
    width: width,
    height: height
  });
  canvas.setChart(chart);

  var option = {
    color: ['#f05b48', '#34b9c2', '#ffcc39'],
    legend: {
      data: ['经典版', '轻骑版', '三文鱼']
    },
    xAxis: [
      {
        type: 'value',
        axisLine: {
          lineStyle: {
            color: '#999'
          }
        },
        axisLabel: {
          color: '#666'
        }
      }
    ],
    yAxis: [
      {
        type: 'category',
        axisTick: { show: false },
        data: ['北京', '上海', '南京'],
        axisLine: {
          lineStyle: {
            color: '#999'
          }
        },
        axisLabel: {
          color: '#666'
        }
      }
    ],
    series: [
      {
        name: '经典版',
        type: 'bar',
        data: [300, 270, 340]
      },
      {
        name: '轻骑版',
        type: 'bar',
        data: [120, 102, 141]
      },
      {
        name: '三文鱼',
        type: 'bar',
        data: [200, 312, 231]
      }
    ]
  };

  chart.setOption(option);
  return chart;
}

Page({
  data: {
    ec: {
      onInit: initChart
    }
  }
});
```

最终效果如图：

![echarts demo mobike](https://user-gold-cdn.xitu.io/2018/7/22/164c21ef1b79d268?w=824&h=1460&f=png&s=95264)

demo 源码可以从[这里](https://github.com/mobikeFE/xiaoce-demo)下载到。


### 与 Highcharts 的不同

由于 Highcharts 底层依赖 `svg` 来画图，但是微信小程序不支持 `svg`，所以只能通过 `web-view` 组件嵌入网页的方式来实现。


## 小结

目前使用 ECharts 开发小程序图表对开发者来说方便一些，一方面已经很成熟，另一方面也提供了小程序组件使用，不过包体积确实不小，当然官方支持在线。当然如果你对 canvas API 很熟悉了，也可以自己封装一套属于自己的 mini 版本。


## 扩展阅读

* [Math.sqrt](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Math/sqrt)
* [Math.floor](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Math/floor)
