---
title: C++使用模版进行元编程
date: 2018-06-08 20:15:30
tags: [template]
categories: c++
---

初步尝试C++模版元编程。元编程考虑的是编译时的逻辑，和运行时不同，有点不太习惯，要时刻记住**编译时**!

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

  //打印：void print_type_arg() [with T = std::integer_sequence<long unsigned int, 2, 2, 2, 2>]
  print_type_arg<IndexSeq1>();

  //打印：void print_type_arg() [with T = std::integer_sequence<long unsigned int, 0, 1, 2, 3>]
  print_type_arg<IndexSeq2>();

  return 0;
}
```

# 常量表达式 (3)

## 常量表达式(3.1)

常量表达式是**编译期已知的不可变值**，有5种：

- **字面量(Literals)**

```cpp
42          // int 字面量
3.14        // double 字面量
"hello"     // 字符串字面量（类型为 const char[6]）
```

- **用constexpr声明的变量**

```cpp
#include <iostream>

int main()
{
  constexpr int a = 10;           // 合法
  constexpr double b = a * 2.5;   // 合法
  // constexpr int c = rand();    // error: call to non-‘constexpr’ function ‘int rand()’

  //打印：10 25
  std::cout << a << " " << b << std::endl;

  return 0;
}
```

- **只涉及常量表达式的运算**

```cpp
#include <iostream>

int main()
{
  constexpr int x = 5 + 3;            // 8
  constexpr int y = x << 2;           // 32
  constexpr int z = sizeof(int) * 8;  // 32（假设 int 为 4 字节）

  //打印：8 32 32
  std::cout << x << " " << y << " " << z << std::endl;

  return 0;
}
```

- **枚举值**

```cpp
#include <iostream>

enum Color
{
  Red = 1,
  Green = 2
};

int main()
{
  constexpr int c = Red; // 合法
  //打印：1
  std::cout << c << std::endl;
  return 0;
}
```

- **constexpr函数（当参数是常量表达式时）**

```cpp
#include <iostream>
#include <stdlib.h>

constexpr int add(int a, int b)
{
  return a + b;
}

int main()
{
  constexpr int   sum1 = add(3, 4);             // 合法，sum1 = 7
  const int       sum3 = add(rand(), rand());   // 合法：但sum3不是常量表达式
  // constexpr int sum2 = add(rand(), rand());  // 非法！error: call to non-‘constexpr’ function ‘int rand()’

  // 打印： 7 -1643747027
  std::cout << sum1 << " " << sum3 << std::endl;

  return 0;
}
```

虽然带有constexpr关键字，函数是否是常量表达式取决于上下文。可以这么理解：constexpr使一个函数可以出现在需要常量表达的地方，但不一定非要在那种情景下使用。见3.3节！

## 常量表达式的应用场景 (3.2)

- 数组大小定义
- 模板参数

```cpp
template <int Size>
struct Array
{
};

Array<5> arr;  // 合法：5 是常量表达式
```

- 静态断言（Static Assert）

```cpp
static_assert(sizeof(int) == 4, "int must be 4 bytes");
```

- 编译时计算

```cpp
#include <iostream>

constexpr int factorial(int n) {
  return (n <= 1) ? 1 : n * factorial(n - 1);
}
constexpr int fact_5 = factorial(5); // 120（编译期计算）

int main()
{
  //打印：120
  std::cout << fact_5 << std::endl;
}
```

## constexpr (3.3)

第3.1节说过，`constexpr`关键字是声明**常量表达式**的方式之一！

- **constexpr变量**：必须在编译期初始化，且initializer必须是常量表达式！

```cpp
constexpr int x = 42;             // 编译期确定
constexpr int y = x + 1;          // 编译期确定
//constexpr int z = rand();       // 错误：rand() 不是常量表达式
```

- **constexpr函数**：可以在编译期调用，也可以在运行时调用，即**取决于上下文**：
    - 编译期调用：当参数是编译期常量且结果用于需要常量表达式的场景（如数组大小、模板参数）。
    - 运行期调用：当参数是运行时变量或结果不用于常量表达式场景!

```cpp
#include <iostream>
#include <string>

constexpr int add(int a, int b)
{
  return a + b;
}

int main()
{
  constexpr int a = add(1, 2);      // 编译期求值
  std::string array[a];             // 合法！
  int b = add(3, 4);                // 可能编译期或运行期求值（取决于优化）
  int c = add(rand(), rand());      // 运行期求值

  //打印：3 7 -1643747027
  std::cout << a << " " << b << " " << c << std::endl;

  return 0;
}
```

如第3.1节所述，虽然带有constexpr关键字，函数是否是常量表达式取决于上下文。可以这么理解：constexpr使一个函数可以出现在需要常量表达的地方，但不一定非要在那种情景下使用。

## static constexpr vs static const (3.4)

```cpp
#include <iostream>

