---
title: LevelDB的table结构
date: 2020-08-07 17:45:20
tags: [leveldb, block, table]
categories: leveldb
---

LevelDB中Table是一个比较复杂的结构。Block负责有序kv-pair的存储、查询及迭代；Table利用Block构造上一层的结构，包含Data Block, Index Block, Filter Block等，并管理这些Block之间的关系。本篇记录这些琐碎细节。

<!-- more -->

<script type="text/x-mathjax-config">
MathJax.Hub.Config({
tex2jax: {inlineMath: [['$','$'], ['\\(','\\)']]}
});
</script>

<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>

# Block (1)

## Block的结构 (1.1)

一个Block可以看做一块连续的空间，布局是这样的：

- 从基地址`data_`开始，是一个个紧密排列的kv-pair；这些kv-pair被分为多个“段”，叫做restart（后面会讲为什么叫么一个奇怪的名字）。默认每个restart包含16个kv-pair；
- 这些kv-pair之后，即从`data_ + restart_offset_`开始，是一个`uint32_t`数组；每个`uint32_t`元素表示一个restart的偏移（相对于基地址`data_`）；所以前面有多少restart，这个数组里就有多少元素；
- 后面是一个`uint32_t`，表示有多少个restart，即前面数组中有多少元素，叫做`NumRestarts`；

以上是Block在内存中的布局；若被存储到磁盘上，还包括：

- 1字节压缩算法。若没压缩则为`kNoCompression`，默认是`kSnappyCompression`；
- 4字节CRC；

它们是Block的存储属性，而不是Block本身的内容，下图没有画出来。

{% asset_img block-format.jpg Block Format %}

解析一个Block的时候，从Block的末尾开始，解析出`NumRestarts`；再往前跳过`NumRestarts`个`uint32_t`，就得到数组开始的地方`data_ + restart_offset_`；然后就知道一个个restart的偏移了。

## kv-pair结构 (1.2)

一个kv-pair的存储结构如下：

{% asset_img kv-pair-format.jpg KV Format %}

注意：为了节省空间，在key的存储中引入了一个空间优化：一个key和它前一个key的公共前缀被省去而只存储后面不同的部分。例如有4个key："abcd", "abce", "abcexy", "amnp"，存储是这样的：

|shared key长度 |non-shared key长度 |non-shared key数据  |
|---------------|-------------------|--------------------|
|0              |4                  |"abcd"              |
|3              |1                  |"e"                 |
|4              |2                  |"xy"                |
|1              |3                  |"mnp"               |

如果知道前一个完整key，并且知道当前key的这些信息（shared长度，non-shared长度，non-shared数据），就可以构造出当前完整key。

如此存储的好处是公共前缀比较长的情况下能有效节省空间（包括磁盘空间和内存空间），但有一个明显的问题：随机读出一个kv-pair，它的key是不完整的（只包含non-shared部分）；要想构造出完整key，就需要前一个完整key，想要前一个完整key，又需要前一个的前一个完整key，以此类推，直到第一个key（没有任何shared）。这对于两种情况是无法忍受的：1.就是前述随机读；2.反向迭代。

为了解决这个问题，就引入了restart：每个restart的第一个kv-pair的key是完整的，不和前一个restart的最后一个key共享任何前缀。这也是restart名字的由来：重新开始。所以，为了获得完整key，回溯到restart的第一个key即可：

- 随机读：定位到被请求的restart，然后在本restart内从前到后搜索；
- 反向迭代：restart从后到前，同一restart内从前到后；例如：$restart_N$有k1,k2,k3三个kv-pair，第一次迭代从k1找到k3返回k3，第二次从k1找到k2返回k2，第3次返回k1。若继续迭代，则以同样的方式在$restart_{N-1}$内进行。可见，反向迭代的代价还是明显比正向高；然而只要restart别太大，代价还是可以接受的。一个restart默认包含16个kv-pair。

## Block Seek和Iterator (1.3)

Block支持3个Seek：

- `SeekToFirst` ：seek到本Block的第一个restart，然后parse出restart内的第一个kv-pair；
- `Seek(target)`：1. 使用二分查找法定位到$restart_X$，满足：$restart_X$的第一个key（注意第一个key是完整的）< `target`，且$restart_{X+1}$的第一个key >= `target`；2. 从$restart_X$的第一个key（注意它是完整的）开始找第一个 >= `target`的kv-pair；
- `SeekToLast`  ：seek到最后一个restart，然后从第一个kv-pair开始解析到最后一个kv-pair；

