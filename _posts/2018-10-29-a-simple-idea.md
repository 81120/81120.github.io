---
layout: post
title: JavaScript中的数据生产者
date: 2018-10-29 13:53:20 +0300
description: JavaScript中的数据生产者. # Add post description (optional)
img: # Add image post (optional)
tags: [javascript]
---

最近数据流被提及得很频繁，听了很多，冒出了一个想法来理解JavaScipt中的数据流，写个笔记记录一下。数据从一个节点流动到另一个节点的过程，也许可以看成是数据从一个节点被生产然后在另一个节点被消费的过程。在JavaScript中，我可以想到的，有四种机制，分别是：Function，Generator，Promise，Observable，下面分别扒一扒。

##### Function
函数是一种可以被调用的特殊对象，在计算的过程中，调用函数是为了获取函数的返回值，从这种意义上来讲，函数是数据的生成者，调用函数的过程是消费数据的过程。如下所示，`f`是生产者，`r`是消费者，`call()`这个方法则是生产者提供的消费数据的机制。

```
const f = () =>  2
let r = f.call()
```
这种数据消费形式有两个特点，首先，数据生产的时机是被消费者决定的，只有当消费者主动调用`call()`的时候，才会触发数据生产的过程。其次，整个生产者只能产生一次数据，假如一个函数定义如下：

```
const f = () => {
  return 1
  return 2
}
f.call()
f.call()
```
两次`f.call()`产生的结果都是1，无论`f`被调用多少次，产生的结果都是1。

##### Generator
Generator在函数的基础上提供了多次返回的能力。和上面类似的例子：

```
const f = function*(){
  yield 1
  yield 2
  yield 3
}
g = f()
g.next()
g.next()
g.next()
```
上面的调用过程会依次返回1，2，3。在这里，生成器是生产者，它可以多次产出数据，但是消费者只有主动调用`next`才能获取到它新产生的数据。从这个意义来讲，函数和生成器都是被动的生产者，它们生产数据的时机都是被消费者决定的，区别在于函数只能生产一次，生成器则可以生产多次。
##### Promise
Promise是主动的生产者。先看一个例子:

```
const p = new Promise((resolve, reject) => {
  resolve(1)
})
p.then((data) => {
  console.log(data)
})
```
在这段程序中，`p`是一个Promise，通过`resolve`来生产数据，`p.then`则是数据的消费过程，与函数和`generator`不同的是，在Promise中，消费数据的时机是由生产者决定的，消费者只需要定义当数据流过来的时候消费的方式就可以了（`then`中做的事情）。而数据何时生产，何时流动过来，都是生产者决定的，消费者端没有任何机制能够主动拉取生产的数据，对比函数中的`call`和generator中的`next`，就更明显了。也就是说，Promise作为数据的生产者，它提供的能力是将数据主动推送到消费者，而消费者不能拉取生成的数据，只需要定义好消费数据的规则即可，所以说Promise是主动的生产者。但是，Promise只能提供一次数据，不能多次返回数据。将上面的例子改成：

```
const p = new Promise((resolve, reject) => {
  resolve(1)
  resolve(2)
})
p.then((d) => {
  console.log(d)
})
.then((d) => {
  console.log(d)
})
```
这样也只会第一次返回1第二次返回undefined，不会在第二次返回2。

##### Observable
Observable是一个待定的特性，但是可以通过`Rx`来使用，作为数据的生产者，它结合了Promise和Generator的特点。Observable是一个主动的生产者，而且它可以多次返回。用一个例子说明：

```
const observable = Rx.Observable.create((observer) => {
  observer.next(1)
  observer.next(2)
  observer.next(3)
})
const observer = {
  next: (x) => {console.log(x)},
  error: (err) => {console.log(err)},
  complete: () => {console.log('finished')}
}
observable.subscribe(observer)
```
这段程序执行，会依次打印出1，2，3，这就说明了，`observable`可以多次返回。可以看出，`observable`通过`next`生产数据，`observer`则只需要定义消费数据的规则即可，生产者和消费者之间通过`subscribe`建立关联。消费者没有主动拉取数据，整个数据的流动都是主动从生产者到消费者的。

##### End
数据的生产者控制数据消费的时机，而数据的消费者只需要关注消费数据的规则这一点在异步的场景下尤其有用。因为在异步的场景下，数据何时返回或者数据何时被生产是不可控的，如果让数据的消费者处理获取数据这一部分的逻辑，带来的问题是，每个消费者都需要为此做相同的事，而且，数据消费的逻辑不能很好地与获取数据的逻辑分离。但是，当数据的生产者能够控制数据的生产过程，并且主动推送到消费者端，那么消费者端就可以关注于消费数据的逻辑，实现了关注点的分离，逻辑也更清晰。Observable是个很有意思的概念，需要进一步挖掘一下，下次好好扒一扒。
