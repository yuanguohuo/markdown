---
title: C++11中的完美转发
date: 2018-05-25 12:04:03
tags: [完美转发, perfect forwarding, 通用引用, universal reference]
categories: c++
---

本文介绍完美转发。第1节和第2节，是翻译[这篇文章](http://thbecker.net/articles/rvalue_references/section_07.html)和[这篇文章](http://thbecker.net/articles/rvalue_references/section_08.html)。

<!-- more -->

# 问题 (1)

我们想写这样一个工厂模板函数，使用参数`Arg`类型的参数`arg`，构造一个`T`的对象，并返回它的`shared_ptr`。我们的理想是**这层工厂函数就像不存在一样**：

* 没有额外的拷贝；就像调用者直接使用`arg`构造`T`的对象一样；
* 若`arg`是右值，则需要保留其右值的特征(匹配`T`的移动构造函数，若`T`有对应的重载)；

## 尝试1 (1.1)

```cpp
template<typename T, typename Arg> 
shared_ptr<T> factory(Arg arg)
{ 
  return shared_ptr<T>(new T(arg));
} 
```

显然，这是一个失败的尝试：

* 引入了额外的拷贝：调用`factory`的时候，`arg`是值传递的，需要实参到形参的拷贝；注意，把`arg`传给`T`的构造函数没有带来额外的拷贝，无论是按值传递还是按引用传递，因为即使没有我们的factory函数，用户直接构造`T`的对象也需要传递。
* 若`T`的构造函数的参数是左值引用类型，就更糟糕：它引用了一个实参(factory函数结束后，实参就消亡了)；

## 尝试2 (1.2)

```cpp
template<typename T, typename Arg> 
shared_ptr<T> factory(Arg& arg)
{ 
  return shared_ptr<T>(new T(arg));
} 
```

`factory`的参数是个引用，因此额外的拷贝是没有了。但是`arg`是一个左值引用，它不能匹配右值，这样的调用无法通过编译：

```cpp
factory<X>(hoo()); // error if hoo returns by value
factory<X>(41); // error
```

## 尝试3 (1.3)

同时提供`factory(Arg&)`和`factory(Arg const &)`：

```cpp
template<typename T, typename Arg> 
shared_ptr<T> factory(Arg& arg)
{ 
  return shared_ptr<T>(new T(arg));
} 

template<typename T, typename Arg> 
shared_ptr<T> factory(Arg const & arg)
{ 
  return shared_ptr<T>(new T(arg));
} 
```

这样，实参若是左值则匹配前者；若是右值则匹配后者；额外拷贝的完全避免了。但是，还有两个问题：

1. 即使参数是右值，也无法匹配`T`的移动构造函数；
2. 若factory有n个参数，每个参数都有const引用和非const引用两种类型，组合起来就需要2^n个重载；

# 解决方案 (2)

回顾一下我们的理想：**这层工厂函数就像不存在一样**：

* 没有额外的拷贝；就像调用者直接使用`arg`构造`T`的对象一样；
* 若`arg`是右值，则需要保留其右值的特征(匹配`T`的移动构造函数，若`T`有对应的重载)；

这要求我们：
1. 必须按引用传递；
2. 必须能引用左值和右值；
3. 若实参是右值，必须保留右值的特征；

第1.3节中使用的*const的左值引用*(`Arg const&`或与之等价的`const Arg&`)可以满足第1和第2点。但实参是右值时，丢失了它右值的特征(无法匹配`T`的移动构造函数)。容易想出，通用引用(见[C++11中的通用引用](http://www.yuanguohuo.com/2018/05/25/cpp11-universal-ref/))能够同时满足这三点。