注意对于`Seek`和`SeekToLast`，都需要先seek到一个restart，然后从它的第一个key开始解析。原因前面已经说过。

对于正向迭代`Next`，直接解析下一个kv-pair；对于反向迭代`Prev`，则如前所述，在一个restart内，每次都要从第一个key（开始）往后遍历，直到`current_`的前一个kv-pair。一个restart结束，则跳到前一个restart继续。

## BlockBuilder (1.4)

构建一个Block，就是往Block中增加kv-pair，其逻辑在`BlockBuilder::Add()`函数：

- 一个restart未满，则计算当前key和前一个key的公共前缀长度`shared`；
- 否则，一个restart已满，则把当时`buffer_.size()`记下来，存到数组中。注意，这个值是下一个restart的起始offset（第一个restart的起始offset为0，初始化的时候就记录到数组中了）。而当前kv-pair是新restart的第一个，`shared`被置0。
- 把一个kv-pair的5个部分存入`buffer_`：shared key长度，non-shared key长度，value长度，non-shared key数据，value数据。

加入kv-pair之后，`BlockBuilder::Finish()`结束Block构建：

- 把数组中的值，即各个restart的起始offset依次存入`buffer_`；
- 把restart的个数存入`buffer_`；

至此，`buffer_`就是一个完整的Block。

# Table (2)

抛却其内部细节，Block就是一个kv-pair的存储，主要支持迭代操作。基于Block，Table构筑上层结构：

{% asset_img table.jpeg TableFormat %}

它本质上是一个树形结构：

{% asset_img table-tree.jpeg TableTreeFormat %}

## DataBlock (2.1)

一个Table包含多个DataBlock；

- Block内是严格按key排序的，没有重复key；
- Block间也是按key排序的，且两个Block没有交集。

它们是Table的主体，后续IndexBlock和FilterBlock等，都是方便索引、寻找、迭代DataBlock以及其中的kv-pair；

## IndexBlock (2.2)

一个Table包含一个IndexBlock。Index Block内包含一系列index，每个index是一个kv-pair，对应着一个DataBlock：

- key：对应DataBlock和下一DataBlock的key的分隔符，即字符串`x`，满足：对应DataBlock中的最大key <= `x` < 下一DataBlock中的最小key；通常`x` = 对应DataBlock中的最大key；
- val：对应DataBlock的handle，即对应DataBlock在Table中的offset和size；

IndexBlock可以用于粗略定位一个key所在的DataBlock，例如`Table::InternalGet`函数：在`index_block`中seek，找到第一个满足`key >= target`的index，记为$P_N$；$P_N$之前的index对应的DataBlock显然不可能包含`target`，因为它们的最大key < `target`；$P_N$对应的DataBlock的最大key >= `target`，所以`target`只可能在这个DataBlock中。然后就在这个DataBlock内寻找`target`。

IndexBlock还用于遍历本Table，见`Table::NewIterator`：它构造一个两层迭代器，上层迭代IndexBlock得到一个个index；下层迭代每个index对应的DataBlock。

## FilterBlock (2.3)

一个Table包含一个FilterBlock；FilterBlock内包含多个filter；filter用于判定一个key有没有可能存在于一个DataBlock中，默认实现是BloomFilter。

值得注意的是，filter和DataBlock不是一一对应的，多个DataBlock可能共用一个filter。这是没问题的：假如$DataBlock_M$，$DataBlock_{M+1}$，$DataBlock_{M+2}$共用一个filter，现在来判定$key_X$有没有可能存在于$DataBlock_{M+1}$中；若结果为false，那么$key_X$不可能存在于这3个DataBlock中的任何一个，当然也不可能存在于$DataBlock_{M+1}$；所以filter正确性是保证的。然而，它增大了false-positive的可能性。为此，需要控制共用的范围，大约2KB数据共用一个filter：

```cpp
// Generate new filter every 2KB of data
static const size_t kFilterBaseLg = 11;
static const size_t kFilterBase = 1 << kFilterBaseLg;
```

在FilterBlock的构造过程中，每当开始一个新DataBlock，不是立即为前面的DataBlock创建一个filter，而是判断这个新DataBlock是否可以和前面的DataBlock共用filter。判断的方式是，看这个DataBlock的offset是否跨越了2KB单元块，逻辑如下：

```cpp
void FilterBlockBuilder::StartBlock(uint64_t block_offset) {
  // DataBlock的offset位于哪个2KB单元块之内?
  uint64_t filter_index = (block_offset / kFilterBase);
  assert(filter_index >= filter_offsets_.size());
  // 已经位于下一个2KB单元块之内了，需要新建一个filter。
  while (filter_index > filter_offsets_.size()) {
    GenerateFilter();
  }
}
```

