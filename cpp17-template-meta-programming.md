---
title: C++使用模版进行元编程
date: 2018-06-08 20:15:30
tags: [template]
categories: c++
---

初步尝试C++模版元编程。元编程考虑的是编译时的逻辑，和运行时不同，常常觉得违法直觉。

<!-- more -->

<script type="text/x-mathjax-config">
MathJax.Hub.Config({
tex2jax: {inlineMath: [['$','$'], ['\\(','\\)']]}
});
</script>

<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>


# 查找一个类型T (1)

假设有一个类型列表：`char`, `short`, `int`, `float`, `double`；要找`float`在这个序列中的位置(从0开始，结果应该是3)：

## 初步尝试 (1.1)

```cpp
#include <iostream>
#include <utility>
#include <string_view>

template <class Needle, class... Ts>
constexpr size_t Find(Needle, Needle, Ts...) {
  return 0;
}

template <class Needle, class T, class... Ts>
constexpr size_t Find(Needle, T, Ts...) {
  return 1 + Find(Needle(), Ts()...);
}

int main()
{
  float  target = 100.0;
  char   c = 'a';
  short  s = 1; 
  int    i = 2;
  float  f = 3.0;
  double d = 4.0;

  std::cout << Find(target, c, s, i, f, d) << std::endl;

  return 0;
}
```

编译运行（本文所有例子只在linux/gcc环境测试过），果然输出3！原理是什么呢？

模版函数Find其实是一个递归，不同的是，**这个递归是在编译时运行的**！

- `Find(Needle, Needle, Ts...)`是递归出口：它的特点是前2个参数类型相同！只要满足这个条件，就递归结束，返回0；注意，参数的值根本没有用，只考虑参数的类型：`Needle`, `Needle`, `Ts...`；
- `Find(Needle, T, Ts...)`是递归中间过程：调用时执行，不，确切的说，是**编译调用语句时执行**；

例如，编译器编译`main()`函数中的语句`Find(target, c, s, i, f, d)`：

- 因为`target`和`c`类型不同，所以匹配`Find(Needle, T, Ts...)`：
    - `Needle=float`, `T=char`, `...Ts={short, int, float, double}`；
    - **编译器生成一个函数**`Find(float, char, short, int, float, double)`；
    - 递归：把`T=char`略去，`1 + Find(float变量，short变量，int变量，float变量，double变量)`；注意这里函数参数都是临时变量，原来的`target`, `s`, `i`, `f`, `d`都丢了：**它们本来就一点用也没有，还会带来问题，后面再填这个坑**！
- 再看`Find(float变量，short变量，int变量，float变量，double变量)`，因为前两个参数类型还是不同，继续匹配`Find(Needle, T, Ts...)`：
    - `Needle=float`, `T=short`, `...Ts={int, float, double}`；
    - **编译器生成函数**`Find(float, short, int, float, double)`；
    - 递归：把`T=short`略去，`1 + 1 + Find(float变量，int变量，float变量，double变量)`；
- 继续`Find(float变量，int变量，float变量，double变量)`，还是匹配`Find(Needle, T, Ts...)`：
    - `Needle=float`, `T=int`, `...Ts={float, double}`；
    - **编译器生成函数**`Find(float, int, float, double)`；
    - 递归：把`T=int`略去，`1 + 1 + 1 + Find(float变量，float变量，double变量)`；
- 最后`Find(float变量，float变量，double变量)`的前2个参数类型一样，匹配递归出口`Find(Needle, Needle, Ts...)`：
    - `Needle=float`, `Needle=float`, `...Ts={double}`
    - **编译器生成函数**`Find(float, float, double)`；
    - 它返回0；

所以，结果是3。看起来很不错！


## 变量值的问题 (1.2)

前面说过，`target`, `s`, `i`, `f`, `d`这些变量会带来问题。什么问题呢？

```cpp

// ...

class Foo
{
  public:
    Foo(int) {}
};

int main()
{
  Foo    target(1);
  char   c = 'a';
  short  s = 1; 
  int    i = 2;
  Foo    f(2);
  double d = 4.0;

  std::cout << Find(target, c, s, i, f, d) << std::endl;

  return 0;
}
```

