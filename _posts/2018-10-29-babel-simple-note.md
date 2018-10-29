---
layout: post
title: babel基础笔记
date: 2018-10-29 13:53:20 +0300
description: babel基础笔记. # Add post description (optional)
img: # Add image post (optional)
tags: [babel, javascript]
---

##### 简介
babel是一个javascript的代码转换工具，在前端开发中用的尤其多。它最初出现，就是为了将es6的语法转换成es5语法从而在让开发者能使用es6的特性来开发。随着react的出现，jsx作为一种语法扩展被引入到了javascript中。自此开始，babel开始逐渐脱离仅仅作为高版本javascript语法转换工具的定位，转变成为一种基于javascript的DSL构建工具。例如flow提供的类型系统也可以通过[preset-flow](https://babeljs.io/docs/plugins/preset-flow/)纳入到babel的生态中。

##### 处理过程
借用一张图来说明babel处理代码的过程：

![](https://img.alicdn.com/tps/TB1nP2ONpXXXXb_XpXXXXXXXXXX-1958-812.png)

以babel7为例，首先利用[@babel/parser](https://github.com/babel/babel/tree/v7.0.0-beta.49/packages/babel-parser)将代码进行词法分析和语法分析形成AST；然后通过[@babel/traverse](https://github.com/babel/babel/tree/v7.0.0-beta.49/packages/babel-traverse)来遍历AST，在这个过程中调用配置的plugin来对原始的AST进行操作，生成新的AST；在新的AST的基础上，利用[@babel/generator](https://github.com/babel/babel/tree/v7.0.0-beta.49/packages/babel-generator)来生成转换后的代码。

为了方便生成AST节点和判断节点的类型，社区有[@babel/types](https://github.com/babel/babel/tree/v7.0.0-beta.49/packages/babel-types)，类似的为了大量产生一批AST节点[@babel/template](https://github.com/babel/babel/tree/v7.0.0-beta.49/packages/babel-template)。在整个过程中，babel采用的AST标准为[AST-SPEC](https://github.com/babel/babel/blob/v7.0.0-beta.49/packages/babel-parser/ast/spec.md)。在测试的过程中可以使用[http://astexplorer.net/](http://astexplorer.net/)来观察生成的AST。

从上面的图可以看出，babel定义了整个代码转换的流程，而且在各个阶段暴露出了相应的标准。在这种设计下，代码转换逻辑可以单独包装成babel-plugin的形式。如果在遍历原始AST的过程中没有调用任何插件，那么babel只会将代码原样返回。babel本身除了[@babel/core](https://github.com/babel/babel/tree/v7.0.0-beta.49/packages/babel-core)和一些其它工具包之外，还维护了[大量plugin](https://github.com/babel/babel/tree/v7.0.0-beta.49/packages)。

![](https://img.alicdn.com/tfs/TB1KmUOwTtYBeNjy1XdXXXXyVXa-2098-966.png)
那么为了做一些好玩的事情，我们需要了解一下如何开发一个babel的pluin。

##### 开发一个简单的babel-plugin
借用babel官网上的例子，实现一个plugin，功能是将变量的名称反转，例如从`XY`变成`YX`。那么首先来看一下，`const hello = 2`对应的AST：

![](https://img.alicdn.com/tfs/TB1QuKZwKuSBuNjy1XcXXcYjFXa-2050-874.png)

要变成的目标`const olleh = 2`对应的AST为：

![](https://img.alicdn.com/tfs/TB1OcoQwTtYBeNjy1XdXXXXyVXa-2066-866.png)

对比发现，只需要将前面的`Identifier`节点的`name`属性的值反转即可，直接看demo:

```javascript
module.exports = () => {
  return {
    visitor: {
      Identifier: (path) => {
        const name = path.node.name
        path.node.name = name.split('').reverse().join('')
      }
    }
  }
}
```

一个babel-plugin典型的结构就是这样，一个函数，返回一个对象，对象里面可以是`visitor`也可以是`enter，exit`，在遍历AST的时候，会在进入节点时首先调用`enter`，在退出时调用`exit`，当只定义了`visitor`时，就相当于只定义了`enter`。`visitor`中的`Identifier`是AST节点中的一类节点，代表当遍历到这种类型的节点时，才会执行定义的逻辑。例子中，`path.node.name = name.split('').reverse().join('')`这一句就完成了将变量名称反转的操作；值得记住的是，在babel-plugin的开发中，对AST的操作都是通过这种副作用的方式实现的。

##### 如何给plugin传入参数
有了这个例子，那么常用的babel的配置中，一般会给babel-plugin传递一些参数，这个又是怎么实现的呢？其实在定义plugin的时候，主体是一个函数，这个函数在执行的过程中，会被传入一些实参，其中，第一个就是包含了babel各种信息的一个context对象，第二个就是配置事传入的参数，那么我们可以在上面的例子上简单改一下，实现一个开关来决定要不要将变量的名称反转，直接看demo:

```javascript
module.exports = (babel, options) => {
  const {
    reverse = true,
    exclude = []
  } = options
  return {
    visitor: {
      Identifier: (path) => {
        const name = path.node.name
        if (reverse && !exclude.includes(name)) {
          path.node.name = name.split('').reverse().join('')
        }
      }
    }
  }
}
```

##### 如何操作AST节点
上面的对AST的操作仅限于在AST节点内部操作一个属性，如果需要改变程序的结构，那么就涉及到以AST节点为单位的操作了，例如替换掉一个AST节点，创建一个AST节点等。利用`@babel/types`中提供的能力我们可以轻松的定义不同的AST节点，然后AST节点本身自带的能力能够支持我们操作AST的位置。下面我们来做一个简单的例子，将程序中的数值类型的值变成对应的字符串类型。依然以前面的例子来说明，`const hello = 2`对应的AST：

![](https://img.alicdn.com/tfs/TB1QuKZwKuSBuNjy1XcXXcYjFXa-2050-874.png)

而`const x = '2'`对应的AST为：

![](https://img.alicdn.com/tfs/TB1R2i4wKuSBuNjy1XcXXcYjFXa-2116-870.png)

可以看出，两者对应的节点的类型已经不同了，那么就需要将原始的AST节点替换掉，看demo:

```javascript
module.exports = (babel, options) => {
  const {
    types
  } = babel
  return {
    visitor: {
      NumericLiteral: (path) => {
        const {
          value
        } = path.node
        path.replaceWith(types.stringLiteral(value.toString()))
      }
    }
  }
}
```

前面提过，第一个参数是babel的整个context，包含了`@babel/types`，用它提供的能力可以创建AST节点，参考[@babel/types doc](https://babeljs.io/docs/core-packages/babel-types/)。而对AST节点的操作，可以参考[manipulation doc](https://github.com/acdlite/babel-plugin-handbook/blob/master/README.md#manipulation)

##### 一个更复杂一点的例子
有了前面的几个例子，可以来展望一下一个复杂一点的例子。例如将`x = 2`这种赋值行为变成一个函数调用`__assign('x', 2)`这种形式。老套路，首先比对一下两者的AST差异：
首先是`x = 2`的AST：

![](https://img.alicdn.com/tfs/TB1_RAYwTtYBeNjy1XdXXXXyVXa-2084-786.png)

再看`__assign('x', 2)`的AST:

![](https://img.alicdn.com/tfs/TB1AG.ZwTtYBeNjy1XdXXXXyVXa-2106-1168.png)

然后修改AST，将`AssignmentExpression`替换成`CallExpression`，这里要替换的是一棵子树：

```javascript
module.exports = (babel, options) => {
  const {
    types
  } = babel
  return {
    visitor: {
      AssignmentExpression: (path) => {
        const {
          left,
          right: valueToAssign
        } = path.node
        const {
          name: variableNameToAssign
        } = left
        const assignFunctionName = types.identifier('__assign')
        const paramName = types.stringLiteral(variableNameToAssign)
        const assignFunctionCall = types.callExpression(assignFunctionName, [paramName, valueToAssign])
        path.replaceWith(assignFunctionCall)
      }
    }
  }
}
```

搞定。
