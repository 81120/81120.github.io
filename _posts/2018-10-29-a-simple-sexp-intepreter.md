---
layout: post
title: a simple s-exp intepreter
date: 2018-10-29 13:53:20 +0300
description: a simple s-exp intepreter. # Add post description (optional)
img: # Add image post (optional)
tags: [s-exp,intepreter]
---

##### S-EXP
S-EXP是一种以文本格式表达半结构化数据的约定，与JavaScript中的JSON格式类似，虽然不同语言的实现有不同的数据类型和语法细节，但是各种实现都通用的规则是使用S-EXP作为括号化的前缀表达式。但是它被人们熟悉更多是因为Lisp系的语言都是以S-EXP作为程序的基本结构，Lisp系语言中程序由表达式构成，而表达式都采用了S-EXP的格式。下面用一个简单的求阶乘的例子说明：

```
(define fact
  (lambda (n)
    (cond
      ((zero? n) 1)
      (else
        (* n (fact (- n 1)))))))
```
翻译成等价的JavaScript代码是：

```
const fact = (n) => {
  if(n == 0){
    return 1
  }
  else{
    return n * fact(n - 1)
  }
}
```
Lisp代码中一个显著的特征是，前缀表示，运算符都放在表达式的第一个位置，和函数的调用格式是一致的（在Lisp中，运算符和函数没有本质区别）。如果把每个表达式的运算符作为一棵树的根节点，后面的子表达式按相同的规则展开，作为它的子树，那么就能得到一个AST，和JavaScript代码生成的AST结构基本一样。换种说法，Lisp语言中开发者是面向AST编程的，这就赋予了Lisp很强的灵活性，其中很大一部分是因为S-EXP的灵活性和同像性，很难说清，是Lisp成就了S-EXP，还是S-EXP成就了Lisp。但是S-EXP的结构很简单，这让词法分析和语法分析的过程很简单，不会像C系的语言，编译器中的词法分析和语法分析就占了大量的篇幅。

##### 词法分析
词法分析是为了将源程序分割成单词，而对S-EXP来说，词法分析相当直接：

```
const tokenize = (program) => {
  return program
         .replace(/\(/g, ' ( ')
         .replace(/\)/g, ' ) ')
         .split(' ')
         .filter((x) => x !== '')
}
```
在这个函数的作用下

```
(* 2 (+ 3 (- 4 5))) 
=> [ '(', '*', '2', '(', '+', '3', '(', '-', '4', '5', ')', ')', ')' ]
```

##### 语法分析
笼统的说，语法分析就是生成AST的过程，这个过程的输入是词法分析的结果，输出是程序对应的AST。逻辑上来讲，AST是树形结构的，具体的表现因使用的语言而异。用JavaScript可以实现如下：

```
const readFromTokens = (tokens) => {
  if(tokens.length === 0){
    throw new Error('unexpected EOF while reading')
  }
  token = tokens.shift()
  if(token === '('){
    let L = []
    while(tokens[0] != ')'){
      L.push(readFromTokens(tokens))
    }
    tokens.shift()
    return L
  }
  else if(token === ')'){
    throw new Error('unexpected')
  }
  else{
    return String(token)
  }
}

const parse = (program) => {
  return readFromTokens(tokenize(program))
}
```

`readFromTokens`函数返回的就是AST，parse只是将词法分析和语法分析的过程结合起来。可以看出，`readFromTokens`所做的就是根据`(`和`)`的匹配关系递归的生成一棵树，依然以上面的一段程序为例，经过语法分析后，生成了如下的结构：

```
[
  "*",
  "2",
  [
    "+",
    "3",
    [
      "-",
      "4",
      "5"
    ]
  ]
]
```
好了，本质上，就是利用JavaScript的类型和数据结构把之前的S-EXP重新表达了一遍，看起来只是玩了一些没用的小把戏，但是，正式因为变成了JavaScript的类型和数据结构，接下来就可以进一步利用JavaScript在这个基础上进行解释。

##### 解释运行
有了语法分析的结果，可以开始尝试解释运行这段程序了，解释运行的策略相当明确，获取AST的根节点，先解释运行它的子树，然后把子树的解释运行结果作为参数，来解释运行根节点，这是一个递归的过程。如果我们的程序不支持自定义变量，那么直接对程序中的字面量做替换就可以逐渐规约整棵AST，但是，当程序支持自定义变量，那么就需要在程序运行期间为变量的名称和值维持一个映射关系，那么我们就引进求值环境的概念来表示这种映射。在这里，环境就是一个JavaScript对象。

```
const standardEnv = () => {
  let env = {
    'pi' : Math.PI,
    'sin' : Math.sin,
    'cos' : Math.cos,
    'sqrt' : Math.sqrt,
    '+' : (x, y) => x + y,
    '-' : (x, y) => x - y,
    '*' : (x, y) => x * y,
    '/' : (x, y) => x / y,
    '>' : (x, y) => x > y,
    '<' : (x, y) => x < y,
    '>=' : (x, y) => x >= y,
    '<=' : (x, y) => x <= y,
    '=' : (x, y) => x === y,
    'abs' : Math.abs,
    'append' : (x, y) => x.concat(y),
    'car' : (x) => x[0],
    'cdr' : (x) => x.slice(1),
    'cons' : (x, y) => [x].concat(y),
    'length' : (x) => x.length,
    'list' : (...args) => Array.apply({}, args),
    'list?' : (x) => x instanceof Array,
    'map' : (x, f) => x.map(f),
    'filter' : (x, pre) => x.filter(pre),
    'max' : Math.max,
    'min' : Math.min,
    'not' : (x) => !x,
    'null?' : (x) => !x || x !== x || x.length === 0,
    'number?' : (x) => typeof(x) === 'number',
    'prcedure?' : (x) => typeof(x) === 'function'
  }
  return env
}
```

