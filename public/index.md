# 安装

  Koa 目前需要 >=0.11.x版本的 node 环境。并需要在执行 node 的时候附带 --harmony 来引入 generators 。 如果您安装了较旧版本的 node ，您可以安装 [n](https://github.com/visionmedia/n) (node版本控制器)，来快速安装 0.11.x

```
$ npm install -g n
$ n 0.11.12
$ node --harmony my-koa-app.js
```

# 应用

  Koa 应用是一个包含一系列中间件 generator 函数的对象。
  这些中间件函数基于 request 请求以一个类似于栈的结构组成并依次执行。
  Koa 类似于其他中间件系统（比如 Ruby's Rack 、Connect 等），
  然而 Koa 的核心设计思路是为中间件层提供高级语法糖封装，以增强其互用性和健壮性，并使得编写中间件变得相当有趣。

  Koa 包含了像 content-negotiation（内容协商）、cache freshness（缓存刷新）、proxy support（代理支持）和 redirection（重定向）等常用任务方法。
  与提供庞大的函数支持不同，Koa只包含很小的一部分，因为Koa并不绑定任何中间件。

  任何教程都是从 hello world 开始的，Koa也不例外^_^  

```js
var koa = require('koa');
var app = koa();

app.use(function *(){
  this.body = 'Hello World';
});

app.listen(3000);
```

## 中间件级联

  Koa 的中间件通过一种更加传统（你也许会很熟悉）的方式进行级联，摒弃了以往 node 频繁的回调函数造成的复杂代码逻辑。
  我们通过 generators 来实现“真正”的中间件。
  Connect 简单地将控制权交给一系列函数来处理，直到函数返回。
  与之不同，当执行到 `yield next` 语句时，Koa 暂停了该中间件，继续执行下一个符合请求的中间件('downstrem')，然后控制权再逐级返回给上层中间件('upstream')。

  下面的例子在页面中返回 "Hello World"，然而当请求开始时，请求先经过 `x-response-time` 和 `logging` 中间件，并记录中间件执行起始时间。
  然后将控制权交给 `reponse` 中间件。当中间件运行到 `yield next` 时，函数挂起并将控制前交给下一个中间件。当没有中间件执行 `yield next` 时，程序栈会逆序唤起被挂起的中间件来执行接下来的代码。

```js
var koa = require('koa');
var app = koa();

// x-response-time

app.use(function *(next){
  var start = new Date;
  yield next;
  var ms = new Date - start;
  this.set('X-Response-Time', ms + 'ms');
});

// logger

app.use(function *(next){
  var start = new Date;
  yield next;
  var ms = new Date - start;
  console.log('%s %s - %s', this.method, this.url, ms);
});

// response

app.use(function *(){
  this.body = 'Hello World';
});

app.listen(3000);
```

## 配置

  应用配置是 `app` 实例属性，目前支持的配置项如下：

  - `app.name` 应用名称（可选项）
  - `app.env` 默认为 __NODE_ENV__ 或者 `development`
  - `app.proxy` 如果为 `true`，则解析 "Host" 的 header 域，并支持 `X-Forwarded-Host`
  - `app.subdomainOffset` 默认为2，表示 `.subdomains` 所忽略的字符偏移量。

## app.listen(...)

  Koa 应用并非是一个 1-to-1 表征关系的 HTTP 服务器。
  一个或多个Koa应用可以被挂载到一起组成一个包含单一 HTTP 服务器的大型应用群。

  如下为一个绑定3000端口的简单 Koa 应用，其创建并返回了一个 HTTP 服务器，为 `Server#listen()` 传递指定参数（参数的详细文档请查看[nodejs.org](http://nodejs.org/api/http.html#http_server_listen_port_hostname_backlog_callback)）。

```js
var koa = require('koa');
var app = koa();
app.listen(3000);
```

  The `app.listen(...)` method is simply sugar for the following:

```js
var http = require('http');
var koa = require('koa');
var app = koa();
http.createServer(app.callback()).listen(3000);
```

  This means you can spin up the same application as both HTTP and HTTPS
  or on multiple addresses:

```js
var http = require('http');
var koa = require('koa');
var app = koa();
http.createServer(app.callback()).listen(3000);
http.createServer(app.callback()).listen(3001);
```

## app.callback()

  Return a callback function suitable for the `http.createServer()`
  method to handle a request.
  You may also use this callback function to mount your koa app in a
  Connect/Express app.

## app.use(function)

  Add the given middleware function to this application. See [Middleware](https://github.com/koajs/koa/wiki#middleware) for
  more information.

## app.keys=

 Set signed cookie keys.

 These are passed to [KeyGrip](https://github.com/jed/keygrip),
 however you may also pass your own `KeyGrip` instance. For
 example the following are acceptable:

```js
app.keys = ['im a newer secret', 'i like turtle'];
app.keys = new KeyGrip(['im a newer secret', 'i like turtle'], 'sha256');
```

  These keys may be rotated and are used when signing cookies
  with the `{ signed: true }` option:

```js
this.cookies.set('name', 'tobi', { signed: true });
```

## Error Handling

  By default outputs all errors to stderr unless __NODE_ENV__ is "test". To perform custom error-handling logic such as centralized logging you
  can add an "error" event listener:

```js
app.on('error', function(err){
  log.error('server error', err);
});
```

  If an error in the req/res cycle and it is _not_ possible to respond to the client, the `Context` instance is also passed:

```js
app.on('error', function(err, ctx){
  log.error('server error', err, ctx);
});
```

  When an error occurs _and_ it is still possible to respond to the client, aka no data has been written to the socket, Koa will respond
  appropriately with a 500 "Internal Server Error". In either case
  an app-level "error" is emitted for logging purposes.


