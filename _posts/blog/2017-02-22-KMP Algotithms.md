---
layout: post
title: "KMP算法"
categories: blog
excerpt:
tags: [Algotithms]
share: true
image:
  feature:
date: 2017-02-22T23:11:56-08:00
modified: 
---

该算法的精髓在于：每次匹配出现不等的字符后，不需要像普通算法一样，从头开始匹配，而是会利用每次匹配得到的部分匹配结果
将模式串尽可能的向前滑行一段距离，继续进行比较。其时间复杂度为O(m+n)。

#### 一、一般情况

假设主串为S<sub>0</sub>S<sub>1</sub>...S<sub>n</sub>，模式串为P<sub>0</sub>P<sub>1</sub>...P<sub>m</sub>.

当主串的第i个字符与模式串的第j个字符“失配”（即两个字符不等）时，假设此时主串的第i个字符应与模式串的第k个字符进行匹配。
如之前所说，我们需要将模式串尽可能的向前滑行，因此不会存在k' > k满足如下等式：

<code>
    P<sub>0</sub>P<sub>1</sub>...P<sub>k-1</sub> = S<sub>i-k</sub>S<sub>i-k+1</sub>...S<sub>i-1</sub>    ········(1)
</code>

既然主串是第i个字符与模式串第j个字符失配，则：

<code>
    P<sub>j-k</sub>P<sub>j-k+1</sub>...P<sub>j-1</sub> = S<sub>i-k</sub>S<sub>i-k+1</sub>...S<sub>i-1</sub>    ········(2)
</code>

由(1)和(2)可得：

<code>
    P<sub>0</sub>P<sub>1</sub>...P<sub>k-1</sub> = P<sub>j-k</sub>P<sub>j-k+1</sub>...P<sub>j-1</sub>    ········(3)
</code>

所以，只要存在k(0<k<j)，使得模式串前j个字符组成的子字符串的前缀(0到k-1)和后缀(j-k到j-1)相等，我们即可在主串S第i个字
符与模式串的第j个字符失配时，使模式串向前滑动j-k继续比较即可。

#### 二、KMP算法伪代码

前提：数组next[m+1]保存了模式串第j个字符与主串失配时，应向前滑动继续与主串比较的字符位置k.

```
/* KMP算法伪代码 */
KMP(S, P, pos) //S为主串， P为模式串， pos为主串开始比较的位置
1 i = pos
2 j = 0
3 while (i < length(S) && j < length(P))
4   if (-1 == j || S[i] == P[j])
5       i = i + 1
6       j = j + 1
7   else
8       j = next[j]
9 if (j == length(P))
10     return i - k
11 return -1
```

#### 三、求解next数组

当主串与模式串第一个字符即失配时，我们易知，主串应前进一位，拿下一个字符接着与模式串的第一个字符进行匹配。因此，我们得到：

```
    next[0] = -1; 我们以-1来表示主串应前进一位
```

假设，`next[j] = k`. 当

1. <code>P<sub>k</sub> = P<sub>j</sub></code> 时，`next[j+1] = k +1`.
2. <code>P<sub>k</sub> != P<sub>j</sub></code> 时，我们可以将模式串的前缀理解为新的模式串，后缀理解为新的主串，新主串
的第j位与新模式串的第k位失配，此时新主串应继续与新模式串的第`next[k]`位进行比较，如果相等，则`next[j+1] = next[k'] + 1`.
否则，以此类推，一直到能匹配上或匹配结束。
如果，直到匹配结束也没有找到满足条件的k'，说明模式串的子字符串<code>P<sub>0</sub>P<sub>1</sub>...P<sub>j</sub></code>
压根就没有前缀和后缀能匹配上,因此主串应继续与模式串的第一个字符相比较，即`next[j+1] = 0`.

```
/* 求next数组伪代码 */
KMP_NEXT(P, m, next) //P为模式串，m为模式串长度
1 i = 0
2 j = -1
3 next[0] = -1
4 while (i < m-1 )
5    if (j == -1 || P[i] == P[j])
6        i = i + 1
7        j = j + 1
8        next[i] = j
9    esle
10       j = next[j]
```

#### 求解next数组的进一步优化

考虑如果P<sub>j</sub>正好等于P<sub>k</sub>呢？P<sub>j</sub>都已经与主串的某个字符不匹配了，那么主串的字符当然不用与P<sub>
k</sub>再进行一次比较了。因此此时，我们可以继续获取`k'=next[k]`来让P<sub>k'</sub>继续与主串的字符进行比较，以此类推。

```
/* 进一步优化后的求解next数组伪代码 */
KMP_NEXT_VAL(P, m, next_val)
1 i = 0
2 j = -1
3 next_val[0] = -1
4 while (i < m-1)
5    if (j == -1 || P[i] == P[j])
6        i = i + 1
7        j = j + 1
8        if (P[i] == P[j])
9            next_val[i] = next_val[j]
10       else
11           next_val[i] = j
12   else
13       j = next_val[j]
```