编译失败：error: no matching function for call to Foo::Foo()！显而易见：`Foo`没有默认构造函数，`Find()`递归过程中（**编译器编译过程中**），`Needle()`找不到合适的构造函数！

其实还隐藏一个问题：假如`Foo`有默认构造函数但很重，例如分配大量内存，打开文件，甚至建立socket连接，就会导致严重的性能问题！前面也说过，那些变量值一点用也没有。

怎么办呢？其实只需要类型，想办法**只传递类型就可以了**！

```cpp

// ...

template <class T>
struct Type {
  using type = T;
};

int main()
{
  std::cout << Find(Type<Foo>(),
                    Type<char>(),
                    Type<short>(),
                    Type<int>(),
                    Type<Foo>(),
                    Type<double>()) << std::endl;
  return 0;
}
```

根据模版的特性, `Type<Foo>`, `Type<char>`, `Type<short>`等都是不同的struct/class！这些class都没有任何资源，且都有默认构造函数（没有任何构造函数的话，编译器会提供默认版本），所以完美的解决了传值的问题。本质上，是传`Type<T>`类型的值，但值没有用，有用的是类型！一言以蔽之：`Type`**用于包装类型，便于传递类型信息**！

## 完善 (1.3)

为了学习更多模版元编程的知识，加入一点优化：假如列表中不包含target类型或者包含多个，编译报错！

```cpp

// ...

template <class T, class... Ts>
using Contains = std::disjunction<std::is_same<T, Ts>...>;

template <class Needle, class... Ts>
constexpr size_t Find(Needle, Needle, Ts...) {
  //确保：匹配之后不会再有相同类型！
  static_assert(!Contains<Needle, Ts...>(), "Duplicate element type");
  return 0;
}

// ...

int main()
{
  //编译报错：static assertion failed: Type not found
  /*
  static_assert(Contains<Type<Foo>,
                         Type<char>,
                         Type<short>,
                         Type<int>,
                         Type<double>>(),
                  "Type not found");
  */

  static_assert(Contains<Type<Foo>,
                         Type<char>,
                         Type<short>,
                         Type<int>,
                         Type<Foo>,
                         Type<double>>(),
                  "Type not found");

  // ...

  return 0;
}
```

重点是`Contains`是如何实现的！显然它的作用是检查类型`T`是否包含在类型列表`Ts...`中。

首先是`template<class T, class U> std::is_same {}`，当`T`和`U`是相同类型时，`std::is_same<T,U>::value`为`true`，否则为`false`！这么表达其实也不准确，准确地说是，**编译器生成了一堆这样的class**：

```cpp
struct false_type {
    static constexpr bool value = false;
};

struct true_type {
    static constexpr bool value = true;
};

class is_same<Foo, char> : public false_type {};

class is_same<Foo, Foo> : public true_type {};

// ...
```

单词disjunction的意思是**析取、逻辑或**，与它对应的是conjunction，**合取、逻辑与**。

这对模版是C++17引入的，这里只看`template<class... B> struct disjunction`（conjunction类似）。说白了`std::disjunction`就是执行逻辑OR运算：从左到右依次检查每个类型，若遇到`true_type`就立即返回它；若最终也没遇到则返回`false_type`。

显然：`Contains<T, Ts...>`就是`std::disjunction<is_same<T, T1>, is_same<T, T2>, ..., is_same<T, Tn>>`（假设`Ts={T1, T2, ..., Tn}`）；其中有一个为true就代表包含！


# integer_sequence (2)

## 编译时整数序列 (2.1)

模版std::integer_sequence是C++14标准库中引入的一个工具，用于在**编译时**生成整数序列。它位于<utility>头文件中，主要用于模板元编程和处理可变参数包（variadic templates）。

```cpp
template<typename T, T... idx>
struct integer_sequence
{
  typedef T value_type;
  static constexpr size_t size() noexcept { return sizeof...(idx); }
};
```

顾名思义，它是编译时生成的**整数序列**，非整数类型如`std::integer_sequence<float, 1.0, 2.0, 3.0>`会编译失败！

