---
layout: post
title: ELM简介
date: 2018-10-29 13:53:20 +0300
description: ELM简介. # Add post description (optional)
img: # Add image post (optional)
tags: [ELM, Front-End]
---

#### 引言
这份笔记起源于某天在知乎上看到的一篇专栏文章[Elm 0.17 新架构解析](https://zhuanlan.zhihu.com/p/21338799?refer=damotou)，后来断断续续看了一些相关的材料，觉得很受启发，就记下了一些笔记，便于梳理思路。[Elm](http://elm-lang.org/)起源于2012年，借鉴了Haskell的部分语法，是一种强类型，静态类型的函数式编程语言，语法简洁，极富表现力；最终编译为JavaScript运行，为前端提供了一个很有启发性的编程模型。

#### Elm的主要特点
##### 1. 类型系统
不同于JavaScript，Elm是一种强类型和静态类型的语言。静态类型和强类型的特点可以让开发者在编译的过程中发现很多潜在的Bug，这种类型的问题在JavaScript出现的频率是非常高的。由于Elm函数式的特点，函数也有确定的类型，这样也能帮开发者发现更多由于参数类型不适合导致的Bug。而且，Elm还支持ADT(代数数据类型)，利用ADT开发者可以构建复杂的类型，实现对结构化数据的建模。这些都是借助于编译器来实现的，经过Haskell社区和PL领域长期的实践和研究，通过类型对世界建模是除了利用对象对世界建模的另一种可行的方案。
##### 2. 编程模型
Elm是函数式的编程语言，最直观的特点有两个，首先，函数是纯函数，意味着相同的输入一定会产生相同的输出；其次，支持高阶函数。第二点属于语法特性，很多语言都支持，在JavaScript中也是非常常用的特性，是一种组合函数的手段；而第一点可以理解为一种约定，只要函数不依赖于外界状态，那么它就是一个纯函数，这么做的好处是函数的状态是可以预测的，便于单元测试。这些特点都是为函数式编程所倡导的编程思想服务的。如果把程序抽象成对数据的操作，而函数式编程则是通过一些纯函数的组合来从原始的数据来构造新的数据，期间，不会对原始数据的状态做任何修改。Elm从语言层面支持了这些特性，而且，在前端UI构建的层面引进了这种思想。在Elm中，快速排序可以实现如下：
> 
```
quicksort : List comparable -> List comparable
quicksort list = 
    case list of
        [] ->
            []
        xs :: rest ->
            let 
                lower = List.filter (\n -> n <= xs) rest
                higher = List.filter (\n -> n > xs) rest
            in
                quicksort lower ++ [xs] ++ quicksort higher
```

##### 3. 组件模型
上面提到，Elm在前端UI构建的层面引进了函数式编程的思想，这体现在Elm将状态的变化过程抽象为一个纯函数。用Elm开发前端时，最核心的是开发状态更新的函数。因为当状态发生变化时，Elm会自动根据状态重新渲染UI。为了实现这一点，Elm需要实现根据状态自动生成UI的功能。在Elm采用的方案是对DOM和事件进行了抽象，提供了类似模板的东西，利用Elm的编译器转换成真实的DOM结构，为了UI渲染的提高性能，在实现的过程中也支持了Virtual DOM。

在Elm中，一个组件通常由三个部分定义model对象，update函数和view函数，签名如下：
>
```
    model : Model
    update : Msg -> Model -> Model
    view : Model -> Html Msg
```

model定义了数据模型，update定义了事件发生时model的更新规则，view则定义了model到UI的对应关系，以一个简单的例子来说明。考虑一个简单的场景，当用户在一个输入框中输入一行字符串时，输入框下面实时显示输入的字符串逆序排列的字符串，那么实现的核心代码如下，完整实现可参考[example](http://elm-lang.org/examples/field):
>
```
    type Msg = NewContent String
    update (NewContent content) oldContent = 
        content
    view content = 
        div []
            [ input [ placeholder "text to reverse", onInput NewContent, myStyle ] [],
             div [ myStyle ] [ text (String.reverse content) ]]
    myStyle =
            style
                [ ("width", "100%")
                , ("height", "40px")
                , ("padding", "10px 0")
                , ("font-size", "2em")
                , ("text-align", "center")
            ]
```

只需要这些就可以完成上面叙述的功能，不需要显式调用其中的update和view函数，这些都是Elm自动完成的。update的第一个参数是事件，第二个参数是当前的状态，每当事件触发，Elm就会把事件和当前的状态传给update函数，来计算新的状态，再将新的状态传给view函数，产生新的UI。这种编程模型的特点在于Elm将状态和事件抽象封装了起来，开发者只需要提供update和view函数，而这两个函数都是纯函数，这就使得组件的状态是很容易预测的，而且函数的开发和测试也相当简单。

#### 启发与总结
对于用惯了JQuery的开发者来说，Elm这种UI构建方案是相当有启发性的。首先编译为JavaScript这种思路为前端开辟了巨大的想象空间，ES6，ClojureScript，Dart，JSX，Vue都采用了这种思路，本质上是为了利用更有表现力的语法，降低工程化的难度。其次，不同于JQuery中构造式的UI生成方式，Elm倡导的声明式的UI生成方式大大降低了开发的复杂度。然后，引入函数式风格，使组件中状态的变化可以预测，大大降低了开发和测试的难度。

从目前的情况来看，Elm不大可能成为一个主流的前端解决方案，一方面是因为Elm的语法相对小众，开发者接受成本高；另一方面是没有一个有影响力的组织来推向市场。归根结底它不流行不是它不够好，就像大家都用JavaScript并不是因为JavaScript本身有多么优美，严谨，很多时候是被动接受市场的选择。但是这些也不妨碍我们借鉴它的思想来更好的使用JavaScript。

综上，只是很浅显的看了一些材料，关于Elm还有很多问题值得进一步研究，如果有机会以后深入研究，先记录如下：

1. 从数据模型自动渲染UI的实现机制，尤其是Virtual DOM的实现机制。
2. 组件组合的方式，目前官方给出的解决方案是嵌套，但是嵌套的方案中组件之间的通信方式是个很重要的问题。
3. 组件之间的通信方式，官方文档中提到使用基于消息传递的并发模型，这种模型在Erlang和Elixir中已经被证实是很有效的并发模型，但是在前端的环境下还没有证据能证明它的有效性。
4. 用JavaScript来模拟这些特性，应该会得到一个很有意思的前端框架。

#### 参考资料
1. [Elm官网](http://elm-lang.org/)
2. [Elm的原始论文](http://elm-lang.org/papers/concurrent-frp.pdf)
3. [知乎专栏文章：Elm 0.17 新架构解析](https://zhuanlan.zhihu.com/p/21338799?refer=damotou)