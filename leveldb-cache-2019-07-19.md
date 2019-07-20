---
title: Leveldb中的LRUCache
date: 2019-07-19 19:00:32
tags: [leveldb, lru, cache]
categories: rocksdb
---

本文简要介绍一下Leveldb中LRUCache的实现。

<!-- more -->

# LRUHandle结构体 (1)

LRUHandle结构是对cache元素的封装，即cache中存的是LRUHandle的实例。它实现了一个key-value对。它的字段的含义显而易见，这里只说明以下几点：

* 它没有使用模板，value通过void指针(`void*`)来表示；
* key是一个char数组(`char key_data[]`)，但结构体中只存储第一个char，后续部分紧跟着结构体。所以，一方面需要一个成员表示key的长度(`size_t key_length`)，另一方面要求`char key_data`是结构体的最后一个数据成员；
* `next_hash`字段：在hash表中，hash值相同的LRUHandle对象(即同一吊桶里的LRUHandle对象)通过`next_hash`链接；
* 构造：不重载构造函数，而是直接malloc，并cast成`LRUHandle*`：

```cpp
LRUHandle* e =
    reinterpret_cast<LRUHandle*>(malloc(sizeof(LRUHandle) - 1 + key.size()));
e->value = value;
e->deleter = deleter;
e->charge = charge;
e->key_length = key.size();
...
```

* 析构：不重载析构函数`~LRUHandle`，而是: 1.调用`deleter`来销毁`value`; 2. free `LRUHandle`实例本身；这是因为：1.本类并不知道如何销毁`value`(因为它是void指针)，需要使用者提供`deleter`; 2.实例本身是通过malloc分配的(如前所述)，需要对应的free来销毁:

```cpp
// LRUHandle* e
(*e->deleter)(e->key(), e->value);
free(e);
```

# HandleTable类 (2)

这个类更简单，就是实现一个hash表。
`LRUHandle** list_`字段 : 吊桶数组。一个数组，数组里的元素是`LRUHandle*`；
`uint32_t length_`字段  : 吊桶数组的长度。首次rehash(Resize)时，初始化为4，以后每次rehash(Resize)时乘以2；
`uint32_t elems_`字段   : hash表中实际存储的元素个数；
`Insert`函数            : 若存在指定key的`LRUHandle`对象，则替换它；若不存在则插入(并`++elems_`，若增加后饱和，则触发rehash(Resize))；
`Remove`函数            : 若存在指定key的`LRUHandle`对象，则删除它(并`--elems_`)；
`Resize`函数            : rehash；

# LRUCache类 (3)

利用hash表(即`HandleTable`)实现的一个lru cache；这是一个单shard的lru cache。多shard的lru cache是`ShardedLRUCache`(见第4节)。注意，rocksdb中也有单shard lru cache和多shard lru cache，但它们的名字不一样：

* leveldb的LRUCache = rocksdb的LRUCacheShard：单shard的lru cache；
* leveldb的ShardedLRUCache = rocksdb的LRUCache：多shard的lru cache；

## 内存对齐 (3.1)

rocksdb的LRUCacheShard通过`alignas`关键字对齐到`CACHE_LINE_SIZE`，而leveldb的这个`LRUCache`并没有对齐；

## LRU列表与LRUHandle的状态 (3.2)

LRUCache本质是一个HandleTable的实例(`HandleTable table_`)加上一个LRU列表(`LRUHandle lru_`)。常见的LRU cache的实现方式是一个hash，然后用一个LRU列表把hash中的所有元素串联起来。一个元素被访问就被移动到LRU列表的头部(这样热的元素在头部，冷的元素在尾部)，淘汰时从尾部移除。然而，这里的实现方式稍有不同：LRU列表`lru_`并不串联所有的元素(一个元素就是一个LRUHandle的实例)，而只串联**可以被淘汰**的元素。什么是**可以被淘汰**的元素呢？

首先，LRUCache有: 
* hash表`HandleTable table_`
* LRU列表`LRUHandle lru_`
* 使用中的元素列表`LRUHandle in_use_`

其次，LRUHandle有: 

* LRUHandle对象是否在LRUCache中`bool in_cache`
* 引用计数`uint32_t refs`

LRUHandle可能处于以下3种状态之一：

* 状态A: 在cache中(被`table_`引用)，并且有外部引用。即：`in_cache=true && refs>1`
* 状态B: 在cache中(被`table_`引用)，但没有外部引用。即：`in_cache=true && refs=1`
* 状态C: 没有在cache中，即`in_cache=false`

状态C的元素不考虑，因为它根本不在LRUCache中。考虑在LRUCache中的元素，状态A的在`in_use_`中，状态B的在`lru_`中；。状态B的元素，也就是`lru_`中的元素，就是**可以被淘汰**的元素。

我们看一个典型的元素的生命周期：
- user-a调用`LRUCache::Insert`创建元素并返回引用：这时元素有两个引用(refs==2)，一个由cache持有，另一个由user-a持有。元素在`in_use_`中，不在`lru_`中；
- user-b调用`LRUCache::Lookup`得到这个元素，refs==3；元素仍然在`in_use_`中，不在`lru_`中；
- user-a调用`LRUCache::Release`释放这个元素，refs==2；元素仍然在`in_use_`中，不在`lru_`中；
- user-b调用`LRUCache::Release`释放这个元素，refs==1；把元素从`in_use_`中移除，并插入`lru_`中，表示在必要时可以淘汰它；
- user-c调用`LRUCache::Lookup`得到这个元素，refs==2；把元素从`lru_`中移除，并插入`in_use_`中，表示元素正在使用，不可以淘汰它；
- user-c调用`LRUCache::Release`释放这个元素，refs==1；把元素从`in_use_`中移除，并插入`lru_`中，表示在必要时可以淘汰它；
- user-d新插入一个元素，导致cache的大小超过预置值，淘汰这个元素：首先从`table_`中移除，从`lru_`中移除，`in_cache`设置为false，然后`refs--`变成0，触发析构(见第1节LRUHandle的析构)；

当一个元素被cache和外部使用者同时引用时(`in_cache=true && refs=2`)，外部使用者可以直接调用`LRUCache::Erase`。这时：元素直接从cache中移除并从`in_use_`中移除(`in_cache=false && refs=1`)。此元素从此脱离cache，它还有一个引用，那是外部使用者持有的。外部使用者再调用`LRUCache::Release`时(已经Erase了，还要Relase是不是有点奇怪？Release要和Insert/Lookup配对使用不管是否Erase？)，refs--变成0，触发析构(见第1节LRUHandle的析构)。

当一个元素在cache中(可能在`lru_`中，也可能在`in_use_`中)，若被另一个key相同的元素覆盖：1. 若在`lru_`中：从`lru_`中移除，把`in_cache`设置为false，然后refs--变成0，触发析构；1. 若在`in_use_`中：从`in_use_`中移除，和Erase类似。

# ShardedLRUCache (4)

如前所述，ShardedLRUCache就是把多个LRUCache组合起来。不细说了。
