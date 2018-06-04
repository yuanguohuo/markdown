---
title: C++11中的值的类型
date: 2018-05-24 11:45:11
tags: [lvalue, xvalue, prvalue, glvalue, rvalue, move]
categories: c++
---

本文介绍了C++11引入的值的类型：lvalue, xvalue, prvalue, glvalue, rvalue，以及如何划分。

<!-- more -->

# C++11之前 (1)

在C++11之前，左值右值（lvalue和rvalue）的概念已经流淌在C/C++人的血液之中。如何划分左值右值呢？有的书上通过列举的方式，把所有左值右值列出来。但一个更容易的规则：
* 一个表达式，如果能对它进行&运算(取地址)，则它是左值；
* 一个表达式，若果它是左值的引用(T&或者const T&)，则它是左值；
* 否则，它是右值;

这样：

* 每一个值，都能被定性为左值或者右值（覆盖所有）；
* 一个值，若为左值，它就不是右值；若是右值，它就不是左值（没有交集）；

也就是说，左值右值，既能覆盖所有值，又不存在交集。这样的分类还是比较理想的。但C++11把事情搞的复杂了。

# C++11之后 (2)

## 五种类型 (2.1)

C++11出现了以下五个名词：

* lvalue
* xvalue
* prvalue
* glvalue
* rvalue

简直傻傻分不清楚。看看它们的定义（个人认为还是不清楚，所以就不翻译了）：

* An lvalue (so called, historically, because lvalues could appear on the left-hand side of an assignment expression) designates a function or an object. [ Example: If E is an expression of pointer type, then *E is an lvalue expression referring to the object or function to which E points. As another example, the result of calling a function whose return type is an lvalue reference is an lvalue. —end example ]
* An xvalue (an “eXpiring” value) also refers to an object, usually near the end of its lifetime (so that its resources may be moved, for example). An xvalue is the result of certain kinds of expressions involving rvalue references (8.3.2). [ Example: The result of calling a function whose return type is an rvalue reference is an xvalue. —end example ]
* A glvalue (“generalized” lvalue) is an lvalue or an xvalue.
* An rvalue (so called, historically, because rvalues could appear on the right-hand side of an assignment expressions) is an xvalue, a temporary object (12.2) or subobject thereof, or a value that is not associated with an object.
* A prvalue (“pure” rvalue) is an rvalue that is not an xvalue. [ Example: The result of calling a function whose return type is not a reference is a prvalue. The value of a literal such as 12, 7.3e5, or true is also a prvalue. —end example ]

## 如何区分 (2.2)

那如何分清它们呢？其实一个变量只有两个独立的属性：

* 是否有身份：是否占用一定的内存，程序员可以判断两个值是否就是同一个。比如局部变量有身份，而临时变量没有身份。
* 是否可移动：自己的资源允许被move。关于move语义，见[C++11的std::move](http://www.yuanguohuo.com/2018/05/24/cpp11-std-move/)。

这样就可以分成4类：

 1. 有身份，不可以移动；
 2. 有身份，可以移动；
 3. 没有身份，可以移动；
 4. 没有身份，不可以移动；

但是，第4类是没有意义的(因为没有身份，一定可以移动)。所以，就剩3类：

 1. 有身份，不可以移动；
 2. 有身份，可以移动；
 3. 没有身份，可以移动；

如何给它们命名呢？

 - 前两个，**接近于**左值，因为有身份； 
 - 后两个，**接近于**右值，因为可移动；

但是，之前说过，C++11之前，左值右值的分类已经深入人心，并且：

* 每一个值，都能被定性为左值或者右值（覆盖所有）。 
* 一个值，若为左值，它就不是右值；若是右值，它就不是左值（没有交集）。

所以，命名的时候，还必须保留这一性质。

* 由于标准库中，rvalue这一词是指可移动，所以，后2和3加在一起应该叫rvalue。
* 左值和右值没有交集，并且要覆盖所有，所以，1应该叫lvalue。
* 1和2加在一起，都有身份，**接近于**左值，叫做generalized lvalue，简称glvalue。
* 3没有身份，比如字面量，临时变量，是一类rvalue，并且是非常纯粹的右值，叫做pure rvalue，简称prvalue。
* 好了，就剩2了，它有身份，又可以被move，叫做将亡值(eXpiring value)，简称xvalue。

总结来说，基础类型只有3类：

* 有身份不可以移动：lvalue，例如局部变量，全局变量，函数参数等；
* 有身份可以移动：xvalue，见下文；
* 没有身份可以移动：prvalue，例如字面量，临时对象；

然后，有衍生出来两种：

 - 1+2 = glvalue 
 - 2+3 = rvalue

真是够烦的。所幸，这个分类里，lvalue和rvalue和C++11之前还能保持一致，能进行&（取值运算）的是左值，否则为右值。并且，它们全覆盖，且没交集。还有，我们又多了一个理解rvalue的角度：可移动。

## xvalue举例 (2.3)

xvalue可能还是不够清晰，哪些值属于此类呢？

 - Case1: 对右值引用函数（返回值是右值引用的函数）的调用.

```cpp
std::move(x);
func(); //func()是一个返回右值引用的函数；
```

 - Case2: 把一个左值强制转换为右值引用.

```cpp
static_cast<Foo&&>(x);
```

 - Case3: xvalue的成员.

```
(static_cast<Foo&&>(x)).m_data;
```

可见，Case 1和2是等价的，std::move()就是通过static_cast实现的。它们都是把一个左值（有身份）变得可移动，得到一个**有身份且可移动**的值，也就是xvalue。Case 3得到的也是**有身份**（因为它所属对象有身份）**且可移动**（因为它所属对象可移动）的值，也就是xvalue。


# 小结 (3)

本文介绍了如何区分C++11引入的五中值的类型。

参考：https://stackoverflow.com/questions/3601602/what-are-rvalues-lvalues-xvalues-glvalues-and-prvalues
