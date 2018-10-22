# 利用腾讯云来管理部署 Node.js 服务

由于前端的快速发展，大家已经开始利用 Node.js 来做轻量服务层。

下面我们会带着大家一步一步从云服务器申请和配置、到 Node.js 环境搭建和配置以及以登录案例为出发点的 Node.js API 部署。（注：本节中的案例采用 `express 4.16.3` 版本）

> 如果你不需要使用腾讯云来管理部署 Node.js 服务，可以略过腾讯云申请和配置部分的内容，但是一些 Node.js 环境搭建和配置以及以登录案例为出发点的 Node.js API 可以适当参考。为了方便学习，本节的部分代码示例我们放到了 GitHub 上，点此访问 [wechat-nodejs-demo](https://github.com/mobikeFE/wechat-nodejs-demo)。

腾讯云包含了很多强大的功能，这里我们主要关注云服务器、关系型数据库、短信，更多功能可查看[腾讯云官网](https://cloud.tencent.com/)。我们可以把 Node.js 服务部署到云服务器上，然后利用 MySQL 来存储用户信息，同时短信服务可以用来向用户发送短信进行登录操作。

## 云服务器

通过云产品 -> 云服务器 -> 云主机进入「云主机」页面，点击**新建**按钮，如图所示。

![新建](https://user-gold-cdn.xitu.io/2018/8/5/16509e36f012ee51?w=1404&h=282&f=jpeg&s=76517)

关于云主机配置，这里选择如下套餐，大家可以根据需求自行选择。

![套餐](https://user-gold-cdn.xitu.io/2018/8/5/16509e36ec499f2a?w=1196&h=590&f=jpeg&s=124455)


### 安装 Node.js

因为后面要部署 express 的服务，所以需要预先安装 Node.js：

1、创建一个目录

```
cd /home/ubuntu
mkdir mobike
```

2、进入目录

```
cd mobike
```

3、拉取 Node.js 包

目前我们选择的是 `8.9.3` 版本

```
wget https://nodejs.org/dist/v8.9.3/node-v8.9.3-linux-x64.tar.xz
```

4、解压
```
tar xvJf node-v8.9.3-linux-x64.tar.xz
```

5、移动到 `/usr/local` 目录下

```
sudo mv node-v8.9.3-linux-x64 /usr/local/node-v8
```

6、设置软链
```
sudo ln -s /usr/local/node-v8/bin/node /bin/node
```

然后你就可以验证一下是否安装成功了，在控制台输入如下命令：
```
node -v
```

控制台输出了 `v8.9.3`。


### 设置 npm

与上面类似，需要设置一下软链：
```
sudo ln -s /usr/local/node-v8/bin/npm /bin/npm
```

然后在控制台输入如下命令：
```
npm -v
```

控制台输出了 `5.5.1`。


### 全局安装 pm2

使用 [pm2](https://www.npmjs.com/package/pm2) 来管理 Node.js 服务，安装方式如下：
```
sudo npm install pm2 -g
```

同样要设置一下软链：
```
sudo ln -s /usr/local/node-v8/bin/pm2 /bin/pm2
```

#### 错误日志

需要查看错误日志的话，可以查看如下路径：
```
/home/ubuntu/.pm2/logs/www-error.log
```

### 安装 Nginx
```
sudo apt-get install nginx
```

默认的配置目录：
```
/etc/nginx/
```

默认的 root 指向地址：
```
/var/www/html
```

可以自行修改默认的 index.nginx-debian.html 内容，如图所示。

![nginx](https://user-gold-cdn.xitu.io/2018/8/5/16509e37357657ae?w=569&h=123&f=jpeg&s=21331)

后面我们可以配置转发到 Node.js 服务对应的端口。


## 短信服务

腾讯云支持 [SMS 服务](https://cloud.tencent.com/document/product/382/18061#.E5.88.9B.E5.BB.BA.E7.AD.BE.E5.90.8D)，申请方式：

![sms](https://user-gold-cdn.xitu.io/2018/8/5/16509e3705483c4d?w=460&h=635&f=jpeg&s=56465)

而且也支持 Node.js SDK 接入，详见文档[腾讯云短信 Node.js SDK](https://github.com/qcloudsms/qcloudsms_js)。


### 添加应用

如下图所示，输入应用名称、类型和简介保存即可。

![sms](https://user-gold-cdn.xitu.io/2018/8/5/16509e36efbd07df?w=511&h=452&f=jpeg&s=29088)

这样我们就有了 `AppID` 和 `AppKey`，当然，一条完整的短信由短信签名和短信正文内容两部分组成。


### 创建短信签名

需要输入签名内容、签名类型还有备注内容，提交之后，需要审核才可以使用，如图所示。

![sms sign](https://user-gold-cdn.xitu.io/2018/8/6/1650d51decddbfc8?w=563&h=660&f=jpeg&s=62336)

这里为了测试，我们选择[公众号或小程序名全称或简称]，由于是个人号，所以审核很快，如图所示。

![sms sign result](https://user-gold-cdn.xitu.io/2018/8/5/16509e36f41e54ff?w=1201&h=202&f=jpeg&s=29574)


### 创建短信正文模板

需要输入模板名称、短信内容，还有申请说明，提交之后，需要审核才可以使用，如图所示。

![sms template](https://user-gold-cdn.xitu.io/2018/8/5/16509e3722405eed?w=467&h=525&f=jpeg&s=64827)

> 可以用 {**} 来放入自定义的内容。

审核之后，如图所示。

![sms template result](https://user-gold-cdn.xitu.io/2018/8/5/16509e3723efc234?w=1165&h=95&f=jpeg&s=23428)


这里也介绍一下带自定义内容的模板，如图所示。

![sms template result](https://user-gold-cdn.xitu.io/2018/8/6/1650d51decee6d08?w=1155&h=93&f=jpeg&s=22998)

我们创建了短信内容：

> 欢迎关注前端新视野，验证码{1}。


### 调用 SDK 发送短信

下面就可以调用 SDK 了，这里我们选择把 SDK 源码拷贝到项目目录下，然后加载 qcloudsms_js SDK：

```js
var QcloudSms = require("qcloudsms_js");
```

传递 appid 和 appkey 进行实例化：

```js
var qcloudsms = QcloudSms(appid, appkey);
```

#### 发送单条

调用封装的 `SmsSingleSender` 方法，传入几个参数：

```js
var ssender = qcloudsms.SmsSingleSender();

// 定义参数和回调
var phoneNumbers = ["***"];
function callback(err, res, resData) {
  if (err) {
    console.log("err: ", err);
  } else {
    console.log("request data: ", res.req);
    console.log("response data: ", resData);
  }
}
ssender.send(0, 86, phoneNumbers[0], "你好，我是前端新视野版主，欢迎关注我们", "", "", callback);
```

#### 自定义参数

带有参数的模板，可以调用封装的 `sendWithParam` 方法，传入几个参数：

```js
var params = ["8888"];

ssender.sendWithParam(86, phoneNumbers[0], templateId,
  params, smsSign, "", "", callback);
```

效果如图：

![sms param](https://user-gold-cdn.xitu.io/2018/8/6/1650d51dece00f48?w=1080&h=1920&f=jpeg&s=104454)



#### 发送多条

调用封装的 `SmsMultiSender` 方法，传入几个参数：

```js
var ssender = qcloudsms.SmsMultiSender();

// 定义参数和回调
var phoneNumbers = ["***"];
function callback(err, res, resData) {
  if (err) {
    console.log("err: ", err);
  } else {
    console.log("request data: ", res.req);
    console.log("response data: ", resData);
  }
}
ssender.send(0, 86, phoneNumbers, "你好，我是前端新视野版主，欢迎关注我们", "", "", callback);
```



## 用 code 换取 openid 和 session_key 等信息

临时登录凭证 code 只能使用一次，官方标记是只有五分钟的有效期。

返回的 openid 为用户的唯一标识。


小程序侧：

> 调用接口 `wx.login()` 获取临时登录凭证（code）。

```js
wx.login({
  success: function(res) {
    if (res.code) {
      console.log(res.code)
    }
  }
})
```

Node.js 侧：

定义路由，获取传递过来的 code，这里以 GET 请求示例。

```js
router.get('/login', function(req, res, next) {
  var code = req.query.code;
})
```

然后调用微信的接口，如需了解可查看[官方文档](https://developers.weixin.qq.com/miniprogram/dev/api/api-login.html#wxloginobject)。

这里我们推荐 2 个工具包来发送请求。

输入以下几个参数：

* appid：小程序唯一标识
* secret：小程序的 app secret
* js_code：wx.login 返回的 code
* grant_type：值为 authorization_code

1. axios

```js
var axios = require('axios')

axios({
  url: 'https://api.weixin.qq.com/sns/jscode2session',
  method: 'GET',
  params: {
    appid: appId,
    secret: appSecret,
    js_code: code,
    grant_type: 'authorization_code'
  }
}).then(res => {
    //...
})
```


2. request

```js
var request = require('request');


request(`https://api.weixin.qq.com/sns/jscode2session?appid=${appId}&secret=${appSecret}&js_code=${code}&grant_type=authorization_code`, function (error, response, body) {
    console.log('body:', body);
}
```

返回的结果如下：

> 测试中使用了我个人的 code，包含了 unionid 返回。

```json
{
  "session_key": "TYA089n******Gp\/YO1EDA==",
  "expires_in": 7200,
  "openid": "**",
  "unionid": "***"
}
```

> 注意：这里因为 code 本身有有效期，所以需要增加异常处理。

代码错误方面，最常见的就是 errcode 为 40029 的错误。

```js
request(`https://api.weixin.qq.com/sns/jscode2session?appid=${appId}&secret=${appSecret}&js_code=${code}&grant_type=authorization_code`, function (error, response, body) {
  if (error) {
    throw new Error(`[查询用户信息失败]，\n${error}`)
  }
  console.log('body:', body); 

  // 错误：{"errcode":40029,"errmsg":"invalid code, hints: [ req_id: 07422030 ]"}
  var data = JSON.parse(body);

  if (data.errcode == 40029) {
    throw new Error(`[查询用户信息失败]，\n${data.errmsg}`)
  }
})
```


## 关系型数据库

这里我们使用了 MySQL 来存储数据，如图所示。

![关系型数据库](https://user-gold-cdn.xitu.io/2018/8/5/1650a11a9a3d2148?w=1199&h=155&f=jpeg&s=34355)

需要进行初始化操作，具体可以点击[参考文档](https://cloud.tencent.com/document/product/236/3128)。


### 存储登录信息到 MySQL

上面获取到了登录信息之后，我们会把它们存储到 MySQL 中：

在通过 Node.js 层代码之前，我们需要先在控制台中创建对应的数据库的表，如下图所示。

登陆腾讯云的数据库管理平台：

![mysql login](https://user-gold-cdn.xitu.io/2018/8/5/16509e373f5b66a4?w=440&h=502&f=jpeg&s=33862)

输入用户名和密码：

![mysql table](https://user-gold-cdn.xitu.io/2018/8/5/16509e37418ed664?w=1199&h=629&f=jpeg&s=79044)

里面我们需要创建表：

定义了表名等信息之后，还需要添加列信息。

![mysql table](https://user-gold-cdn.xitu.io/2018/8/5/16509e374607c8df?w=769&h=635&f=jpeg&s=52945)



### 方案一：MySQL 工具包

我们采用 [MySQL](https://www.npmjs.com/package/mysql)。

第一步：安装
```
npm install mysql --save
```

第二步：初始化

调用 `createConnection` 方法：

```js
var mysql  = require('mysql');

var connection = mysql.createConnection({     
  host     : configs.mysql.host,    
  user     : configs.mysql.user,           
  password : configs.mysql.pass,
  port: configs.mysql.port,             
  database: configs.mysql.db
});
```

第三步：插入数据

```js
var userInsertSql = 'INSERT INTO userSessionInfo(uuid, create_time, open_id, session_key) VALUES(?,?,?,?)';
var data = {
  uuid: uuid,
  create_time: create_time,
  open_id: open_id,
  session_key: session_key
};

// 调用 query
connection.query(userInsertSql, data, function (err, result) {
  
})


// 调用 end
connection.end();
```


### 方案二：Knex 工具包

不熟悉的同学可以先看一下 [Knex 官方文档](https://knexjs.org/)。

第一步：安装
```
npm install knex --save
```

当然因为我们这里要使用 MySQL 来存储数据，还需要安装：
```
npm install mysql --save
```

第二步：初始化

主要配置了：

* `client` 很简单，配置：mysql
* `connection` 这里的信息可以在腾讯云的关系型数据库 - MySQL 实例列表里面找到，如图：

![mysql](https://user-gold-cdn.xitu.io/2018/8/5/16509e37557fe32a?w=829&h=343&f=jpeg&s=62005)

```js
require('knex')({
  client: 'mysql',
  connection: {
    host: configs.mysql.host,
    port: configs.mysql.port,
    user: configs.mysql.user,
    password: configs.mysql.pass,
    database: configs.mysql.db,
    charset: configs.mysql.char
  }
})
```

第三步：插入数据

这里我们存储如下字段：

* uuid
* create_time
* open_id
* session_key


```js
knex('userSessionInfo').insert({
    uuid, 
    create_time,  
    open_id, 
    session_key
})
```

**uuid 生成规则**

我们使用 uuid 工具包

```js
const uuidGenerator = require('uuid/v4')
const uuid = uuidGenerator()
```

**时间生成规则**

方式一：直接存储时间戳

方式二：使用 moment 工具包

```js
const moment = require('moment')
const create_time = moment().format('YYYY-MM-DD HH:mm:ss')
```

方式三：使用 Day.js 工具包

```js
const dayjs = require('dayjs')
const create_time = dayjs().format('YYYY-MM-DD HH:mm:ss')
```

第四步：更新 `session_key`

`session_key` 有一定的有效时间，一般使用 `checkSession` 校验用户当前 `session_key` 是否有效。

```js
wx.checkSession({
  success: function(){
    //session_key 未过期，并且在本生命周期一直有效
  },
  fail: function(){
    // session_key 已经失效，需要重新执行登录流程
    // 重新登录
    ....
  }
})
```

所以我们把上面的方法完善一下，增加更新策略：

1. 先查询是否存储了同样的 openid

```js
knex('userSessionInfo')
.count('open_id as hasUser')
.where({
  open_id
})
.then(res => {
  if (res[0].hasUser) {
    // 存在，更新
  } else {
    // 不存在，新建
  }
})
```

调用 knex 的 [count 方法](https://knexjs.org/#Builder-count)，相当于：

```sql
select count(`open_id`) as `hasUser` from `userSessionInfo`
```

2. 更新对应的 openid 那条记录

我们扩展一下之前的表字段，增加一个 skey 来存每次的新 session_key

先调用 [where 方法](https://knexjs.org/#Builder-where) 查询对应的 open_id 的记录，再调用 [update 方法](https://knexjs.org/#Builder-update) 更新以下字段。

```js
knex('userSessionInfo')
.where({
  open_id
})
.update({
  skey, 
  last_visit_time, 
  session_key
})
```

相当于：

```sql
update `userSessionInfo` set `skey` = skey, `last_visit_time` = last_visit_time, `session_key` = last_visit_time and  where `open_id` = open_id
```

3. 根据 openid 查询用户信息

我们定义了一个 `getUserInfoByOpenId` 方法，支持从库里面查询用户信息

```js
function getUserInfoByOpenId (openId) {
  if (!openId) throw new Error('参数 openId 缺少')

  return knex('userSessionInfo').select('*').where({ open_id: openId }).first()
}
```

这里调用了 [select 方法](https://knexjs.org/#Builder-select)，然后 [first 方法](https://knexjs.org/#Builder-first) 代表匹配的第一条记录，流程如下：

```js
knex('userSessionInfo')
.select('*')
.where({ open_id: openId })
.first()
```


## 小结

当然，在 MySQL ORM 部分也推荐同学们使用 Sequelize，参考文档请点击 [Manual | Sequelize](http://docs.sequelizejs.com/)，大家可以自由选择。期望通过本节内容的学习，你能够掌握如何用腾讯云来管理和运维一个 Node.js 服务，同时提高 MySQL 和 SMS 服务的使用能力。







