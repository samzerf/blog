---
title: koa2核心源码分析
date: 2020-12-25 10:48:05
tags:
---
koa是一个轻量级的web应用框架。其实现非常精简和优雅，核心代码仅有区区一百多行，非常值得我们去细细品味和学习。
<!-- more -->
在开始分析源码之前先上demo～

#### DEMO 1 

``` javascript
const Koa = require('../lib/application');
const app = new Koa();

app.use(async (ctx, next) => {
  console.log('m1-1');
  await next();
  console.log('m1-2');
});

app.use(async (ctx, next) => {
  console.log('m2-1');
  await next();
  console.log('m2-2');
});

app.use(async (ctx, next) => {
  console.log('m3-1');
  ctx.body = 'there is a koa web app';
  await next();
  console.log('m3-2');
});

app.listen(8001);
```
上面代码最终会在控制台依次输出
```
m1-1
m2-1
m3-1
m3-2
m2-2
m1-2
```
当在中间件中调用`next()`时，会停止当前中间件的执行，转而进行下一个中间件。当下一个中间件执行完后，才会继续执行`next()`后面的逻辑。

#### DEMO 2

我们改一下第一个中间件的代码，如下所示：
``` javascript
app.use(async (ctx, next) => {
  console.log('m1-1');
  // await next();
  console.log('m1-2');
});
```
当把第一个中间件的`await next()`注释后，再次执行，在控制台的输出如下：
```
m1-1
m2-1
```
显然，如果不执行`next()`方法，代码将只会执行到当前的中间件，不过后面还有多少个中间件，都不会执行。

这个`next`为何会具有这样的魔力呢，下面让我们开始愉快地分析koa的源码，一探究竟～

#### 代码结构
> 分析源码之前我们先来看一下koa的目录结构，koa的实现文件只有4个，这4个文件都在lib目录中。

