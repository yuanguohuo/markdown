---
title: Rust语法笔记 
date: 2019-07-28 18:34:21
tags: [rust]
categories: rust 
---

看Programing Rust一书记录的一些细节，供查阅。

<!-- more -->

# Array and Vector (1)

Array的语法是`[T; length]`，length必须是常量(编译时能够确定其值)。这和C/C++的静态数组(即`T a[length];`定义的数组，而不是`T *a = new T[length];`定义的数组)很相似。问题：array能在heap上么？答：不能，array只能在栈上，就像C/C++的局部静态数组一样。

Vector的数据一定在heap上。可以使用以下3中方式创建vector:
- `Vec::new()`: 创建一个空的vector；
- `Vec::with_capacity(cap)`: 创建一个空的vector，但有cap个空间；
- `vec![2, 3]`: 等价于`Vec::with_capacity()`并`push`元素2和3；

Vector变量是一个三元组: (pointer, capacity, length)，和golang的slice有点像。从功能上类似于C/C++的动态数组`T *a = new T[length];`

注意：一些有用的方法(迭代元素，搜索，排序等)不是array和vector的，而是slice reference的。rust能够把array和vector隐士的转换为slice reference，所以，我们可以直接对array和vector调用这些方法。
```
let mut chaos = [3, 5, 4, 1, 2];
chaos.sort();
assert_eq!()chaos, [1, 2, 3, 4, 5];

let mut v = vec!["a man", "a plan", "a cancel", "panama"];
v.reverse();
assert_eq!(v, vec!["panama", "a cancel", "a plan", "a man"]);
```
这里sort和reverse都是slice reference的方法。


# Reference and Slice (2)

Slice是vector或者array的一个片段，记作`[T]`，但slice永远是按引用传递的，换言之，不存在slice类型（`[T]`）的变量，只存在slice reference类型（`&[T]`）的变量。因为前者不存在，所以书中常常把`&[T]`简单地叫做slice类型。为了精确起见，本文还是把`&[T]`叫做**slice reference类型**。我们要看的是reference和slice reference的区别。

|类型               |指向的内容         |Owning         |值                    |
|-------------------|-------------------|---------------|----------------------|
|reference          |单个对象           |No             |地址                  |
|slice reference    |连续的多个对象     |No             |(地址,对象数)二元组   |

这样理解是不对的：slice是(地址,对象数)二元组(受golang的slice影响，golang的slice是三元组)，slice reference是指向二元组的指针。
和C/C++类比的话，reference `s`相当于`T *s = new T;`而slice reference `s`相当于`T *s = new T[length];`。只是在C/C++中指针`s`不区分这两者，而rust中是区分的: 前者是一个单独的地址；后者是(地址,对象数)。


# Char and String (3)

和C/C++不同的char不同，而和golang的Rune非常类似，rust的char是4个字节，值是字符的unicode码点；而string不是char的序列(这样的话，一个char占4个字节，比较浪费空间)，而是UTF-8编码的字符序列(这样，ASCII只占一个字节，其他的字符有占2-4个字节，总体上节约空间)；

String有以下3中：

- a. string字面量

存在全局数据区

```
"\"Hello world\" is my first program."          //escape
r"\d+(\.\d+)*"                                  //no escape
r###"no escape, and " is a legal character"###
```

- b. string slice (&str)

```
let s1 = "你好ABC";             // &str, 引用全局数据区，里面必须是合法UTF-8字符序列；
let s2 = "你好XYZ".to_string(); // std::string::String, 数据在堆区(从全局数据区拷贝到堆区)，里面是合法UTF-8字符序列；
let s3 = &s2[0..3];             // &str, 引用堆区的数据s2的空间，里面必须是合法UTF-8字符序列；
let s4 = &s2[0..2];             // 编译不通过，因为里面不是合法UTF-8字符序列。"你"是4个字节，取其前3字节不是合法UTF-8字符；
```

这里的s1和s3都是&str变量。如何理解&str呢？其实&str可以理解为&[u8]，即u8的slice reference(见第2节)，唯一要求是u8序列必须可以解析为UTF-8字符序列。也就是说，&str是(地址, 对象数)二元组。另外，&str的`len()`返回的是字节数，而不是字符数。

- c. String

String可以理解为Vec<u8>，唯一要求是u8序列必须可以解析为UTF-8字符序列。也就是说，String的变量是一个(pointer, capacity, length)三元组，其数据一定在heap上。另外，&str的`len()`返回的也是字节数，而不是字符数。

通过下表可以看出: String和&str的关系，非常类似于Vec<u8>和&[u8]的关系。

|                                     |Vect<u8>                |String                  |
|-------------------------------------|------------------------|------------------------|
|automatically frees buffers          |Yes                     |Yes                     |
|growable                             |Yes                     |Yes                     |
|`::new()` and `::with_capacity()`    |Yes                     |Yes                     |
|`.reserve()` and `.capacity()`       |Yes                     |Yes                     |
|`.push()` and `.pop()`               |Yes                     |Yes                     |
|range syntax v[start..stop]          |Yes, returns &[u8]      |Yes, returns &str       |
|automatic conversion                 |&Vec<u8> to &[u8]       |&String to &str         |
|inherits methods                     |from &[u8]              |from &str               |

