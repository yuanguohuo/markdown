---
title: C++11中的通用引用
date: 2018-05-25 12:03:23
tags: [universal reference, 通用引用]
categories: c++
---

本文主要介绍C++11中的通用引用，包括构成通用引用的条件，几种类型的通用引用及各自使用场景。也列举了一些容易被误认为是通用引用的情形。

<!-- more -->

# 左值右值都可引用(1)

熟悉右值引用的话，就会知道右值引用是通过"&&"来声明的。可能看见"&&"的第一反应就是右值引用。可惜，有些情况下竟然不是。在[C++11中的右值引用](http://www.yuanguohuo.com/2018/05/25/cpp11-rvalue-ref/)，我们看到了std::move的源代码：

```cpp
template <typename T>
typename remove_reference<T>::type&& move(T&& t)
{
    return static_cast<typename remove_reference<T>::type&&>(t);
}
```

这里std::move的参数t，就不是右值引用。若是右值引用的话，你怎么传一个左值给它当参数呢？它的作用可是把左值转换成右值啊。所以，t必须能匹配左值。另一方面，这样的代码也不会出错：

```cpp
Foo f("123");
Foo&& r1 = std::move(std::move(f));

Foo&& r2 = std::move(Foo("456"));
```

也就是说，std::move也可以接受右值作参数，t也能够匹配右值。实际上，参数t就是一个**通用引用**，**可以使用左值表达式初始它，也可以使用右值表达式初始化它。匹配左值表达式时，它表现的像一个左值引用；匹配右值表达式时，它表现的像一个右值引用**。


# 构成通用引用的条件(2)

出现"&&"的时候，如何判断它是右值引用，还是通用引用呢？通用引用有两个条件：

1. 形如"T&&"；
2. T的类型需要推导；

也就是说，若T是已知类型，不需要推导，则是右值引用；T需要推导，则是通用引用。先看一些通用引用的例子：

```cpp
int lvalue = 3;
auto&& r1 = lvalue;

auto&& r2 = 4;

int&& r3 = 5;
auto&& r4 = r3; //等价于int& r4 = r3;

std::vector<int> vect;
auto&& r5 = vect[0];

template<typename T>
void func(T&& p);

func(lvalue);
func(10);
```

这里r3是右值引用，因为没有类型推导；相反，r1, r2, r4, r5和p都是通用引用：
* r1**等价于**左值引用；
* r2**等价于**右值引用；
* r4**等价于**左值引用；没错，是左值引用，因为r3是左值(虽然它是右值引用类型的变量)；
* r5**等价于**左值引用；因为std::vector<int>::operator[]返回左值引用(左值引用是左值)；
* func(lvalue)中，p**等价于**左值引用；
* func(10)中，p**等价于**右值引用；

这两个条件看起来不难，但实际上还是比看起来苛刻。看下面的例子：

例1：必须形如"T&&"

```cpp
int lvalue = 3;

const auto&& r = lvalue; //错误：r不是通用引用，不能通过左值初始化！

template<typename T>
void func(const T&& p);

func(lvalue);            //错误：p不是通用引用，不能通过左值初始化！
```

必须是"T&&"，"const T&&"都不行。当然，"T"可以换成别的类型参数名，或者"auto"；

例2：必须存在类型推导

```cpp
template<typename T>
class Bar
{
  private:
    T _m;
  public:
    Bar(Bar&& other); //不是通用引用，是右值引用
};
```

虽然是类模板，但没有类型推导，other的类型就是Bar&&，所以也不是通用引用。甚至，下面也不是通用引用：

```cpp
template<typename T>
class Bar
{
  private:
    T _m;
  public:
    Bar(T&& t); //不是通用引用，是右值引用
};


template <class T, class Allocator = allocator<T> >
class vector
{
public:
    void push_back(T&& x); //不是通用引用，是右值引用 
};
```

注意，这里也不存在类型推导：Bar()和push_back()不可能脱离它们的class而独立存在，然而，当它们的class产生的时候(也即是，类模板的实例化的时候)，类型参数已经确定了。所以，调用Bar()和push_back()的时候，不存在类型推导。

而下面这种情况，却是有类型推导的(调用Bar()和emplace_back()的时候，需要推导类型)，是通用引用；

```cpp
template<typename T>
class Bar
{
  private:
    T _m;
  public:
    template<typename S>
    Bar(S&& s);   // s是通用引用
};

template <class T, class Allocator = allocator<T> >
class vector
{
public:
    template <class... Args>
    void emplace_back(Args&&... args);  // args中的每一个参数都是是通用引用
};
```

例3：推导的必须是"T&&"中的T的类型
```cpp
template<typename S>
void func(std::vector<S>&& p);
```

虽然这里存在类型推导，但推导是std::vector的类型参数，而不是"T&&"中的"T"。实际上，p的类型是已知的std::vector<>。

# 通用引用类型的变量(3)

一个通用引用类型的变量，**不管它是通过左值初始化的，还是通过右值初始化的，它本身是一个左值**。这类似于**右值引用类型的变量(有名字)是左值**，见[C++11中的右值引用](http://www.yuanguohuo.com/2018/05/25/cpp11-rvalue-ref/)。

如果它引用的是左值(通过左值初始化的)，我们当然不想把它当做右值处理；如果它引用的是右值(通过右值初始化的)，我们通常要利用它的右值的特点的(否则，赋给它右值干什么呢？)。你拿到一个通用引用类型的左值，它引用的可能是左值，也可能是右值，你不清楚(例如，你在写一个摸板函数，它的参数是通用引用，你当然不知道调用者传进左值还是右值)，但你还想保留它左值或右值特性，怎么办呢？见[C++11中的完美转发]()。

# 引用折叠(4)

## 引用的引用是非法的(4.1)

首先，必须说明，代码中是不能有"引用的引用"这样的类型的，例如：

```cpp
int f = 3;

int & & r1 = f;   //error: cannot declare reference to ‘int&’, which is not a typedef or a template type argument
int & && r2 = f;  //error: cannot declare reference to ‘int&’, which is not a typedef or a template type argument
int && & r3 = f;  //error: cannot declare reference to ‘int&&’, which is not a typedef or a template type argument
int && && r4 = f; //error: cannot declare reference to ‘int&&’, which is not a typedef or a template type argument
```

编译错误指出，不能声明"int&"或者"int&&"的引用(即"引用的引用")。但，后半句又说，它不是一个typedef或者模板类型参数。实际上，在**编译过程中**，typedef或者模板类型参数，可能产生"引用的引用"这种情况。例如：

```cpp
template<typename T>
void func(T&& p);
```

根据类型推导规则：
* 若p匹配的是右值，那么T被推导成那个右值的类型；例如，func(3); T是int；
* 若p匹配的是左值，那么T被推导成那个左值的引用类型；例如，int i=3; func(i); T是int&；

也就是说：
* 对于func(3); T的推导结果是int，模板实例化的结果是void func(int&& p);
* 对于int i=3; func(i); T的推导结果是int&，模板实例化的结果是void func(int& && p);

对于后一种情况，实例化的代码是非法的(存在"引用的引用"), 编译器会对它进行引用折叠(reference collapsing)。

## 引用折叠规则(4.2)

左值引用和右值引用可以组合出四种"引用的引用"：
1. 左值引用的左值引用；
2. 左值引用的右值引用；
3. 右值引用的左值引用；
4. 右值引用的左值引用；

折叠规则比较简单：
1. 右值引用的右值引用折叠成(collapses into)右值引用；
2. 其他情况折叠成(collapses into)左值引用；

通过这个规则，上例中，模板实例化得到的void func(int& && p)被折叠成void func(int& p)。

也就是说：
* 对于func(3); T的推导结果是int，模板实例化的结果是void func(int&& p); 无须折叠；通用引用p最终等价于右值引用；
* 对于int i=3; func(i); T的推导结果是int&，模板实例化的结果是void func(int& && p); 折叠结果是void func(int& p)；通用引用p最终等价于左值引用；

## 注意引用剥除(4.3)

其实，引用剥除和引用折叠是一毛钱关系没有的。在这里提它，纯粹是为了消除直观上的混淆。

在[C++11中的auto关键字](http://www.yuanguohuo.com/2018/05/25/cpp11-auto/)中，我们已经见到过引用剥除(reference stripping)，auto类型推导的时候，需要进行引用剥除，模板类型参数推导也需要。

```cpp
template<typename T>
void func(T&& p);

int&& r1 = 3;  
int x = 3;
int& r2 = x; 

func(r1);
func(r2);
```

1. 在func(r1)中，p的类型是"int&& &&"，然后折叠成"int&&"吗？

不是。在推导类型T的时候，r1的引用是被剥除的。又因为r1是一个左值(说过很多次了，见[C++11中的右值引用](http://www.yuanguohuo.com/2018/05/25/cpp11-rvalue-ref/))，所以T的推导结果是int&。所以，模板实例化的结果是void func(int& && p); 折叠结果是void func(int& p)。

2. 在func(r2)中，p的类型是"int& &&"，然后折叠成"int&"吗？

是的。但是，这和"int& r2=x"中的"&"无关，因为它也被剥除了。T被推导成"int&"完全是因为r2是左值。

可见，由于引用剥除的缘故，r1和r2声明中的"&&"和"&"起不到任何作用。不注意的话，可能认为类型折叠和它们有关。
