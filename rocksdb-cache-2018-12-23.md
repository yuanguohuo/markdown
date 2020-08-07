---
title: RocksDB中的LRUCache
date: 2018-12-23 19:10:05
tags: [rocksdb, lru, cache]
categories: leveldb 
---

本文简要介绍一下RocksDB中LRUCache的实现。

<!-- more -->

# LRUHandle结构体 (1)

LRUHandle结构是对cache元素的封装，即cache中存的是LRUHandle的实例。它实现了一个key-value对。它的字段的含义显而易见，这里只说明以下几点：

* 它没有使用模板，value通过void指针(`void*`)来表示；
* key是一个char数组(`char key_data[]`)，但结构体中只存储第一个char，后续部分紧跟着结构体。所以，一方面需要一个成员表示key的长度(`size_t key_length`)，另一方面要求`char key_data`是结构体的最后一个数据成员；
* Free()是实例的destructor，但它不是通过`delete this`来实现(调用析构函数`~LRUHandle`)，而是调用使用者提供的`deleter`来销毁`value`，并通过`delete[] reinterpret_cast<char*>(this)`来销毁实例本身。这是因为：1.本类并不知道如何销毁`value`(因为它是void指针)，需要使用者提供`deleter`; 2.实例本身是通过`reinterpret_cast<LRUHandle*>(new char[...])`分配的(见`LRUCacheShard::Insert`)，需要对应的`delete[] char*`来销毁。注意：`delete[] reinterpret_cast<char*>(this)`不调用this的析构函数。


# LRUHandleTable类 (2)

这个类更简单，就是实现一个hash表。不赘述。

# LRUCacheShard类 (3)

这个类比较麻烦。LRUCache包含多个shard，每个shard就是一个本类的实例。也就是说，把这个类搞清楚，LRUCache就清楚了。

## 内存对齐 (3.1)

首先要说的是，这个类使用c++11提供的`alignas`关键字对齐到`CACHE_LINE_SIZE`。shard的实现，一般都会对齐到`CACHE_LINE_SIZE`，因为高频访问才需要引入shard。高频情况下，对齐到`CACHE_LINE_SIZE`更高效。

## LRU列表与LRUHandle的状态 (3.2)

LRUCacheShard本质是一个LRUHandleTable的实例加上一个LRU列表。常见的LRU cache的实现方式是一个hash，然后用一个LRU列表把hash中的所有元素串联起来。一个元素被访问就被移动到LRU列表的头部(这样热的元素在头部，冷的元素在尾部)，淘汰时从尾部移除。然而，这里的实现方式稍有不同：
- LRU列表`lru_`并不串联所有的元素(一个元素就是一个LRUHandle的实例)，而只串联**可以被淘汰**的元素；
- 什么是**可以被淘汰**的元素呢？答案是满足这两个条件的元素：1.被cache引用(在cache里，即LRUHandle的InCache()==true); 2.只被cache引用(即LRUHandle的refs==1)。所以，如果一个元素的refs为1，但这个引用不是由cache持有，它就不在LRU列表里(元素都不在cache里，谈何淘汰呢)；或者，被cache引用，同时也被外部使用者引用(refs>1)，它也不在LRU列表里(虽然元素在cache里，但还正被使用着，不能淘汰)。总之，LRU用来记录cache中的可以被淘汰的元素，当cache的大小超过设定值时，就从LRU中找元素进行淘汰。

基于上面的描述，可以发现一个元素(LRUHandle实例)可能处于以下3种状态之一：
* 状态A: 同时被cache和外部使用者引用(InCache()==true并且refs>1)；
* 状态B: 只被cache引用(InCache()==true并且refs==1)；
* 状态C: 只被外部使用者引用(InCache()==false)；

可见：只有处于状态B的元素才在LRU列表里。我们看一个典型的元素的生命周期：
- user-a调用`LRUCacheShard::Insert`创建元素并返回引用：这时元素有两个引用(refs==2)，一个由cache持有，另一个由user-a持有。元素在cache中但不在LRU列表里；
- user-b调用`LRUCacheShard::Lookup`得到这个元素，refs==3；元素在cache中但不在LRU列表里；
- user-a调用`LRUCacheShard::Release`释放这个元素，refs==2；元素在cache中但不在LRU列表里；
- user-b调用`LRUCacheShard::Release`释放这个元素，refs==1；这时把元素插入LRU列表，表示在必要时可以淘汰它；
- user-c调用`LRUCacheShard::Lookup`得到这个元素，refs==2；从LRU列表中移除。元素在cache中但不在LRU列表里；
- user-c调用`LRUCacheShard::Release`释放这个元素，refs==1；这时把元素插入LRU列表，表示在必要时可以淘汰它；
- user-d新插入一个元素，导致cache的大小超过预置值，`LRUCacheShard::EvictFromLRU()`被调用，淘汰这个元素(从cache和LRU列表中移除，然后其Free函数被调用)；

当一个元素被cache和外部使用者同时引用时(InCache()==true并且refs==2)，外部使用者可以直接调用`LRUCacheShard::Erase`。这时：元素直接从cache中移除(InCache()==false并且refs==1)，注意它这时候本来就不在LRU中，所以无需从LRU移除。另外注意，cache的usage中还记着它的charge，当外部使用者调用`LRUCacheShard::Release`时，refs==0，才从usage中扣除。

当一个元素在cache中(InCache()==true)，若被另一个key相同的元素覆盖时，和Erase类似：元素从cache中移除。若之前只有cache引用(在LRU里)，也从LRU里移除，并调用Free()销毁；否则不在LRU里，说明还有外部引用，refs--，处于状态C。

## LRU的高低优先级 (3.3)

LRU是一个双向链表，其中由一个dummy元素`lru_`，可以把它看作双向链表的尾。另外还有一个`lru_low_pri_`指针，它和`lru_`把这个双向链表分成两部分：一部分是高优先级的；另一部分是低优先级的。淘汰时，从低优先级的开始。什么样的是高优先级的呢？一种是显示设置为为高优先级(`LRUHandle::SetInHighPriPool(true)`)，另一种是被命中过的元素(`LRUHandle::SetHit`)重新进入LRU时。

## LRUCache (4)

如前所述，LRUCache就是把多个LRUCacheShard组合起来。不细说了。
