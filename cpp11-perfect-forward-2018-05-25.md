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
* 若`arg`是右值，则需要保留其右值的特征(匹配`T`的移动构造函数——若有)；

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

这样，实参若是非const的左值则匹配前者；若是const的左值或者右值则匹配后者；额外拷贝的完全避免了。但是，还有两个问题：

1. 即使实参是右值，匹配后者，`arg`引用的是右值，`new T(arg)`也无法匹配`T`的移动构造函数；
2. 若factory有n个参数，每个参数都有const引用和非const引用两种类型，组合起来就需要2^n个重载；

# 解决方案 (2)

回顾一下我们的理想：**这层工厂函数就像不存在一样**：

* 没有额外的拷贝；就像调用者直接使用`arg`构造`T`的对象一样；
* 若`arg`是右值，则需要保留其右值的特征(匹配`T`的移动构造函数——若有)；

这要求我们：
1. 必须按引用传递；且左值和右值必须都能引用；
2. 若引用右值，必须保留右值的特征；

第1.3节中使用的*const的左值引用*(`Arg const&`或与之等价的`const Arg&`)可以满足第1点。但如前所述，无法满足第2点。容易想出，通用引用(见[C++11中的通用引用](http://www.yuanguohuo.com/2018/05/25/cpp11-universal-ref/))能够同时满足这两点。

首先这个模板函数声明应该是这样的：

```cpp
template<typename T, typename Arg> 
shared_ptr<T> factory(Arg&& arg);
```

因为`arg`是一个通用引用，匹配左值时它是左值引用，匹配右值时它是右值引用。乍一看，认为这样可以了：

```cpp
template<typename T, typename Arg> 
shared_ptr<T> factory(Arg&& arg)
{
  return shared_ptr<T>(new T(arg));
}
```

其实，这不行。看过[C++11中的右值引用](http://www.yuanguohuo.com/2018/05/25/cpp11-rvalue-ref/)和[C++11中的通用引用](http://www.yuanguohuo.com/2018/05/25/cpp11-universal-ref/)就会知道，**无论右值引用类型的变量还是通用引用类型的变量，都是左值，因为它们有名字**。这里名字为`arg`的变量(函数参数)，当然是左值。那么`new T(arg)`无论如何也不能匹配移动构造函数。我们想要的是：

* 当左值`arg`引用的是左值时，把被引用的左值传给`new T()`；
* 当左值`arg`引用的是右值时，把被引用的右值传给`new T()`；由于`arg`本身是左值，我们需要使用static_cast把它强制为右值；

C++11提供的`std::forward()`，使用一行代码，就给我们实现了。它的玄妙之处在于类型推导和引用折叠(见[C++11中的通用引用](http://www.yuanguohuo.com/2018/05/25/cpp11-universal-ref/))。

## std::forward (2.1)

```cpp
template<class S>
S&& forward(typename remove_reference<S>::type& a) noexcept
{
  return static_cast<S&&>(a);
}
```

还是通过`factory`例子来看它是如何处理左值和右值的吧。有了它，我们的`factory`函数可以这样写：

```cpp
template<typename T, typename Arg> 
shared_ptr<T> factory(Arg&& arg)
{ 
  return shared_ptr<T>(new T(std::forward<Arg>(arg)));
} 
```

** forwad左值：**

```cpp
X x;
factory<A>(x);
```

根据类型推导规则(见[C++11中的通用引用](http://www.yuanguohuo.com/2018/05/25/cpp11-universal-ref/))，函数模板`factory`的类型参数`Arg`的推导结果是`X&`，那么`factory`展开为：

```cpp
shared_ptr<A> factory(X& && arg)
{ 
  return shared_ptr<A>(new A(std::forward<X&>(arg)));
} 
```

用`Arg`(`X&`)替换`std::forward`类型参数`S`：

```cpp
X& && forward(remove_reference<X&>::type& a) noexcept
{
  return static_cast<X& &&>(a);
} 
```

经过remove_reference和引用折叠(见[C++11中的通用引用](http://www.yuanguohuo.com/2018/05/25/cpp11-universal-ref/))，最终结果是：

```cpp
shared_ptr<A> factory(X& arg)
{ 
  return shared_ptr<A>(new A(std::forward<X&>(arg)));
} 

X& std::forward(X& a) noexcept
{
  return static_cast<X&>(a);
} 
```

可以看出：当左值`arg`引用的是左值时，传递给`new T()`的是被引用的左值(被引用的左值再经过`static_cast<TX&>`得到的还是左值引用)。

** forwad右值：**

```cpp
X foo();
factory<A>(foo());
```

根据类型推导规则(见[C++11中的通用引用](http://www.yuanguohuo.com/2018/05/25/cpp11-universal-ref/))，函数模板`factory`的类型参数`Arg`的推导结果是`X`，那么`factory`展开为：

```cpp
shared_ptr<A> factory(X&& arg)
{ 
  return shared_ptr<A>(new A(std::forward<X>(arg)));
} 
```

这种情况下，factory不需要引用折叠。然后，用`Arg`(`X`)替换`std::forward`类型参数`S`：

```cpp
X&& forward(typename remove_reference<X>::type& a) noexcept
{
  return static_cast<X&&>(a);
}
```

经过remove_reference，`forward`简化成这样：

```cpp
X&& forward(X& a) noexcept
{
  return static_cast<X&&>(a);
}
```

可以看出：当左值`arg`引用的是右值时，传递给`new T()`的是被引用的右值(static_cast<X&&>把`arg`强制为右值，因为没有名字)。

**完美转发的两步：**

可见，完美转发中，有两个关键步骤：

1. 使用通用引用来接收参数；这样，左值右值都能接收；
2. 使用std::forward()转发给被调函数；这样，左值作为左值传递，右值作为右值传递；

## std::forward和std::move比较 (2.2)

对比[C++11中的通用引用](http://www.yuanguohuo.com/2018/05/25/cpp11-universal-ref/)中*std::move的工作原理*一节，我们发现，std::forward和std::move还是有一些相似之处的：

1. 它们都使用static_cast()来完成任务；
2. 它们都使用通用引用(及背后的类型推导和引用折叠机制)；

所不同的是：

1. `std::forward`参数是左值引用，返回的是通用引用；`std::move`参数是通用引用，返回的是右值引用；

这很好理解：
* `std::forward`拿到的是一个左值(所以参数是左值引用)；然后看这个左值是左值引用类型还是右值引用类型，若为前者则返回左值，若为后者则返回右值(所以返回值是通用引用)；
* `std::move`拿到的可能是左值也可能是右值(所以参数是通用引用)，但一定返回右值(所以返回值是右值引用)；

# (3)
