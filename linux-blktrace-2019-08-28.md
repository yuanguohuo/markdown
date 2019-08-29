---
title: linux blktrace
date: 2019-08-28 20:48:12
tags: [linux,blktrace]
categories: linux 
---

[iostat](http://www.yuanguohuo.com/2019/08/17/linux-iostat/)可以获取到特定device的IO请求信息，如read/write数，合并数，request size，**等待时间**等。但这些都是基于device统计的，我们无法获取到基于IO的详细信息，特别是**等待时间**都花在哪里了？比如多少时间是在IO scheduler中排队，从发送到device driver到完成又是多少时间。blktrace可以提供这些信息。

<!-- more -->

# blktrace简介 (1)

blktrace是block层的trace机制，它可以跟踪IO请求的生成、进入队列、发到driver以及完成等事件。blktrace包含3个组件：

- 一个内核模块
- 一个把内核模块跟踪到的IO信息导出到用户态的工具(blktrace命令)
- 一些分析和展示IO信息的工具(blkparse, btt命令)

所以广义的blktrace包含这3个部分，狭义的blktrace只是值blktrace命令。本文介绍广义的blktrace：包括使用blktrace命令导出IO信息，然后使用blkparse和btt分析展示。blktrace命令通过 debug file system 从内核导出数据：

- blktrace从kernel接收数据并通过debug file system的buffer传递到用户态；debug file system默认是/sys/kernel/debug，可以使用blktrace命令的`-r`选项来覆盖。buffer的大小和数量默认是512KiB和4，可以通过`-b`和`-n`来覆盖。blktrace运行的的时候，可以看见debug file system里有一个block/{device}(默认是/sys/kernel/debug/block/{device})目录被创建出来，里面有一些trace{cpu}文件。
- blktrace默认地搜集所有trace到的事件，可以使用blktrace命令的`-a`选项来mask。
- blktrace把从内核接收到的数据写到当前目录，格式是`{device}.blktrace.{cpu}`。例如`blktrace -d /dev/sdc`会生成sdc.blktrace.0, sdc.blktrace.1, ... sdc.blktrace.N-1个文件，N为cpu数。也可使用`-o`选项来自定义`{device}`部分，这方便和blkparse结合使用：blktrace的`-o`参数对应blkparse的`-i`参数。
- blktrace默认地会一直运行，直到被`ctrl-C`停掉。可以使用`-w`选项来指定运行时间，单位是秒。

blktrace会区分两类请求:

* 文件系统请求(fs requests)：通常是用户态进程产生的，读写disk的特定位置和特定大小的数据。当然，也有可能由内核产生：flust脏页或sync文件系统的元数据(super block或journal block等)。
* SCSI命令(pc requests)：blktrace直接把SCSI命令发送到用户态，blkparse可以解析它。

# IO流程与event (2)

一个IO可能发起于:

- Filesystem (PageCache)
- Application
- remap (md/dm). 这种情况，blktrace会trace到，并为IO生成一个A(remap)事件。

IO发起之后，主要会经历以下阶段(事件)：

* Q: queued. request尚未生成，只是queue到指定的位置(不清楚)。
* G: get request. 分配一个`struct request`实例，生成request。
* I: inserted. request被发送到IO scheduler。
* D: issued. IO scheduler中的请求被发送到driver。
* C: complete. 发送到driver中的请求已经完成了。这个事件会描述IO的初始sector和size(位置和大小)，以及是成功还是失败。

其中阶段I(inserted)可能被合并取代，也就是说，不是作为一个独立的request被发送到IO scheduler，而是合并到已经处于IO scheduler中的某个request中。

* M: back merge. 当前请求被合并到已存在于IO scheduler中的某个请求之后。
* F: front merge. 当前请求被合并到已存在于IO scheduler中的某个请求之前。

另外，一个请求还可能经历plug和unplug阶段。在I(inserted)之后D(issued)之前，即在IO scheduler中，若一个request到来的时候，device的队列为空，linux会plug这个队列(即堵住队列的出口)一段时间，期待有更多的request进来(这样可以合并？)。当队列中有一定数量的request之后，或者等待超时，linux就会unplug(打开队列出口，可以从IO scheduler出去进入driver)这个队列。

* P: plug. request进来的时候，device的队列为空，plug直到一定数量的request进来，或超时。
* U: unplug. 一定数量的request进来，或超时，unplug队列。

所以，一个完整的IO流程是：

[!IO流程](io-flow-1.jpg)

其中灰色的阶段（事件）可能经历也可能不经历（多数情况不经历）；而红色和绿色阶段（事件）是多种可能，且最可能是绿色。所以，最典型的IO流程是：

[!简化的IO流程](io-flow-2.jpg)


# 使用blktrace命令导出原始数据 (3)

如前所述，运行blktrace可以导出内核组件trace到的IO events，并把信息写到当前目录下。常用的选项有：

- `-d`: 指定被trace的device。
- `-o`: 指定生成文件名`{device}.blktrace.{cpu}`的`{device}`部分。不指定时，默认为形如`sda`,`sdb`的设备名。这个通常用于和blkparse配合使用：1. 实时trace：blktrace的`-o`选项为`-`，blkparse的`-i`选项也为`-`；2. 事后处理：blktrace的`-o`选项为`x`，blkparse的`-i`选项也必须为`x`(若blktrace不指定`-o`选项，则使用默认的形如`sda`的设备名，此时blkparse的`-i`选项也必须是形如`sda`的设备名)。
- `-a`: mask，也就是让blktrace只trace指定的IO。

支持的mask有：

```
barrier: barrier attribute
complete: completed by driver
fs: requests
issue: issued to driver
pc: packet command events
queue: queue operations
read: read traces
requeue: requeue operations
sync: synchronous attribute
write: write traces
notify: trace messages
drv_data: additional driver specific trace
```

例如，`-a write`过滤写事件；`-a sync`过滤sync事件。多个`-a`选项是**并**的关系。一般情况下，blktrace不用指定mask，而是把所有的事件都trace下来。blkparse有同样的`-a`选项，可以使用它来指定只解析某些事件。当然，影响性能的时候除外。



# 使用blkparse命令分析数据 (4)