```cpp
#include <iostream>
#include <utility>
#include <string_view>

int main()
{
  //编译失败：error: ‘float’ is not a valid type for a template non-type parameter
  //using FloatSeq = std::integer_sequence<float, 1.0, 2.0, 3.0>;
  //std::cout << FloatSeq::size() << std::endl;

  //打印 "c : 3"
  using CharSeq = std::integer_sequence<char, 'c', 'y', 'x'>;
  std::cout << typeid(CharSeq::value_type).name() << " : " << CharSeq::size() << std::endl;

  //打印 "s : 4"
  using Int16Seq = std::integer_sequence<int16_t, 9, 5, 2, 7>;
  std::cout << typeid(Int16Seq::value_type).name() << " : " << Int16Seq::size() << std::endl;

  //打印 "i : 5"
  using IntSeq = std::integer_sequence<int, 5, 8, 1, 1, 1>;
  std::cout << typeid(IntSeq::value_type).name() << " : " << IntSeq::size() << std::endl;

  return 0;
}
```


## 生成integer_sequence的实例类 (2.2)

生成实例类，更常见的使用方式是`make_integer_sequence`:

```cpp
//打印 "l : 6"
using LongSeq = std::make_integer_sequence<long int, 6>;
std::cout << typeid(LongSeq::value_type).name() << " : " << LongSeq::size() << std::endl;
```

