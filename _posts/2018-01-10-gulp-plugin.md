---
layout: post
title: 如何开发一个gulp插件
date: 2018-01-10 14:33:20 +0300
description: 开发一个gulp插件. # Add post description (optional)
img: # Add image post (optional)
tags: [gulp, plugin]
---
gulp的api简单灵活，利用这几个api就可以实现绝大多数前端资源的处理，`gulp.src`，`gulp.pipe`，`gulp.dest`提供了对资源流式处理的框架级别的支持，但是在每个任务节点中，是具体的插件来实现对流的处理和转化。那么，如何开发一个gulp插件呢？

下面开发一个简单的插件，功能是在文件的开头，加上作者和时间戳。

首先引入`through2`，这个库对stream进行了包装，提供了更方便的使用方式。为了方便起见，也引入gulp-util库。
```javascript
const through = require('through2')
const gulpUtil = require('gulp-util')
```

然后定义插件
```javascript
const addAuthor = (options) => through.obj(function(file, encode, callback) {
   if (file.isNull()) {
      this.push(file)
      return callback()
   }
   if (file.isStream()) {
      this.emit('error', new gulpUtil.PluginError(PLUGIN_NAME, 'stream is not supported'))
      this.push(file)
      return callback()
   }
   const fileContent = file.contents.toString()
   const {author='leo'} = options
   const updatedFileContent = `
    /*
    * by ${author}
    * at ${date}
    * /
    ${fileContent}`
   file.contents = new Buffer(updatedFileContent)
   this.push(file)
   callback()
})
```

其中，第一步判断文件是否为空和第二步判断file对象是否为stream，这个是模式化的程序。对文件的操作在第三部分，首先取出文件的内容，然后转化成字符串，在对字符串进行一系列操作后，将最终得到的字符串转换成buffer，赋值给file对象。交给下一个插件来处理。其中callback就是代表了下一个插件。如果把这个过程抽象成一个`transformString`的过程，那么，处理的流程就是
```javscript
const fileContent = file.contents.toString()
const updatedFileContent = transformString(fileContent)
file.contents = new Buffer(updatedFileContent)
```

可以在`transformString`中做很多事，比如格式化，编译，压缩等等，例如调用babel来编译es6语法，调用lint来格式化。


最后，对外暴露接口即可
```
module.exports = {addAuthor}
```

测试一下，在gulpfile.js中，引入插件
```javascript
const gulp = require('gulp')
const {addAuthor} = require('./gulp-add-author-plugin.js')
gulp.task('test', () => {
  gulp.src([''])
      .pipe(addAuthor({author: 'author'}))
      .pipe(gulp.dest('dist'))
})
```