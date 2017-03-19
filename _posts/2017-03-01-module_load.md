---
layout: blog
title: Node How Module load
type: Node
time: 2017-03-01

---

# MODULE  [代码](https://github.com/nodejs/node/blob/master/lib/module.js)

## 简述

Node应用由模块组成，采用CommonJS模块规范。每个文件就是一个模块，有自己的作用域。在一个文件里面定义的变量、函数、类，都是私有的，对其他文件不可见。

```
// example.js
var x = 5;
var addX = function (value) {
  return value + x;
};

```

上面代码中，变量x和函数addX，是当前文件example.js私有的，其他文件不可见。

## exports vs module.exports
CommonJS规范规定，每个模块内部，module变量代表当前模块。
这个变量是一个对象，它的`exports`属性（即`module.exports`）是对外的接口。
加载某个模块，其实是加载该模块的`module.exports`属性。

### 初始情况下 exports 与 module.exports 是指向相同一块内存地址。最终还是加载该模块的`module.exports`属性

```
//a.js

exports.a = 1

//b.js

var a = require('./a')
console.log(a)

```
输出： `{a:1}`

```
//a.js

exports = {a:1}

//b.js

var a = require('./a')
console.log(a)

```
输出： `{}`

```
//a.js

module.exports = {a:1}

//b.js

var a = require('./a')
console.log(a)

```
输出： `{a:1}`


## module对象
Node内部提供一个Module构建函数。所有模块都是Module的实例

```
function Module(id, parent) {
  this.id = id;
  this.exports = {};
  this.parent = parent;
  // ...
```
每个模块内部，都有一个module对象，代表当前模块。它有以下属性。

- module.id 模块的识别符，通常是带有绝对路径的模块文件名。
- module.filename 模块的文件名，带有绝对路径。
- module.loaded 返回一个布尔值，表示模块是否已经完成加载。
- module.parent 返回一个对象，表示调用该模块的模块。
- module.children 返回一个数组，表示该模块要用到的其他模块。
- module.exports 表示模块对外输出的值。

```
//a.js
console.log('module.id:', module.id)
console.log('module.filename:',module.filename)
console.log('module.loaded:',module.loaded)
console.log('module.parent:',module.parent)
console.log('module.children:',module.children)

exports = module.exports = {a:1}

//a.js end

执行 node a

输出： 

module.id: .
module.filename: /Users/mario/nodejs_module/a.js
module.loaded: false
module.parent: null
module.children: []

```
=====================我是分割线=============================

```
//a.js
console.log('module.id:', module.id)
console.log('module.filename:',module.filename)
console.log('module.loaded:',module.loaded)
console.log('module.parent:',module.parent)
console.log('module.children:',module.children)

exports = module.exports = {a:1}

//a.js end

//b.js

var ajs = require('./a')
console.log(ajs)

//b.js end

执行 node b

输出： 

module.id: /Users/mario/nodejs_module/a.js
module.filename: /Users/mario/nodejs_module/a.js
module.loaded: false
module.parent: Module {
  id: '.',
  exports: {},
  parent: null,
  filename: '/Users/mario/nodejs_module/b.js',
  loaded: false,
  children:
   [ Module {
       id: '/Users/mario/nodejs_module/a.js',
       exports: {},
       parent: [Circular],
       filename: '/Users/mario/nodejs_module/a.js',
       loaded: false,
       children: [],
       paths: [Object] } ],
  paths:
   [ '/Users/mario/nodejs_module/node_modules',
     '/Users/mario/node_modules',
     '/Users/node_modules',
     '/node_modules' ] }
module.children: []
{ a: 1 }

```
=====================我是分割线=============================

## require命令
### 基本用法
require命令的基本功能是，读入并执行一个JavaScript文件，然后返回该模块的exports对象。如果没有发现指定模块，会报错

```
 //a.js

  var invisible = function () {
    console.log("invisible");
  }

  exports.message = "hi";

  exports.say = function () {
    console.log(message);
  }

//b.js 

var example = require('./example.js');
console.log(example)
// {
//   message: "hi",
//   say: [Function]
// }
```

