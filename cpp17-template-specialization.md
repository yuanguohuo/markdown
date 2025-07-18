---
title: C++的模版特化 
date: 2018-06-07 18:32:08
tags: [template]
categories: c++
---

介绍C++模版的特化与偏特化。

<!-- more -->

<script type="text/x-mathjax-config">
MathJax.Hub.Config({
tex2jax: {inlineMath: [['$','$'], ['\\(','\\)']]}
});
</script>

<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>

# 主模版 (1)

```cpp
#include <iostream>
#include <string>
#include <string_view>
#include <tuple>
#include <utility>

using namespace std;

template<typename T1, typename T2, typename T3>
class MyTemp {
  public:
    static void print_type_info() {
      std::string_view name = __PRETTY_FUNCTION__;
      name.remove_prefix(name.find("print_type_info() ") + 18);
      std::cout << "generalized-type: " << name << std::endl;
    }
};
```

主模版有3个类型参数：`T1`, `T2`和`T3`；静态函数(类函数)`print_type_info()`用于打印出实例化的类型实参，细节可以忽略。它借助于`__PRETTY_FUNCTION__`；这是函数的签名，里面包含类的信息，即实例化的类型实参。

假如模版没有定义，即：

```cpp
template<typename T1, typename T2, typename T3>
class MyTemp;
```

它也可以实例化，但实例化的类不能构造对象。这是一个细节。

```cpp
using T1 = MyTemp<int, char, float>;  //OK
T1 t1;  //error: ‘T1 t1’ has incomplete type and cannot be defined
```

# 全特化 (2)

特化比较简单，就是针对`T1`, `T2`, `T3`为特定类型提供一个特别的实现。实例化时，若类型参数刚好匹配“特定类型”，就实例化为这个特别实现；否则就用主模版的实现。

```cpp
//全特化
template<>
class MyTemp<char, int, float> {
  public:
    static void print_type_info() {
      std::cout << "full-specialized: [char, int, float]" << std::endl;
    }
};
```

实例化：

```cpp
using G = MyTemp<std::string, std::string, std::string>; //主模版
using F = MyTemp<char, int, float>; //全特化

G::print_type_info();
F::print_type_info();
```

MacOS/clang编译器输出(c++17)：

```
generalized-type: [T1 = std::string,
                   T2 = std::string,
                   T3 = std::string]
full-specialized: [char, int, float]
```

Linux/gcc编译器输出(c++17)：

```
generalized-type: [with T1 = std::basic_string<char>;
                        T2 = std::basic_string<char>;
                        T3 = std::basic_string<char>]
full-specialized: [char, int, float]
```

# 偏特化 (3)

先看简单形态：

```cpp
//偏特化1
template<typename T1, typename T2>
class MyTemp<T1, T2, float> {
  public:
    static void print_type_info() {
      std::string_view name = __PRETTY_FUNCTION__;
      name.remove_prefix(name.find("print_type_info() ") + 18);
      std::cout << "partial-specialized-1: " << name << std::endl;
    }
};

//偏特化2
template<typename T1>
class MyTemp<T1, int, float> {
  public:
    static void print_type_info() {
      std::string_view name = __PRETTY_FUNCTION__;
      name.remove_prefix(name.find("print_type_info() ") + 18);
      std::cout << "partial-specialized-2: " << name << std::endl;
    }
};
```

这里提供了2个偏特化：

- 偏特化1：只固定第3个类型参数为`float`；
- 偏特化2：固定第2个类型参数为`int`，固定第3个类型参数为`float`；

那么，实例化的的时候，如匹配偏特化2必然匹配偏特化1(例如`MyTemp<string, int, float>`)，最终选择哪个呢？当然是**选择匹配最多的，遵循最长匹配原则**，即偏特化2;

```cpp
using P1 = MyTemp<double, double, float>; //偏特化1
using P2 = MyTemp<double, int, float>;    //偏特化2
P1::print_type_info();
P2::print_type_info();
```

MacOS/clang编译器输出(c++17)：

```
partial-specialized-1: [T1 = double, T2 = double, T3 = float]
partial-specialized-2: [T1 = double, T2 = int, T3 = float]
```