template<class T, T v>
struct MyTemp
{
  static constexpr T value1 = v;
  static const     T value2 = v;
};

int main()
{
  using TempInst1 = MyTemp<int, 3>;

  const int* p1 = &TempInst1::value1;

  // const int* p2 = &TempInst1::value2; //导致链接错误：undefined reference to `MyTemp<int, 3>::value2'

  //打印：3
  std::cout << *p1 << std::endl;
  return 0;
}
```

- static constexpr T value1 = v;

    - 编译期常量：constexpr强制要求变量在编译期初始化，且必须用常量表达式赋值。
    - 隐式内联（C++17起）：允许T是任何字面类型，无需在类外定义即可直接使用（包括取地址），编译器自动处理存储。注意：C++17前，若T是类类型，仍需在类外定义！


- static const T value = v;

    - 运行时常量：const仅表示变量不可修改，初始化可以发生在运行时，也可以发生在编译期！
    - ODR依赖：若T是整型或枚举类型，允许类内初始化，其他类型（如double、类类型）必须在类外定义（即使有类内初始化）！若程序中使用其地址（ODR-used），必须在类外定义，否则可能引发链接错误。


ODR（One Definition Rule，单一定义规则）依赖：同一个实体（变量、函数、类等）在程序中必须有且仅有一个定义。违反 ODR 会导致未定义行为（UB），常见表现是编译或链接错误。

也就是说，当`T`**不为整型或枚举类型时**，`value2`必须在类外定义！

```cpp
MyTemp::value2 = ...
```

而无论`T`是什么类型`value1`都不必在类外定义（C++17起）！

## constexpr vs. const (3.5)

|  特性              | constexpr                           | const                                     |
|--------------------|-------------------------------------|-------------------------------------------|
|变量初始化时机      | 编译期，一定是常量表达式            | 编译期或运行时，不一定（依赖initializer） |
|函数调用时机        | 编译期或运行时(取决于上下文)        | 始终运行期调用（普通函数行为）            |
|static成员变量      | 编译期常量，隐式内联（C++17起）     | 运行时常量，ODR依赖                       |
|适用场景            | 需要编译期确定的常量或逻辑          | 运行时常量                                |

# SFINAE原则 (4)

SFINAE（Substitution Failure Is Not An Error） 是 C++ 模板元编程中的核心原则，它允许在模板参数替换失败时不触发编译错误，而是将该模板候选从重载集中静默剔除。

## SFINAE的核心机制 (4.1)

当编译器尝试实例化一个模板时，会经历以下步骤：

- 替换（Substitution）：将模板参数代入模板声明，生成具体函数/类的签名。
- 检查有效性：验证代入后的模板代码是否合法（例如类型是否匹配、成员是否存在等）。
- 处理失败：
    - 如果替换失败（如访问不存在的成员、类型不兼容等），且失败发生在 直接替换过程 中（而非函数体内部），则根据 SFINAE 原则，忽略该候选模板，继续查找其他候选。
    - 如果所有候选均失败，才会报错。


看下面这段代码：


```cpp
#include <iostream>
#include <type_traits>

// 仅对整数类型有效
template<typename T>
typename std::enable_if<std::is_integral<T>::value, void>::type
process(T value)
{
  std::cout << "处理整数：" << value << std::endl;
}

// 仅对浮点类型有效
template<typename T>
typename std::enable_if<std::is_floating_point<T>::value, void>::type
process(T value)
{
  std::cout << "处理浮点：" << value << std::endl;
}

int main() {

  // 打印：处理整数：3
  process(3);          // 匹配整数版本（浮点版本被 SFINAE 剔除）

  // 打印：处理浮点：3.14
  process(3.14);       // 匹配浮点版本（整数版本被 SFINAE 剔除）

  // process("hello"); // 无匹配版本：no matching function for call to ‘process(const char [6])’
}
```

这是一段合法代码，可编译运行。编译器在处理`process(42)`时，**替换阶段**会产生：

```cpp
typename std::enable_if<true, void>::type
process(T value) { /* 处理整数 */ }

typename std::enable_if<false, void>::type
process(T value) { /* 处理浮点数 */ }
```

注意`std::enable_if<B, T>`在`B=false`时，根本就没有`type`成员！也就是说，**替换阶段**就失败了。假如没有SFINAE机制，那么，就会导致编译失败，终止！

而有了SFINAE机制，编译器只会把这个**失败的替换**静默丢弃，不视为错误。

同理，编译器处理`process(3.14)`时，也会产生2个替换，并丢弃**失败的替换**。最终，两个正确的替换被保留下来。

所以，与编译错误的关键区别是：SFINAE失败仅影响重载选择，若最终无合法候选，才会触发编译错误。

## SFINAE的替代方案 (4.2)

- if constexpr：在函数体内部实现编译期条件分支，避免多份重载（C++17起）

```cpp
#include <iostream>
#include <type_traits>

