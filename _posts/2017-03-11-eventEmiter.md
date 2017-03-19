---
layout: blog
title: Node eventEmiter
type: Node
time: 2017-03-11

---

# Events

## 概述(基本用法)

Event Emitter 是一个接口，可以在任何对象上部署。这个接口由events模块提供。

```
var EventEmitter = require('events').EventEmitter;
var emitter = new EventEmitter();

```
events模块的EventEmitter是一个构造函数，可以用来生成事件发生器的实例emitter。
然后，事件发生器的实例方法on用来监听事件，实例方法emit用来发出事件。

```
emitter.on('someEvent', function (msg) {
  console.log('event has occured message: '+ msg);
});

function f() {
  console.log('start');
  emitter.emit('someEvent');
  console.log('end');
}

f()
// start
// event has occured message: someEvent
// end

```
上面代码还表明，EventEmitter对象的事件触发和监听是同步的，即只有事件的回调函数执行以后，函数f才会继续执行。

EventEmitter 提供的方法

- emiter.addListener （同 emiter.on）添加监听事件
- emiter.prependListener 给监听的事件callback栈开头添加callback函数
- emiter.once 添加只执行一次的事件，函数
- emiter.prependOnceListener 给监听的事件callback栈开头添加执行一次的callback函数
- emiter.removeListener 删除listener
- emiter.removeAllListeners
- emiter.listeners 获取指定事件名的所有listeners
- emiter.listenerCount 获取指定事件名的所有listeners总数
- emiter.eventNames 获取所有事件名

## [代码](https://github.com/nodejs/node/blob/master/lib/events.js)
## [精华cnode](https://cnodejs.org/topic/571e0c445a26c4a841ecbcf1)