---
title: Webpack打包文件分析与动态加载原理
date: 2021-04-19 11:09:14
tags:
---

### 一、构建结果分析

先从一个简单的模块开始。

假设我们有一个`hello`模块如下，
```javascript
function hello() {
  console.log('hello world')
}
hello()
```
使用webpack进行构建后，得到的结果如下，
<!-- more -->
```javascript
(function(modules) {
    var installedModules = {};  //  模块缓存
    function __webpack_require__(moduleId) {...}
    __webpack_require__.m = modules;
    __webpack_require__.c = installedModules;
    // define getter function for harmony exports
    __webpack_require__.d = function(exports, name, getter) {...};
    // define __esModule on exports
    __webpack_require__.r = function(exports) {...};
    __webpack_require__.t = function(value, mode) {...};
    // getDefaultExport function for compatibility with non-harmony modules
    __webpack_require__.n = function(module) {...};
    // Object.prototype.hasOwnProperty.call
    __webpack_require__.o = function(object, property) { return Object.prototype.hasOwnProperty.call(object, property); };
    // __webpack_public_path__
    __webpack_require__.p = "";
    // Load entry module and return exports
    return __webpack_require__(__webpack_require__.s = 0);
})
([
/* 0 */
(function(module, exports) {
    function hello() {
         console.log('hello world')
    }
    hello()
})]);
```
我们的代码经过webpack构建后，生成的是一个IIFE。该自执行函数先定义了一系列的方法，然后在最后执行了
```javascript
__webpack_require__(__webpack_require__.s = 0)
```
下面我们来看看`__webpack_require__`到底做了什么。该函数的定义如下：

```javascript
// 模块缓存
var installedModules = {};
function __webpack_require__(moduleId) {
  // 检查当前require的模块id是否在cache中存在，如果存在，则直接返回对应模块的exports
  if (installedModules[moduleId]) {
    return installedModules[moduleId].exports;
  }
  // 创建一个模块，
  var module = installedModules[moduleId] = {
    i: moduleId,
    l: false,
    exports: {}
  };
  // 执行模块方法
  modules[moduleId].call(module.exports, module, module.exports, __webpack_require__);
  // 标记当前模块已经被加载过
  module.l = true;
  // 返回模块的exports
  return module.exports;
}
```
通过分析上面的代码，我们可以知道，`__webpack_require__`的作用其实就是定义了一个`module`，同时将这个`module`和`__webpack_require__`本身作为参数传给指定的函数并执行。这样在函数内部给`module`或`module.exports`赋值时，就能同时修改`installedModules`里对应的模块。而且在执行函数内部如果使用`__webpack_require__`方法，也能获取到`modules`中相应的模块。`__webpack_require__(0)`表示执行第一个模块，也就是我们的`hello`模块。`hello`的代码比较简单，只是简单的执行一下函数。下面我们来看一个稍微复杂一点的🌰。

我们把代码修改一下，定义一个`log`模块，然后在`hello`中使用它。
```javascript
// log.js
export default function log(str) {
  console.log(str);
}

// index.js
import log from './log';
function hello() {
  log('hello world')
}

hello()
```
使用webpack重新构建，得到的`bundle`如下：
```javascript
(function (modules) { // webpackBootstrap
  // 省略了一堆代码
  return __webpack_require__(__webpack_require__.s = 0);
})
  ([
    /* 0 */
    (function (module, __webpack_exports__, __webpack_require__) {
      "use strict";
      __webpack_require__.r(__webpack_exports__);
/* harmony import */ var _log__WEBPACK_IMPORTED_MODULE_0__ = __webpack_require__(1);
      function hello() {
        Object(_log__WEBPACK_IMPORTED_MODULE_0__["default"])('hello world')
      }
      hello()
    }),
    /* 1 */
    (function (module, __webpack_exports__, __webpack_require__) {
      "use strict";
      __webpack_require__.r(__webpack_exports__);
/* harmony export (binding) */ __webpack_require__.d(__webpack_exports__, "default", function () { return log; });
      function log(str) {
        console.log(str);
      }
    })
  ]);
```
通过分析发现，相比第一次构建的代码，`modules`中多了一个模块，而这个模块正是我们后面添加的`log`模块。而且`hello`模块编译出来的代码也有了一些变化：
```javascript
function (module, __webpack_exports__, __webpack_require__) {
  "use strict";
  __webpack_require__.r(__webpack_exports__);
/* harmony import */ var _log__WEBPACK_IMPORTED_MODULE_0__ = __webpack_require__(1);
  function hello() {
    Object(_log__WEBPACK_IMPORTED_MODULE_0__["default"])('hello world')
  }
  hello()
})
```
这里我们看到多了一个`__webpack_require__.r`的方法，这个方法是初始化的时候定义的，它的功能其实很简单，就是标记一下当前的模块为`esModule`，我们分析模块执行时，可以忽略这些不会对流程造成影响的逻辑。`hello`模块中，通过`__webpack_require__(1)`加载下一个模块的`exports`，并将其缓存到变量`_log__WEBPACK_IMPORTED_MODULE_0__`上，然后在`hello`方法中传递参数给`log__WEBPACK_IMPORTED_MODULE_0__["default"]`并执行。

