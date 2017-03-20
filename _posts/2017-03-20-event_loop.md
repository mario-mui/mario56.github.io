---
layout: blog
title: "[译]The Node.js Event Loop, Timers, and process.nextTick()"
type: Node
time: 2017-03-20

---

## 什么事 Event Loop

Evebt Loop 就是那个允许 Nodejs 执行非阻塞I/O操作的东西。尽管JavaScript是单线程的。
现代大多数的内核都是多线程的，它们可以在后台处理多I/O操作。单其中一个操作执行完了，它会告诉Node.js 把适当的 callback 添加到 poll 阶段的队列中，最终这个callback会被执行。

## 解析 Event Loop

当node 启动时候，会初始化 Event Loop。执行提供的输入的脚本文件，这文件中可能包含了一些异步操作（async API calls, schedule timers, or call process.nextTick()），然后开始执行 Event Loop。

下图显示了一个简化版的事件循环的操作执行顺序。


```

   ┌───────────────────────┐
┌─>│        timers         │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     I/O callbacks     │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     idle, prepare     │
│  └──────────┬────────────┘      ┌───────────────┐
│  ┌──────────┴────────────┐      │   incoming:   │
│  │         poll          │<─────┤  connections, │
│  └──────────┬────────────┘      │   data, etc.  │
│  ┌──────────┴────────────┐      └───────────────┘
│  │        check          │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
└──┤    close callbacks    │
   └───────────────────────┘


```

_注意:每个事件箱子将被称为“阶段”（phase)_

每个阶段都有一个先进先出的 callbacks 队列，每一个阶段都有自己特殊的地方，通常来说，当Event Loop 进入一个给定的阶段，它将执行该阶段特定的任何操作，然后执行这个阶段队列里的 callbacks， 直到这个队列里的 callbacks 都被执行完，或者达到队列的callbacks最大执行数目。这个时候，Event Loop 将会进入下一个阶段。

由于这些操作中的任何一个可以调度更多的操作，并且在轮询阶段中处理的新事件被内核排队，所以轮询事件可以在轮询事件被处理时排队。 因此，长时间运行的回调可以允许 pool 阶段运行比 timer 的阈值长得多。 有关更多详细信息，请参阅 timer 和 poll 部分。

## 阶段概述

- timers: 这个阶段执行 setTimeout() 和 setInterval() 的 callbacks。
- I/O callbacks: 执行大部分的 callbacks 除了 close callbacks，那些被 timer 或 setImmediate() 设置的callbacks
- idle, prepare: 仅供内部使用。
- pool: 检索新的 I/O 事件，适当的时候 node 会在这个阶段阻塞。
- check: setImmediate() 的 callbacks 被执行。
- close callbacks: 例如 socket.on('close', ...).

_在事件循环的每次运行之间，Node.js检查它是否正在等待任何异步I / O或计时器，如果没有任何异常I / O或计时器完全关闭Node。_

## 阶段详情

### timer

计时器指定阈值，在该阈值之后，可以执行提供的回调，而不是人们希望其被执行的确切时间。 计时器回调将在指定的时间过去之后尽早运行; 但是，操作系统调度或其他回调的运行可能会延迟它们。

_注：技术上，pool 阶段控制何时执行定时器_

例如，假设您计划在超过100毫秒的阈值后执行超时，那么您的脚本以异步方式读取需要95毫秒的文件：

```
var fs = require('fs');

function someAsyncOperation (callback) {
  // Assume this takes 95ms to complete
  fs.readFile('/path/to/file', callback);
}

var timeoutScheduled = Date.now();

setTimeout(function () {

  var delay = Date.now() - timeoutScheduled;

  console.log(delay + "ms have passed since I was scheduled");
}, 100);


// do someAsyncOperation which takes 95 ms to complete
someAsyncOperation(function () {

  var startCallback = Date.now();

  // do something that will take 10ms...
  while (Date.now() - startCallback < 10) {
    ; // do nothing
  }

});

```

当事件循环进入pool 阶段时，它有一个空队列（fs.readFile()尚未完成），因此它将等待剩余的毫秒数，直到达到最快的计时器阈值。 当它等待95 ms通过时，fs.readFile()完成读取文件，它的回调需要10毫秒来完成，它被添加到 pool 队列并执行。 当回调完成时，队列中没有更多的回调，所以事件循环将看到已经到达最快定时器的阈值，然后回到 timer 阶段以执行定时器的回调。 在此示例中，您将看到正在调度的计时器与正在执行的回调之间的总延迟将为105ms。


