# pseudo-worker

浏览器-web-worker-polyfill

[![explain](http://llever.com/explain.svg)](https://github.com/chinanf-boy/Source-Explain)
    
Explanation

> "version": "1.0.0"

[github source](https://github.com/nolanlawson/pseudo-worker)

~~[english](./README.en.md)~~

---

- [出发前准备](#确定主文件)

---

1. [定义局部变量](#定义局部变量)

2. [定义-生命周期-钩子](#生命周期-钩子)

- [executeEach](#executeEach)

- [callErrorListener](#callErrorListener)

- [addEventListener](#addEventListener)

- [removeEventListener](#removeEventListener)

- [postError](#postError)

- [runPostMessage](#runPostMessage)

- [postMessage](#postMessage)

- [terminate](#terminate)

- [workerPostMessage](#workerPostMessage)

- [workerAddEventListener](#workerAddEventListener)

3. [主逻辑 - 开启 - 最好从这里开始](#主逻辑)

4. [有关-web-worker的资料](https://developer.mozilla.org/zh-CN/docs/Web/API/Worker/Worker)

---

- ~~polyfill.js~~

---

说明一下，两个关键点

1. `Myworker` 

本项目 `pseudo.worker`, 作为 `Worker` 的 polyfill

> Myworker = new Worker(`worker.js`) // Worker == pseudo-worker

2. `worker.js`

这就是-**Worker-Api**-引入的, 脚本文件

[可以看看有关-web-worker的资料](https://developer.mozilla.org/zh-CN/docs/Web/API/Worker/Worker)

---

## 确定主文件

package.json

``` json
"main": "index.js",
```

## index

如果-你是一开始看-[请点击从定义变量开始](#定义局部变量)

### doEval

代码 1-9

``` js
'use strict';

function doEval(self, __pseudoworker_script) {
  /* jshint unused:false */
  (function () {
    /* jshint evil:true */
    eval(__pseudoworker_script);
  }).call(global);
}

```

> 运行-`worker.js`中脚本

请注意，主线程中必须以`myWorker.onmessage`方式调用,  反之`worker.js` 脚本中, 只需定义 onmessage, 因为worker.js全域有效(`DedicatedWorkerGlobalScope`).

该 `DedicatedWorkerGlobalScope` 对象(也就是 Worker 全局作用域)可以通过 `self` 关键字来访问.

这也就是说

``` js
        workerSelf = {
          postMessage: workerPostMessage,
          addEventListener: workerAddEventListener,
        };
        // doEval(workerSelf,
        // 变 workerSelf == self
        // function doEval(self, 
```

### pseudo-Worker

#### 定义局部变量

代码 11-21

``` js
function PseudoWorker(path) {
  var messageListeners = [];
  var errorListeners = [];
  var workerMessageListeners = [];
  var workerErrorListeners = [];
  var postMessageListeners = [];
  var terminated = false;
  var script;
  var workerSelf;

  var api = this;


```

#### 生命周期-钩子

这部分都是函数定义，可选择[下一小段「主逻辑」](#主逻辑)

代码 23-136

#### executeEach

``` js
  // custom each loop is for IE8 support
  function executeEach(arr, fun) {
    var i = -1;
    while (++i < arr.length) {
      if (arr[i]) {
        fun(arr[i]);
      }
    }
  }

```

对`arr`每个值，运行`fun(arr[i])`

---

#### callErrorListener

``` js
  function callErrorListener(err) {
    return function (listener) {
      listener({
        type: 'error',
        error: err,
        message: err.message
      });
    };
  }

``` 

更换-`err`-数据结构

---

#### addEventListener

作为 `Myworker.addEventListener`

``` js
  function addEventListener(type, fun) {
    /* istanbul ignore else */
    if (type === 'message') {
      messageListeners.push(fun);
    } else if (type === 'error') {
      errorListeners.push(fun);
    }
  }

```

``` js
Myworker = new Worker()
```

- `messageListeners` message信息-触发函数->存储

- `errorListeners` err信息-触发函数->存储

---

#### removeEventListener

作为 `Myworker.removeEventListener`

``` js
  function removeEventListener(type, fun) {
      var listeners;
      /* istanbul ignore else */
      if (type === 'message') {
        listeners = messageListeners;
      } else if (type === 'error') {
        listeners = errorListeners;
      } else {
        return;
      }
      var i = -1;
      while (++i < listeners.length) {
        var listener = listeners[i];
        if (listener === fun) {
          delete listeners[i];
          break;
        }
      }
  }
```

``` js
Myworker = new Worker()
```

- listeners

对`type` = 'message'|'error'

然后遍历 `while listeners`

删除相等的函数 `if (listener === fun) {`

---

#### postError

规范错误输出, 并执行 `Myworker.onerror` 触发函数

``` js
  function postError(err) {
    var callFun = callErrorListener(err);
    if (typeof api.onerror === 'function') {
      callFun(api.onerror);
    }
    if (workerSelf && typeof workerSelf.onerror === 'function') {
      callFun(workerSelf.onerror);
    }
    executeEach(errorListeners, callFun);
    executeEach(workerErrorListeners, callFun);
  }
```

- [callErrorListener 改变数据存储结构-返回函数](#callerrorlistener)

- [executeEach-遍历第一个变量 arr, 运行第二个变量fun(arr[i])](#executeeach)

---

#### runPostMessage

传输-在`worker.js`-onmessage->postMessage(`msg`) 的 信息

``` js
 function runPostMessage(msg) {
    function callFun(listener) {
      try {
        listener({data: msg});
      } catch (err) {
        postError(err); 
      }
    }

    if (workerSelf && typeof workerSelf.onmessage === 'function') {
      callFun(workerSelf.onmessage);
    }
    executeEach(workerMessageListeners, callFun);
  }

```

- [postError 触发Myworker.onerror](#posterror)

- [executeEach-遍历第一个变量 arr, 运行第二个变量fun(arr[i])](#executeeach)

---

#### postMessage

Myworker.postMessage 的 api

``` js
  function postMessage(msg) {
    if (typeof msg === 'undefined') {
      throw new Error('postMessage() requires an argument');
    }
    if (terminated) { // 如果-停
      return;
    }
    if (!script) { // 并没有脚本
      postMessageListeners.push(msg); // 存储一下信息
      return;
    }
    runPostMessage(msg);
  }

```

``` js
Myworker = new Worker()
```

> 作为 `Myworker.postMessage`

- runPostMessage

---

#### terminate

设置-传输停止

``` js
  function terminate() {
    terminated = true;
  }
```

``` js
Myworker = new Worker()
```

> 作为 `Myworker.terminate`

---

#### workerPostMessage

作为 `worker.js 中的 postMessage`

``` js
// worker.js
onmessage = function(e) {
  console.log('Message received from main script');
  var workerResult = 'Result: ' + (e.data[0] * e.data[1]);
  console.log('Posting message back to main script');
  postMessage(workerResult); // <---
  // 输出信息给 worker.onmessage
}
```

``` js
  function workerPostMessage(msg) {
    function callFun(listener) {
      listener({
        data: msg
      });
    }
    if (typeof api.onmessage === 'function') {
      // Myworker = new Worker(path)
      // Myworker.onmessage 有所定义
      callFun(api.onmessage); // 传输到外面
    }
    executeEach(messageListeners, callFun);
  }
```

我们结合- 本节`worker.js` 例子说明一下，

- [`executeEach(messageListeners, callFun);`](#executeeach)


---

#### workerAddEventListener

作为 `worker.js 中的 addEventListener`

``` js
  function workerAddEventListener(type, fun) {
    /* istanbul ignore else */
    if (type === 'message') {
      workerMessageListeners.push(fun);
    } else if (type === 'error') {
      workerErrorListeners.push(fun);
    }
  }
```


---

#### 主逻辑

代码 138-171

``` js
  var xhr = new XMLHttpRequest();  // 准备 ajax

  xhr.open('GET', path); // path == worker.js
  xhr.onreadystatechange = function () {
    if (xhr.readyState === 4) {
      if (xhr.status >= 200 && xhr.status < 400) {
        script = xhr.responseText; // worker.js 内容
        workerSelf = {
          postMessage: workerPostMessage,
          addEventListener: workerAddEventListener,
        };
        doEval(workerSelf, script);  // --- 1
        var currentListeners = postMessageListeners; // 一开始是 []
        postMessageListeners = [];
        for (var i = 0; i < currentListeners.length; i++) {
          runPostMessage(currentListeners[i]); // --- 2
        }
      } else {
        postError(new Error('cannot find script ' + path));
      }
    }
  };

  xhr.send(); // 开启

  api.postMessage = postMessage; 
  // worker.postMessage
  api.addEventListener = addEventListener; 
  // worker.addEventListener
  api.removeEventListener = removeEventListener;
  // worker.removeEventListener
  api.terminate = terminate;
  // worker.terminate

  return api;
}

module.exports = PseudoWorker;
```

---

在这里，我们再理清一下`worker`的信息传输顺序

> Myworker = new Worker(path)

- 从`Myworker.postMessage(message)`

- message - e.data -> `worker.js` 中 `onmessage(e)`

- `worker.js`- `onmessage(e)` 中 `postMessage(msg)` ->

- 传输-msg-到 e.data -> `Myworker.onmessage(e)`

---

1. [`doEval(workerSelf, script); 请先看这个 >>`](#doeval)

在`xhr.send();` - 我们对·path·的获取 - 触发`onreadystatechange` 函数 - 运行`doEval 函数`

2. `workerPostMessage` 赋予 `worker.js` - postMessage

> [workerPostMessage ](#workerpostmessage)

3. `workerAddEventListener` 赋予 `worker.js` - addEventListener

> [workerAddEventListener](#workeraddeventlistener)

