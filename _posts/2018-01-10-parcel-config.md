---
layout: post
title: Parcel的配置入门
date: 2018-01-10 01:32:20 +0300
description: Parcel的简单配置. # Add post description (optional)
img: i-rest.jpg # Add image post (optional)
tags: [parcel]
---
[parcel](https://parceljs.org/)是一个打包工具，它最大的卖点就是0配置，相比于`webpack.config.js`里复杂的配置，0配置简直就是福音。尝试了一下，发现确实很方便，就记一笔流水账。

首先安装：
```shell
  yarn global add parcel-bundler
```
执行完之后，测试一下：`parcel -h`，如果能看到正常的输出提示，就表示安装成功。

安装成功，下一步。

创建应用：
```shell
  mkdir test
  cd test
  yarn init
```
这是为了后续的更复杂的包的编译准备的，理论上不执行`yarn init`也不影响。

在目录下创建两个文件，`index.html,index.js`，在`'index.html`中通过script标签引入`index.js`就可以了，在`index.js`中写一段简单的测试代码`console.log('hello world')`。做完这些之后，执行`parcel index.html`，就可以在`http://localhost:1234`打开这个`html`页面，在`console`下也能看到`hello world`被输出了。整个过程中，没有做任何配置。

那么接下来，为了支持`react`，首先安装`react`的依赖环境。
```shell
  yarn add react 
  yarn add react-dom
  yarn add prop-types
  yarn add babel-polyfill
```

为了支持`es6`，`es7`，`jsx`的编译，需要配置`babel`的编译规则，在目录下创建一个`.babelrc`文件，然后写入相应的配置：
```javascript
  {
    "presets": ["env", "react", "stage-0"]
  }
```

然后安装相应的依赖包：
```
  yarn add babel-preset-env --dev
  yarn add babel-preset-react --dev
  yarn add babel-preset-stage-0 --dev
```

到此为止，配置已经能够满足开发的需求了。

写一个简单的`demo`来验证一下
```html
   <!DOCTYPE html>
   <html lang="en">
   <head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
   </head>
   <body>
     <div id='target'></div>
     <script src="./home.js"></script>
   </body>
  </html>
```

```javascript
  import React from 'react'
  import ReactDOM from 'react-dom'
  const Test = (props) => <div>{props.text}</div>
  const target = document.getElementById('target')
  ReactDOM.render(<Test text={"hello world"} />, target)
```

然后执行`parcel index.html`，打开`http://localhost:1234`就能看到熟悉的`hello world`。

至此，一个功能完整的`react`开发环境就搭建好了，整个过程顺滑流畅，一气呵成，不禁想起了被`webpack`支配的恐惧。从`github`上`star`数量增长的速度也能看出大家对这样的工具已经期待很久了。后面小的项目，感觉这个工具都可以hold住了。