Linux/gcc编译器输出(c++17)：

```
partial-specialized-1: [with T1 = double; T2 = double]
partial-specialized-2: [with T1 = double]
```

似乎可以总结：特化就是把`template<type-arg-list-1> class MyTemp<type-arg-list-2>`里的“type-arg-list-1”缩短，把“type-arg-list-2”中形参固化成真实类型。当“type-arg-list-1”为空的时候，“type-arg-list-2”里的形参全部固化成真实类型，这时就变成“full-specialization”了(见第2节)。

但实际上，固定“type-arg-list-2”的时候，不一定非要缩短“type-arg-list-1”！**特化不是从“type-arg-list-1”中移除，在“type-arg-list-2”中固化的过程**！看下面的例子；

```cpp
//偏特化3
template<typename U, typename V, typename W, int a, int b>
class MyTemp<std::tuple<U,V,W>, char[a], double[b]> {
  public:
    static void print_type_info() {
      std::string_view name = __PRETTY_FUNCTION__;
      name.remove_prefix(name.find("print_type_info() ") + 18);
      std::cout << "partial-specialized-3: " << name << std::endl;
    }
};
```

当“type-arg-list-2”固定的时候，“type-arg-list-1”不但没有缩短，反而曾长了！特化/偏特化确实是固化“type-arg-list-2”中的类型参数，但不是为了缩短“type-arg-list-1”！实际上，**“type-arg-list-1”为“type-arg-list-2”提供类型信息**：即**“type-arg-list-2”中的一切未固定信息，例如类型参数，非类型参数（例如int参数）等，都要由“type-arg-list-1”提供**。

- 对于偏特化1和偏特化2的简单情形，在“type-arg-list-2”固化的过程中，需要的类型信息减少了，所以“type-arg-list-1”缩短了。甚至在“full-specialization”中“type-arg-list-1”缩为0，因为“type-arg-list-2”不需要任何类型信息；
- 但对于偏特化3，为了固化“type-arg-list-2”：`T1`固化为`std::tuple<U,V,W>`需要3个类型参数`U`, `V`和`W`；`T2`固化为`char[a]`，`T3`固化为`double[b]`还需要2个`int`类型的型参`a`和`b`；这些都需要“type-arg-list-1”来提供。

在为一个模版提供特化/偏特化的时候也是这个思路：**先想着去固定`T1`, `T2`和`T3`；当这些类型能全部固定的时候，就是全特化，“type-arg-list-1”为空；当这些类型中有不可以固定的东西，无论是类型参数，还是`int`这样的非类型参数，都放到“type-arg-list-1”中，这就是偏特化**。

那么如何实例化偏特化3呢？

```cpp
using P3 = MyTemp<std::tuple<int, float, double>, char[100], double[200]>;
P3::print_type_info();
```

MacOS/clang编译器输出(c++17)：

```
partial-specialized-3: [T1 = std::tuple<int, float, double>, T2 = char[100], T3 = double[200]]
```

Linux/gcc编译器输出(c++17)：

```
partial-specialized-3: [with U = int; V = float; W = double; int a = 100; int b = 200]
```

Linux/gcc的输出中明确地推导出了`U`, `V`和`W`以及`a`, `b`；不过从MacOS/clang的输出，可以更清楚地看到`T1`, `T2`和`T3`的实参。

注意：

- 实例化时不是要提供`U`, `V`和`W`以及`a`, `b`，而是要提供与`std::tuple<U,V,W>, char[a], double[b]`匹配的`T1`, `T2`和`T3`；
- 编译器会推导出`U`, `V`和`W`以及`a`, `b`；**可以利用编译器的推导能力来提取基础类型**；例如在偏特化3中添加`using type_u = U;`，那么在上面的实例化中，就从`std::tuple<int, float, double>`中提取出`type_u = int`；就是说，`T1`又是一个模版类，模版类有自己的类型参数`X`, `Y`, `Z`(这时，`X`, `Y`, `Z`要放在“type-arg-list-1”中)，编译器会推导出它们的实际类型，所以`using type_x = X;`就可以得到真实的`X`类型；

