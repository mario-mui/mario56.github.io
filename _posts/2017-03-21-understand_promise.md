---
layout: blog
title: "理解 Promise 以及实现"
type: JavaScript
time: 2017-03-21

---

# JavaScript Promises
Promise 是在 JavaScript中解决回调地狱的方法之一，当然要有一些规范[promise/A+](https://promisesaplus.com/), promise 如何使用的？[promise](https://www.promisejs.org/), [bluebird](http://bluebirdjs.com/docs/getting-started.html).这里就不说了。

## 让我们以最简单的形式开始,从下面这种形式

```
doSomething(function(value) {
  console.log('Got a value:' + value);
});

```

到这种形式

```
doSomething().then(function(value) {
  console.log('Got a value:' + value);
});

```

为了实现上面的这种形式转变，我们要做的是把 doSomething() 的实现从

```
function doSomething(callback) {
  var value = 42;
  callback(value);
}
```
变成这样

```
function doSomething() {
  return {
    then: function(callback) {
      var value = 42;
      callback(value);
    }
  };
}

```
到这里我们得到一个信息，promise 把所有可能性的返回包装成一个对象。

```
function Promise(fn) {
  var ctx = this;
  ctx.callback = null;
  function resolve(value) {
    ctx.callback(value);
  }
  fn(resolve);
}

Promise.prototype.then = function(cb) {
  this.callback = cb;
};

```

重新实现doSomething()

```
function doSomething() {
  return new Promise(function(resolve) {
    var value = 42;
    resolve(value);
  });
}

```
这里有一个问题。如果你跟踪执行，你会看到resolve()在then()之前被调用，这意味着回调将为null。我们使用setImmediate()。为什么这么做请看[这里](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/#setimmediate-vs-settimeout)

```
function Promise(fn) {
  var ctx = this;
  ctx.callback = null;
  function resolve(value) {
    setImmediate(()=>{
      ctx.callback(value);
    })
  }
  fn(resolve);
}

Promise.prototype.then = function(cb) {
  this.callback = cb;
};

```

这是利用异步的方式，当然这是比较脆弱的，假如我们在异步中调用then(),callback 还是会是null比如：

```

setImmediate(()=>{
  promise.then((res)=>{
  console.log(res)
  })
})

```

经过我们脆弱的代码，我们发现，Promise 是需要状态的，我们需要知道他们在被执行之前的状态，然后确定我们是正确的切换状态。

- promise 可以在 pending 状态，等待一个值，也可以 resolved 一个值(当然还有reject一个错误)
- 一旦Promise resolve 一个值，它将一直保持在那个值，不会再resolve一次

重新写一下我们的Promise

```
function Promise(fn) {
  var state = 'pending';
  var value;
  var deferred;

  function resolve(newValue) {
    value = newValue;
    state = 'resolved';

    if(deferred) {
      handle(deferred);
    }
  }

  function handle(onResolved) {
    if(state === 'pending') {
      deferred = onResolved;
      return;
    }

    onResolved(value);
  }

  this.then = function(onResolved) {
    handle(onResolved);
  };

  fn(resolve);
}

```

假如 resolve 的调用是同步的

```

var promise = new Promise((resolve)=>{
    resolve('111')
})

```

当调用promise的then 方法时候，promise 对象的state就是'resolved',这时候就直接callback(value)

如果 resolve 的调用在异步中

```
var p= new P((resolve)=>{
  setTimeout(()=>{
    resolve('111')
  },5000)
})

```
当调用promise的then 方法时候，promise 对象的state就是'pending', deferred 值会被复制为 onResolved（就是在then方法里的callback）当5秒之后，触发resolve的时候，state的值为'resolved' value的值为resolve传进来的值, `if(deferred)` 这条判断就是 `true` handle方法被调用，这个时候会执行 onResolved(value)。

这个时候我们的promise还只能处理一个then

```
promise.then((res)=>{
  console.log('first then',res)
})

promise.then((res)=>{
  console.log('second then',res)
})

```
我们只会得到`second then`,解决这个问题的就是把我们的deferred变为数组就可以了。

```
var deferred = [];

  function resolve(newValue) {
    value = newValue;
    state = 'resolved';
    if(deferred.length) {
      for(var i = 0,j=deferred.length; i<j; i++ ){
        handle(deferred[i]);
      }
    }
  }

  function handle(onResolved) {
    if(state === 'pending') {
      deferred.push(onResolved)
      return;
    }

    onResolved(value);
  }
```

## 链式的 promise
以下是我们想要的链式使用promise的方式

```
getSomeData()
.then(step1)
.then(step2)
.then(step3);

```
这就要求我们的then方法也是要返回一个promise的对象的。
下面就是我们添加链式调用支持的promise(第一版)

```
function Promise(fn) {
  var state = 'pending';
  var value;
  var deferred;
  var _next;

  function resolve(newValue) {
    value = newValue;
    state = 'resolved';
    if(_next)
      handle(deferred, _next);
  }

  function handle(onResolved, next) {
    if(state === 'pending') {
      deferred = onResolved;
      _next = next;
      return;
    }

    if(!onResolved){
      next(value)
      return;
    }

    var ret = onResolved(value);
    next(ret)
  }

  this.then = function(onResolved) {
    return new Promise((_resolve)=>{
      handle(onResolved, _resolve);
    })
  };
  fn(resolve);
}


```

我们把 `handle(onResolved, resolve)` 中的 `onResolved` 和 `resolve` 封装成一个对象

```
function Promise(fn) {
  var state = 'pending';
  var value;
  var deferred = null;

  function resolve(newValue) {
    value = newValue;
    state = 'resolved';

    if(deferred) {
      handle(deferred);
    }
  }

  function handle(handler) {
    if(state === 'pending') {
      deferred = handler;
      return;
    }

    if(!handler.onResolved) {
      handler.resolve(value);
      return;
    }

    var ret = handler.onResolved(value);
    handler.resolve(ret);
  }

  this.then = function(onResolved) {
    return new Promise(function(resolve) {
      handle({
        onResolved: onResolved,
        resolve: resolve
      });
    });
  };

  fn(resolve);
}

```

这个时候我们的Promis在then的函数里还只能处理return value，假如 value 是一个 promise 的话，就会发生一些我们不想看到的事情。
```
doSomething().then(function(result) {
  // doSomethingElse returns a promise
  return doSomethingElse(result);
}).then(function(res) {
  console.log("the 2 result is", res);
}).then(function(finalResult) {
  console.log("the final result is", finalResult);
});

// console.log("the final result is", finalResult);
// console.log("the 2 result is", res);

```

显然上面的打印顺序不是我们想要的。当然我们也可以这样去写

```
doSomething().then(function(result) {
  // doSomethingElse returns a promise
  return doSomethingElse(result);
}).then(function(anotherPromise) {
  anotherPromise.then(function(finalResult) {
    console.log("the final result is", finalResult);
  });
});

```
这样的话，如果anotherPromise里再次返回一个promise的话，又会进入回调的地狱了，这也不是我们想要的。

那么修改我们的resolve函数

```

function resolve(newValue) {
  if(newValue && typeof newValue.then === 'function') {
    newValue.then(resolve);
    return;
  }
  state = 'resolved';
  value = newValue;

  if(deferred) {
    handle(deferred);
  }
}

```

如果 newValue 是 promise 的时候，直接 resolve() 掉 newValue.then 的返回值。简写的话就是 `newValue.then(resolve)`
到现在，我们已经基本完成了 resolve 的功能。当然别忘了，还有reject用了处理出错的情况，我们还没有实现。

当出现错误的时候，需要给出一个原因去 rejected ，让调用者知道当出现错的时候，所以我们在then函数里加入第二个参数。

```
doSomething().then(function(value) {
  console.log('Success!', value);
}, function(error) {
  console.log('Uh oh', error);
});
```

下面是增加了错误处理的doSomething实现

```
function doSomething() {
  return new Promise(function(resolve, reject) {
    var result = somehowGetTheValue();
    if(result.error) {
      reject(result.error);
    } else {
      resolve(result.value);
    }
  });
}
```
 
promise 的实现

```

function Promise(fn) {
  var state = 'pending';
  var value;
  var deferred = null;

  function resolve(newValue) {
    if(newValue && typeof newValue.then === 'function') {
      newValue.then(resolve, reject);
      return;
    }
    state = 'resolved';
    value = newValue;

    if(deferred) {
      handle(deferred);
    }
  }

  function reject(reason) {
    state = 'rejected';
    value = reason;

    if(deferred) {
      handle(deferred);
    }
  }

  function handle(handler) {
    if(state === 'pending') {
      deferred = handler;
      return;
    }

    var handlerCallback;

    if(state === 'resolved') {
      handlerCallback = handler.onResolved;
    } else {
      handlerCallback = handler.onRejected;
    }

    if(!handlerCallback) {
      if(state === 'resolved') {
        handler.resolve(value);
      } else {
        handler.reject(value);
      }

      return;
    }

    var ret;
    try {
      ret = handlerCallback(value);
    } catch(e) {
      handler.reject(e);
      return;
    }
    handler.resolve(ret);
  }

  this.then = function(onResolved, onRejected) {
    return new Promise(function(resolve, reject) {
      handle({
        onResolved: onResolved,
        onRejected: onRejected,
        resolve: resolve,
        reject: reject
      });
    });
  };


  //this.catch = function(onRejected) {
  //  return this.then(undefined, onRejected)
  //}

  fn(resolve, reject);
}


```

完成，catch 的方法就是调用then方法，第一个参数传入 undefined