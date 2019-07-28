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


# Reference and Slice

本来reference和slice没什么关系的，但是书里面有一个不精确的表达：slice是vector或者array的一个片段，记作`[T]`，但slice永远是按引用传递的，即没有`[T]`类型的变量，只有`&[T]`类型的变量。`&[T]`类型的变量本该叫做slice reference，但是因为不存在`[T]`的变量，所以，`&[T]`常常被简单地叫做slice。为了精确，本文中把`&[T]`叫做**slice reference**。所以，我们要看的是reference和slice reference的区别。

|类型               |指向的内容         |Owning         |值                    |
|-------------------|-------------------|---------------|----------------------|
|reference          |单个对象           |No             |地址                  |
|slice reference    |连续的多个对象     |No             |(地址,对象数)二元组   |

这样理解是不对的：slice是(地址,对象数)二元组(受golang的slice影响，golang的slice是三元组)，slice reference是指向二元组的指针。
和C/C++类比的话，reference `s`相当于`T *s = new T;`而slice reference `s`相当于`T *s = new T[length];`。只是在C/C++中指针`s`不区分这两者，而rust中是区分的: 前者是一个单独的地址；后者是(地址,对象数)。


