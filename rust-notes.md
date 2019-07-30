---
title: rust语法笔记 
date: 2019-07-28 18:34:21
tags: [rust]
categories: rust 
---

看Programing Rust一书记录的一些细节，供查阅。

<!-- more -->

# Array and Vector (1)

Array的语法是`[T; length]`，length必须是常量(编译时能够确定其值)。这和C/C++的静态数组(即`T a[length];`定义的数组，而不是`T *a = new T[length];`定义的数组)很相似。问题：array能在heap上么？

Vector的数据一定在heap上。可以使用以下3中方式创建vector:
- `Vec::new()`: 创建一个空的vector；
- `Vec::with_capacity(cap)`: 创建一个空的vector，但有cap个空间；
- `vec![2, 3]`: 等价于`Vec::with_capacity()`并`push`元素2和3；
Vector变量是一个三元组: (pointer, capacity, length)，和golang的slice有点像。从功能上类似于C/C++的动态数组`T *a = new T[length];`

注意：一些有用的方法(迭代元素，搜索，排序等)不是array和vector的方法，而是slice reference。rust能够把array和vector隐士的转换为slice reference，所以，我们可以直接对array和vector调用这些方法。
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

本来reference和slice没什么关系的，但是书里面有一个不精确的表达：slice是vector或者array的一个片段，记作`[T]`，但slice永远是按引用传递的，即没有`[T]`类型的变量，只有`&[T]`类型的变量。`&[T]`类型的变量本该叫做slice reference，但是因为不存在`[T]`的变量，所以，`&[T]`常常被简单地叫做slice。为了精确，本文中把`&[T]`叫做**slice reference**。所以，我们要看的是reference和slice reference的区别。

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
