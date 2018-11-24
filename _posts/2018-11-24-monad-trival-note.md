---
layout: post
title: monad笔记
date: 2018-11-14 15:42:20 +0300
description: monad 笔记. # Add post description (optional)
img: # Add image post (optional)
tags: [monad, haskell]
---

#### Monad
在haskell中，monad的定义如下：
```haskell
class Monad m where
  return :: a -> m a
  (>>=) :: m a -> (a -> m b) -> m b
```

满足以下的规范：
 ```haskell
 1. return x >>= f = f x
 2. x >>= return = x 
 3. (m >>= f) >>= g = m >>= (\x -> f x >>= g)
 ```


这三条规范本质上是在约束`a -> ma`这种类型的函数在组合时必须遵守的规律，其中`return, f, g`都是它们的代表。第一条和第二条是的意思是`return`是这种组合的单位元，第三条的本质是`f, g`在组合时满足结合律。在`>>=`这种形式下，形式不能清晰的表现出这种组合的关系。
我们可以仔细分析一下这个组合的本质，从第一条开始看：`return x >>= f = f x`，其中有两个函数`return :: (Monad m) => a -> m a`，`f :: (Monad m) => a -> m b`，将等式做一些等价变形：
```
return x >>= f = f x 
=> (\t -> return t >>= f) x = f x 
=> (\t -> return t >>= f) = f
```
左边的`(\t -> return t >>= f)`是一个函数，是将`return, f`组合的结果，那么将这种组合的模式抽象出来就能得到如下的运算：
```haskell
g `m_compose` f = (\x -> f x >>= g)
```
上面的过程就能表达为：
```haskell
f `m_compose` return == f
```
再看这个运算的类型签名：
```haskell
m_compose :: (Monad m) => (b -> m c) -> (a -> m b) -> (a -> m c)
```

考虑计算一下：
```haskell
return `m_compose` f 
```
按照上面的定义：
```haskell
return `m_compose` f = (\x -> f x >>= return)
考虑到 x >>= return = x 
那么可以进一步化简为
return `m_compose` f 
=> (\x -> f x >>= return) 
=> (\x -> f x) 
=> f
```
也就是说：
```haskell
f `m_compose` return == f
```

再来考虑三个函数组合的情况：
```
(f `m_compose` g) `m_compose` h
=> (\x -> h x >>= (f `m_compose` g))
=> (\x -> h x >>= (\y -> g y >>= f))
结合上面的第三条：
=> (\x -> (h x >>= g) >>= f)

f `m_compose` (g `m_compose` h)
=> (\x -> (g `m_compose` h) x >>= f)
=> (\x -> (\y -> h y >>= g) x >>= f)
=> (\x -> (h x >>= g) >>= f)
```

可以看出函数`f :: (Monad m) => a -> m b`组成的集合以及`return :: (Monad m) => a -> m a`在运算`m_compose`下组成一个幺半群。

也就是说，从上面的三个条件能够推出如下结论：
```haskell
定义运算`(<=<)`如下：
(<=<) :: (Monad m) => (b -> m c) -> (a -> m b) -> (a -> m c)
(<=<) g f = (\x -> f x >>= g)
定义函数如下：
return :: (Monad m) => a -> m a
那么所有满足`f :: (Monad m) => a -> m b`类型的函数组成的集合G，在单位元return和运算(<=<)下组成一个幺半群。
```

容易证明这个条件和上面的三个条件等价。