### I/O callbacks

此阶段执行某些系统操作的回调，例如TCP错误的类型。 例如，如果TCP套接字尝试连接时收到ECONNREFUSED，某些 *nix系统会等待报告错误。 这将排队在I / O回调阶段执行。

### poll

这个 poll 阶段主要有两个功能：

- 为阈值已过的定时器执行脚本
- 处理轮询队列中的事件

当事件循环进入 pool 阶段并且没有计划任何计时器时，会发生以下两种情况之一：

- 如果 pool 队列不为空，则事件循环将遍历其回调队列，同步执行它们，直到队列耗尽或达到系统相关的硬限制。
- 如果 pool 队列为空
  - 如果脚本已由setImmediate()调度，则事件循环将结束 pool 阶段，并继续执行 check 阶段以执行这些调度的脚本。
  - 如果脚本没有被setImmediate()调度，事件循环将等待回调被添加到队列，然后立即执行它们。

一旦 pool 队列为空，事件循环将检查已达到其时间阈值的计时器。如果一个或多个定时器就绪，事件循环将返回到 timer 阶段以执行那些定时器的回调。

### check

此阶段允许人员在 pool 阶段完成后立即执行回调。如果 pool 阶段变为空闲并且脚本已使用setImmediate()排队，则事件循环可以继续到 check 阶段，而不是等待。

setImmediate()实际上是一个在事件循环的单独阶段运行的特殊计时器。它使用一个libuv API，调度的回调在 pool 阶段完成后执行。

通常，当代码被执行时，事件循环最终将到达 pool 阶段，它将等待传入连接，请求等等。然而，如果回调已经被setImmediate()调度并且 pool 阶段变为空闲， 将结束并继续到 pool 阶段，而不是等待轮询事件。

### close callbacks

如果套接字或句柄突然关闭（例如socket.destroy()），则在此阶段将发出“关闭”事件。否则它将通过process.nextTick() 触发。


## setImmediate() vs setTimeout()

setImmediate和setTimeout() 是相似的，但是行为方式不同，取决于它们被调用的时间。

- `setImmediate()` 设计是为了在当前 pool 阶段完成后执行脚本。
- `setTimeout()`调度要在以ms为单位的最小阈值过后运行的脚本。

计时器的执行顺序将根据它们被调用的上下文而变化。 如果两者都在主模块内调用，则时间将受进程的性能（其可受到在机器上运行的其他应用影响）的约束。

例如，如果我们运行不在I/O周期（即主模块）内的以下脚本，两个定时器的执行顺序是非确定性的，因为它受进程的性能的限制：

```
// timeout_vs_immediate.js
setTimeout(function timeout () {
  console.log('timeout');
},0);

setImmediate(function immediate () {
  console.log('immediate');
});

```

```
$ node timeout_vs_immediate.js
timeout
immediate

$ node timeout_vs_immediate.js
immediate
timeout

```

但是，如果在I/O周期内移动两个调用，则总是首先执行立即回调：

```

// timeout_vs_immediate.js
var fs = require('fs')

fs.readFile(__filename, () => {
  setTimeout(() => {
    console.log('timeout')
  }, 0)
  setImmediate(() => {
    console.log('immediate')
  })
})

```

```

$ node timeout_vs_immediate.js
immediate
timeout

$ node timeout_vs_immediate.js
immediate
timeout

```

使用setImmediate() 多于 setTimeout() 的主要优点是setImmediate()将总是在任何定时器之前执行，如果在I/O周期内调度，独立于存在多少定时器。

## process.nextTick()

### 理解 process.nextTick()

您可能已经注意到，process.nextTick()没有显示在图表中，即使它是异步API的一部分。 这是因为process.nextTick()在技术上不是事件循环的一部分。 相反，nextTickQueue将在当前操作完成后被处理，而不管事件循环的当前相位。

回顾我们的Event loop图，任何时候在给定阶段调用process.nextTick()，传递给process.nextTick()的所有回调都将在事件循环继续之前解决。 这可以创建一些不好的情况，**因为它允许您通过递归process.nextTick()调用来“挨饿”您的I/O，这会阻止事件循环到达轮询阶段**。

### process.nextTick() 为什么被允许呢？

什么会这样的东西包含在Node.js中？它的一部分是一个设计哲学，其中一个API应该始终是异步的，即使不是必须的。以下面的代码片段为例：

```
function apiCall (arg, callback) {
  if (typeof arg !== 'string')
    return process.nextTick(callback,
      new TypeError('argument should be string'));
}

```

