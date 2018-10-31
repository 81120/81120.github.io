---
layout: post
title: Electron笔记
date: 2018-10-31 21:33:20 +0300
description: Electron 笔记. # Add post description (optional)
img: # Add image post (optional)
tags: [electron]
---

#### Electron简介
[Electron](https://electronjs.org/)是一个开源的跨平台桌面应用开发工具，它最大的特色是利用html, javascript, css就能开发跨平台的桌面应用。很多著名的应用例如VS Code就是利用Electron来开发的。

桌面应用避免不了要和本地OS进行交互，Electron就通过整合Chromium和NodeJS来为Elctron中运行的javascript提供一个扩展的执行环境，让javascript能够同时拥有浏览器中和NodeJS中的能力。借一张图来说明：
![undefined](https://cdn.nlark.com/lark/0/2018/png/7311/1540815664056-3a7b176d-5806-49d6-858f-43e2616aefe2.png) 

但是一个应用在Electron中运行的时候，会分别运行在一个主进程和若干个渲染进程中。在主进程中运行的javascript的上下文等同于NodeJS的上下文，而在渲染进程中运行的javascript则是运行在一个扩展了Chromium，加入了一些Electron扩展的上下文中，让渲染进程中的Javascript能够拥有超越浏览器的能力。各进程之间共享状态通过IPC实现：
![undefined](https://cdn.nlark.com/lark/0/2018/png/7311/1540815490873-a57280d3-4c34-4b98-b421-7412d90c492d.png) 

#### 一个hello world
先来看一个简单的例子：
首先创建一个javascript包，并安装依赖环境：
```
yarn init && yarn add electron && touch index.js
```
在`index.js`中，写下如下代码：
```javascript
const {app, BrowserWindow} = require('electron');

let mainWindow = null;

const createWindow = () => {
  mainWindow = new BrowserWindow({
    width: 1200,
    height: 720
  })

  mainWindow.loadURL('https://www.google.com')

  mainWindow.on('closed', () => {
    mainWindow = null
  })
}

app.on('ready', createWindow)
```
在目录下执行：
```
./node_modules/electron/cli.js index.js
```
就能看到执行的效果。
electron执行index.js的时候，就利用index.js创建了electron应用的主进程，然后利用`BrowserWindow`创建了一个浏览器环境，利用loadUrl来加载一个网页进行渲染。而网页中的javascrip就运行在一个渲染子进程中。

#### 进程间的通信
electron应用中，进程之间的通信一般按照和主进程通信，主进程转发消息的模式。那么就可以专注于渲染子进程和主进程之间的通信，存在以下三种方式：
##### ipcMain&ipcRenderer
这种模式下，主进程利用ipcMain，渲染子进程利用icpRenderer来进行通信，以一段代码为例：
在主进程中：
```javascript
const {ipcMain} = require('electron')
......
ipcMain.on('message-from-render', (event, args) => {
  .... // do something about args
  event.sender.send('message-from-main', '') // unicast
})
ipcMain.send('message-from-main', '') // broad cast
```
在渲染子进程中：
```javascript
const {ipcRenderer} = require('electron')
......
ipcRenderer.on('message-from-main', (event, args) => {
   ....
})
ipcRender.send('message-from-render', '')
```

##### remote
在渲染子进程中，可以引入remote模块来完成向主进程中通信的过程:
```javascript
const {remote} = require('electron')
const BrowserWindow = remote.BrowserWindow
const win = new BrowserWindow({ width: 800, height: 600 })
win.loadURL('https://www.google.com')
```
就可以在渲染子进程中创建一个新的渲染子进程，如果不使用这种模式，就需要利用ipcRenderer向主进程发送消息，然后在主进程中创建新的渲染进程。但是这种机制是单向的，只能从渲染子进程向主进程传递信息。
在remote上取到的每个对象本质上都是主进程中的一个对象，当尤其是函数对象，当调用函数对象时，本质上就是在向主进程发送ipc信息。

##### webContents.executeJavaScript
在主进程中，可以利用`webContents.executeJavaScript`在渲染子进程中执行一段javascript代码来改版子进程中的上下文信息：
```javascript
const {BrowserWindow} = require('electron')
const win = new BrowserWindow(options)
const _webContents= win.webContents
_webContents.executeJavaScript(code, callback)
```
code是一段javascript代码的文本，会在渲染子进程中执行，callback是可选的，会在代码执行完后通过回调函数将执行的结果返回给主进程。这种机制也是单向的，只能从主进程向渲染子进程传递信息。

#### 引入React
利用普通的javascript, css, html可以实现UI，但是怎么折腾一下能把React放进去呢，毕竟复杂的应用，只用原生的javascript的实现很吃力。理清了Electron应用工作的流程之后，整个过程也不麻烦。核就只需要在页面上引入将React和自己写的代码构建打包之后的结果，然后在渲染进程中渲染整个html页面即可。比较麻烦的是如何在开发时能够保存，构建，刷新直接看到效果，实现在浏览器上一样的开发体验。
为了方便，直接利用`create-react-app`来作为脚手架。
首先，创建应用：
```
create-react-app my-electron-app
```
完成后，react相关的依赖已经声明好，接下来引入electron相关的依赖：
```
cd my-electron-app
yarn add -D electron 
```
此时，缺少electron应用的主进程的入口，那么需要一个主进程的入口：
```
touch index.js
```
在index.js中，需要创建一个渲染进程，加载html页面渲染，在开发环境下，`create-react-app`会在3000端口启动一个本地的web服务器，那么可以在开发环境下加载这个页面进行渲染，最终打包构建时，引用构建之后`build/index.html`即可：
```javascript
const {app, BrowserWindow, ipcMain} = require('electron');
const path = require('path');
const url = require('url');

let mainWindow = null;

const createWindow = () => {
  mainWindow = new BrowserWindow({
    width: 1200,
    height: 720
  });
  const contentPath = process.env.DEV ? `http://localhost:3000` : url.format({
      pathname: path.join(__dirname, './../build/index.html')

  mainWindow.loadURL(
      contentPath,
      protocol: 'file:',
      slashes: false
    })
  );

  mainWindow.on('closed', () => {
    mainWindow = null;
  });
};

