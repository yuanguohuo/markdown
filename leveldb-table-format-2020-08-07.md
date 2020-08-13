---
title: LevelDB的table结构
date: 2020-08-07 17:45:20
tags: [leveldb, block, table]
categories: leveldb
---

LevelDB中table的结构。

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

一个Block可以看做一块连续的空间，可以在磁盘上，也可以加载到内存中。它的布局是这样的：

- 从基地址`data_`开始，是一个个紧密排列的kv-pair；这些kv-pair被分为多个“段”，叫做restart（后面会讲为什么叫么一个奇怪的名字）。默认每个restart包含16个kv-pair；
- 这些kv-pair之后，即从`data_ + restart_offset_`开始，是一个`uint32_t`数组；每个`uint32_t`元素表示一个restart的偏移（相对于基地址`data_`）；所以前面有多少restart，这个数组里就有多少元素；
- 最后是一个`uint32_t`，表示有多少个restart，即前面数组中有多少元素，叫做`NumRestarts`；

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
- 否则，一个restart已满，则把当时`buffer_.size()`记下来，存到数组中。注意，这个值是下一个restart的起始offset（第一个restart的起始offset为0，初始化的时候就记录到数组中了）。而当前kv-pair是新restart的第一个，故`shared`为0。
- 把一个kv-pair的5个部分存入`buffer_`：shared key长度，non-shared key长度，value长度，non-shared key数据，value数据。

加入kv-pair之后，`BlockBuilder::Finish()`结束Block构建：

- 把数组中的值，即各个restart的起始offset依次存入`buffer_`；
- 把restart的个数存入`buffer_`；

至此，`buffer_`就是一个完整的Block。

# Table (2)

## 数据Block (2.1)
## Index Block (2.2)
## Filter Block (2.3)
