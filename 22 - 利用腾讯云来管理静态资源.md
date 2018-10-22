# 利用腾讯云来管理静态资源

由于小程序有包体积限制，同时随着业务的飞速增长，特别是运营活动的频繁更新，如何减少包体积慢慢开始变成一个极需优化的问题。

目前摩拜的静态资源是托管在**腾讯云**上的，通过它我们整个工程的静态资源都在 CDN 上，包内的体积自然就减少了一些。

下面我们会带着大家一步一步从腾讯云注册开始、再到配置以及工程化完成文件上传。

> 如果你不需要使用腾讯云来托管静态资源，可以忽略本节腾讯云注册和配置部分的内容，但是一些上传脚本操作可以适当参考。为了方便大家学习，本节的部分代码示例我们放到了 GitHub 上面，可以点击直接下载 [xiaoce-qcloud-demo](https://github.com/mobikeFE/xiaoce-qcloud-demo)。


## 腾讯云注册与配置

腾讯云包含很多强大的功能，我们这里主要关注：对象存储和 CDN 加速。更多信息可查看[腾讯云官网](https://cloud.tencent.com/)。

对象存储可以便捷地云端托管我们的静态资源，CDN 加速在文件下载、分发的场景中起到了很重要的作用。同时对于我们特别关注的安全性：腾讯云账号分为主账号和 CAM 用户，CAM 用户包含协作者、子用户、消息接收人。所以对于静态资源操作方面还是很安全的，具体可以查看[官方链接](https://cloud.tencent.com/document/product/598/14985)。

### 腾讯云注册

注册方式非常简单，如有需要可以查看[官方说明文档](https://cloud.tencent.com/document/product/378/17985)，支持四种注册方式，如图所示：

![注册](https://user-gold-cdn.xitu.io/2018/7/9/1647d7e01d107b21?w=827&h=542&f=jpeg&s=50639)

一般我们会选择 **QQ 登录**，具体操作大家可以直接点击[注册入口](https://cloud.tencent.com/register)。


### 腾讯云配置

这里我们会从基础信息、配置 Bucket、资源文件监控以及配置域名这几个方向介绍，让大家快速了解和使用腾讯云的配置，为后面调用接口进行上传操作做好准备。

#### 基础信息

因为调用上传 SDK 会用到以下信息：

* APPID

这是成功申请腾讯云账户后，系统分配的账户标识之一。
在[账号信息页面](https://console.cloud.tencent.com/developer)可以看到

* SecretId 与 SecretKey

在上传的时候需要这两个参数，如果你还没有设置，点击**新建密钥**，如图所示：

![云API密钥-API密钥管理](https://user-gold-cdn.xitu.io/2018/7/9/1647d80b0d684ccf?w=1438&h=382&f=jpeg&s=79715)

> 注册成功之后，页面内不会直接有「云 API 秘钥」，需要在「云产品」-> 「管理工具」里找到「云 API 秘钥」并点击使用，页面左侧才会出现「云 API 秘钥」。


#### 配置对象存储

在「云产品」-> 「存储」里找到「对象存储」，点击进入「对象存储」，新建 Bucket（存储桶）的操作如下图所示。

![新建 Bucket](https://user-gold-cdn.xitu.io/2018/7/11/16486e19bca86e7a?w=731&h=538&f=jpeg&s=53446)

我们需要输入几项内容。

##### 名称

一般我们取名为英文字母，比如这里的 mobike。它要求：仅支持小写字母、数字和 - 的组合，不能超过 40 字符。

##### 地域

目前腾讯云支持的地域如下图所示：

![地域](https://user-gold-cdn.xitu.io/2018/7/9/1647d7e01d3befb1?w=150&h=157&f=jpeg&s=15816)

一般建议根据业务场景就近选择，比如我们这里选择了**北京**，可以提高对象上传、下载速度。

> Bucket 创建后`不能修改`所属地域。

##### 访问权限

腾讯云支持私有和公用，我们目前会针对**敏感**文件使用私有，比如认证类的文件。

##### CDN 加速

默认是关闭的，这里我们设置开启，腾讯云提供免费额度，可以[参考这里](https://user-gold-cdn.xitu.io/2018/7/9/1647d7e01eea429c?w=912&h=252&f=jpeg&s=40168)。

很简单的操作之后，一个用来存储我们资源文件的 Bucket 就创建完成啦。

#### 资源文件监控

在腾讯云我们可以通过监控报表，清晰地看到如下数据。

![免费](https://user-gold-cdn.xitu.io/2018/7/9/1647d7e01d5381bf?w=1217&h=635&f=jpeg&s=91017)

1. CDN 回源统计的数据也对我们很有帮助
2. 读请求次数会提醒我们某一天业务是否有大流量激增
3. 写请求次数会提醒我们某一天文件上传是否有问题

同时它还支持返回码统计，我们可以看到多少链接出现了死链（404返回码）等等。

![免费](https://user-gold-cdn.xitu.io/2018/7/9/1647d7e0459534a2?w=1134&h=337&f=jpeg&s=35044)


#### 配置域名

默认域名在创建好存储桶后，可通过对象存储控制台的存储桶「域名管理」查看。

默认域名规则如下，一般有 2 个：

1. 适用 JSON API 的：

```
<bucketname>-<APPID>.cos<region>.myqcloud.com
```

2. 适用 XML API 的：

```
<bucketname>-<APPID>.cos.<region>.myqcloud.com
```

参照上面我们创建的 Bucket，APPID 为 1252758967 的用户创建了一个名为 mobike、所属地域为北京的存储桶，其默认域名则为：

JSON API：

```
mobike-1252758967.cosbj.myqcloud.com
```

XML API：

```
mobike-1252758967.cos.ap-beijing.myqcloud.com
```

访问存储桶 mobike 根目录下的对象 test.txt ，其访问地址为：

JSON API：

```
mobike-1252758967.cosbj.myqcloud.com/test.txt
```

XML API：

```
mobike-1252758967.cos.ap-beijing.myqcloud.com/test.txt
```

当然因为我们前面选择了 CDN 加速，还可以通过加速域名进行访问：

[http://mobike-1252758967.file.myqcloud.com/qcloud.json](http://mobike-1252758967.file.myqcloud.com/qcloud.json)

我们可以在「云产品」->「CDN与加速」->「CDN」->「域名管理」，设置访问控制、缓存配置等，在高级配置里面也可以开启 HTTPS、HTTP2.0 和设置全局的 Header 等操作：

![](https://user-gold-cdn.xitu.io/2018/7/12/1648e4e3e9389785?w=1042&h=522&f=jpeg&s=58857)

> 关于自定义域名的方式具体参考[官网](https://cloud.tencent.com/document/product/436/6252#%E5%BC%80%E5%90%AF%20CDN%20%E5%8A%A0%E9%80%9F)，本文就不做更多叙述，我们目前就采用的这种自定义域名配置。


## 上传文件

我们要通过腾讯云的 Node.js SDK 把本地文件批量上传到腾讯云，这里我们提供 3 个方案：

- cos-nodejs-sdk-v5
- 基于 fis
- 基于 gulp

### cos-nodejs-sdk-v5

它是腾讯云提供的 COS Node.js SDK，我们可以调用它的 API 进行文件和 Bucket 的操作。

请注意，`新版本`的参数发生了变化，本示例建议大家使用**最新版本**。

> warning: AppId has been deprecated, Please put it at the end of parameter Bucket(E.g: "test-1250000000").

现在我们把 `dist/images` 目录下面的所有文件（包含子文件夹里面的）上传到之前创建的 example bucket 里面。

##### 1. 加载 cos-nodejs-sdk-v5

```js
var COS = require('cos-nodejs-sdk-v5');
```

##### 2. 实例化

直接通过 new COS 的方式，这里的参数前面都已经介绍过了。

`ap-beijing` 代表地域是`北京`。

```js
var cos = new COS({
    Region: 'ap-beijing',
    SecretId: '**',
    SecretKey: '**',
    Bucket: 'mobike-1252758967'
})
```

##### 3. 循环要上传的目录

先看一下目录结构：

```
├─dist
  ├─images
     ├─mobike
       ├─WechatIMG228.jpeg
     ├─WechatIMG1215.jpeg
     ├─****
```

这里我们推荐 `ndir` 这个工具包来帮我们完成循环目录的操作。

```js
// 目录结构请相应调整
var dest = path.resolve(__dirname, './dist/images');

ndir.walk(dest, function onDir(dirpath, files) {
  // console.log(' * %s', dirpath)
  for (var i = 0, l = files.length; i < l; i++) {
    var info = files[i];
    if (info[1].isFile()) {
      // console.log('isFile')
      // console.log(info[0])
      // console.log(info[1])
      // 这里可以拿到文件的很多信息
    }
  }
}, function end() {
  console.log('walk end.')
}, function error(err, errPath) {
  console.error('%s error: %s', errPath, err)
});
```


##### 4. 上传文件

我们调用 `putObject` 这个 API 进行文件上传，代码如下：

```js
// 文件路径
var fileWithPath = '***';
var fileName = '***';

// 调用 putObject 方法，传入参数
cos.putObject({
    Bucket: '**',
    Region: '**',
    Key: 'juejin/xiaoce/' + fileName,
    Body: fs.createReadStream(fileWithPath), 
    ContentLength: fs.statSync(fileWithPath).size, 
    onProgress: function (progressData) {
        console.log(JSON.stringify(progressData))
    },
}, function (err, data) {
    if (err) {
        console.log('err', err)
    } else {
        console.log(data)
    }
});
```

最终我们可以访问到 [http://mobike-1252758967.file.myqcloud.com/juejin/xiaoce/images/WechatIMG1215.jpeg](http://mobike-1252758967.file.myqcloud.com/juejin/xiaoce/images/WechatIMG1215.jpeg)。


> 本案例的代码我们已经放到 GitHub，具体可以自行下载，替换你的配置进行测试，地址请点击[这里](https://github.com/mobikeFE/xiaoce-qcloud-demo)。


### 基于 fis

fis 支持插件 [fis3-deploy-qcloud](https://www.npmjs.com/package/fis3-deploy-qcloud)，内部也是依赖上面提到的 `cos-nodejs-sdk-v5`，使用方式如下：

```js
fis.plugin('qcloud', {
  Region: 'ap-beijing',
  SecretId: '**',
  SecretKey: '**',
  Bucket: 'mobike-1252758967'
})
```

> 考虑到之前的 fis3-deploy-qcloud 很久没有维护了，而且使用了老版本 1.2.5，所以我们把它升级了一下，上传到 npm 上，大家可以使用新版本 [fis3-deploy-qcloud-latest](https://github.com/mobikeFE/fis3-deploy-qcloudv2#readme)。

> 本案例的代码我们已经放到 GitHub，具体可以自行下载，替换你的配置进行进行测试，地址请点击[这里](https://github.com/mobikeFE/xiaoce-qcloud-demo)。


### 基于 gulp

推荐使用 [gulp-upload-qcloud](https://github.com/yingye/gulp-upload-qcloud) 插件，内部也是依赖上面提到的 `cos-nodejs-sdk-v5`，使用方式如下：

```js
const gulp = require('gulp');
const uploadQcloud = require('gulp-upload-qcloud');

gulp.task('upload', () => {
  return gulp.src(['test/**/*.*'])
    .pipe(uploadQcloud({
      SecretId: '**',
      SecretKey: '**',
      Bucket: 'mobike-1252758967',
      Region: 'ap-beijing',
      Prefix: 'demo/sub'
      OverWrite: false
    }));
});
```

> 本案例的代码我们已经放到 GitHub，具体可以自行下载，替换你的配置进行进行测试，地址请点击[这里](https://github.com/mobikeFE/xiaoce-qcloud-demo)。


## 优化

### 优化缓存文件的时间

参考如下配置，设置一个更长的 CacheControl 值：

```
CacheControl: 'max-age=604800'
```

我们从浏览器 Network 里面打开，如图：

![](https://user-gold-cdn.xitu.io/2018/7/9/1647ef2e1c3e1ad0?w=585&h=328&f=jpeg&s=50730)

当然也可以在「云产品」->「CDN与加速」->「CDN」->「域名管理」中的[缓存配置](https://cloud.tencent.com/document/product/228/6290)，进行更定制化的缓存设置，可以按文件类型、文件夹、全路径文件等，如图所示。

![](https://user-gold-cdn.xitu.io/2018/7/23/164c2e8a0d6349e0?w=974&h=428&f=jpeg&s=51603)

比如这里，我们可以对之前的 json 后缀文件设置不缓存，如图所示。

![](https://user-gold-cdn.xitu.io/2018/7/23/164c2ed8bcc79d80?w=1071&h=450&f=jpeg&s=66750)

如果是多个文件类型后缀，可以用 `;` 隔开，内容区分大小写，必须是以 `.` 开头。这里我们设置刷新时间为 0 就代表不缓存，所有请求转发至用户源站。配置保存之后大约需要 5 分钟的部署时间。

当部署完成之后，[json文件](http://mobike-1252758967.file.myqcloud.com/qcloud.json)我们从浏览器 Network 里面打开，如图：

![](https://user-gold-cdn.xitu.io/2018/7/23/164c2f3f2d099a26?w=605&h=350&f=jpeg&s=54661)

可以看到，原来的 `Cache-Control：max-age = 600` 消失了（下图中加框部分）。

![](https://user-gold-cdn.xitu.io/2018/7/23/164c5af6b2a49af3?w=358&h=133&f=jpeg&s=23889)


## 小结

通过以上内容的介绍以及案例代码，相信里面提到的一些上传插件也会帮助到你的日常开发，读者也可以清晰的了解如何用腾讯云来管理静态资源。