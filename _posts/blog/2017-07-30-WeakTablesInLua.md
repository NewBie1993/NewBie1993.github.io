---
layout: post
title: "Lua中的weak表"
categories: blog
excerpt: "像其他高级语言一样，Lua提供了内存自动管理功能，使用垃圾回收机制来自动删除已成为垃圾的对象。然而，有时即使非常聪明的回收器也需要你的帮助。没有任何垃圾回收器能做到使你完全不用关心资源管理。"
tags: [Lua]
share: true
image:
  feature:
date: 2017-07-30T17:22:29-08:00
modified: 
---

# weak表（弱表）的定义
**weak表**是用来告诉Lua某一对象的某一引用不应阻止垃圾回收器回收该对象的一种机制。（Weak tables are the mechanism that you use to tell Lua that a reference should not prevent the reclamation of an object, that is weak tables allow the collection of Lua objects that are still accessible to the program.）

弱引用是不被垃圾回收器考虑的一种对某对象的引用。 如果某对象的所有引用都是弱引用， 那么该对象将会被回收， 并且相应的弱引用将会被删除。

# 为什么要有weak表这种机制
我们都知道， 在Lua中， 如果我们不再使用某对象， 可以将所有对该对象的引用赋为nil值， 这样Lua的垃圾回收器会在下次运行时回收该对象。 但是简单的清除所有的引用并不总是有用的。 一个经典的例子是， 假设我们希望在一个表liveTable中维护当前程序中所有正在使用的对象， 我们该如何实现它呢？ 答案貌似很简单： 每当我们有一个新的对象时， 就将该对象插入到liveTable中。 然而， 该对象一旦插入到该table中， 该对象将永远不会被回收。 因为即使已经没有别的引用在指向它， 但是liveTable表在引用。 Lua并不知道该引用不应该阻止对某一对象的回收， 这需要我们去告诉它。

# 怎样去实现一个weak表
正常情况下， 某个表的key或者value都是强引用， 它们都会阻止垃圾回收器回收它们所引用的对象。 在weak表中， key或者value都可以是弱引用， 也就是说会有三种类型的weak表：
1. 只有key是弱引用的weak表.
2. 只有value是弱引用的weak表.
3. key和value都是弱引用的weak表.

weak表的弱引用性质是由其metatable(元表)中的__mode字段给出的。 __mode字段对应的value是一个string值：
1. 如果value值是"k"， 则该表中的key字段是弱引用.
2. 如果value值是"v"， 则该表中的value字段是弱引用.
3. 如果value值是"kv"， 则该表中的key字段和value字段均为弱引用.

一个具体的例子：
```
a = {}
setmetatable(a, {__mode = "k"})    -- 设置a为弱表， 且其key是弱引用（其value不是弱引用）
key = {}                           -- 创建第一个对象
a[key] = 1
key = {}                           -- 创建第二个对象（与第一个对象不同）  
a[key] = 2
collectgarbage()
for k, v in pairs(a) do print (v) end -- 预期只输出2，  因为随着第一个对象被回收， 该表中的第一条记录整个被删除
```

**注意：**只有对象才可以从弱表中被回收。 其他的值， 比如numbers、 booleans是不会被回收的。 但是虽然strings值是可以被回收的， 它却有一点儿比较细微的差别。 从实现的角度来说， strings不同于table或者thread的显示创建。 比如， Lua在计算{}表达式时，会显示创建一个table。 但是， 如果Lua在计算"a".."b"呢？ 如果系统中已经存在一个"ab"了呢？ Lua会重新创建一个还是在运行程序之前创建一个？ 这些并不重要， 这只是一个实现细节。 从程序员的角度来说， strings是一个值而不是一个对象， 因此像numbers或者booleans一样， strings在弱表中是不会被回收的（除非对应的整条记录被移除）。

# weak表的几种应用
weak表可以用来实现*记忆函数(Memoize function)*、 *关联对象属性(Object Attributes)*、 *默认值(default value)*等。

## 记忆函数(Memoize function)
这是一种典型的以空间换取时间的处理方案。 假设这里有一个用Lua编写的服务器程序，用来处理用户请求。 每次接收到用户请求之后， 该程序调用load函数处理该请求， 获取对应的处理函数。 load是一种消耗比较大的函数， 并且某些请求可能会比较频繁， 我们可以利用一个辅助table来记忆对应的处理函数以提高性能。
```
results = {}
setmetatable(results, {__mode = "v"})
function mem_loadstring(s)
    local res = results[s]
    if nil == res then
        res = assert(load(s))
        results[s] = res
    end
    return res
end
```

## 关联对象属性(Object Attributes)
一般，我们可以直接在table中设置一个相应的key来保存对应的属性。 然而， 如果一个对象不是用table来实现的呢？ 即使是用table来实现的，如果我们并不想在对象本身的table中保存属性呢？ 比如， 我们希望一个对象的某个属性是私有的， 或者我们不希望某个属性去干扰对象的遍历。 这时候， 我们可以额外创建一个key为弱引用的weak表（value当然不能是弱引用，防止某个未被回收对象的属性丢失）， 以对象本身为key值，value为属性值。 这样， 当对象释放之后， 关联属性表中的对应记录也会在下次垃圾回收时被回收。

## 默认值(default value)
我们可以使用前面两个*记忆函数(Memoize function)*和*关联对象属性(Object Attributes)*方式来实现该特性。

### 关联对象属性(Object Attributes)方式
```
-- weak表中保存每个对象的默认值， 以对象为key， 默认值为value， 弱表中的key为弱引用。 每个对象设置一个metatable（元表）。
defaults = {}
setmetatable(defaults, {__mode = "k"})
mt = {__index = function(t) return defaults[t] end}
function setDefault(t, d)
    defaults[t] = d
    setmetatable(t, mt)
end
```

### 记忆函数(Memoize function)方式
```
-- 每个默认值生成一个metatable（元表）， 然后保存到一个value为弱引用的weak表中， 具有相同默认值的对象共享同一个metatable（元表）
metas = {}
setmetatable(metas, __mode = "v")
function setDefault(t, d)
    local mt = metas[d]
    if nil == mt then
        mt = {__index = function() return d end}
        metas[d] = mt 
    end
    setmetatable(t, mt)
end
```

#### 总结
两种实现方式具有相同的复杂度和性能。 第一种方式每个对象仅需要一小部分内存在关联属性表中保存自己的默认值（weak表中的一条记录）。 第二种方式为每一个默认值生成一个metatable（元表）。 所以， 如果实际中有大量的表默认值都相同， 建议使用第二种方式。 否则， 我们还是应该使用第一种方式实现。

### Ephemeron Tables（蜉蝣表？）
考虑在key为弱引用， value为强引用的weak表中， value引用了他所对应的key这种情况。 从weak的标准解释来看， 尽管该weak表的key为弱引用， 但它的value并不是， 因此可以理解为还有value这个强引用在引用该key值， 整条记录将不会被移除。 但这么严格的解释并没有什么作用， 因为在实际使用中， 我们都是希望通过key去获取它对应的value。

Lua5.2为了解决该问题引入了Ephemeron Tables的概念。 **key为弱引用， value为强引用的weak表即为Ephemeron Tables(A table with weak keys and strong values is an *ephemeron table*)。** 在一个ephemeron table中， value是否可以获取到， 取决于其对应的key是否可以被获取到。 具体来说， ephemeron table中的一条记录(k, v), 只有在有其他强引用引用k时， 其对应的v才可以被认为是强引用。 否则， 即使v在（直接或间接）引用该k值， 整条记录也会被从该表中移除。