template<typename T>
void process(T value) {
  if constexpr (std::is_integral_v<T>) {
    std::cout << "处理整数: " << value << std::endl;
  } else if constexpr (std::is_floating_point_v<T>) {
    std::cout << "处理浮点: " << value << std::endl;
  } else {
    static_assert(std::is_integral_v<T> || std::is_floating_point_v<T>, "Unsupported type");
  }
}

int main()
{
  // 打印：处理整数: 3
  process(3);

  // 打印：处理浮点: 3.14
  process(3.14);

  // process("hello");  // error: static assertion failed: Unsupported type
  return 0;
}
```

- Concepts：通过requires直接约束模板参数（C++20起）

注：下面的代码Linux/gcc环境`g++ --std=c++2a`无法编译! 而Clang环境(macOS/Linux)下`clang++ --std=c++2a`可以编译运行!

```cpp
#include <iostream>
#include <type_traits>

template<typename T>
  requires std::is_integral_v<T> || std::is_floating_point_v<T>
void process(T value)
{
  std::cout << "处理整型或浮点: " << value << std::endl;
}

int main()
{
  // 打印：处理整型或浮点: 3
  process(3);

  // 打印：处理整型或浮点: 3.14
  process(3.14);

  // process("hello");  // error: no matching function for call to 'process'
  return 0;
}
```

## enable_if (4.3)

- 基本定义

当B为true：`enable_if_t<B, T>`等价于T；当B为false：`enable_if<B, T>` 无`type`成员，导致模板替换失败（SFINAE）。

```cpp
template<bool B, class T = void>
struct enable_if {};

template<class T>
struct enable_if<true, T> {
    using type = T;  // 当条件为 true 时定义 type 成员
};

// C++14 起提供的别名模板简化版本
template<bool B, class T = void>
using enable_if_t = typename enable_if<B, T>::type;
```

- 核心用途1：控制函数模板的重载

比`if constexpr`可读性差，见4.2节。

```cpp
#include <iostream>
#include <type_traits>

// 实现1：针对整数类型
template<typename T>
std::enable_if_t<std::is_integral_v<T>, void>
process(T value) {
  std::cout << "处理整数: " << value << std::endl;
}

// 实现2：针对浮点类型
template<typename T>
std::enable_if_t<std::is_floating_point_v<T>, void>
process(T value) {
  std::cout << "处理浮点: " << value << std::endl;
}

int main()
{
  //打印：处理整数: 3
  process(3);

  //打印： 处理浮点: 3.14
  process(3.14);

  // process("hello"); //error: no matching function for call to ‘process(const char [6])’
}
```

- 核心用途2：约束类模板参数

```cpp
#include <iostream>
#include <string>
#include <type_traits>

// 仅当T是算术类型时可用
template<typename T, typename = std::enable_if_t<std::is_arithmetic_v<T>>>
struct MyContainer {
};

int main() {
  MyContainer<int> c1;             // 编译通过

  // MyContainer<std::string> c2;  // error: template argument 2 is invalid

  return 0;
}
```

首先，`typename = std::enable_if_t<...>`是**未命名类型参数，即类型参数名被省略，因为不需要在类内使用**。补全的话是:

```cpp
template<typename T, typename Dummy = std::enable_if_t<std::is_arithmetic_v<T>>>
```

其次，满足`is_arithmetic_v`时，得到的是`void`；其实是什么类型都是无所谓，例如改写成这样，是等效的：

```cpp
template<typename T, typename = std::enable_if_t<std::is_arithmetic_v<T>, std::string>>
```

因为这个**未命名类型参数的作用是约束模版只对算术类型的T有效**，若是非算术类型，`enable_it<false>::value`没有定义。

不过，上面这个代码可读性差（见4.2节），下面是使用`requires`重新实现：

注：下面的代码Linux/gcc环境`g++ --std=c++2a`无法编译! 而Clang环境(macOS/Linux)下`clang++ --std=c++2a`可以编译运行!

```cpp
#include <iostream>
#include <string>
#include <type_traits>

// 仅当T是算术类型时可用
template<typename T>
  requires std::is_arithmetic_v<T>
struct MyContainer {
};

int main() {
  MyContainer<int> c1;             // 编译通过

  // MyContainer<std::string> c2;  // error: constraints not satisfied for class template 'MyContainer' [with T = std::basic_string<char>]

  return 0;
}
```

- 常见错误：条件重叠导致歧义

```cpp
#include <string>
#include <type_traits>

template<typename T>
std::enable_if_t<std::is_integral_v<T>, void> func(T) {}

template<typename T>
std::enable_if_t<std::is_arithmetic_v<T>, void> func(T) {}

int main()
{
  // 条件重叠（整数也是算术类型）
  // func(3); //error: call of overloaded ‘func(int)’ is ambiguous
  return 0;
}
```