另外，似乎3个类型参数都固化了，看上去像一个“full-specialization”？其实不是的，**“full-specialization”不是那么定义的，是看模版还有没有自由度**。显然这里还有自由度，`std::tuple<U,V,W>, char[a], double[b]`中的`U`, `V`和`W`以及`a`, `b`都可以随意选择。

最后，再看一个更复杂的例子：

```cpp
//偏特化4
template<typename U, typename V, typename... W, size_t... a, size_t... b>
class MyTemp<std::tuple<U, V, W...>, std::index_sequence<a...>, std::index_sequence<b...>> {
  public:
    static void print_type_info() {
      std::string_view name = __PRETTY_FUNCTION__;
      name.remove_prefix(name.find("print_type_info() ") + 18);
      std::cout << "partial-specialized-4: " << name << std::endl;
    }
};
```

这里甚至引入了“参数包”：固化“type-arg-list-2”中的`T1 = std::tuple<U, V, W...>`，`T2 = std::index_sequence<a...>`, `T3 = std::index_sequence<b...>`需要“type-arg-list-1”提供`U`, `V`, `W...`以及`a...`和`b...`；

实例化：

```cpp
using P4 = MyTemp<std::tuple<int, float, double, char, std::string>, std::index_sequence<1,2,3>, std::index_sequence<4,5,6,7,8,9>>;
P4::print_type_info();
```

MacOS/clang编译器输出(c++17)：

```
partial-specialized-4: [T1 = std::tuple<int, float, double, char, std::string>,
                        T2 = std::integer_sequence<unsigned long, 1, 2, 3>,
                        T3 = std::integer_sequence<unsigned long, 4, 5, 6, 7, 8, 9>]
```

Linux/gcc编译器输出(c++17)：

```
partial-specialized-4: [with U = int; 
                             V = float; 
                             W = {double, char, std::basic_string<char, std::char_traits<char>, std::allocator<char> >};
                             long unsigned int ...a = {1, 2, 3};
                             long unsigned int ...b = {4, 5, 6, 7, 8, 9}]
```

同样，Linux/gcc的输出中明确地推导出`U`, `V`, `W...`以及`a...`, `b...`；不过MacOS/clang的输出中，可以更清楚地看到`T1`, `T2`和`T3`的实参。


# 特化模版会继承主模版的基类吗 (4)

先说答案：不会！

```cpp
#include <iostream>
#include <utility>
#include <string>
#include <string_view>

template <class T, size_t N>
struct Aligned;

//主模版
template <class T>
struct NotAligned {
  static_assert(std::is_same<T, float>::value, "only NotAligned<float> allowed");
};

//偏特化
template <class T, size_t N>
struct NotAligned<const Aligned<T, N>> {
  static_assert(std::is_same<T, double>::value, "only NotAligned<const Aligned<double, N>> allowed");
};

//主模版
template <class T>
struct Type : NotAligned<T> {
  using type = T;
  static void info() {
    std::cout << "general-type: " << __PRETTY_FUNCTION__ << std::endl;
  }
};

//偏特化
template <class T, size_t N>
struct Type<Aligned<T, N>> {
  using type = T;
  static void info() {
    std::cout << "specialized-type: " << __PRETTY_FUNCTION__ << std::endl;
  }
};

int main()
{
  // 打印：
  // general-type: static void Type<T>::info() [with T = float]
  using TpFloat = Type<float>; //必须为float
  TpFloat::info();

  // 打印：
  // general-type: static void Type<T>::info() [with T = const Aligned<double, 16>]
  using TpConstAlignedDouble = Type<const Aligned<double, 16>>; //必须为double
  TpConstAlignedDouble::info();

  // 打印：
  // specialized-type: static void Type<Aligned<T, N> >::info() [with T = int; long unsigned int N = 32]
  using TpAlignedX = Type<Aligned<int, 32>>; //int可以随意换：short, size_t, string, ...
  TpAlignedX::info();

  return 0;
}

```

MacOS/clang编译器输出(c++17)：

