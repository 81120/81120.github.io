---
layout: post
title: 一个简单的模块加载器
date: 2018-01-10 01:39:20 +0300
description: 一个简单的CMD木块加载器. # Add post description (optional)
img: # Add image post (optional)
tags: [cmd, loader]
---

##### 基本信息
CMD风格的模块加载器是解决`javascript`模块化加载的一个方案，典型代表是[sea.js](http://www.zhangxinxu.com/sp/seajs/)，它的[规范](https://github.com/seajs/seajs/issues/242)也很简洁。在研究原理的时候，可以自己动手写一个简单的`loader`。为了方便，就叫它`testLoader`。 

##### 基本实现
`testLoader`有这样的功能：
  1. 通过`testLoader.config({...})`来配置一些全局的信息
  2. 通过`testLoader.define(function(){})`来定义模块，但是不指定模块的`id`，用模块所在文件的路径来作为模块的`id`
  3. 在模块内部通过`require`来加载别的模块
  4. 在模块内部通过`exports`和`module.exports`来对外暴露接口
  5. 通过`testLoader.bootstrap(id, callback)`来作为入口启动

首先，定义`testLoader`的基本信息：
```javascript
window.testLoader = {
  define: define,
  bootstrap: bootstrap,
  require: require,
  modules: {},
  config: {
    root: ''
  },
  config: function(obj) {
    this.config = {...this.config, ...obj}
  },
  MODULE_STATUS: {
    PENDING: 0,
    LOADING: 1,
    COMPLETED: 2,
    ERROR: 3
  }
}
```
从`testLoader.bootstrap`开始看。`testLoader.bootstrap(id, callback)`在执行时，首先是根据`id`来加载模块，加载模块完成后，将模块暴露出的对象作为参数，执行`callback`。在加载模块的时候，首先是从`testLoader`的模块缓存中查找有没有相应的模块，如果有，就直接取，否则，就创建一个新的模块，并将这个模块缓存。代码如下：
```javascript
const generatePath = (id) => `${testLoader.config.root}${id}`

const load = (id) => new Promise((resolve, reject) => {
  let mod = testLoader.modules[id] || (new Module(id))
  mod.on('complete', () => {
    let exp = getModuleExports(mod)
    resolve(exp)
  })
  mod.on('error', reject)
})

const bootstrap = (ids, callback) => {
  ids = Array.isArray(ids) ? ids : [ids]
  Promise.all(ids.map((id) => load(generatePath(id))))
  .then((list) => {
    callback.apply(window, list)
  }).catch((error) => {
    throw error
  })
}
```

`getModuleExports`时是用于获取模块暴露出的接口，实现如下：
```javascript
const getModuleExports = (mod) => {
  if (!mod.exports) {
    mod.exports = {}
    mod.factory(testLoader.require, mod.exports, mod)
  }
  return mod.exports
}
```
当模块的`exports`属性为空的时候，执行`mod.factory(testLoader.require, mod.exports, mod)`，因为传入的`mod.exports`是一个引用类型，在`factory`执行的过程中会因为副作用，为`mod.exports`提供值。

而`Module`则是一个用来生成模块对象的`Class`，定义如下：
```javascript

class Module {
  constructor(id) {
    this.id = id 
    testLoader.modules[id] = this
    this.status = testLoader.MODULE_STATUS.PENDING
    this.factory = null
    this.dependences = null
    this.callbacks = {}
    this.load()
  }

  load() {
    let id = this.id
    let script = document.createElement('script')
    script.src = id
    script.onerror = (event) => {
      this.setStatus(testLoader.MODULE_STATUS.ERROR, {
        id: id,
        error: new Error('module can not load')
      })
    }
    document.head.appendChild(script)
    this.setStatus(testLoader.MODULE_STATUS.LOADING)
  }

  on(event, callback) {
    (this.callbacks[event] || (this.callbacks[event] = [])).push(callback)
    if (
      (this.status === testLoader.MODULE_STATUS.LOADING && event === 'load') || 
      (this.status === testLoader.MODULE_STATUS.COMPLETED && event === 'complete')
    ) {
      callback(this)
    }

    if (this.status === testLoader.MODULE_STATUS.ERROR && event === 'error') {
      callback(this, this.error)
    }
  }

  emit(event, arg) {
    (this.callbacks[event] || []).forEach((callback) => {
      callback(arg || this)
    })
  }

  setStatus(status, info) {
    if (this.status === status) return
    if (status === testLoader.MODULE_STATUS.LOADING) {
      this.emit('load')
    }
    else if (status === testLoader.MODULE_STATUS.COMPLETED) {
      this.emit('complete')
    }
    else if (status === testLoader.MODULE_STATUS.ERROR) {
      this.emit('error', info)
    }
    else return
  }
}
```
在创建一个模块对象的时候，首先是给模块赋予一些基本的信息，然后通过`script`标签来加载模块的内容。这个模块对象只是提供了一个模块的基本的属性和简单的事件通信机制，但是模块的内容，模块的依赖这些信息，需要通过`define`来提供。`define`为开发者提供了定义模块的能力，`Module`则是提供了`testLoader`描述表示模块的方式。

通过`define`定义模块，在`define`执行的时候，首先需要为模块定义一个`id`，这个`id`是模块在`testLoader`中的唯一标识。在前面已经说明了，在`testLoader`中，不能指定`id`，只是通过路径来生成`id`，那么通过获取当前正在运行的`script`代码的路径来生成`id`。获取到`id`之后，从`testLoader`的缓存中取出对应的模块表示，然后解析模块的依赖。由于`define`的时候，不能指定`id`和依赖，对依赖的解析是通过匹配关键字`require`来实现的，通过解析`require('x')`获取所有的依赖模块的`id`，然后加载所有依赖。就完成了模块的定义，代码如下：
```javascript
const getCurrentScript = () => document.currentScript.src

const getDependence = (factoryString) => {
  let list = factoryString.match(/require\(.+?\)/g) || []
  return list.map((dep) => dep.replace(/(^require\(['"])|(['"]\)$)/g, ''))
}

const define = (factory) => {
  let id = getCurrentScript().replace(location.origin, '')
  let mod = testLoader.modules[id]
  mod.dependences = getDependence(factory.toString())
  mod.factory = factory
  if(mod.dependences.length === 0) {
    mod.setStatus(testLoader.MODULE_STATUS.COMPLETED)
    return
  }
  Promise.all(mod.dependences.map((id) => new Promise((resolve, reject) => {
      id = generatePath(id) 
      let depModule = testLoader.modules[id] || (new Module(id))
      depModule.on('complete', resolve)
      depModule.on('error', reject)
    })
  )).then((list) => {
    mod.setStatus(testLoader.MODULE_STATUS.COMPLETED)
  }).catch((error) => {
    mod.setStatus(testLoader.MODULE_STATUS.ERROR, error)
  })
}
```
那么依赖别的模块是通过`require`来实现的，它核心的功能是获取一个模块暴露出来的接口，代码如下：
```javascript
const require = (id) => {
  id = generatePath(id)
  let mod = testLoader.modules[id]
  if (mod) {
    return getModuleExports(mod)
  }
  else {
    throw 'can not get module by id: ' + id
  }
}
```
从上面解析依赖的方式可以看出，在通过`define`定义模块的时候，匿名函数有三个参数
```javascript
testLoader.define(function(requrie, exports, module){})
```
`exports`本质上是`module.exports`的引用，所以通过`exports.a=x`是可以暴露接口的，但是`exports={a:x}`则不行，因为后一种方式本质上是改变了将`exports`作为一个值类型的参数，修改它的值，这种操作，在函数调用结束后，是不会生效的。按照这种原理，`module.exports={a:x}`是可以达到效果的。

##### 测试例子
  1. `index.js`
      ```javascript
      testLoader.define(function(require, exports, module) {
        var a = require('a.js')
        var b = require('b.js')
        a(b)
        module.exports = {
          a: a,
          b: b
        }
      })
      ```
  2. `a.js`
      ```javascript
      testLoader.define(function(requrie, exports, module) {
        module.exports = function(msg) {
          console.log('in the a.js')
          document.body.innerHTML = msg
        }
      })
      ```
  3. `b.js`
      ```javascript
      testLoader.define(function(require, exports, module) {
        console.log('in the b.js')
        module.exports = 'Wonderful Tonight'
      })
      ```
  4. `index.html`
      ```html
      <html lang="en">
        <head>
          <title></title>
          <meta charset="UTF-8">
          <meta name="viewport" content="width=device-width, initial-scale=1">
          <script src='./test-cmd-loader.js'></script>
          <script>
            testLoader.config({
              root: '/Users/guangyi/code/javascript/sass/lib/test/'
            })
            testLoader.bootstrap('index.js', (list) => {
              console.log(list)
            })
          </script>
        </head>
        <body></body>
      </html>
      ```
用浏览器打开，在调试窗口能看到打印的`log`，页面上也渲染出了`Wonderful Tonight`。<html lang="en">
  <head>
    <title></title>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <script src='./mini-cmd-loader.js'></script>
    <script>
      testLoader.config({
        root: '/Users/guangyi/code/javascript/sass/lib/test/'
      })
      testLoader.bootstrap('index.js', (list) => {
        console.log(list)
      })
    </script>
  </head>
  <body></body>
</html>

##### 总结
通过这个简单的`loader`，可以了解`CMD`的规范，以及`CMD`规范的`loader`工作的基本流程。但是，和专业的`loader`相比，还有很多没有考虑到，比如`define`的时候，支持指定模块的`id`和依赖，不过在上面的基础上，也很容易实现，在生成`id`的时候将自动生成的`id`作为默认值，在决定依赖的时候，将参数中定义的依赖和解析生成的依赖执行一次`merge`处理即可。但是，这些能力本质上还是一样的，因为这种机制定义的依赖是静态依赖，在这个模块的内容执行之前，依赖的模块已经被加载了，所以类似
```javascript
if (condition) {
  require('./a.js')
}
else {
  require('./b.js)
}
```
这种加载依赖的方式是不生效的，不论`condition`的值是什么，两个模块都会被加载。要实现动态加载，或者说运行时加载，一个可行的方案是在上面的基础上，提供一个新的声明依赖的关键字，然后这个关键字代表的函数，在执行的时候再创建模块，加载代码。

还有，当模块之间存在循环依赖的情况，还没有处理。理论上，通过分析模块之间的静态依赖关系，就可以发现循环依赖的情况。也可以在运行的时候，根据模块的状态决定模块是否返回空来结束循环依赖。

还有跨语言的支持也是一个很有意思的问题。