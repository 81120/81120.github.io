---
layout: post
title: [翻译]Javascript高阶技巧-memoization
date: 2018-10-29 13:53:20 +0300
description: 翻译. # Add post description (optional)
img: # Add image post (optional)
tags: [javascript, memoization]
---

#### 译者按
这篇[文章](http://www.fatalerrors.org/a/javascript-advanced-skill-memoization.html?utm_source=tuicool&utm_medium=referral)本身并不高深，逻辑也不复杂，但是着笔点很巧妙，集中于一个很常用的策略，然后举出了很多例子来佐证。其实这种策略非常通用，在`python`中结合`decorator`的[写法](https://docs.python.org/3/library/functools.html)有非常优雅的实现，在ESX中，也有了`decorator`的语法，结合这种语法，代码能写得更优雅。而且，近期facebook开源了[skip](http://skiplang.com/)，更是将这种策略扩散到一个更广的层面，细细体味，妙不可言啊。

memoization是取自拉丁语`memorandum`(备忘录)，区别于`memorization`(记忆)。

首先，我们来看维基百科上的介绍：
```
In computing, memoization or memoisation is an optimization technique used primarily to speed up computer programs by storing the results of expensive function calls and returning the cached result when the same inputs occur again.
```

简单来说，在计算机领域，`memoization`或者`memoisation`是一种常见的程序优化技巧。通过把一个需要消耗大量计算资源的计算结果缓存起来，当再次出现相同的输入时，便直接返回缓存的结果，以加速程序的执行。

在这篇文章中，我们受限介绍了一个是用memoization优化的例子，然后解读`underscore`中的memoization的代码，最后解读`reselect`中的memoization来加深理解。

#### 阶乘
##### 不使用memoization
我们可以不假思索的写出下面的代码：
```javascript
const factorial = n => {
    if (n === 1) {
        return 1
    } else {
        return factorial(n - 1) * n
    }
}
```

##### 使用memoization
```javascript
const cache = []
const factorial = n => {
    if (n === 1) {
        return 1
    } else if (cache[n - 1]) {
        return cache[n - 1]
    } else {
        let result = factorial(n - 1) * n
        cache[n - 1] = result
        return result
    }
}
```

##### 使用闭包和memoization
最常用的方式是结合闭包和`memoization`
```javascript
const factorialMemo = () => {
    const cache = []
    const factorial = n => {
        if (n === 1) {
            return 1
        } else if (cache[n - 1]) {
            console.log(`get factorial(${n}) from cache...`)
            return cache[n - 1]
        } else {
            let result = factorial(n - 1) * n
            cache[n - 1] = result
            return result
        }
    }
    return factorial
}
const factorial = factorialMemo()
```

最通用的写法一般是这样的：
```javascript
const factorialMemo = func => {
    const cache = []
    return function(n) {
        if (cache[n - 1]) {
            console.log(`get factorial(${n}) from cache...`)
            return cache[n - 1]
        } else {
            const result = func.apply(null, arguments)
            cache[n - 1] = result
            return result
        }
    }
}

const factorial = factorialMemo(function(n) {
    return n === 1 ? 1 : factorial(n - 1) * n
})
```

从上面计算阶乘的例子中可以看出，`memoization`是一个`利用空间换时间`的方案。通过将计算结果缓存，当再次出现相同的输入时，通过直接返回缓存的计算结果来加快计算速度。

#### underscore中的memoization实现
```javascript
// Memoize an expensive function by storing its results.
_.memoize = function(func, hasher) {
    var memoize = function(key) {
        var cache = memoize.cache
        var address = '' + (hasher ? hasher.apply(this, arguments) : key)
        if (!_.has(cache, address)) cache[address] = func.apply(this, arguments)
        return cache[address]
    };
    memoize.cache = {}
    return memoize
}
```

可以利用`_.memoize`来实现上面的阶乘：
```javascript
const factorial = _.memoize(function(n) {
    return n === 1 ? 1 : factorial(n - 1) * n
})
```

参考这个实现，阶乘的计算过程可以进一步被转换为如下的形式：
```javascript
const factorialMemo = func => {
    const memoize = function(n) {
        const cache = memoize.cache
        if (cache[n - 1]) {
            console.log(`get factorial(${n}) from cache...`)
            return cache[n - 1]
        } else {
            const result = func.apply(null, arguments)
            cache[n - 1] = result
            return result
        }
    }
    memoize.cache = []
    return memoize
}

const factorial = factorialMemo(function(n) {
    return n === 1 ? 1 : factorial(n - 1) * n
})
```

#### reselect中的memoization实现
```javascript
export function defaultMemoize(func, equalityCheck = defaultEqualityCheck) {
    let lastArgs = null
    let lastResult = null
    // we reference arguments instead of spreading them for performance reasons
    return function () {
        if (!areArgumentsShallowlyEqual(equalityCheck, lastArgs, arguments)) {
            // apply arguments instead of spreading for performance.
            lastResult = func.apply(null, arguments)
        }

        lastArgs = arguments
        return lastResult
    }
}
```

从这段代码可以看出，当`lastArgs`和`arguments`相同时，`func`将不会执行。

#### 总结
`memoization`是一种优化技巧，它能够避免不必要的重复计算从而提高计算速度。

#### 参考
* [Memoization wiki](https://en.wikipedia.org/wiki/Memoization)
* [Understanding JavaScript Memoization In 3 Minutes](https://codeburst.io/understanding-memoization-in-3-minutes-2e58daf33a19)
* [Underscore](https://github.com/jashkenas/underscore)
* [reselect](https://github.com/reduxjs/reselect)
* [Implementing Memoization in JavaScript](https://www.sitepoint.com/implementing-memoization-in-javascript/)
