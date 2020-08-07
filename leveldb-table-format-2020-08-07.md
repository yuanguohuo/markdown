---
title: LevelDB的table结构
date: 2020-08-07 17:45:20
tags: [leveldb, lru, cache]
categories: leveldb
---

LevelDB中table的结构。

<!-- more -->

# Block (1)

## Block的结构 (1.1)

一个Block可以看做一块连续的空间，可以在磁盘上，也可以加载到内存中。它的布局是这样的：

- 从基地址`data_`开始，是一个个紧密排列的kv-pair；这些kv-pair被分为多个“段”，叫做restart（后面会讲为什么叫么一个奇怪的名字）。默认每个restart包含16个kv-pair；
- 这些kv-pair之后，即从`data_ + restart_offset_`开始，是一个`uint32_t`数组；每个`uint32_t`元素表示一个restart的偏移（相对于基地址`data_`）；所以前面有多少restart，这个数组里就有多少元素；
- 最后是一个`uint32_t`，表示有多少个restart，即前面数组中有多少元素，叫做`NumRestarts`；

---------------------------------
      图
---------------------------------

解析一个Block的时候，从Block的末尾开始，解析出`NumRestarts`；再往前跳过`NumRestarts`个`uint32_t`，就得到数组开始的地方`data_ + restart_offset_`；然后就知道一个个restart的偏移了。

为了节省空间，对于kv-pair的key，它和前一个key的共享前缀被省去，而只存储后面不同的部分。例如有4个key："abcd", "abce", "abcexy", "amnp"，存储是这样的：

|共享前缀长度 |非共享长度 |非共享部分  |
|-------------|-----------|------------|
|0            |4          |abcd        |
|3            |1          |e           |
|4            |2          |xy          |
|1            |3          |mnp         |


因为kv-pair需要存储key和value，所以一个kv-pair的存储结构如下：

- shared key长度；
- non-shared key长度；
- value长度；
- non-shared key数据；
- value数据；

---------------------------------
      图
---------------------------------

如果知道前一个完整key，并且知道当前key的这些信息（shared长度，non-shared长度，non-shared数据），就可以构造出当前完整key。

如此存储的好处是节省空间（包括磁盘空间和内存空间），但有一个明显的问题：随机读出一个kv-pair，它的key是不完整的（只包含non-shared部分）；要想构造出完整key，就需要前一个完整key，想要前一个完整key，又需要前一个的前一个完整key，以此类推，直到第一个key（没有任何shared）。这对于两种情况是无法忍受的：1.就是前述随机读；2.反向迭代。

为了解决这个问题，就引入了restart：每个restart的第一个kv-pair的key是完整的，不和前一个restart的最后一个key共享任何前缀。所以，为了获得完整key，回溯到restart的第一个key即可：

- 随机读：定位到被请求的restart，然后在本restart内从前到后搜索；
- 反向迭代：restart从后到前，同一个restart内从前到后；可见，反向迭代`Prev()`还是明显比正向迭代代价高的；

只要restart别太大，代价还是可控的。默认一个restart包含16个kv-pair。

## Block的Iterator (1.2)
## BlockBuilder (1.3)

# Table (2)

## 数据Block (2.1)
## Index Block (2.2)
## Filter Block (2.3)
