---
layout: post
title: "C++11之Rvalue References"
categories: blog
excerpt: "C++11新特性中的右值引用。"
tags: [C++, C++11]
share: true
image:
  feature:
date: 2017-08-25T20:01:33-08:00
modified: 
---

# Rvalue References的概念
引用(References)在C++11之前我们都说的是左值引用。C++中的左值概念继承自C语言，指的是可以放在赋值表达式左边的变量，比如有名对象、从堆或栈上申请下来的对象、某对象的成员等。右值概念同样继承自C，只能放在赋值表达式的右边，比如literals、临时对象等。

理论上来讲，左值引用只能绑定到左值对象上，不可以绑定到右值对象。比如`int& i = 42`会导致编译错误。然而，C++标准中却故意留下了一个例外:`const int& i = 42`是可以编译通过的。引入这个例外是考虑可以允许我们向一个接受左值引用的函数传入一个临时对象作为参数。比如：
```
void print(const std::string&);
print("hello world!");
```

无论如何，C++11现在引入了右值引用(lvalue references)的概念，只能绑定到右值对象，不允许绑定到左值对象。右值引用需要用两个&来表示，比如：
```
int&& i = 42;
int j = 2;
int&& k = j; /* 该行会导致编译错误 */
```

左值引用和右值引用可以用以区分两个重载函数。右值概念的引入同时引出了move semantics概念。

# Move semantics
右值一般是一个临时对象，所以我们可以考虑在某种情况下将其里面的内容“偷”过来同时不影响程序的正确执行。这意味着，如果我们可以确定函数的入参是一个右值对象，我们可以不用去copy右值对象中的内容，而是直接move过来。这样，在一个大的动态结构中，我们可以避免手动申请大块内存的开销获得了很大的优化空间。

比如，我们考虑一个接受std::vector<int>作为入参的函数需要在内部修改vector中的内容而不能影响原来的vector。在C++11之前，我们必须在函数内从原来的vector拷贝出来一份再进行修改。如下：
```
void process_copy(const std::vector<int>& _vec)
{
    std::vector<int> vec(_vec);
    vec.push_back(42);
}
```
这样，不论在入参是左值还是右值时，我们必须手动拷贝一份出来再进行修改。现在，我们可以重载一个接受右值引用的函数来处理入参是右值的情况。如下：
```
void process_copy(const std::vector<int>&& _vec)
{
    _vec.push_back(42);
}
```
我们可以直接在右值对象上进行修改。

现在，考虑我们有一个class如下：
```
class X
{
    private:
        int *data;
    public:
        X(): data(new int[100000]){}
        ~X(){ delete [] data;}
        X(const X& other): data(new int[100000])
        {
            std::copy(other.data, other.data + 100000, data);
        }
        X(X&& other): data(other.data)
        {
            other.data = nullptr;
        }
}
```
接受右值引用最为形参的构造函数我们称之为move constructor，通过这个构造函数我们可以节省中间申请内存的时间开销。

对class X来说，move constructor仅仅是一种优化手段。但是在某种情况下，它不止是一种优化手段。比如std::unique_ptr<>类，copy constructor是无意义的。但是通过move constructor我们可以将其所指向对象的所有权移交(move)给另一个std::unique_ptr<>。

如果我们希望将一个我们明确知道不使用的左值move出去，我们可以通过static_cast<X&&>或者std::move()将其强制转换为右值对象。如下：
```
X x1;
X x2 = std::move(x1);
X x3 = static_cast<X&&>(x2);
```

右值引用有三个重要的性质：
1. For overload resolution, **lvalues prefer binding to lvalue references and rvalues prefer binding to rvalue references.** Hence why temporaries prefer invoking a move constructor / move assignment operator over a copy constructor / assignment operator.
2. **rvalue references will implicitly bind to rvalues and to temporaries that are the result of an implicit conversion.** i.e. `float f = 0f; int&& i = f;` is well formed because float is implicitly convertible to int; the reference would be to a temporary that is the result of the conversion.
3. **Named rvalue references are lvalues. Unnamed rvalue references are rvalues.** This is important to understand why the `std::move` call is necessary in: `foo&& r = foo(); foo f = std::move(r);`.

```
void do_stuff(X&& x)
{
    X a(x_); /* copy */
    X b(std::move(x_)); /* move */
}
do_stuff(X());
X x;
do_stuff(x); /* compile error. lvalue can't bind to rvalue reference */
```

# Rvalue References and function templates
如果一个函数模板的形参是一个右值引用模板形参，当传入的参数是左值时，自动模板参数类型推断会将模板形参类型推断为左值引用；当传入的参数是右值时，会按正常理解推断。

比如：
```
template <typename T>
void foo(T&& t){}

foo(42); /* calls foo<int>(42) */
foo(3.14159); /* calls foo<double>(3.14159) */
foo(std::string()); /* calls foo<std::string>(std::string()) */

int i = 42;
foo(i); /* calls foo<int&>(i) */
```

Because the function parameter is declared T&&, this is therefore a reference to a reference, which is treated an just the original reference type. The signature of foo<int&>() is thus `void foo<int&>(int& t);`.