---
title: C++11中的右值引用
date: 2018-05-25 09:12:31
tags: [rvalue reference, 右值引用, std::move]
categories: c++
---

本文主要介绍C++11中的右值引用，如何初始化，以及参数匹配上的特点，并总结了和左值引用的相似与不同。然后简单介绍了一下std::move。

<!-- more -->

# 左值引用 (1)

在开始研究右值引用之前，先说一下左值引用。左值引用，就是C++11之前我们所说的引用。为了区分右值引用，现在把它明确的叫做**左值**引用。所以，无须多说，只提一点(因为后面会用到这点)：

```cpp
int foo(int& a);       //不能匹配右值，例如foo(3)会编译失败。
int bar(const int& a); //可以匹配右值。
```

以后若说*右值引用只能引用右值，左值引用只能引用左值*，是指*非const的左值引用*。

# 右值引用 (2)

有关右值引用(rvalue reference)的产生逻辑，见[C++11的std::move](http://www.yuanguohuo.com/2018/05/24/cpp11-std-move/)，这里我们只关注右值引用本身。下文的例子都基于一个测试class Foo，它的定义在文末"测试用到的Foo类"一节。

## 右值引用是一种新的类型 (2.1)

Foo&&是一个不用于Foo&和Foo的新类型，如下三个函数可以同时出现：

```cpp
int func(Foo f)
{
  cout << "int func1(Foo)" << endl;
  return 0;
}

int func(Foo& f)
{
  cout << "int func1(Foo&)" << endl;
  return 0;
}

int func(Foo&& f)
{
  cout << "int func1(Foo&&)" << endl;
  return 0;
}
```

**注意：**能够同时定义这三个版本的重载，证明了Foo&&、Foo&和Foo是三个不同的类型。但调用的时候，总是出现歧义：

* 左值作参数 

```cpp
int main()
{
  Foo foo("xyz");
  func(foo);
  return 0;
}
```

编译失败，错误如下：

```
error: call of overloaded ‘func(Foo&)’ is ambiguous
   func(foo);
           ^
candidates are:
int func(Foo)
 int func(Foo f)
     ^
int func(Foo&)
 int func(Foo& f)
     ^
int func(Foo&&) <near match>
 int func(Foo&& f)
```

删掉func(Foo)或者func(Foo&)则编译通过。值得一提的是，编译器认为func(Foo&&)也是一个近似匹配的候选对象。但事实上，它是匹配不了的：你若真把func(Foo)和func(Foo&)都删了，让func(Foo&&)去匹配，则会报错: error: cannot bind 'Foo' lvalue to 'Foo&&'.

* 右值作参数

```cpp
int main()
{
  func(Foo("abc"));
  return 0;
}
```

编译失败，错误如下：

```
error: call of overloaded ‘func(Foo)’ is ambiguous
   func(Foo("abc"));
                  ^
candidates are:
int func(Foo)
 int func(Foo f)
     ^
int func(Foo&&)
 int func(Foo&& f)
```

func(Foo&)匹配不了右值(注意func(const Foo&)可以匹配右值，见"左值引用"一节)，而func(Foo)和func(Foo&&)都可以。删掉两者中的一个，则编译成功。

先不管这两种歧义问题(不定义func(Foo)这个版本，只保留另两个？)，回到主题上: Foo&&是一种不同于Foo&和Foo的新类型。

## 右值引用如何初始化 (2.2)

```cpp
Foo func2()
{
  Foo f1("AAA");
  return f1;
}

int main()
{
  int && r1 = 3;
  Foo && r2 = func2();
  Foo && r3 = Foo("BBB");

  Foo f("CCC");
  Foo && r4 = std::move(f);

  return 0;
}
```

从上面的例子，总结一下右值引用如何初始化：

* r1：引用字面量；
* r2：引用临时变量(函数返回的)；
* r3：引用临时变量(匿名变量)；
* r4：引用经std::move强制的左值；

## 有名是左值无名是右值 (2.3)

我们把上一节的例子加以扩充(注意，本例需要认真看一下class Foo的实现，见文末"测试用到的Foo类"一节):

```cpp
Foo func2()
{
  Foo f1("AAA");
  return f1;
}

int main()
{
  int && r1 = 3;
  Foo && r2 = func2();
  Foo && r3 = Foo("BBB");

  Foo f("CCC");
  Foo && r4 = std::move(f);

  r2.dump();
  r3.dump();

  r2 = Foo("DDD");
  Foo f3("EEE");
  r3 = f3;

  return 0;
}
```

编译：

```
# g++ -fno-elide-constructors --std=c++11 test.cpp
```

运行，输出如下：

```
Foo(const char * s)    0x7ffc4a430330/0x21b0010/AAA
Foo(Foo && other)    0x7ffc4a430330/0x21b0010/AAA-->0x7ffc4a430380/0x21b0010/AAA
~Foo()    0x7ffc4a430330/0/NULL
Foo(const char * s)    0x7ffc4a430390/0x21b0030/BBB
Foo(const char * s)    0x7ffc4a430360/0x21b0050/CCC
0x7ffc4a430380/0x21b0010/AAA
0x7ffc4a430390/0x21b0030/BBB
Foo(const char * s)    0x7ffc4a4303a0/0x21b0070/DDD
Foo& operator= (Foo && other)    0x7ffc4a4303a0/0x21b0070/DDD==>0x7ffc4a430380/0x21b0070/DDD
~Foo()    0x7ffc4a4303a0/0/NULL
Foo(const char * s)    0x7ffc4a430350/0x21b0010/EEE
Foo& operator= (const Foo & other)    0x7ffc4a430350/0x21b0010/EEE==>0x7ffc4a430390/0x21b0030/EEE
~Foo()    0x7ffc4a430350/0x21b0010/EEE
~Foo()    0x7ffc4a430360/0x21b0050/CCC
~Foo()    0x7ffc4a430390/0x21b0030/EEE
~Foo()    0x7ffc4a430380/0x21b0070/DDD
```

* 输出第1行：构造func2中的局部变量f1；
* 输出第2行：构造临时对象。f1的data被move到临时对象（通过移动拷贝构造函数）；它被r2引用；
* 输出第3行：析构f1。data已经被move，故为0/NULL；
* 输出第4行：构造临时对象Foo("BBB")；它被r3引用；
* 输出第5行：构造main中的局部变量f；
* 输出第6行：r2.dump()；
* 输出第7行：r3.dump()；
* 输出第8行：构造临时对象Foo("DDD");
* 输出第9行：临时对象Foo("DDD")被赋值到r2（通过移动赋值函数）;
* 输出第10行：析构临时对象Foo("DDD")。data已经被move，故为0/NULL；
* 输出第11行：构造main中的局部变量f3；
* 输出第12行：f3被赋值到r3。通过赋值函数(非移动赋值)；
* 输出第13行：析构f3。它的data没有被move。
* 输出第14行：析构main中的局部变量f；
* 输出第15行：析构r3引用的临时对象(匿名变量)；
* 输出第16行：析构r2引用的临时对象(函数返回的)；


例子比较复杂，但我们关注的重点是r2和r4，它们引用两个临时对象。要说这俩临时对象，若没有r2和r4引用它们，它们一经构造就会立即被析构。但现在不同了：1. 它们有了更长的生命周期，直到引用它们的**右值引用类型的变量**生命周期结束时，才被析构；2. 它们可以出现在"="左边(r2=Foo("DDD"); r3=f3;)。等等！这不是左值才有的特征吗？

是的！这俩临时对象(右值)，变成了左值。其实应该换个角度说，**右值引用类型的变量，r2和r3，是左值**。再说一遍，**右值引用类型的变量是左值！**只不过：
* 这个左值的类型是右值引用类型；
* 只能赋给它右值(从上例可以看出，虽然赋值时一定要是右值，但赋值完成之后，右值变成了左值)；

为什么这样呢？其实这是C++11的一个规则：**右值引用类型(形如Foo&&)的表达式，有名字的是左值；没有名字的是右值。**

例1：
```cpp
Foo&& getFoo();
Foo f = getFoo(); //表达式getFoo()返回值没有名字，是右值，这里调用Foo(Foo&&)
```

例2：
```cpp
void func(Foo&& f)
{
  Foo f2 = f; //f有名字，是左值，这里调用Foo(Foo const&)
}
```

例3：
```cpp
int func(Foo& f)
{
  cout << "int func1(Foo&)" << endl;
  f.dump();
  return 0;
}

int func(Foo&& f)
{
  cout << "int func1(Foo&&)" << endl;
  f.dump();
  return 0;
}

int main()
{
  Foo f("123");
  Foo&& rvref = std::move(f);

  func(std::move(f));
  func(rvref);
  func(std::move(rvref));

  return 0;
}
```

编译：

```
# g++ -fno-elide-constructors --std=c++11 test.cpp
```

运行，输出如下：

```
Foo(const char * s)    0x7ffd802c59c0/0x1216010/123
int func1(Foo&&)
0x7ffd802c59c0/0x1216010/123
int func1(Foo&)
0x7ffd802c59c0/0x1216010/123
int func1(Foo&&)
0x7ffd802c59c0/0x1216010/123
~Foo()    0x7ffd802c59c0/0x1216010/123
```

可见：
* std::move(f)匹配的是func(Foo&&)；这符合预期；
* rvref匹配的是func(Foo&)；这就奇怪了，rvref明明就是Foo&&类型的，为什么没有匹配func(Foo&&)？
* std::move(rvref)匹配的是func(Foo&&)；恩，这也符合预期；

看了这些例子，我们知道，**左值的引用肯定是左值，但右值的引用却不一定是右值，这取决于表达式有没有名字**。重申一遍：**右值引用类型(形如Foo&&)的表达式，有名字的是左值；没有名字的是右值。**背后的动机是什么呢？

因为一个右值，一经拷贝(无论是做拷贝构造函数的参数，还是出现在"="的右边)，便失去了它的资源(被move走了)。假如存在有名字的右值，那么一个不经意的拷贝(在"="右边)，就把它修改了，让它丢失资源从而变成变成空壳子。这会带来严重的后果：想想我们看代码的时候，要找一个变量在哪里被修改过，都是找它在"="左边的出现，不是吗？谁会关心它在"="右边的出现呢？也就是说，假如存在有名字的右值，我们要找它在哪里被修改过，就不得不注意它所有的出现。如例2中，f在"="右边出现一次，就不是原来的f了。这非常不合理。

若程序员真想让有名字的变量当右值，怎么办呢？答案就是std::move()。虽然经std::move()强制过的变量，可能丢失了资源，但是，情况要好多了：有std::move()来提醒你，这个变量可能被修改了(资源被move走)。

另外，一个右值引用类型的变量(左值)，例如，例3中的rvref，可以再次传给std::move()来得到右值。

为了加深印象，再看一个例子。假如class B重载了移动构造函数，class D继承clss B。为了确保移动语义适用于D从B继承来的那部分，我们需要为D提供移动构造函数。这样是不对的：

```cpp
D(D&& other) : B(other)
{
  ...
}
```

因为other是一个左值，B(other)会调用B(const B&)函数，而不是移动构造函数。正确的写法是：
```cpp
D(D&& other) : B(std::move(other))
{
  ...
}
```

**注意：**父类的右值引用类型的变量，可以引用子类的右值。

## 参数匹配 (2.4)

从前面可以看出，func(Foo&)匹配左值(重申：有名字的右值引用表达式是左值)，func(Foo&&)匹配右值。在本节，我们主要强调：

```cpp
int func(const Foo&);
```

是能够匹配右值的(注意是const的左值引用)。这个问题说过啊(见"左值引用"一节)，有什么新鲜的？还真有一点需要提。考虑构造函数和赋值函数：

```cpp
Foo(const Foo&);
Foo& operator=(const Foo&);
```

它们是能够匹配右值的。若在class Foo中，没有移动构造和移动赋值函数，下面对构造和赋值的调用，会匹配上面的两个函数。

```cpp
Foo f1("123");
Foo f2(std::move(f1));

Foo f3("456");
Foo f4("789");
f4 = std::move(f3);
```

这是有好处的：假如你确定f1和f3不再需要，可以被move，你就可以这么这样写代码，而不用关系class Foo是否重载了移动构造和移动赋值函数。若暂时没有重载，那也不会出错；将来重载了，则会带来性能提升。

## 和左值引用的相似与不同 (2.5)

相似之处：
* **右值引用类型的变量**和**左值引用类型的变量**，本身都是左值；
* **右值引用类型的变量**和**左值引用类型的变量**，都是引用另一个变量，本身不占内存；
* 对**右值引用类型的变量**和**左值引用类型的变量**进行&运算，得到的都是被引用对象的地址；
* 和返回局部变量的左值引用是非法的一样，**返回局部变量的右值引用也是非法的**；道理也相同；
* 和左值引用一样，父类的右值引用，也能够引用子类的右值；

不同之处：
* 初始化不同：**右值引用类型的变量**只能用右值进行初始化，而**左值引用类型的变量**必须用左值(这里指非const情形)。
* 类型不同：**右值引用类型**是一种新的类型，当然和**左值引用类型**不同。

也就是说：**右值引用类型的变量和左值引用类型的变量，除了初始化不同和类型不同之外，其他都是相同的**(对吗？还有什么不同？)

## std::move (2.6)

std::move的作用，就是把一个左值强制转换为右值，然后就可以用来初始化一个右值引用类型的变量了，例如函数形参，或者局部变量Foo&& f。它的实现如下：

```cpp
template <typename T>
typename remove_reference<T>::type&& move(T&& t) noexcept
{
    return static_cast<typename remove_reference<T>::type&&>(t);
}
```

"typename remove_reference<T>::type"就是把T中的引用去掉，无论T是"Foo&"还是"Foo"，它的结果都是Foo。这样就明确了，std::move()就是通过static_cast<>把参数t强制为右值的。现在，先记住这个简单的理解吧。我将在[C++11中的通用引用](http://www.yuanguohuo.com/2018/05/25/cpp11-universal-ref/)中*std::move的工作原理*介绍它是如何工作的。

## 区分右值和右值引用 (2.7)

下面都是右值引用类型的表达式，区分一下右值和右值引用:

* Foo&& f中的f是右值引用(或者说右值引用类型的变量)，本身是左值；
* std::move(g)是右值(假设g是Foo类型的左值)，所以可以赋给一个右值引用类型的变量；
* bar()是右值(假设bar的原型是Foo&& bar();)，这种情形和std::move()一样；
* static_cast<Foo&&>(g)是右值(假设g是Foo类型的左值)，这种情形和std::move()一样；

# 测试用到的Foo类 (3)

```cpp
#include <iostream>
#include <string.h>

using namespace std;

class Foo
{
  private:
    char * data;
  public:
    virtual ~Foo()
    {
      std::cout << "~Foo()    " 
                << (void*)this << "/" 
                << (void*)data << "/" 
                << (data==NULL?"NULL":data) << std::endl;

      if(data != NULL)
      {
        delete []data;
        data = NULL;
      }
    }

    Foo(const char * s) : data(NULL)
    {
      std::cout << "Foo(const char * s)    ";

      if(s)
      {
        int len = strlen(s);
        data = new char[len+1];
        memcpy(data, s, len+1);
      }

      std::cout << (void*)this << "/" 
                << (void*)data << "/" 
                << (data==NULL?"NULL":data) << std::endl;
    }

    Foo(const Foo & other) : data(NULL)
    {
      std::cout << "Foo(const Foo & other)    " 
                << (void*)&other << "/" 
                << (void*)other.data << "/" 
                << (other.data==NULL?"NULL":other.data) << "-->";


      if(other.data)
      {
        int len = strlen(other.data);
        data = new char[len+1];
        memcpy(data, other.data, len+1);
      }

      std::cout << (void*)this << "/" 
                << (void*)data << "/" 
                << (data==NULL?"NULL":data)<< std::endl;
    }

    Foo& operator= (const Foo & other)
    {
      std::cout << "Foo& operator= (const Foo & other)    " 
                << (void*)&other << "/" 
                << (void*)other.data << "/" 
                << (other.data==NULL?"NULL":other.data) << "==>";

      if(this != &other)
      {
        if(NULL != data)
        {
          delete []data;
          data = NULL;
        }

        if(NULL != other.data)
        {
          int len = strlen(other.data);
          data = new char[len+1];
          memcpy(data, other.data, len+1);
        }
      }

      std::cout << (void*)this << "/" 
                << (void*)data << "/" 
                << (data==NULL?"NULL":data) << std::endl;

      return *this;
    }

    Foo(Foo && other) noexcept
    {
      std::cout << "Foo(Foo && other)    " 
                << (void*)&other << "/" 
                << (void*)other.data << "/" 
                << (other.data==NULL?"NULL":other.data) << "-->";

      data = other.data;
      other.data = NULL;

      std::cout << (void*)this << "/" 
                << (void*)data << "/" 
                << (data==NULL?"NULL":data) << std::endl;
    }

    Foo& operator= (Foo && other) noexcept
    {
      std::cout << "Foo& operator= (Foo && other)    " 
                << (void*)&other << "/" 
                << (void*)other.data 
                << "/" << (other.data==NULL?"NULL":other.data) << "==>";

      if(this != &other)
      {
        if(data != NULL)
        {
          delete []data;
          data = NULL;
        }

        data = other.data;
        other.data = NULL;
      }

      std::cout << (void*)this << "/" 
                << (void*)data << "/" 
                << (data==NULL?"NULL":data) << std::endl;

      return *this;
    }

    void dump()
    {
      std::cout << (void*)this << "/" 
                << (void*)data << "/" 
                << (data==NULL?"NULL":data) << std::endl;
    }
};
```

# 小结 (4)

本文研究右值引用的本质。它和左值引用有很多相似之处，除了初始化。所以，可以使用类比的方式来理解它。
