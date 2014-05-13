# 安装

  Koa 目前需要 >=0.11.x版本的node环境。并需要在执行node的时候附带 --harmony来引入generators。 如果您安装了较旧版本的node，您可以安装 [n](https://github.com/visionmedia/n) (node版本控制器)，来快速安装 0.11.x

```
$ npm install -g n
$ n 0.11.12
$ node --harmony my-koa-app.js
```

# 应用

  Kao应用是一个包含一系列中间件generator函数的对象。
  这些中间件函数以一个类似于栈的结构在request请求上组成并依次执行。
  Koa类似于其他中间件系统(比如Ruby's Rack、Connect等)。
  Koa的核心设计思路是提供低层中间件的高层语法糖封装，其增强了互用性、健壮性，并使得编写中间件相当有趣。

  Koa包含了像content-negotiation(内容控制)、cache freshness(缓存刷新)、proxy support(代理支持)和redirection(重定向)等常用任务方法。
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

  Koa的中间件通过一种更加传统(你也许会很熟悉)的方式进行级联，摒弃了以往node频繁的回调函数造成的复杂代码逻辑。
  我们通过generators来实现“真正的”中间件。
  Connect简单地将控制权交给一系列函数来处理，直到函数返回。
  与之不同，当执行到 `yield next` 语句时，Koa 暂停了该中间件，继续执行下一个符合请求的中间件('downstrem')，然后控制权再逐级返回给上层中间件('upstream')。

  下面的例子在页面中返回 "Hello World"，然而当请求开始时，请求先经过 `x-response-time` 和 `logging` 中间件，并记录中间件执行起始时间。
  然后将控制权交给 `reponse` 中间件。当中间件运行到 `yield next` 时，函数挂起并将控制前交给下一个中间件。当没有中间件执行 `yield next`时，程序栈会逆序唤起被挂起的中间件来执行接下来的代码。

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

## Settings

  Application settings are properties on the `app` instance, currently
  the following are supported:

  - `app.name` optionally give your application a name
  - `app.env` defaulting to the __NODE_ENV__ or "development"
  - `app.proxy` when true proxy header fields will be trusted
  - `app.subdomainOffset` offset of `.subdomains` to ignore [2]

## app.listen(...)

  A Koa application is not a 1-to-1 representation of a HTTP server.
  One or more Koa applications may be mounted together to form larger
  applications with a single HTTP server.

  Create and return an HTTP server, passing the given arguments to
  `Server#listen()`. These arguments are documented on [nodejs.org](http://nodejs.org/api/http.html#http_server_listen_port_hostname_backlog_callback). The following is a useless Koa application bound to port `3000`:

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