### 加载规则
require命令用于加载文件，后缀名默认为.js。
根据参数的不同格式，require命令去不同路径寻找模块文件。
- 如果参数字符串以“/”开头，则表示加载的是一个位于绝对路径的模块文件。比如，require('/home/mario/a.js')将加载/home/mario/a.js。

- 如果参数字符串以“./”开头，则表示加载的是一个位于相对路径（跟当前执行脚本的位置相比）的模块文件。比如，require('./a')a.js。

- 如果参数字符串不以“./“或”/“开头，则表示加载的是一个默认提供的核心模块（位于Node的系统安装目录中），或者一个位于各级node_modules目录的已安装模块（全局安装或局部安装）。

```
  举例来说，脚本/home/user/projects/b.js执行了require('a.js')命令，Node会依次搜索以下文件。
  * /usr/local/lib/node/a.js
  * /home/user/projects/node_modules/a.js
  * /home/user/node_modules/a.js
  * /home/node_modules/a.js
  * /node_modules/a.js
```
- 如果参数字符串不以“./“或”/“开头，而且是一个路径，比如require('example-module/path/to/file')，则将先找到example-module的位置，然后再以它为参数，找到后续路径。

- 如果指定的模块文件没有发现，Node会尝试为文件名添加.js、.json、.node后，再去搜索。.js件会以文本格式的JavaScript脚本文件解析，.json文件会以JSON格式的文本文件解析，.node文件会以编译后的二进制文件解析。

### 模块的缓存
第一次加载某个模块时，Node会缓存该模块。以后再加载该模块，就直接从缓存取出该模块的module.exports属性。

如果想要多次执行某个模块，可以让该模块输出一个函数，然后每次require这个模块的时候，重新执行一下输出的函数。

```
//a.js

module.exports = function(){
  return {
    a: 1;
    b: function(){}
  }

}

//b.js
var a = require('./a')()

//c.js
var a = require('./a')()

```

所有缓存的模块保存在require.cache之中，如果想删除模块的缓存，可以像下面这样写。

```
// 删除指定模块的缓存
delete require.cache[moduleName];

// 删除所有模块的缓存
Object.keys(require.cache).forEach(function(key) {
  delete require.cache[key];
})

```

## 模块的加载机制
require命令是CommonJS规范之中，用来加载其他模块的命令。它其实不是一个全局命令，而是指向当前模块的module.require命令，而后者又调用Node的内部命令Module._load。

```
Module._load = function(request, parent, isMain) {
  // 1. 检查 Module._cache，是否缓存之中有指定模块
  // 2. 检查是否是node native module，若果是调用NativeModule.require()
  // 3. 如果缓存之中没有，就创建一个新的Module实例
  // 4. 将它保存到缓存
  // 5. 使用 module.load() 加载指定的模块文件，
  //    读取文件内容之后，使用 module._compile() 执行文件代码
  // 6. 如果加载/解析过程报错，就从缓存删除该模块
  // 7. 返回该模块的 module.exports
};

```

上面的第4步，采用module.compile()执行指定模块的脚本，逻辑如下。

```
Module.prototype._compile = function(content, filename) {
  // 1. 生成 wrapper , 包装闭包函数
  // 2. 生成一个compiledWrapper函数，##
    var compiledWrapper = vm.runInThisContext(wrapper, {
      filename: filename,
      lineOffset: 0,
      displayErrors: true
  });
  // 3. 加载其他辅助方法到require， 生成 args
  // 4. 将文件内容放到一个函数之中，该函数可调用 require
  // 5. 执行该函数
};

```
上面的第1步和第2步，require函数及其辅助方法主要如下。

- require(): 加载外部模块
- require.resolve()：将模块名解析到一个绝对路径
- require.main：指向主模块
- require.cache：指向所有缓存的模块
- require.extensions：根据文件的后缀名，调用不同的执行函数

一旦require函数准备完毕，整个所要加载的脚本内容，就被放到一个新的函数之中，这样可以避免污染全局环境。该函数的参数包括require、module、exports，以及其他一些参数。

```
(function (exports, require, module, __filename, __dirname) {
  // YOUR CODE INJECTED HERE!
});

```