![](https://user-gold-cdn.xitu.io/2018/10/11/166636886e81f287?w=306&h=226&f=png&s=20686)

* `application.js`  — 定义了一个类，这个类定义了koa实例的方法和属性
* `context.js`  — 定义了一个proto对象，并对proto中的属性进行代理。中间件中使用的ctx对象，其实就是继承自proto
* `request.js`  —  定义了一个对象，该对象基于原生的req拓展了一些属性和方法
* `response.js` - 定义了一个对象，该对象基于原生的res拓展了一些属性和方法

通过package.json文件得知，koa的入口文件是lib/application.js，我们先来看一下这个文件做了什么。

#### 定义koa类
打开`application.js`查看源码可以发现，这个文件主要就是定义了一个类，同时定义了一些方法。

``` javascript
module.exports = class Application extends Emitter {

  constructor() {
    super();
    this.middleware = []; // 中间件数组
  }
  
  listen (...args) {
    // 启用一个http server并监听指定端口
    const server = http.createServer(this.callback());
    return server.listen(...args);
  }
  
  use (fn) {
    // 把中间添加到中间件数组
    this.middleware.push(fn);
    return this;
  }
  
}
```

我们创建完一个koa对象之后，通常只会使用两个方法，一个是`listen`，一个是`use`。listen负责启动一个http server并监听指定端口，use用来添加我们的中间件。

当调用`listen`方法时，会创建一个http server，这个http server需要一个回调函数，当有请求过来时执行。上面代码中的`this.callback()`就是用来返回这样的一个函数：这个函数会读取应用所有的中间件，使它们按照传入的顺序依次执行，最后响应请求并返回结果。

`callback`方法的核心代码如下：
``` javascript
  callback() {
    const fn = compose(this.middleware);
    const handleRequest = (req, res) => {
      const ctx = this.createContext(req, res);
      return this.handleRequest(ctx, fn);
    };
    return handleRequest;
  }
```
#### 回调函数callback的执行流程
`callback`函数会在应用启动时执行一次，并且返回一个函数`handleRequest`。每当有请求过来时，`handleRequest`都会被调用。我们将`callback`拆分为三个流程去分析：

1. 把应用的所有中间件合并成一个函数`fn`，在`fn`函数内部会依次执行`this.middleware`中的中间件（是否全部执行，取决于是否有调用`next`函数执行下一个中间件）
2. 通过`createContext`生成一个可供中间件使用的`ctx`上下文对象
3. 把ctx传给`fn`，并执行，最后对结果作出响应

#### koa中间件执行原理
``` javascript
const fn = compose(this.middleware);
```
源码中使用了一个`compose`函数，基于所有可执行的中间件生成了一个可执行函数。当该函数执行时，每一个中间件将会被依次应用。`compose`函数的定义如下：
``` javascript
function compose (middleware) {
  if (!Array.isArray(middleware)) throw new TypeError('Middleware stack must be an array!')
  for (const fn of middleware) {
    if (typeof fn !== 'function') throw new TypeError('Middleware must be composed of functions!')
  }

  return function (context, next) {
    let index = -1
    return dispatch(0) // 开始执行第一个中间件
    function dispatch (i) {
      if (i <= index) return Promise.reject(new Error('next() called multiple times')) // 确保同一个中间件不会执行多次
      index = i
      let fn = middleware[i]
      if (i === middleware.length) fn = next //个人认为对在koa中这里的fn = next并没有意义
      if (!fn) return Promise.resolve() // 执行到最后resolve出来
      try {
        return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));
      } catch (err) {
        return Promise.reject(err)
      }
    }
  }
}
```

它会先执行第一个中间件，执行过程中如果遇到`next()`调用，就会把控制权交到下一个中间件并执行，等该中间件执行完后，再继续执行`next()`之后的代码。这里的`dispatch.bind(null, i + 1)`就是`next`函数。到这里就能解答，为什么必须要调用`next`方法，才能让当前中间件后面的中间件执行。（有点拗口…）匿名函数的返回结果是一个`Promise`，因为要等到中间件处理完之后，才能进行响应。

#### context模块分析
中间件执行函数生成好之后，接下来需要创建一个`ctx`。这个`ctx`可以在中间件里面使用。`ctx`提供了访问`req`和`res`的接口。
创建上下文对象调用了一个`createContext`函数，这个函数的定义如下：
``` javascript
/**
 * 创建一个context对象,也就是在中间件里使用的ctx，并给ctx添加request, respone属性
 */
  createContext(req, res) {
    const context = Object.create(this.context); // 继承自context.js中export出来proto
    const request = context.request = Object.create(this.request); // 把自定义的request作为ctx的属性
    const response = context.response = Object.create(this.response);// 把自定义的response作为ctx的属性
    context.app = request.app = response.app = this;
    // 为了在ctx, request, response中，都能使用httpServer回调函数中的req和res
    context.req = request.req = response.req = req;
    context.res = request.res = response.res = res;
    request.ctx = response.ctx = context;
    request.response = response;
    response.request = request;
    context.originalUrl = request.originalUrl = req.url;
    context.state = {};
    return context;
  }
```

`ctx`对象实际上是继承自`context`模块中定义的`proto`对象，同时添加了`request`和`response`两个属性。`request`和`response`也是对象，分别继承自`request.js`和`response.js`定义的对象。这两个模块的功能是基于原生的`req`和`res`封装了一些`getter`和`setter`，原理比较简单，下面就不再分析了。

我们重点来看看`context`模块。
``` javascript
const proto = module.exports = {

  inspect() {
    if (this === proto) return this;
    return this.toJSON();
  },
  
  toJSON() {
    return {
      request: this.request.toJSON(),
      response: this.response.toJSON(),
      app: this.app.toJSON(),
      originalUrl: this.originalUrl,
      req: '<original node req>',
      res: '<original node res>',
      socket: '<original node socket>'
    };
  },

  assert: httpAssert,

  throw(...args) {
    throw createError(...args);
  },
  
  onerror(err) {
    if (null == err) return;
    if (!(err instanceof Error)) err = new Error(util.format('non-error thrown: %j', err));
    let headerSent = false;
    if (this.headerSent || !this.writable) {
      headerSent = err.headerSent = true;
    }
    // delegate
    this.app.emit('error', err, this);
    if (headerSent) {
      return;
    }
    const { res } = this;
    // first unset all headers
    /* istanbul ignore else */
    if (typeof res.getHeaderNames === 'function') {
      res.getHeaderNames().forEach(name => res.removeHeader(name));
    } else {
      res._headers = {}; // Node < 7.7
    }

    // then set those specified
    this.set(err.headers);

    // force text/plain
    this.type = 'text';

    // ENOENT support
    if ('ENOENT' == err.code) err.status = 404;

    // default to 500
    if ('number' != typeof err.status || !statuses[err.status]) err.status = 500;

    // respond
    const code = statuses[err.status];
    const msg = err.expose ? err.message : code;
    this.status = err.status;
    this.length = Buffer.byteLength(msg);
    this.res.end(msg);
  },

  get cookies() {
    if (!this[COOKIES]) {
      this[COOKIES] = new Cookies(this.req, this.res, {
        keys: this.app.keys,
        secure: this.request.secure
      });
    }
    return this[COOKIES];
  },

  set cookies(_cookies) {
    this[COOKIES] = _cookies;
  }
};
```
`context`模块定义了一个`proto`对象，该对象定义了一些方法（eg: `throw`）和属性(eg: `cookies`)。我们上面通过`createContext`函数创建的`ctx`对象，就是继承自`proto`。因此，我们可以在中间件中直接通过`ctx`访问`proto`中定义的方法和属性。

值得一提的点是，作者通过代理的方式，让开发者可以直接通过`ctx[propertyName]`去访问`ctx.request`或`ctx.response`上的属性和方法。

> 实现代理的关键逻辑

``` javascript
/**
 * 代理response一些属性和方法
 * eg: proto.response.body => proto.body
 */
delegate(proto, 'response')
  .method('attachment')
  .method('redirect')
  .access('body')
  .access('length')
  // other properties or methods
  
/**
 * 代理request的一些属性和方法
 */
delegate(proto, 'request')
  .method('acceptsLanguages')
  .method('acceptsEncodings')
  .method('acceptsCharsets')
  .method('accepts')
  .method('get')
  // other properties or methods
```
实现代理的逻辑也非常简单，主要就是使用了`__defineGetter__`和`__defineSetter__`这两个对象方法，当`set`或`get`对象的某个属性时，调用指定的函数对属性值进行处理或返回。

#### 最终的请求与响应
当`ctx`（上下文对象）和`fn`(执行中间件的合成函数)都准备好之后，就能真正的处理请求并响应了。该步骤调用了一个`handleRequest`函数。
``` javascript
  handleRequest(ctx, fnMiddleware) {
    const res = ctx.res;
    res.statusCode = 404; // 状态码默认404
    const onerror = err => ctx.onerror(err);
    const handleResponse = () => respond(ctx);
    onFinished(res, onerror);
    // 执行完中间件函数后，执行handleResponse处理结果
    return fnMiddleware(ctx).then(handleResponse).catch(onerror);
  }
```

`handleRequest`函数会把`ctx`传入`fnMiddleware`并执行，然后通过`respond`方法进行响应。这里默认把状态码设为了`404`，如果在执行中间件的过程中有返回，例如对`ctx.body`进行负责，`koa`会自动把状态码设成`200`，这一部分的逻辑是在`response`对象的`body`属性的`setter`处理的，有兴趣的朋友可以看一下`response.js`。

`respond`函数会对`ctx`对象上的`body`或者其他属性进行分析，然后通过原生的`res.end()`方法将不同的结果输出。

#### 最后
到这里，koa2的核心代码大概就分析完啦。以上是我个人总结，如有错误，请见谅。欢迎一起交流学习！