接下来我们再看一下log模块的实现：
```javascript
/* 1 */
(function (module, __webpack_exports__, __webpack_require__) {
  "use strict";
  __webpack_require__.r(__webpack_exports__);
  /* harmony export (binding) */ __webpack_require__.d(__webpack_exports__, "default", function () { return log; });
  function log(str) {
    console.log(str);
  }
}
```
`log`模块中使用了一个`__webpack_require__.d`方法，该方法的定义如下：
```javascript
__webpack_require__.d = function(exports, name, getter) {
  if(!__webpack_require__.o(exports, name)) {
    Object.defineProperty(exports, name, { enumerable: true, get: getter });
  }
};
__webpack_require__.o = function(object, property) { return Object.prototype.hasOwnProperty.call(object, property); };
```
不难发现，这个`__webpack_require__.d`方法的作用就是：检查指定对象上是否具有某个属性，如无，就为这个对象的指定属性绑定一个`getter`。所以，当执行下面的代码时，其实就是在`module.exports`添加了一个`default`的方法，执行该方法会返回`log`函数。
```javascript
__webpack_require__.d(__webpack_exports__, "default", function () { return log; });
```

因此，在`hello`模块中，就能通过`__webpack_require__(1)`获取`log`模块的`exports`。通过`__webpack_require__(1)["default"])()`就能调用`log`方法。

综上所述，我们可以了解到，webpack打包出来的IIFE，其实核心就只有两个，一个是定义的内置方法，另一个是依赖的`modules`数组，模块之间通过`__webpack_require__`引用。我们写的模块代码，会作为依赖传递给`webpack`定义的函数，同时，第一个模块就是入口文件。


### 二、 动态加载
当我们在编写大型单页`vue`应用时，一般会用到路由懒加载（当访问到特定路由时，才会去加载相应的模块）。其实这里主要是利用了**动态 import**这个特性。把上面的例子改一下，看看`webpack`是如何处理**动态 import**的。
```javascript
// index.js
function hello() {
  import('./dynamic.js').then(res => console.log(res))
}
hello()

// dynamic.js
export default function log(str) {
  console.log(str);
}
```
经过`webpack`构建后，我们发现，除了`app.bundle.js`以外，还多了一个`1.bundle.js`，这就是我们要动态导入的模块。另外，`app.bundle.js`中还多了几个先前没出现过的方法。
```javascript
(function (modules) { // webpackBootstrap
   // 省略部分代码
// 处理jsonp
function webpackJsonpCallback(data) {  //...}; 

// 构造jsonp的请求地址
function jsonpScriptSrc(chunkId)  {// ...}

// 动态加载方法，返回promise，在promise.then中可以使用__webpack_require__加载动态模块
webpack_require__.e = function requireEnsure(chunkId) {   // ...};
var jsonpArray = window["webpackJsonp"] = window["webpackJsonp"] || [];
var oldJsonpFunction = jsonpArray.push.bind(jsonpArray);
jsonpArray.push = webpackJsonpCallback;
jsonpArray = jsonpArray.slice();
for (var i = 0; i < jsonpArray.length; i++) webpackJsonpCallback(jsonpArray[i]);
var parentJsonpFunction = oldJsonpFunction;
return __webpack_require__(__webpack_require__.s = 0);
})([/* 0 */
       (function (module, exports, __webpack_require__) {
		 function hello() {
		 __webpack_require__.e(/* import() */ 1).then(__webpack_require__.bind(null, 1)).then(res => console.log(res))
		 }
		 hello()
	 })
]);
```
还是先从入口文件看起，当执行`hello`方法时，会执行以下逻辑：
```javascript
__webpack_require__.e(/* import() */ 1).then(__webpack_require__.bind(null, 1)).then(res => console.log(res))
```
但我们发现，webpack打包出来的依赖数组中，就只有`hello`模块，并没有其他模块了，如果在`hello`中直接使用`__webpack_require__(1)`，那就会报错。但在`__webpack_require__.e(/* import() */ 1)`之后就能正常引用了。`__webpack_require__.e`这个方法到底做了什么呢？我们先来看看它的定义：
```javascript

  // 用来存储已加载和加载中的chunk
  // undefined表示chunk未加载，null表示chunk preloaded/prefetched
  // Promise = chunk loading, 0 = chunk loaded
  var installedChunks = {
	0: 0 // 表示模块0已加载
  };
__webpack_require__.e = function requireEnsure(chunkId) {
  var promises = [];
  // JSONP chunk loading for javascript
  var installedChunkData = installedChunks[chunkId]; // 当前chunkId为1，值为undefined，表示未加载,
  if (installedChunkData !== 0) { // 0 表示chunk已经安装了
    // installedChunkData只有0, undefined, null ,promise4种类型的值。如果installedChunkData布尔值为true，表示installedChunkData为一个数组，chunk加载中
    if (installedChunkData) { // 
      promises.push(installedChunkData[2]);
    } else {
      // setup Promise in chunk cache
      var promise = new Promise(function (resolve, reject) {
        installedChunkData = installedChunks[chunkId] = [resolve, reject];
      });
      promises.push(installedChunkData[2] = promise); // installedChunkData = [resolve, reject，promise]，例如installedChunkData保存新创建的promise以及他的resolve和reject
      // promises => [promise]
      // 开始加载chunk
      var script = document.createElement('script');
      var onScriptComplete;
      script.charset = 'utf-8';
      script.timeout = 120;
      if (__webpack_require__.nc) {
        script.setAttribute("nonce", __webpack_require__.nc);
      }
      script.src = jsonpScriptSrc(chunkId);
      // create error before stack unwound to get useful stacktrace later
      var error = new Error();
      onScriptComplete = function (event) {
        // avoid mem leaks in IE.
        script.onerror = script.onload = null;
        clearTimeout(timeout);
        var chunk = installedChunks[chunkId]; // [resolve, reject，promise]
        if (chunk !== 0) {
          if (chunk) { // reject erro
            var errorType = event && (event.type === 'load' ? 'missing' : event.type);
            var realSrc = event && event.target && event.target.src;
            error.message = 'Loading chunk ' + chunkId + ' failed.\n(' + errorType + ': ' + realSrc + ')';
            error.name = 'ChunkLoadError';
            error.type = errorType;
            error.request = realSrc;
            chunk[1](error);
          }
          installedChunks[chunkId] = undefined;
        }
      };
      var timeout = setTimeout(function () {
        onScriptComplete({ type: 'timeout', target: script });
      }, 120000);
      script.onerror = script.onload = onScriptComplete; // load事件会在script加载并执行完才触发
      document.head.appendChild(script);
    }
  }
  return Promise.all(promises);
};
```
`__webpack_require__.e`的作用是判断模块是否已加载（或加载中），如果都不是，就利用`jsonp`加载模块。该方法会返回一个`promise`，当`script`加载并执行成功时会`resolve`，当加载失败时`reject`。这里需要注意的是，**`script`的`load`事件会在脚本下载并执行完之后触发**。

