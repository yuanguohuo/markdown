---
title: C++11中的通用引用
date: 2018-05-25 12:03:23
tags: [universal reference, 通用引用]
categories: c++
---

本文主要介绍C++11中的通用引用，包括构成通用引用的条件，几种类型的通用引用及各自使用场景。也列举了一些容易被误认为是通用引用的情形。

<!-- more -->

# 构成通用引用的条件

熟悉右值引用的话，就会知道右值引用是通过"&&"来声明的。可能看见"&&"的第一反应就是右值引用。可惜，有些情况下竟然不是。在[C++11中的右值引用](http://www.yuanguohuo.com/2018/05/25/cpp11-rvalue-ref/)，我们看到了std::move的源代码：

```cpp
template <typename T>
typename remove_reference<T>::type&& move(T&& t)
{
    return static_cast<typename remove_reference<T>::type&&>(t);
}
```