这个过程初始化了一个全局的环境，按环境的作用来理解，在初始化时，没有任何自定义的变量，那么环境是空的。我们在初始化的时候，定义了一些基本的变量和过程，可以理解为语言的标准库。那么，求值的过程定义如下：

```
const evaluator = (x, env) => {
  if(typeof(x) === 'string'){
    if(isNaN(x)){
      return env[x]
    }
    else{
      return Number(x)
    }
  }
  else if(!(x instanceof Array)){
    return x
  }
  else if(x[0] === 'if'){
    let [_, test, conseq, alt] = x
    let exp = alt
    if(evaluator(test, env)){
      exp = conseq
    }
    return evaluator(exp, env)
  }
  else if(x[0] === 'define'){
    let [_, variable, exp] = x
    console.log(variable)
    console.log(exp)
    env[variable] = evaluator(exp, env)
    console.log(env)
  }
  else{
    let proc = evaluator(x[0], env)
    let args = x.slice(1).map((x) => evaluator(x, env))
    return proc.apply({}, args)
  }
}

const interpret = (program) => {
  return evaluator(parse(program), globalEnv)
}
```

`evaluator`就完成了解释运行的任务，输入是AST，interpret只是为了把词法分析的过程和求值运行的过程组合起来。可以看到，分了4种情况来求值，第一中种是对字面量和变量的求值，第四种是标准库种函数的调用，第二种和第三种情形本质上是一种情形，因为它们对应的AST子树的根节点不是操作符或者函数，而是一种语法关键字，`if`和`define`，分别代表了逻辑分支和自定义变量的规则。到这里，一个粗糙的S-EXP解释器久完成了，把前面定义的代码片段放在这里运行可以得到结果：

```
(* 2 (+ 3 (- 4 5)))  => 4

(define r 10)
(* pi (* r r))) => 314.1592653589793

(if (< 2 3) 2 3) => 2
```
从上面的例子来看，这个解释器已经支持简单的自定义变量（define），分支判断（if）和标准库种的函数调用了（*，+）。还不够，在这个基础上，我们还可以试着给它加一个repl，毕竟现在的语言repl是标配了（顺便吐槽C系的语言，毕竟Lisp系的50年前久有了）。

##### REPL
所幸在Node中实现REPL也很容易，基本的策略是，在一个Loop中读取表达式，然后对表达式进行求值，然后打印表达式的求值结果，然后进入下一轮循环，直接看代码：

```
const repl = require("repl")
const tinyLambda = require('./tiny.js').tinyLambda

repl.start({
  prompt: ">>>> ",
  eval: function(cmd, context, filename, callback) {
    if (cmd !== "(\n)") {
      cmd = cmd.trim();
      console.log(cmd)
      var ret = tinyLambda(cmd);
      callback(null, ret);
    } else {
      callback(null);
    }
  }
});
```

前面解释器的部分都放在tiny.js中，然后在tiny.js中定义`exports.tinyLambda = interpret`。

##### 总结
选择S-EXP的原因有两个，第一，S-EXP做词法分析和语法分析都很简单，不需要很专业的Lex + Yacc来做，这一部分就是纯粹的文本处理，语言的解释，最核心的还是在解释执行的部分。第二，S-EXP和Lisp特别相似，正是由于Lisp的存在，S-EXP可以有除了文件格式的作用，因为在里面可以定义逻辑，相反的例子参照JSON；能亲手实现一个简单的Lisp语言解释器，这种成就感，大家都懂的。这里的简单并不是在谦虚，而是在陈述一个事实，这个解释器还存在以下几个方面的问题：

1. 还不能自定义函数，只能使用标准库里定义的函数。
2. 没有宏的支持，P.S. 没有宏，根本不好意思说是Lisp解释器好吗？
3. 求值环境是全局共用的，还没有实现求值环境的层次结构，话说，求值环境不就是变量的作用域吗？层次的作用域在实现闭包时很重要。
4. 没有尾递归优化。
5. 没有Continuation支持。

这个简单的例子只是为了说明，实现一个语言的解释器，或者一个形式系统的规约规则并没有想象的那么困难，反而很好玩，我们要保持信心。当你的小系统逐渐能够玩一些魔法的时候，你会不由自主的想要赋予这个系统更强的能力，这时，当你单枪匹马地闯进无数前辈耗费数百年心血创建的逻辑世界时，你不是感到迷茫和恐惧，而是能怀着敬畏的心欣赏逻辑系统那种如同雕塑一般严谨的美感，那么这个才是我们做这件事的价值所在。祝玩的开心。