通过`jsonp`加载的`1.bundle.js`的实现如下：
```javascript
(window["webpackJsonp"] = window["webpackJsonp"] || []).push([[1], [/* 0 */,  /* 1 */
  (function (module, __webpack_exports__, __webpack_require__) {
    "use strict";
    __webpack_require__.r(__webpack_exports__);
/* harmony default export */ __webpack_exports__["default"] = (function () {
      return 'dynamic'
    });
  })
]]);
```
不难发现，`js`加载完成后，调用了` window["webpackJsonp"]`的`push`方法。当这个`push`方法可不是数组原生当`push`方法，初始化时`webpack`做了一个小动作，关键代码如下：
```javascript
var jsonpArray = window["webpackJsonp"] = window["webpackJsonp"] || [];
var oldJsonpFunction = jsonpArray.push.bind(jsonpArray); // oldJsonpFunction -> push，代理jsonpArray的push方法
jsonpArray.push = webpackJsonpCallback; 
```
所以在执行push方法时，实际上是执行webpackJsonpCallback。我们来看一下这个函数的定义：
```javascript
function webpackJsonpCallback(data) {
  var chunkIds = data[0];
  var moreModules = data[1];
  // 把新加载的模块加到modules数组中
  // 把模块标记为已加载，然后resolve
  var moduleId, chunkId, i = 0, resolves = [];
  for (; i < chunkIds.length; i++) {
    chunkId = chunkIds[i];
    if (Object.prototype.hasOwnProperty.call(installedChunks, chunkId) && installedChunks[chunkId]) {
      resolves.push(installedChunks[chunkId][0]); // 把chunk的resolve方法放到resolves数组中，后面统一resolve
    }
    installedChunks[chunkId] = 0; // 把模块标记为已加载
  }
  for (moduleId in moreModules) {
    if (Object.prototype.hasOwnProperty.call(moreModules, moduleId)) {
      modules[moduleId] = moreModules[moduleId]; // 把新加载回来的模块添加到之前的模块数组中
    }
  }
  if (parentJsonpFunction) parentJsonpFunction(data); // 把模块数据放到window["webpackJsonp"] 数组中
  while (resolves.length) {
    resolves.shift()(); // resolve the promise
  }
};
```
这个函数其实就做了几件事：
- 把模块标记为已加载，并追加到modules数组中
- 把模块数据放到window["webpackJsonp"] 数组中
- 调用之前那些promise的resolve

因此，当`__webpack_require__.e(/* import() */ 1).then`时，就能通过`__webpack_require__`拿到动态获取的模块了。
```javascript
__webpack_require__.e(/* import() */ 1).then(__webpack_require__.bind(null, 1)).then(res => console.log(res))
```