app.on('ready', createWindow);

app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') {
    app.quit();
  }
});

app.on('activate', () => {
  if (mainWindow) {
    return;
  }
  createWindow();
});
```
接下来让整个项目运行起来，在package.json中加入以下内容：
```json
"homepage": "./",
"main": "src/index.js"
```
在scripts下加入：
```
"electron": "./node_modules/electron/cli.js ."
```
这样当执行`yarn electron`时，就会执行到`electron .`，就会执行`electron index.js`，这样就启动了。可是在测试环境下，需要先启动react应用的测试服务，所以在测试时需要同时开两个窗口，分别执行`yarn start`和`yarn electron`

然后就可以愉快的玩耍了～

这里用来渲染网页的引擎是chromium，所以，对最新的html, javsctipt, css的特性的支持都很好，在构建React代码时其实大量的构建都不需要。

#### 结合MonacoEditor
在上面的基础上，开始做我们想做的事情了，例如引入[monaco-editor](https://microsoft.github.io/monaco-editor/)来做一个简单的编辑器，monaco-editor是基于web-worker的，在webpack体系构建时，需要引入一个plugin来支持。
```
yarn add monaco-editor
yarn add -D monaco-editor-webpack-plugin
```
然后在webpack的配置中加入以下配置：
```javascript
const MonacoWebpackPlugin = require('monaco-editor-webpack-plugin')
```
在plugins下加入：
```javascript
new MonacoWebpackPlugin()
```
`create-react-app`生成的项目中，webpack的配置被隐藏了，在应用根目录下执行一下`yarn eject`导出webpack配置，然后在`config/webpack.config.dev.js`中加入上面的内容即可。

#### 构建打包
最后一步就是将应用打包成原生系统的安装包，有`electron-packager`和`electron-builder`两个可选，后者比较方便，就选后者。
首先安装依赖：
```
yarn add -D electron-builder
```
然后在package.json中加入构建相关的配置：
```json
"build": {
    "appId": "com.alibaba.cbu.just-studio",
    "mac": {
      "target": ["dmg","zip"]
    }
}
```
在scripts中加入构建的命令：
```
"dist-mac": "electron-builder --mac"
```
然后，执行一下`yarn dist-mac`就能将应用打包

#### 总结&存在的问题
可以看出，electron可以理解为一个扩展的javasctipt, css, html执行环境，利用这个工具可以利用web的技术栈，开发出一个跨平台的桌面应用。如果代码架构分层合理的话，能够和web复用大量代码，是一个功能完全b/s结构之前的很好的选择。就上面的一些列操作之后，存在以下几个问题：
  * electron提供的环境对web的新特性支持很好，构建的内容可以针对这个特点做更多的优化，降低构建后内容的体积
  * electron打包出来的应用体积很大，一个简单的应用就有100+M，需要剔除不用的资源来降低应用的体积
  * 一个比较大的electron应用需要一个合理的分层，来屏蔽electron在渲染子进程中的扩展特性，保证代码能够尽量迁移到web
  * 还有更多一个桌面应用的特性还需要更深入的研究一下

#### 引用
 [electron](https://electronjs.org/)
