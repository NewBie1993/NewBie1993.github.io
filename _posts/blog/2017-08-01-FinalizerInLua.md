---
layout: post
title: "Lua中的Finalizers"
categories: blog
excerpt: "finalizer允许释放不在垃圾回收器直接控制下的资源。"
tags: [Lua]
share: true
image:
  feature:
date: 2017-08-01T22:04:03-08:00
modified: 
---

# Finalizers的定义
Finalizers是某对象在被回收时调用的与其关联在一起的函数。(A finalizer is a function associated with an object that is called when that object is about to be collected.)

# Finalizers的用法
通过实现某对象元表中的__gc我们可以实现Finalizers机制。 具体如下：
```
o = {x = "hi"}
setmetatable(o, {__gc = function (o) print(o.x) end})
o = nil
collectgarbage() --> hi
```

**注意：**当我们设置某对象的metatable(元表)时， 如果其metatable(元表)的__gc值为非nil， 那么该对象将会被标记。 如果没有被标记， 该对象将不会触发finalization。 也就是说， 在设置metatable(元表)时， metatable(元表)中的__gc值就必须为非nil， 否则即使稍后给metatable的__gc赋值也不会有任何作用。 因为Lua在赋值过程中并不会有任何特殊处理。 如下面一个例子：
```
o = {x = "hi"}
mt = {}
setmetatable(o, mt)
mt.__gc = function (o) print(o.x) end
o = nil
collectgarbage() --> (prints nothing)
```

即使你真的想稍后再设置__gc方法， 你可以先给__gc提供一个任意值， 俗称占位符(placeholder)， 然后再设置具体的函数。 总的来说， 就是要保证设置metatable(元表)的时候对象能够被标记。

当垃圾回收器在一个周期内finalize几个对象时， 将会按照对象被标记的想法顺序来调用它们的finalizer。

# 对finalizer的几个误解
1. 一个常见的误解是认为对象之前的关联关系会影响它们之间被finalize的顺序。 比如想象用Lua实现的一个链表3-->2-->1， 大家可能觉得2必须    在1之前被finalize， 因为2引用了1。 然而， 考虑如果是双向链表呢。 所以说， 对象之间的关联关系不会影响它们之间被finalize的顺序，     仍旧是按照它们被标记的顺序进行。

2. 另一个关于finalizer的复杂点是复活(resurrection)。 当finalizer被调用时， 会将被finalize的对象作为参数传入。 因此， 该对象必须     至少在finalization期间是活着的， 我们称其为短暂复活(transient resurrection)。 (While the finalizer runs, nothing stops it    from storing the object in a global variable, for instance, so that it remains accessible after the finalizer returns.    I call this a permanent resurrection.----目前还没有理解这一段话， 大家有谁明白可以私信我微博[newbie1993](https://www.weibo.com/bangencao1993)^_^)。

   因为有复活的存在， 因此有finalizer的对象被回收时会被分为两个阶段。 第一阶段， 垃圾回收器检测关联finalizer的对象已经不被引用， 将其复活并放入待finalize的队列中。 待其finalizer运行之后， 将其标记为已被finalize。 第二阶段，  垃圾回收器检测到其不被引用， 就删除该对象。 如果你想确保程序中的所有垃圾都被回收， 你应该确保垃圾回收器被调用两次， 第二次将会删除在第一次调用中被finalize的对象。

# 利用finalizer机制可以实现的一些功能
当直到程序结束， 某些对象也没有被回收时， Lua将会在Lua state被close前调用这些对象的finalizer。 我们可以利用该机制实现Lua中的      atexit功能。 示例如下：
```
_G.AA = {__gc = function ()
-- your 'atexit' code comes here
print("finishing Lua program")
end}
setmetatable(_G.AA, _G.AA)
```

另一个比较有意思的技术是， 可以在每一个垃圾回收周期结束的时候调用一个函数。 因为对象只能被finalize一次， 我们可以在finalizer中生成一个新对象使其在下一次中运行finalizer。 具体如下：
```
do
local mt = {__gc = function (o)
-- whatever you want to do
print("new cycle")
-- creates new object for next cycle
setmetatable({}, getmetatable(o))
end}
-- creates first object
setmetatable({}, mt)
end
collectgarbage() --> new cycle
collectgarbage() --> new cycle
collectgarbage() --> new cycle
```

# finalizer与weak表之间的一些小细节
垃圾回收器在复活对象之前清除value值， 在复活对象之后清除key值。
```
-- a table with weak keys
wk = setmetatable({}, {__mode = "k"})
-- a table with weak values
wv = setmetatable({}, {__mode = "v"})
o = {} -- an object
wv[1] = o; wk[o] = 10 -- add it to both tables
setmetatable(o, {__gc = function (o)
print(wk[o], wv[1])
end})
o = nil
collectgarbage() --> 10 nil
```

这种行为的理论依据是， 我们会经常使用key为弱引用的weak表去关联对象的某些属性， 并且finalizer可能会获取这些属性。 原文如下：
The rationale for this behavior is that frequently we use tables with weak keys to keep properties of an object, and finalizers may need to access those attributes. However, we use tables with weak values to reuse live objects; in this case, objects being finalized are not useful anymore.
