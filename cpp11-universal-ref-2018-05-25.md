---
title: C++11中的通用引用
date: 2018-05-25 12:03:23
tags: [通用引用, universal reference, 引用折叠, reference collapsing, 引用剥除, reference stripping]
categories: c++
---

*通用引用*在语法上很容易和右值引用混淆，所以本文介绍了构成通用引用条件。然后，着重介绍类型推导和引用折叠(reference collapsing)是如何演绎出*通用引用*的。

<!-- more -->

# 左值右值都可引用 (1)

熟悉右值引用的话，就会知道它是通过`&&`来声明的。可能看见`&&`的第一反应就是右值引用。可惜，有些情况下竟然不是。在[C++11中的右值引用](http://www.yuanguohuo.com/2018/05/25/cpp11-rvalue-ref/)一篇中，我们看到了`std::move`的源代码：

```cpp
template <typename T>
typename remove_reference<T>::type&& move(T&& t) noexcept
{
  return static_cast<typename remove_reference<T>::type&&>(t);
}
```

这里`std::move`的参数`t`，就不是右值引用。若是右值引用的话，你怎么能传一个左值给它当参数呢？它的作用可是把左值转换成右值啊。所以，`t`必须能匹配左值。另一方面，这样的代码也不会出错：

```cpp
Foo f("123");
Foo&& r1 = std::move(std::move(f));

Foo&& r2 = std::move(Foo("456"));
```

也就是说，`t`也能够匹配右值。实际上，参数`t`就是一个**通用引用**。**通用引用，可以使用左值表达式初始化，也可以使用右值表达式初始化。匹配左值表达式时，它表现的像一个左值引用；匹配右值表达式时，它表现的像一个右值引用**。

# 构成通用引用的条件 (2)

出现`&&`的时候，如何判断它是右值引用，还是通用引用呢？通用引用有两个条件：

1. 形如`T&&`；
2. T的类型需要推导；

先看一些例子：

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

这里`r3`是右值引用，因为没有类型推导；相反，`r1, r2, r4, r5`和`p`都是通用引用：
* `r1`**等价于**左值引用；
* `r2`**等价于**右值引用；
* `r4`**等价于**左值引用；没错，是左值引用，因为r3是左值(虽然它是右值引用类型的变量)；
* `r5`**等价于**左值引用；因为`std::vector<int>::operator[]`返回左值引用(左值引用是左值)；
* `func(lvalue)`中，`p`**等价于**左值引用；
* `func(10)`中，`p`**等价于**右值引用；

也就是说，形如`T&&`的类型中，若`T`是已知类型，则是右值引用；若`T`需要推导，则是通用引用。这两个条件看起来不难，但实际上还是比看起来要苛刻：

**例1：必须形如`T&&`**

```cpp
int lvalue = 3;

const auto&& r = lvalue; //错误：r不是通用引用，而是右值引用，故不能通过左值初始化！

template<typename T>
void func(const T&& p);

func(lvalue);            //错误：p不是通用引用，而是右值引用，故不能通过左值初始化！
```

注：必须是`T&&`，`const T&&`都不行。当然，`T`可以是`auto`或者别的类型参数名；

**例2：必须存在类型推导**

```cpp
template<typename T>
class Bar
{
  private:
    T _m;
  public:
    Bar(Bar&& other); //不是通用引用，而是右值引用
};
```

虽然是类模板，但没有类型推导，`other`的类型就是`Bar&&`，所以也不是通用引用。甚至下面的也不是：

```cpp
template<typename T>
class Bar
{
  private:
    T _m;
  public:
    Bar(T&& t); //不是通用引用，而是右值引用
};

template <class T, class Allocator = allocator<T> >
class vector
{
public:
    void push_back(T&& x); //不是通用引用，而是右值引用 
};
```

这里也不存在类型推导：`Bar()`和`push_back()`不可能脱离它们的class而独立存在，然而，当它们的class产生的时候(也即是，类模板的实例化的时候)，类型参数已经确定了。所以，调用`Bar()`和`push_back()`的时候，不存在类型推导。

而下面这种情况，却是通用引用，因为调用`Bar()`和`emplace_back()`的时候，需要推导类型：

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

**例3：推导的必须是`T&&`中的T的类型**
```cpp
template<typename S>
void func(std::vector<S>&& p);
```

虽然这里存在类型推导，但推导是std::vector的类型参数，而不是`T&&`中的`T`(实际上，p的类型是已知的`std::vector`)，所以，`p`也不是通用引用。

# 通用引用类型的变量 (3)

一个通用引用类型的变量，**不管它是通过左值初始化的，还是通过右值初始化的，它本身是一个左值**。这类似于**右值引用类型的变量是左值(因为有名字)**，见[C++11中的右值引用](http://www.yuanguohuo.com/2018/05/25/cpp11-rvalue-ref/)。

如果它引用的是左值(通过左值初始化的)，我们当然想把它当作左值处理；相反，如果它引用的是右值(通过右值初始化的)，我们又想保留它右值的特征(例如，能作为移动拷贝的参数)。你拿到一个通用引用类型的左值，它引用的可能是左值，也可能是右值，你不清楚(例如，你在写一个摸板函数，它的参数是通用引用，你当然不知道调用者传进左值还是右值)，但你还想保留它原有特征(左值或右值)，怎么办呢？`std::forward`可以帮你，见[C++11中的完美转发](http://www.yuanguohuo.com/2018/05/25/cpp11-perfect-forward/)。

# 类型推导和引用折叠 (4)

前面我们看到，通用引用在引用左值时变成左值引用，在引用右值时变成右值引用。这种奇特的效果是如何产生的呢？

1. 编译器允许它匹配左值和右值；
2. 类型推导和引用折叠(reference collapsing)最终让它变成左值引用或者右值引用；

## 类型推导规则 (4.1)

首先，通用引用`auto&&`和`T&&`中的类型推导规则是这样的：

* 若匹配的是左值lv，`auto`或`T`被推导成lv的引用类型；例如匹配`i`时(`int i=3;`)，推导结果是`int&`；
* 若匹配的是右值rv，`auto`或`T`被推导成rv的类型；例如匹配3时，推导结果是`int`；

## 引用的引用是非法的 (4.2)

其次，必须说明，*引用的引用*是非法的，例如：

```cpp
int f = 3;

int & & r1 = f;   //error: cannot declare reference to ‘int&’, which is not a typedef or a template type argument
int & && r2 = f;  //error: cannot declare reference to ‘int&’, which is not a typedef or a template type argument
int && & r3 = f;  //error: cannot declare reference to ‘int&&’, which is not a typedef or a template type argument
int && && r4 = f; //error: cannot declare reference to ‘int&&’, which is not a typedef or a template type argument
```

编译错误指出，不能声明`int&`或者`int&&`的引用(即*引用的引用*)。

## 编译过程中产生*引用的引用* (4.3)

由于第4.1节的类型推导规则(下面例1和例2)，模板类型参数实例化(下面例3)和typedef展开(下面例4)等原因，**在编译的过程中**，可能产生*引用的引用*这种情况：

**例1：**

```cpp
template<typename T>
void func(T&& p);

func(3);

int i=3;
func(i);
```

根据类型推导规则：
* 对于`func(3)`，`T`的推导结果是`int`，模板实例化的结果是`void func(int&& p)`；
* 对于`func(i)`，`T`的推导结果是`int&`，模板实例化的结果是`void func(int& && p)`；

前一种情况，通用引用变成右值引用；后一种情况，模板实例化的过程中产生了*引用的引用*。

**例2：**

```cpp
auto&& r1 = 3;

int i = 3;
auto&& r2 = i;
```

根据类型推导规则：
* `r1`引用的是右值，所以`auto`的推导结果是`int`，`r1`的类型是`int&&`；
* `r2`引用的是左值，所以`auto`的推导结果是`int&`，`r2`的类型是`int& &&`；

通用引用`r1`变成右值引用；`r2`的类型推导过程中出现了*引用的引用*。

**例3：**

```cpp
template<typename T>
class Bar {
  public:
    typedef T& LVRefType;
};
```

使用引用类型(左值引用或右值引用)去实例化Bar：

```cpp
int i=1;
int j=2;

Bar<int&>::LVRefType ri = i;
Bar<int&&>::LVRefType rj = j;
```

这样，模板类实例化过程中也产生*引用的引用*：

```cpp
typedef int& & LVRefType;   //Bar<int&>
typedef int&& & LVRefType;  //Bar<int&&>
```

**例4：**

```cpp
void func(Bar<int>::LVRefType&& p);
void func(Bar<int&>::LVRefType&& p);
void func(Bar<int&&>::LVRefType&& p);
```
因为`Bar<int>::LVRefType`，`Bar<int&>::LVRefType`和`Bar<int&&>::LVRefType`都等价于`int&`，所以，typedef展开得到的结果都是：

```cpp
void func(int& && p);
```

即，产生了*引用的引用*。

第4.2节说了，*引用的引用*是非法的。那我们这些例子中都存在*引用的引用*怎么办呢？好在，这些*引用的引用*不是存在于源代码中(否则编译失败)，而是在编译过程中临时产生的。编译器会立即消除它们，手段就是**引用折叠(reference collapsing)**。

## 引用折叠规则 (4.4)

左值引用和右值引用可以组合出四种*引用的引用*：

1. 左值引用的左值引用；
2. 左值引用的右值引用；
3. 右值引用的左值引用；
4. 右值引用的右值引用；

折叠(collapsing rules)规则比较简单：

1. 右值引用的右值引用折叠成(collapses into)右值引用；
2. 其他情况折叠成(collapses into)左值引用；

基于折叠规则，看看上面的四个例子：

* 例1：对于`func(i)`，T的推导结果是`int&`，模板实例化过程中产生`void func(int& && p)`，最终折叠成`void func(int& p)`，也就是说，通用引用`p`最终变成左值引用；
* 例2：r2引用的是左值，所以`auto`的推导结果是`int&`，`r2`的类型展开成`int& &&`，最终折叠成`int&`，也就是说，通用引用`r2`最终变成左值引用；
* 例3：`int& &`和`int&& &`都折叠成`int&`，LVRefType最终变成`int&`；这么定义是正确的：无论取代`T`的是不是引用，是左值引用还是右值引用，`LVRefType`最终都是左值引用类型，和它的名字相符；
* 例4：`int& && p`被折叠成`int& p`；

可见，**类型推导和引用折叠在通用引用机制中起着重要作用**：编译器允许`T&&`匹配左值和右值，然后经过类型推导和(或)引用折叠，`T&&`才最终变成左值引用或者右值引用。

## std::move的工作原理 (4.5)

回头看`std::move`的实现，你发现，它的工作原理不过是类型推导和引用折叠的另一个例子：

```cpp
template <typename T>
typename remove_reference<T>::type&& move(T&& t) noexcept
{
  return static_cast<typename remove_reference<T>::type&&>(t);
}
```

**move左值：**

```cpp
Foo f;
std::move(f);
```

根据类型推导规则(第4.1节)，`std::move`的类型参数`T`推导为`Foo&`，`std::move`展开为：

```cpp
typename remove_reference<Foo&>::type&& move(Foo& && t) noexcept
{
  return static_cast<typename remove_reference<Foo&>::type&&>(t);
}
```

经过remove_reference和引用折叠：

```cpp
Foo&& move(Foo& t) noexcept
{
  return static_cast<Foo&&>(t);
}
```

可见：左值经过std::move()变为右值；

**move右值：**

```cpp
std::move(Foo("xyz"));
```

根据类型推导规则(第4.1节)，`std::move`的类型参数`T`推导为`Foo`，`std::move`展开为：

```cpp
typename remove_reference<Foo>::type&& move(Foo&& t) noexcept
{
  return static_cast<typename remove_reference<Foo>::type&&>(t);
}
```

经过remove_reference：

```cpp
Foo&& move(Foo&& t) noexcept
{
  return static_cast<Foo&&>(t);
}
```

可见：右值经过std::move()还是右值；

## 引用剥除 (4.6)

其实，引用剥除(reference stripping)和引用折叠(reference collapsing)没有一毛钱关系。之所以在这里提它，纯粹是为了消除直观上的混淆。

在[C++11中的auto关键字](http://www.yuanguohuo.com/2018/05/25/cpp11-auto/)中，我们已经见到过引用剥除，auto类型推导的时候，需要进行引用剥除，模板类型参数推导也需要。

```cpp
template<typename T>
void func(T&& p);

int&& r1 = 3;

int x = 3;
int& r2 = x; 

func(r1);
func(r2);
```

1. 问题：在`func(r1)`中`p`的类型是`int&& &&`，然后折叠成`int&&`，对吗？答：不对。在推导类型T的时候，`r1`的右值引用是被剥除的(`int&& r1=3`中的`&&`被剥除)。又因为`r1`是一个左值(说过很多次了，右值引用类型的变量有名字，因此是左值，见[C++11中的右值引用](http://www.yuanguohuo.com/2018/05/25/cpp11-rvalue-ref/))，所以`T`推导为`int&`。因此，模板实例化为`void func(int& && p)`，最终折叠成`void func(int& p)`。
2. 问题：在`func(r2)`中，`p`的类型是`int& &&`，然后折叠成`int&`，对吗？答：对，但是，这和`int& r2=x`中的`&`无关，因为它也被剥除了。`T`被推导成`int&`完全是因为`r2`是左值的缘故。

可见，由于引用剥除的缘故，`r1`和`r2`声明中的`&&`和`&`起不到任何作用；模板参数类型的推导，完全取决于函数实参是左值还是右值。

# 小结 (5)

通用引用既能引用左值又能引用右值。实质上，是类型推导和引用折叠(reference collapsing)导致这种现象产生的。我们知道，const类型的左值引用(`const T&`)也能够同时引用左值和右值，那通用引用的优势是什么呢？见[C++11中的完美转发](http://www.yuanguohuo.com/2018/05/25/cpp11-perfect-forward/)。
