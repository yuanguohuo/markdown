---
title: C++11中的decltype关键字
date: 2018-05-26 16:55:06
tags: [decltype]
categories: c++
---

本文前两节翻译自[这篇文章](http://thbecker.net/articles/auto_and_decltype/section_06.html)和[这篇文章](http://thbecker.net/articles/auto_and_decltype/section_07.html)。

<!-- more -->

# 类型推导 (1)

decltype的类型推导和auto([C++11中的auto关键字](http://www.yuanguohuo.com/2018/05/25/cpp11-auto/))不同。为了方便说明类型如何推导，先做如下定义：

1. 简单表达式：不带括号的变量，函数参数，类成员；例如，x，x.m_data，px->m_data等；
2. 复杂表达式：非简单表达式；例如，`(x)`，`(x.m_data)`，`(px->m_data)`，`x*y`，`func()`等等；

## 对于简单表达式 (1.1)

`expr`是一个简单表达式，那么`decltype(expr)`的类型是：源代码中`expr`的声明类型(当然，typedef展开、模板类型参数实例化，还是需要的)；

```cpp
struct S
{
  int m_x;

  S()
  {
    m_x = 42;
  }
};

int x;
const int cx = 42;
const int& crx = x;
const S* p = new S();

typedef decltype(x) x_type;  //x_type是"int"
auto a = x;                  //a的类型是"int"

typedef decltype(cx) cx_type; //cx_type是"const int"
auto b = cx;                  //b的类型是"int"

typedef decltype(crx) crx_type; //crx_type是"const int&"
auto c = crx;                   //c的类型是"int"

typedef decltype(p->m_x) m_x_type; //虽然p->m_x是const的(因为p是const的)，但m_x_type是"int"(因为m_x的声明类型是"int"，没const) 
auto d = p->m_x;                   //d的类型是"int"
```

## 对于复杂表达式 (1.2)

先搞明白lvalue，xvalue和prvalue，见[C++11中的值的类型](http://www.yuanguohuo.com/2018/05/24/cpp11-value-types/)。然后，看规则：

`expr`是一个复杂表达式，且其类型为`T`：
* 若`expr`是prvalue，则`decltype(expr)`是`T`；
* 若`expr`是lvalue，则`decltype(expr)`是`T&`；
* 若`expr`是xvalue，则`decltype(expr)`是`T&&`；

**例1：**把第1.1节中的简单表达式加上括号，变成复杂表达式

```cpp
struct S
{
  int m_x;

  S()
  {
    m_x = 42;
  }
};

int x;
const int cx = 42;
const int& crx = x;
const S* p = new S();

typedef decltype((x)) x_with_parens_type;  //x_with_parens_type是"int&"；x类型为"int"，是lvalue(故添加"&")；
typedef decltype(x) x_type;                //x_type是"int"
auto a_p = (x);                            //a_p的类型是"int"
auto a = x;                                //a的类型是"int"

typedef decltype((cx)) cx_with_parens_type;  //cx_with_parens_type是"const int&"；cx类型为"const int"，是lvalue(故添加"&")；
typedef decltype(cx) cx_type;                //cx_type是"const int"
auto b_p = (cx);                             //b_p的类型是"int"
auto b = cx;                                 //b的类型是"int"


typedef decltype((crx)) crx_with_parens_type; //crx_with_parens_type是"const int&"；crx类型为"const int&"，是lvalue(故再添一个"&"，然后引用折叠)
typedef decltype(crx) crx_type;               //crx_type是"const int&"
auto c_p = (crx);                             //c_p的类型是"int"
auto c = crx;                                 //c的类型是"int"

typedef decltype((p->m_x)) m_x_with_parens_type; //m_x_with_parens_type是"const int&"；p->m_x类型为"const int"，是lvalue(故添加"&")
typedef decltype(p->m_x) m_x_type;               //虽然p->m_x是const的(因为p是const的)，但m_x_type是"int"(因为m_x的声明类型是"int"，没const)
auto d_p = (p->m_x);                             //d_p的类型是"int"
auto d = p->m_x;                                 //d的类型是"int"
```

**例2：**更复杂的情形

```cpp
const S foo();
const int& foobar();
std::vector<int> vect = {42, 43};

typedef decltype(foo()) foo_type; //foo_type是"const S"；foo()的类型为"const S"，是prvalue(故不加"&")
auto a = foo();                   //a的类型是"S"，const被剥除

typedef decltype(foobar()) foobar_type; //foobar_type是"const int&"；foobar()的类型是"const int&"，是lvalue(故再添一个"&"，然后引用折叠)
auto b = foobar();                      //b的类型是"int"，const和"&"被剥除；

typedef decltype(vect.begin()) iterator_type; //iterator_type是"vector<int>::iterator"；vect.begin()的类型为"vector<int>::iterator"，是prvalue(故不加"&")
auto iter = vect.begin();                     //iter的类型是"vector<int>::iterator"

decltype(vect[0]) first_element = vect[0]; //类型是"int&"；operator[]返回类型为"int&"，是lvalue(故再添一个"&"，然后引用折叠)
auto second_element = vect[1];  //类型是"int"，因为"&"被剥除；
```

注意：
1. 最后的`first_element`是vect[0]的引用；而`second_element`是vect[1]的拷贝；
2. 这里提到的引用折叠，见[C++11中的通用引用](http://www.yuanguohuo.com/2018/05/25/cpp11-universal-ref/)；const和&剥除，见[C++11中的auto关键字](http://www.yuanguohuo.com/2018/05/25/cpp11-auto/)；

**例3：**二元和三元运算

```cpp
int x = 0;
int y = 0;
const int cx = 42;
const int cy = 43;
double d1 = 3.14;
double d2 = 2.72;

typedef decltype(x * y) prod_xy_type; //prod_xy_type是"int"；乘积的类型为"int"，且为prvalue(故不加"&")
auto a = x * y;                       //a的类型是"int"

typedef decltype(cx * cy) prod_cxcy_type; //prod_cxcy_type是"int"；乘积的类型为"int"(注意不是const int)，且为prvalue(故不加"&")
auto b = cx * cy;                         //b的类型是"int"

typedef decltype(d1 < d2 ? d1 : d2) cond_type; //cond_type是"double&"；运算结果类型是"double"，且为lvalue(故添加"&")
auto c = d1 < d2 ? d1 : d2;                    //c的类型是"double"

//注意：x < d2 ? x : d2的结果是prvalue，因为这个表达式可能返回x，也可能返回d2，它们是不同的类型，为此
//编译器分配一个临时变量(double)来存结果；若返回x，x被升级为double；
typedef decltype(x < d2 ? x : d2) cond_type_mixed; //cond_type_mixed是"double"；运算结果类型是"double"，且为prvalue(故不加"&")
auto d = x < d2 ? x : d2;                          //d的类型是"double"
```

注意：对于`x < y ? x : y`：
* 若x和y类型相同，它是左值表达式(返回x或y的引用，lvalue)；
* 若x和y类型不同，它是右值表达式(返回临时变量，prvalue)；

这很危险啊！看这么个例子：由于`std::min`要求两个参数类型必须相同，我们要写一个`fmin()`，能够从不同类型的数字(`int`，`double`等)中取最小值。

```cpp
//失败的尝试
template<typename T, typename S>
auto fpmin(T x, S y) -> decltype(x < y ? x : y)
{
  return x < y ? x : y;
}
```

暂不提`auto -> decltype(...)`这种语法，下文会介绍。为什么这是个失败的尝试呢？

因为返回类型`decltype(x < y ? x : y)`可能是引用(T和S相同的时候)，也可能不是(T和S不同的时候)。而若是前一种情况(返回引用的时候)，大错就铸成了：因为返回的是局部变量(函数参数x或y)的引用。玩C++的人都知道，这是很幼稚的错误。然而，在这里它却如此的隐晦。正确的版本是：

```cpp
//正确的版本
template<typename T, typename S>
auto fpmin(T x, S y) -> typename std::remove_reference<decltype(x < y ? x : y)>::type
{
  return x < y ? x : y;
}
```

# decltype的性质 (2)

## 类型推导的时候不求值 (2.1)

在推导`decltype(expr)`的类型的时候，不会对`expr`进行求值(难道类型推导不是编译的时候做的吗？编译的时候肯定不会求值啊)。例如，这段代码没有问题：

```cpp
std::vector<int> vect;
assert(vect.empty());
typedef decltype(vect[0]) integer; //vect是空的，vect[0]超界，但没问题；
```


# 使用场景 (3)

## 基础用法 (3.1)

在[C++11中的auto关键字](http://www.yuanguohuo.com/2018/05/25/cpp11-auto/)中，我们看到这样一个例子：

```cpp
template<typename T, typename S>
void foo(T lhs, S rhs)
{
  auto prod = lhs * rhs;
  //...
}
```

假如不是定义`lhs * rhs`类型的变量，而是就要那个类型，怎么办呢？C++11之前没有标准的办法，一些编译器(例如linux使用的g++)有扩展的关键字`typeof`帮我们做到：

```cpp
template<typename T, typename S>
void foo(T lhs, S rhs)
{
  typedef typeof(lhs * rhs) product_type;
}
```

由于`typeof`关键字是编译器的扩展，换个编译器，这段代码可能就不能编译了。现在C++11为我们提供了通用的办法(为了区别于`typeof`，它使用了`decltype`)：

```cpp
template<typename T, typename S>
void foo(T lhs, S rhs)
{
  typedef decltype(lhs * rhs) product_type;
}
```

## Trailing return type (3.2)

现在，考虑另一种情况：假如`foo`的返回的是`lhs * rhs`的类型呢？这样试试：

```cpp
template<typename T, typename S>
decltype(lhs * rhs) foo(T lhs, S rhs)
{
  return lhs * rhs;
}
```

可是它无法通过编译，因为在`decltype(lhs * rhs)`中，`lhs`和`rhs`是没有声明的。C++11引进了一种叫"trailing return type"的语法：

```cpp
template<typename T, typename S>
auto foo(T lhs, S rhs) -> decltype(lhs * rhs) 
{
  return lhs * rhs;
}
```

注意：

1. 在这种语法中，类型推导完全按照decltype的规则进行(见第1节)，和auto的推导规则无关；
2. 和[C++11中的auto关键字](http://www.yuanguohuo.com/2018/05/25/cpp11-auto/)中提到的"Function return type deduction"进行区别：函数声明中，一个有`->decltype(...)`，一个没有；

## Alternate type deduction on declaration in C++14 (3.3)

## Type deduction for lambda arguments in C++14 (3.4)