注意：`make_integer_sequence`生成的是一个类型，不是一个对象！这个类型是`integer_sequence<long int, 0, 1, 2, 3, 4, 5>`；如何生成的呢？一般编译器有builtin实现，假如没有的化，[cppreference.com](https://en.cppreference.com/w/cpp/utility/integer_sequence)给了一个实现：

```cpp
template<class T, T I, T N, T... integers>
struct make_integer_sequence_helper
{
  using type = typename make_integer_sequence_helper<T, I + 1, N, integers..., I>::type;
};

template<class T, T N, T... integers>
struct make_integer_sequence_helper<T, N, N, integers...>
{
  using type = std::integer_sequence<T, integers...>;
};

template<class T, T N>
using my_make_integer_sequence = typename make_integer_sequence_helper<T, 0, N>::type;

int main()
{
  //打印 "s : 3"
  using ShortSeq = my_make_integer_sequence<short, 3>;
  std::cout << typeid(ShortSeq::value_type).name() << " : " << ShortSeq::size() << std::endl;

  return 0;
}

```

这又是一个递归，和第1节有点类似，不过这次编译器面对的不是模版函数，而是模板类（其实差不多）：

- `make_integer_sequence_helper`是主模版:
    - 类型参数：`T`，是一个整数类型(`char`, `short`, `int`等)；
    - 后面是一个`T`类型的整数值序列：`I`, `N`, `integers...` (模版参数可以为类型，也可以为值)；
    - 它有一个“类型成员”：`type`；这是递归的入口；

- `make_integer_sequence_helper<T, N, N, integers...>`是模板偏特化：
    - 同样，类型参数：`T`，是一个整数类型(`char`, `short`, `int`等)；
    - 但后面的整数值序列，**要前2个相等才匹配这个偏特化**！
    - 它的"类型成员"：`type`就是递归出口了；

看一下编译器如何编译`my_make_integer_sequence<short, 3>`，它展开是`make_integer_sequence_helper<short, 0, 3>::type`；

- 看`make_integer_sequence_helper<short, 0, 3>`，0和3不相等，匹配主模版：
    - `I=0`，`N=3`，`...integers={}`
    - 那么它的`type`就是`make_integer_sequence_helper<short, 1, 3, 0>::type`；
- 继续看`make_integer_sequence_helper<short, 1, 3, 0>`，1和3不相等，还是匹配主模板：
    - `I=1`，`N=3`，`...integers={0}`
    - 那么它的`type`就是`make_integer_sequence_helper<short, 2, 3, 0, 1>::type`；
- 继续看`make_integer_sequence_helper<short, 2, 3, 0, 1>`，2和3不相等，还是匹配主模板：
    - `I=2`，`N=3`，`...integers={0, 1}`
    - 那么它的`type`就是`make_integer_sequence_helper<short, 3, 3, 0, 1, 2>::type`；
- 最终`make_integer_sequence_helper<short, 3, 3, 0, 1, 2>`匹配偏特化，因为3和3相等：
    - `N=3`，`N=3`，`...integers={0, 1, 2}`
    - 所以，它的`type`就是`std::integer_sequence<short, 0, 1, 2>`


在这个过程中，编译器生成了4个中间类实例，假如它们分别是`A`, `B`, `C`, `D`，那么`A::type=B`，`B::type=C`，`C::type=D`；`D::type`才是最终我们要的`std::integer_sequence`!


## 提取整数值序列 (2.3)

有个有意思的问题：`std::integer_sequence`实例类中只有`value_type`和`size()`静态函数，并没有那个整数值序列（没有"ShortSeq::seq"这样的东西）！那编译器为什么还要一顿递归呢？何不直接生成一个class，其`value_type=short`且`static size()`返回3呢？

其实也不是，**编译器还是知道整数值序列的**，因为编译器真的生成了不同的class实例；换句话说：`std::integer_sequence<short, 1, 2, 3>`和`std::integer_sequence<short, 3, 2, 1>`是不同的class实例，虽然它们的`value_type`都是`short`且`size()`都返回3；这可以通过如下辅助模版函数来证实：

```cpp

// ...

template <typename T>
void print_type_arg()
{
  std::cout << __PRETTY_FUNCTION__ << std::endl;
}

int main()
{
  //打印：void print_type_arg() [with T = std::integer_sequence<short int, 1, 2, 3>]
  print_type_arg<std::integer_sequence<short, 1, 2, 3>>();

  //打印：void print_type_arg() [with T = std::integer_sequence<short int, 3, 2, 1>]
  print_type_arg<std::integer_sequence<short, 3, 2, 1>>();

  //打印：void print_type_arg() [with T = std::integer_sequence<int, 5, 8, 1, 1, 1>]
  using IntSeq = std::integer_sequence<int, 5, 8, 1, 1, 1>;
  print_type_arg<IntSeq>();

  return 0;
}
```

也可以利用编译器的推导能力来提取：

```cpp
template<typename T>
struct SeqExtractor;

template<short... args>
struct SeqExtractor<std::integer_sequence<short, args...>> {
  static void foo() {
    std::cout << __PRETTY_FUNCTION__ << std::endl;
    (std::cout << ... << args) << std::endl;
  }

  void bar() {
    std::cout << __PRETTY_FUNCTION__ << std::endl;
    (std::cout << ... << args) << std::endl;
  }
};

int main()
{
  using Extractor = SeqExtractor<std::integer_sequence<short, 9, 1, 1>>;
  Extractor extractor;

  //打印：
  //  static void SeqExtractor<std::integer_sequence<short int, args ...> >::foo() [with short int ...args = {9, 1, 1}]
  //  911
  Extractor::foo();

  //打印：
  //  void SeqExtractor<std::integer_sequence<short int, args ...> >::bar() [with short int ...args = {9, 1, 1}]
  //  911
  extractor.bar();

  return 0;
}
```

使用`std::integer_sequence<short, 9, 1, 1>`去匹配偏特化，编译器推导出`...args={9, ,1, 1}`；**在`SeqExtractor`偏特化模版范围内，`args...`就是整数值序列包**！


## index_sequence (2.4)

其实不必多说，`std::index_sequence`就是`std::integer_sequence`的模版别名，把`T`固定为`size_t`；并且对应地，标准库也提供`std::make_integer_sequence`的别名`std::make_index_sequence`。

```cpp
template <typename T>
void print_type_arg()
{
  std::cout << __PRETTY_FUNCTION__ << std::endl;
}

int main()
{
  using IndexSeq1 = std::index_sequence<2, 2, 2, 2>;
  using IndexSeq2 = std::make_index_sequence<4>;

  //打印：
  //  void print_type_arg() [with T = std::integer_sequence<long unsigned int, 2, 2, 2, 2>]
  print_type_arg<IndexSeq1>();

  //打印：
  //  void print_type_arg() [with T = std::integer_sequence<long unsigned int, 0, 1, 2, 3>]
  print_type_arg<IndexSeq2>();

  return 0;
}
```
