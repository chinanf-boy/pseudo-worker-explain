# pseudo-worker

浏览器-web-worker-polyfill

[![explain](http://llever.com/explain.svg)](https://github.com/chinanf-boy/Source-Explain)
    
Explanation

> "version": "1.0.0"

[github source](https://github.com/)

~~[english](./README.en.md)~~

---



- [有关-web-worker的资料](https://developer.mozilla.org/zh-CN/docs/Web/API/Worker/Worker)

---

## 确定主文件

package.json

``` json
"main": "index.js",
```

## index

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

### pseudoWorker

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

  function callErrorListener(err) {
    return function (listener) {
      listener({
        type: 'error',
        error: err,
        message: err.message
      });
    };
  }

  function addEventListener(type, fun) {
    /* istanbul ignore else */
    if (type === 'message') {
      messageListeners.push(fun);
    } else if (type === 'error') {
      errorListeners.push(fun);
    }
  }

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

  function postMessage(msg) {
    if (typeof msg === 'undefined') {
      throw new Error('postMessage() requires an argument');
    }
    if (terminated) {
      return;
    }
    if (!script) {
      postMessageListeners.push(msg);
      return;
    }
    runPostMessage(msg);
  }

  function terminate() {
    terminated = true;
  }

  function workerPostMessage(msg) {
    function callFun(listener) {
      listener({
        data: msg
      });
    }
    if (typeof api.onmessage === 'function') {
      callFun(api.onmessage);
    }
    executeEach(messageListeners, callFun);
  }

  function workerAddEventListener(type, fun) {
    /* istanbul ignore else */
    if (type === 'message') {
      workerMessageListeners.push(fun);
    } else if (type === 'error') {
      workerErrorListeners.push(fun);
    }
  }

```

#### executeEach

#### callErrorListener

#### addEventListener

#### removeEventListener

#### postError

#### runPostMessage

#### postMessage

#### terminate

#### workerPostMessage

worker.js

``` js
onmessage= // ..
```
#### workerAddEventListener

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

- 从`Myworker.postMessage(message)`

- message - e.data -> `worker.js` 中 `onmessage(e)`

- `worker.js`- `onmessage(e)` 中 `postMessage(msg)`

- 传输-msg-到 e.data -> `Myworker.onmessage(e)`

---

1. [`doEval(workerSelf, script); 解释 >>`](#doeval)

在`xhr.send();` - 我们对·path·的获取 - 触发 - `onreadystatechange` 函数- `doEval`


2. `workerPostMessage` 赋予 `worker.js` - postMessage

> [workerPostMessage](#workerpostmessage)
