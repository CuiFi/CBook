# 高级篇 1：WePY 设计细节

在 WePY 使用案例章节中，介绍了 WePY 的项目构建及基本使用，那 WePY 是怎么转换成小程序语言的呢？本节将从这一问题入手，来介绍 WePY 框架的设计以及对小程序细节的优化实现。

## 编译过程

项目代码最终是要转换成小程序语言的，先来看一下 WePY 编译流程图：

![wepy](https://user-gold-cdn.xitu.io/2018/7/11/1648957b9fe10aca?w=785&h=589&f=png&s=29043)

### wepy-cli

使用 `wepy-cli` 工具对每个 `.wpy` 文件进行编译，其目录结构如下：

<img src="https://user-gold-cdn.xitu.io/2018/7/1/16453d8899b602a9?w=375&h=800&f=jpeg&s=44398">

`compile.js` 是入口编译文件，其包含的 `compile` 方法会根据不同的文件后缀名分发到不同的编译方法，部分源码如下：

```js
// - src/compile.js
export default {
  ...
  compile(opath) {
    let src = cache.getSrc();
    let dist = cache.getDist();
    let ext = cache.getExt();
    let config = util.getConfig();

    if (!util.isFile(opath)) {
      util.error('文件不存在：' + util.getRelative(opath));
      return;
    }

    switch(opath.ext) {
      case ext:
        cWpy.compile(opath);
        break;
      case '.less':
        cStyle.compile('less', opath);
        break;
      case '.sass':
        cStyle.compile('sass', opath);
        break;
      case '.scss':
        cStyle.compile('scss', opath);
        break;
      case '.js':
        cScript.compile('babel', null, 'js', opath);
        break;
      case '.ts':
        cScript.compile('typescript', null, 'ts', opath);
        break;
      default:
        util.output('拷贝', path.join(opath.dir, opath.base));
      ...
    }
    ...
  }
    ...
}
```

除了 `cWpy` 执行了 `.wpy` 模板文件的编译，还有 `cStyle`、`cScript` 等编译方法。如果没有匹配到任何编译方法，就会走 `default` 逻辑，进行拷贝输出文件。

### .wpy 文件拆解

上面我们提到编译 `.wpy` 文件使用的是 `cWpy` 方法，代码再往前翻：

```js
import cWpy from './compile-wpy';
```

`compile-wpy.js` 便是核心编译 `.wpy` 文件方法所在，其同样拥有 `compile` 方法：

```js
// - src/compile-wpy.js
export default {
  ...
  compile (opath) {
    let filepath = path.join(opath.dir, opath.base);
    let src = cache.getSrc();
    let dist = cache.getDist();
    let wpyExt = cache.getExt();
    let pages = cache.getPages();

    let type = '';
    let relative = path.relative(util.currentDir, filepath);

    if (filepath === path.join(util.currentDir, src, 'app' + wpyExt)) {
      type = 'app';
      util.log('入口: ' + relative, '编译');
    } else if (pages.indexOf(relative) > -1) {
      type = 'page';
      util.log('页面: ' + relative, '编译');
    } else if (relative.indexOf(path.sep + 'components' + path.sep) > -1){
      type = 'component';
      util.log('组件: ' + relative, '编译');
    } else {
      util.log('Other: ' + relative, '编译');
    }
  }
  ...
}
```

`.wpy`主要分为三类：

- app：小程序入口文件`src/app.wpy`，最终编译为根目录下的 `app.js`
- page：页面文件，如 `src/pages/index.wpy`
- component：组件文件，如 `src/components/dialog.wpy`

我们继续往后看：

```js
// - src/compile-wpy.js
let wpy = this.resolveWpy(opath);
if (!wpy) {
  return;
}
```

分辨出不同的 `type` 后，直接转到 `resolveWpy` 方法，核心的处理逻辑如下：

```js
// - src/compile-wpy.js
export default {
  ...
  resolveWpy (xml, opath) {
    ...
    xml = this.createParser(opath).parseFromString(content);
    [].slice.call(xml.childNodes || []).forEach((child) => {
      const nodeName = child.nodeName;
      if (nodeName === 'style' || nodeName === 'template' || nodeName === 'script') {
        let rstTypeObj;

        if (nodeName === 'style') {
          rstTypeObj = {code: ''};
          rst[nodeName].push(rstTypeObj);
        } else {
          rstTypeObj = rst[nodeName];
        }

        rstTypeObj.src = child.getAttribute('src');
        rstTypeObj.type = child.getAttribute('lang') || child.getAttribute('type');
        if (nodeName === 'style') {
          // 针对于 style 增加是否包含 scoped 属性
          rstTypeObj.scoped = child.getAttribute('scoped') ? true : false;
        }
        ...
      }
    })
    ...
  }
  ...
}
```

其中 `xml` 是使用 `xmldom` 包提供的 `DOMParser` 处理完成，分析出三种模板处理类型（rstType）：

- template：wxml 模板，交给 `compile-template` 处理
- style：wxss 样式，交给 `compile-style` 处理
- script：js脚本，交给 `compile-script` 处理

这也是我们可以将三者写进同一个 `.wpy` 文件的原因。

## 运行原理

先看下入口文件 `app.wpy`：

```js
<script>
import wepy from 'wepy';
export default class extends wepy.app {
  ...
}
</script>
```

经过 `compile-script` 编译后结果如下：

```js
// dist/app.js
App(require('./npm/wepy/lib/wepy.js').default.$createApp(_default, {}));

// dist/pages/index.js
Page(require('./../npm/wepy/lib/wepy.js').default.$createPage(Index , 'pages/index'));
```

可以看出：

- 都有 `require('./npm/wepy/lib/wepy.js')`
- 分别调用了 `$createApp` 和 `$createPage` 核心方法

进而可以去 `./npm/wepy/lib/wepy.js` 探个究竟：

注意：WePY 为了支持 npm 安装包文件，会将 `node_mudules` 中的相应文件拷贝到项目编译后目录中，所以我们直接可以在 `dist` 中找到 `wepy.js` 。

```js
// -lib/wepy.js
import base from './base';
...
export default {
  event: event,
  app: app,
  component: component,
  page: page,
  mixin: mixin,

  $createApp: base.$createApp,
  $createPage: base.$createPage,
  ...
};
```

看来 `wepy.js` 只是提供所有核心方法的出口文件，想要找到 `$createApp` 和 `$createPage` 还得去 `base.js` 中：

```js
// -lib/base.js
import event from './event';
import util from './util';

let PAGE_EVENT = ['onLoad', 'onReady', 'onShow', 'onHide', 'onUnload', 'onPullDownRefresh', 'onReachBottom', 'onShareAppMessage', 'onPageScroll', 'onTabItemTap'];
let APP_EVENT = ['onLaunch', 'onShow', 'onHide', 'onError'];

export default {
  $createApp (appClass, appConfig) {
    let config = {};
    let app = new appClass();
    ...
    app.$wxapp = getApp();

    APP_EVENT = APP_EVENT.concat(appConfig.appEvents || []);
    PAGE_EVENT = PAGE_EVENT.concat(appConfig.pageEvents || []);

    APP_EVENT.forEach((v) => {
      config[v] = function (...args) {
        ...
      };
    });
    return config;
  },
  $createPage (pageClass, pagePath) {
    ...
    config.onLoad = function (...args) {
      ...
      args.push(secParams);

      [].concat(page.$mixins, page).forEach((mix) => {
        mix['onLoad'] && mix['onLoad'].apply(page, args);
      });

      page.$apply();
    };

    config.onShow = function (...args) {
      ...
      [].concat(page.$mixins, page).forEach((mix) => {
        mix['onShow'] && mix['onShow'].apply(page, args);
      });
      ...
      page.$apply();
    };

    PAGE_EVENT.forEach((v) => {
      if (v !== 'onLoad' && v !== 'onShow') {
        config[v] = (...args) => {
          ...
          page.$apply();

          return rst;
        };
      }
    });
    ...
    return $bindEvt(config, page, '');
  },
}
```

代码有点长，但结构还是很清晰，两个方法主要对小程序事件（`onLoad`, `onShow` 等）进行了封装，除了添加了一些自定义参数 `secParams` 并执行对应的事件函数外，有一行我们需要注意：

```js
page.$apply();
```

这是 WePY 进行数据检查的方式，如果熟悉其他 web 开发框架，对其应该不陌生。

## 数据绑定

WePY 使用脏数据检查对原生小程序 `setData` 进行封装，在函数运行周期结束时执行脏数据检查。如果在异步函数中更新数据时，则需要手动执行 `$apply()`，官方的流程图如下：

![](https://user-gold-cdn.xitu.io/2018/7/6/1646fdcbdbbe6ac0?w=420&h=800&f=jpeg&s=35892)

`$apply` 定义：

```js
$apply (fn) {
  if (typeof(fn) === 'function') {
    fn.call(this);
    this.$apply();
  } else {
    if (this.$$phase) {
      this.$$phase = '$apply';
    } else {
      this.$digest();
    }
  }
}
```

`$$phase` 标识是否有 **脏数据检查** 在运行，如果没有，则执行 `$digest()`。

```js
$digest() {
  let k;
  let originData = this.$data;
  this.$$phase = '$digest';
  this.$$dc = 0;
  while (this.$$phase) {
    this.$$dc++;
    if (this.$$dc >= 3) {
        throw new Error('Can not call $apply in $apply process');
    }
    ...
    this.$$phase = (this.$$phase === '$apply') ? '$digest' : false;
  }
}
```

`$digest()` 执行时，主要是遍历 `originData`，将 `originData[k]` 和 `this[k]` 做对比，如果不一样，放到 `readyToSet` 中，在循环之后，统一执行 `setData` 方法。

最后，再检查 `$$phase` 是否有被设置为 `$apply'`，如果是，则再做一次数据检查。

同时，WePY 也实现了 `computed` 和 `watch`：

```js
$digest () {
  ...
  let readyToSet = {};
  if (this.computed) {
    for (k in this.computed) { // If there are computed property, calculated every times
      let fn = this.computed[k], val = fn.call(this);
      if (!util.$isEqual(this[k], val)) { // Value changed, then send to ReadyToSet
        readyToSet[this.$prefix + k] = val;
        this[k] = util.$copy(val, true);
      }
    }
  }
  ...
}
```

`readyToSet` 表示需要被设置的数据，每次数据检查的时候都会判断 `computed` 属性，如果有就执行计算函数 `this.computed[k]` ，再对计算结果和原始值进行对比。

`watch` 也是类似：

```js
$digest () {
  ...
  let readyToSet = {};
  ...
  for (k in originData) {
    if (!util.$isEqual(this[k], originData[k])) { // compare if new data is equal to original data
      // data watch trigger
      if (this.watch) {
        if (this.watch[k]) {
          if (typeof this.watch[k] === 'function') {
            this.watch[k].call(this, this[k], originData[k]);
          } else if (typeof this.watch[k] === 'string' && typeof this.methods[k] === 'function') {
            this.methods[k].call(this, this[k], originData[k]);
          }
        }
      }
      // Send to ReadyToSet
      readyToSet[this.$prefix + k] = this[k];
      ...
    }
  }
}
```

同样是对 `readyToSet` 值进行扩充，最终统一进行 `setData`：

```js
if (Object.keys(readyToSet).length) {
  this.setData(readyToSet, () => {
    ...
  });
}
```

注意：上面的 `setData` 并不是直接调用了小程序中的方法，而是进行了一层封装：

```js
setData (k, v) {
  ...
  for (t in k) {
    let noPrefix = t.replace(reg, '');
    this.$data[noPrefix] = util.$copy(k[t], true);
    // 1.9.2 do not allow to set a undefined value
    if (k[t] === undefined) {
      delete k[t];
    }
  }
  ...
  return this.$root.$wxpage.setData(k);
}
```

## 小程序细节优化

### 请求队列

微信限制了**请求**的并发数量，最开始限制在5个，后来放开至10个。通过数据分析发现，摩拜单车小程序的并发请求数10个以上的，平均占比在0.5%左右。WePY解决方案和我们是一致的：采用请求队列控制并发数量，当并发请求超过10个的时候，后续请求延迟执行，具体实现可以参考 [app.js](https://github.com/Tencent/wepy/blob/1.7.x/packages/wepy/src/app.js)。

### 事件优化，支持 Promise 写法

`$initAPI` 方法实现的该功能，主要利用 `Object.defineProperty` 的 get 方法，将返回封装为 Promise，具体实现可以参考 [app.js](https://github.com/Tencent/wepy/blob/1.7.x/packages/wepy/src/app.js)。

### 支持 npm 包

在上面的运行原理分析中，我们有提到，是将 node_mudules 中的入口文件拷贝到 dist/npm 目录中实现的。接下来，我们将以加载 **wepy-redux** 的 npm 包为例，从 wepy-cli 的源码中分析这一过程的实现方式：

1、 项目编译过程中， 会将 import 语法转换为 require

```js
// 转换前 `import 'wepy-redux'`
var _wepyRedux = require('wepy-redux');
```

2、 递归遍历代码中的 require， `resolveDeps` 实现替换 require 为**相对路径**

```js
// - src/compile-script.js
exports.default = {
  ...
  resolveDeps: function resolveDeps(code, type, opath) {

    ...

    return code.replace(/require\(['"]([\w\d_\-\.\/@]+)['"]\)/ig, function (match, lib) {

        var resolved = lib;

        var target = '',
            source = '',
            ext = '',
            needCopy = false;

        if (config.output === 'ant' && lib === 'wepy') {
            lib = 'wepy-ant';
        }
        lib = _resolve2.default.resolveAlias(lib);
        if (_path2.default.isAbsolute(lib)) {
          ...
        } else if (lib[0] === '.') {
          ...
        } else if (lib.indexOf('/') === -1 || lib.indexOf('/') === lib.length - 1 || lib[0] === '@' && lib.indexOf('/') !== -1 && lib.lastIndexOf('/') === lib.indexOf('/')) {

              var mainFile = _resolve2.default.getMainFile(lib);

              if (!mainFile) {
                  throw Error('找不到模块: ' + lib + '\n被依赖于: ' + _path2.default.join(opath.dir, opath.base) + '。\n请尝试手动执行 npm install ' + lib + ' 进行安装。');
              }
              npmInfo = {
                  lib: lib,
                  dir: mainFile.dir,
                  modulePath: mainFile.modulePath,
                  file: mainFile.file
              };
              source = _path2.default.join(mainFile.dir, mainFile.file);
              target = _path2.default.join(npmPath, lib, mainFile.file);

              lib += _path2.default.sep + mainFile.file;
              ext = '';
              needCopy = true;
        } else {
          var o = _resolve2.default.walk(lib);
          source = _path2.default.join(o.modulePath, lib);
          target = _path2.default.join(npmPath, lib);
          ext = '';
          needCopy = true;
        }
        ...
        resolved = resolved.replace(/\\/g, '/').replace(/^\.\.\//, './');
        return 'require(\'' + resolved + '\')';
    });
  },
  ...
}
```

其中，`needCopy` 标记了需要拷贝的依赖文件，`checkBuildCache` 检查项目根目录下 .wepycache 文件内是否已有 wepy-redux 的拷贝记录，没有的话就将 node_modules 文件下 wepy-redux 文件拷贝至 dist/npm 下并保存记录。

```js
if (needCopy) {
  if (!_cache2.default.checkBuildCache(source)) {
      _cache2.default.setBuildCache(source);
      _util2.default.log('依赖: ' + _path2.default.relative(process.cwd(), target), '拷贝');
      var newOpath = _path2.default.parse(source);
      newOpath.npm = npmInfo;
      _this.compile('js', null, 'npm', newOpath);
  }
}
```

针对 npm 依赖包内代码做了一些 hack 处理。

```js
// - src/compile-script.js
exports.default = {
  ...
  npmHack: function npmHack(opath, code) {
      code = code.replace(/process\.env\.NODE_ENV/g, JSON.stringify(process.env.NODE_ENV));
      switch (opath.base) {
          case 'lodash.js':
          case '_global.js':
              code = code.replace('Function(\'return this\')()', 'this');
              break;
          case '_html.js':
              code = 'module.exports = false;';
              break;
          case '_microtask.js':
              code = code.replace('if(Observer)', 'if(false && Observer)');

              code = code.replace('Promise && Promise.resolve', 'false && Promise && Promise.resolve');
              break;
          case '_freeGlobal.js':
              code = code.replace('module.exports = freeGlobal;', 'module.exports = freeGlobal || this;');
      }
      var config = _util2.default.getConfig();
      if (config.output === 'ant' && opath.dir.substr(-19) === 'wepy-async-function') {
          code = '';
      }
      return code;
  },
  ...
}
```

3、 require 路径被替换为相对路径，实现引入 npm 包的功能。

```js
var _wepyRedux = require('./../npm/wepy-redux/lib/index.js');
```

注：本文分析的源码以 wepy@1.7.1 版本为准。


## 小结

阅读源码并不是很高深的事情，一遍遍看下去，每次都可能有不同的收获。随着小程序越发成熟，框架也越来越多，不管 WePY 今后发展如何，至少目前来看，还是很优秀也是值得考虑使用的框架。