该代码段进行参数检查，如果它不正确，它会将错误传递给回调。 API最近更新，允许传递参数到process.nextTick(),允许它传递回传递的任何参数作为参数传播到回调，所以你不必嵌套函数。

我们所做的是将错误传回给用户，但只有在我们允许用户代码的其余部分执行后。 通过使用process.nextTick()，我们保证apiCall()总是在用户代码的剩余部分之后，允许继续事件循环之前运行它的回调。 为了实现这一点，允许JS调用栈释放，然后立即执行提供的回调，允许一个人递归调用process.nextTick()，而不会达到RangeError：从v8超出的最大调用堆栈大小。

```
// this has an asynchronous signature, but calls callback synchronously
function someAsyncApiCall (callback) { callback(); };

// the callback is called before `someAsyncApiCall` completes.
someAsyncApiCall(() => {

  // since someAsyncApiCall has completed, bar hasn't been assigned any value
  console.log('bar', bar); // undefined

});

var bar = 1;

```

用户定义someAsyncApiCall()以具有异步性？但它实际上是同步操作。 当它被调用时，提供给someAsyncApiCall()的回调在事件循环的同一阶段被调用，因为someAsyncApiCall()实际上并不异步地做任何事情。 因此，回调尝试引用bar，即使它可能尚未有范围中的该变量，因为脚本无法运行到完成。

通过将回调放在process.nextTick()中，脚本仍然能够运行到完成，允许在调用回调之前初始化所有的变量，函数等。 它还具有不允许事件循环继续的优点。 在事件循环被允许继续之前向用户警告错误可能是有用的。 下面是使用process.nextTick()的上一个示例：

```

function someAsyncApiCall (callback) {
  process.nextTick(callback);
};

someAsyncApiCall(() => {
  console.log('bar', bar); // 1
});

var bar = 1;

```

一个真实的例子：

```

const server = net.createServer(() => {}).listen(8080);

server.on('listening', () => {});

```

## process.nextTick() vs setImmediate()

对于用户来说，我们有两个类似的调用，但是他们的名字很混乱。

- process.nextTick() 在同一个阶段立即触发
- setImmediate() 在事件循环的以下迭代或“tick”上触发

实质上，名称应该被交换。 process.nextTick() 比 setImmediate() 更快地触发，但这是一个不可能改变的过去的工件。 做这个开关会在npm打破很大一部分的包。 每天都有更多的新模块被添加，这意味着每天我们等待，更多的潜在破损发生。 虽然他们很困惑，名字本身不会改变。

我们建议开发人员在所有情况下使用setImmediate()，因为它更容易推理（并且它导致代码与更广泛的环境兼容，如浏览器JS）。

## 为什么使用 process.nextTick()?

两个主要原因：

- 允许用户处理错误，清除任何不需要的资源，或者可以在事件循环继续之前再次尝试请求。
- 有时，有必要允许回调在调用堆栈展开后但在事件循环继续之前运行。

一个例子是匹配用户的期望。简单示例：

```
var server = net.createServer();
server.on('connection', function(conn) { });

server.listen(8080);
server.on('listening', function() { });
```

就是说 listen()在事件循环的开始运行，但监听回调放在setImmediate()中。 现在，除非主机名被传递，绑定到端口将立即发生。 现在对于事件循环继续它必须击中轮询阶段，这意味着有一个非零的机会，一个连接可能已经接收允许连接事件在侦听事件之前被触发。

另一个例子是运行一个函数构造函数，例如，继承自EventEmitter，并且它想在构造函数中调用一个事件：

```
const EventEmitter = require('events');
const util = require('util');

function MyEmitter() {
  EventEmitter.call(this);
  this.emit('event');
}
util.inherits(MyEmitter, EventEmitter);

const myEmitter = new MyEmitter();
myEmitter.on('event', function() {
  console.log('an event occurred!');
});
```

您不能立即从构造函数中发出事件，因为脚本不会处理到用户为该事件分配回调的点。 所以，在构造函数本身内，你可以使用process.nextTick()设置一个回调函数，在构造函数完成后发出事件，这提供了预期的结果：

```

const EventEmitter = require('events');
const util = require('util');

function MyEmitter() {
  EventEmitter.call(this);

  // use nextTick to emit the event once a handler is assigned
  process.nextTick(function () {
    this.emit('event');
  }.bind(this));
}
util.inherits(MyEmitter, EventEmitter);

const myEmitter = new MyEmitter();
myEmitter.on('event', function() {
  console.log('an event occurred!');
});

```

[原文](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)