正常情况比较容易理解，除了那个`while`。为什么是`while`而不是`if`呢？因为中间可能跳过一些2KB单元块：前一个DataBlock太大了，例如10KB，导致当前DataBlock的offset一下跨到多个2KB单元块之后，这就需要创建空的filter。举个例子，顺便画出FilterBlock的结构：

- DataBlock-0 offset = 0;
- DataBlock-1 offset = 0.5K;
- DataBlock-2 offset = 1.2K; 这个DataBlock很大，直到13K-1处;
- DataBlock-3 offset = 13K;
- DataBlock-4 offset = 14.8K;
- DataBlock-5 offset = 15.1K;

{% asset_img filter-block.jpeg TableTreeFormat %}

- `base_lg_`表示共享范围，默认为2K，这个值就是11（2^11=2K）；
- 因为`0/2K`=`0.5K/2K`=`1.2K/2K`=`0`，所以DataBlock 0,1,2共用filter-0。也就是，若两个DataBlock的offset在同一个2KB单元块内，则它们就共用同一个filter；
- DataBlock-3开始于13K，占用filter-6，所以需要填入5个空filter。实际上它们不存在，只是在`offset_`部分填入5个空索引。这是为了保持`offset_`部分是一个数组，也就是可通过下标来查找：例如，一个DataBlock的offset是13K，那么它对应的就是filter-6，而filter-6的数据在offset_[6]处。如下：

```cpp
bool FilterBlockReader::KeyMayMatch(uint64_t block_offset, const Slice& key) {
  // filter的索引；
  uint64_t index = block_offset >> base_lg_;
  if (index < num_) {
    // start: 根据filter的索引，查offset_数组，得到的filter数据的位置;
    // limit: 下一个filter数据的位置，也就是当前filter数据结束的地方;
    uint32_t start = DecodeFixed32(offset_ + index * 4);
    uint32_t limit = DecodeFixed32(offset_ + index * 4 + 4);
    if (start <= limit && limit <= static_cast<size_t>(offset_ - data_)) {
      Slice filter = Slice(data_ + start, limit - start);
      return policy_->KeyMayMatch(key, filter);
    } else if (start == limit) {
      // Empty filters do not match any keys
      return false;
    }
  }
  return true;  // Errors are treated as potential matches
}
```

## TableBuilder (2.4)

Table构建的逻辑在`TableBuilder::Add`中:

1. 初始时`r->pending_index_entry`为`false`；当kv-pair添加进来，就把它添加到DataBlock和FilterBlock。当越来越多的kv-pair被添加进来，一个DataBlock就满了。
2. DataBlock满了之后，就调用`Flush`把它写到文件中，其中做了几个比较重要的事。
    * a. 构建DataBlock，压缩并写入文件；
    * b. 记录这个DataBlock的index handle，即这个DataBlock在Table中的offset和size；
    * c. 把`r->pending_index_entry`设置为`true`，表示需要在IndexBlock中添加一个index（为本DataBlock建索引）；
    * d. 调用`filter_block->StartBlock`，告诉FilterBlock一个新的DataBlock要开始了。若新的DataBlock的offset跨越了2KB单元块，则为前面累积的kv-pair生成filter，并清空以便重新积累；
3. 满DataBlock之后的第一个kv-pair被添加时，`r->pending_index_entry`为`true`（2.c步），为它建立index，其中index handle已经在2.b步记录了。


## Table的对外接口 (2.5)

主要是以下3个函数：

- `Open`函数从一个文件恢复Table的内存结构；
- `NewIterator`函数返回一个迭代器，迭代本Table存储的所有kv-pair。它是一个两层迭代器，上层在IndexBlock中迭代，得到一个个index（每个index对应一个DataBlock）；下层在DataBlock中迭代，得到一个个kv-pair。当下层迭代器耗尽时，从上层迭代器获取一个index，打开它对应的DataBlock。`Seek`过程也是类似，先Seek上层，得到一个index；然后再Seek这个index对应的DataBlock；
- `InternalGet`是一个私有函数，所以只有friend类`TableCache`可以调用。这个函数和Seek类似，先Seek上层再Seek下层，然后对找到的kv-pair调用给定的回调函数。

# 小结 (3)

本篇介绍Block和Table的结构，以及它们对外提供的接口（主要是Iterator）。为后续version和version set打下基础。