# Ownership, move and lifetime (4)

对于变量的生命周期，目前主流语言在a.由程序员控制和b.自动gc中二选其一。前者如C/C++，或者如Java和Go。前者容易出bug：dangling pointer, double free等；后者程序员失去了对变量生命周期的控制，有的时候很难搞清楚该释放的内存为什么没有释放。释放时机不可依赖，释放造成stw等问题也比较棘手(还有，内存以外的其他资源，例如网络连接和文件句柄的生命周期，还是需要程序员控制。理论上讲，内存和网络连接及文件句柄都是资源，采取两种方式去管理似乎也不够优雅)。总之，这两种方式都不够理想，理想的是：程序员能够控制变量的生命周期并且语言应该是安全的(没有dangling pointer, double free等问题)。这就是rust的努力方向。

## Ownership (4.1)

- 每一个值有唯一一个owner，但一个值可以own其他它个值。当owner生命周期结束，它own的所有值的生命周期都结束；
- 值之间的owning关系形成一棵树；
- rust程序中的每个值都属于唯一的某一棵树，树的根是某个变量；
- 当根变量的生命周期结束，树中所有值的生命周期都结束，被drop掉；

可见ownership的规则相当的严苛，但有3个途径来放松这些规则：move，Rc/Arc以及borrowing reference。

## Move (4.2)

和C++11中的move类似，只不过C++11中只有右值(prvalue和xvalue)才能被move，若左值也可以被move，那么它出现在等号的右边(或作为函数参数，作为函数返回值)一次，就改变了(丢失了资源)，很不符合常理：要找一个变量可能发生改变的地方，只找它在等号左边的出现就行了，谁会管它在等号右边的出现呢？

然而，rust就是这么干的：不分左值右值，都能被move。变量出现在等号的右边也会发生改变(丢失自己的资源)。好在，rust会在编译时做检查，确保丢失了资源(被move，变成uninitialized状态)的变量不能再被使用。当然，作为函数参数或函数返回值和出现在等号右边是一样的。

好吧，事情也没那么绝对：rust的类型分为Move和Copy两类，上面说的这些都是真对Move类型的。只是rust中绝大多数类型是Move，小部分是Copy。
- Copy类型：逐位拷贝就足够的，drop时不需要额外操作的(例如unlock，释放堆空间，关闭文件句柄，断开连接等)。例如：整数类型，浮点类型，`char`，`bool`；以及这些类型组成的tuple好固定大小的array。还有，若自定义的struct或enum只包含Copy类型的字段，程序员可以用`#[derive(Copy, Clone)]`把它声明为Copy类型的(注意：默认不是Copy的而是Move的)。
- Move类型：其他。例如：`String`，`Box<T>`，`File`，`MutexGuard`等。程序员自定义类型默认是Move的，即使它只包含Copy类型的字段。
本质上讲：在rust中，赋值(以及参数传递和函数返回)，都是按字节拷贝(例如拷贝整数，Vec三元组等)，不同之处在于：对于Copy类型，赋值(以及参数传递和函数返回)之后，源变量还保持为initialized，而对于Move类型源变量变成了uninitialized。

另外，C++中赋值可以被程序员重载，里面可重可轻，重的如深拷贝大块内存甚至分配别的资源，轻的如move，这使得赋值，参数传递，函数返回等基本操作的代价不可预测。相反，rust中这些基本操作都是内存的按字节拷贝(拷贝的量比较小，例如一个整数，一个Vec三元组等)，代价比较明晰；而重的操作，如Clone，是显性的。

注意：

```
    let mut str_vec = Vec::new();
    for i in 101..106 
    {
        str_vec.push(i.to_string());
    }

    //let _third = str_vec[2];     //编译不通过
    
    let mut i_vec = Vec::new();
    for i in 1..16
    {
        i_vec.push(i)
    }

    let _forth = i_vec[3];         //OK，没问题
```

对于Move类型，indexed对象不能被move，否则导致Vec内有uninitialized空洞。可以用`pop`, `swap_remove`, `std::mem::replace`等操作。对于Copy类型则没有这个问题。

```
    let v = vec![
                    "hello".to_string(),
                    "rust".to_string(),
                    "programmers".to_string()
                ];

    for mut s in v 
    {
        s.push('!');
        println!("{}", s);
    }
```

这段代码的特殊之处在于：不是v的元素一个个的被move到s，而是v直接被move到一个隐藏变量中(v变成uninitialized)，然后这个隐藏变量的元素被一个个move到s(s是owner所以能够修改字符串)。在循环的过程中，这个隐藏变量是有空洞的(部分元素被move)，但这个状态是不可见的。

## Rc and Arc (4.3)

## Borrow reference (4.4)

### 类型的lifetime (4.4.1)

### 变量的lifetime (4.4.2)