```
general-type: static void Type<float>::info() [T = float]
general-type: static void Type<const Aligned<double, 16>>::info() [T = const Aligned<double, 16>]
specialized-type: static void Type<Aligned<int, 32>>::info() [T = Aligned<int, 32>]
```

Linux/gcc编译器则编译出错，输出(c++17)：

```
general-type: static void Type<T>::info() [with T = float]
general-type: static void Type<T>::info() [with T = const Aligned<double, 16>]
specialized-type: static void Type<Aligned<T, N> >::info() [with T = int; long unsigned int N = 32]
```

首先，`NotAligned`的主模版只允许实例化`NotAligned<float>`；而偏特化`NotAligned<const Aligned<T, N>>`只允许实例化`T=double`；

然后，看`main()`中的几个实例化类：

- `TpFloat`：匹配`Type`的主模板，并且编译器推导出`T=float`；继承`NotAligned<T>`时，也匹配其主模版；主模版只允许`NotAligned<float>`，这里`T`刚好是`float`，满足`static_assert`！
- `TpConstAlignedDouble`：也匹配`Type`的主模版，是的，没错，`const`是类型的一部分，所以不能匹配`Type<Aligned<...>>`特化！匹配`Type<T>`时，编译器推导出`T=const Aligned<double, 16>`；继承`NotAligned<T>`时，匹配其偏特化`NotAligned<const Aligned<T, N>>`！在这个偏特化中，编译器又推导出`T=double`！也满足`static_assert`!
- `TpAlignedX`：匹配`Type`的偏特化`Type<Aligned<T, N>>`。假如这个偏特化也继承主模版的基类`NotAligned`，无论匹配`NotAligned`的主模版还是偏特化，`static_assert`都不会满足！

所以，**一旦匹配特化或偏特化，就和主模板没有任何关系，不会继承主模版的基类**！

# 显示实例化 (5)

```cpp
#include <iostream>
#include <typeinfo>

using namespace std;

template<typename T>
void foo()
{
  std::cout << "foo("  << typeid(T).name() << ")" << std::endl;
}

template void foo<int>();

int main()
{
  foo<int>();
  foo<double>();
  return 0;
}
```

MacOS/clang编译器(c++17)和Linux/gcc编译器(c++17)都输出：

```
foo(i)
foo(d)
```

注意：`template void foo<int>();`的作用是什么呢？**其实并没有直接作用**：`foo<int>()`和`foo<double>`行为是一样的！而并没有声明`template void foo<double>();`

这是一种**显式实例化**（Explicit Instantiation）语法。它的作用是**强制要求编译器在当前位置生成目标实例**。特点是，`template`后面没有`<>`，并且函数没有body！

在本例中，就是强制要求编译器生成`foo<int>(){...}`函数实例；虽然没有强制要求生成`foo<double>(){...}`，但编译器在编译`foo<double>();`的时候，**也能够自动推导出需要那样一个实例**！

显式实例化的作用：

- 控制编译单元：在大型项目中，显式实例化可以用于集中管理模板实例化，减少编译时间。Yuanguo：假如在moduleA中定义模版而在moduleB, moduleC, ...中使用；在moduleA中显示实例化可以让这些模版实例集中在一起(moduleA中)？
- 提前生成代码：编译器会在此处生成foo<int>的具体实现代码。
- 避免隐式实例化：后续代码中如果调用foo<int>()，将直接使用此处生成的实例化版本，而不会隐式生成重复代码。


# 对比模版别名 (6)

其实，模版(偏)特化和模版别名没有关系：**特化是为特定参数定制行为，产生了新的实现**；而**模版别名只是简化名称，不改变行为**！
只是，模版别名定义时和偏特化有点像：

```cpp
template<type-arg-list-2>
using TempAlias = Temp<type-arg-list-2>;
```

要填写“type-arg-list-2”，其中的一切未固定信息（类型参数、非类型参数）都要写到“type-arg-list-1”中！
假如所有信息都固定了，则不需要写`template<>`，这和模版完全特化有点区别。

# 小节 (7)

总结C++模版的特化以及偏特化，特别是偏特化中的复杂情况。
