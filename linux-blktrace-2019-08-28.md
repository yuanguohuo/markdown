---
title: linux blktrace
date: 2019-08-28 20:48:12
tags: [linux,blktrace]
categories: linux 
---

[iostat](http://www.yuanguohuo.com/2019/08/17/linux-iostat/)可以获取到特定device的IO请求信息，如read/write数，合并数，request size，**等待时间**等。但这些都是基于device统计的，我们无法获取到基于IO的详细信息，特别是**等待时间**都花在哪里了？比如多少时间是在IO scheduler中排队，从发送到device driver到完成又是多少时间。blktrace可以提供这些信息。

<!-- more -->

<script type="text/x-mathjax-config">
MathJax.Hub.Config({
tex2jax: {inlineMath: [['$','$'], ['\\(','\\)']]}
});
</script>

<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>

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

简单情况下，可以**实时解析**：一边导出IO事件，一边解析。

```
blktrace -d /dev/sde -o - | blkparse -i -
```

更常见的是**事后解析**：先用blktrace把trace到的IO事件导出来保存在文件中，然后使用`blkparse`进行解析。

```
blktrace -d /dev/sde
blkparse -i sde 
```

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

下图粗略显示一个IO的流程。左边是比较详细，其中灰色的阶段（事件）可能经历也可能不经历（多数情况不经历）；而红色和绿色阶段（事件）是多种可能，且最可能是绿色。所以，右边是典型的IO流程。

<div align=center>![IO流程](io-flow-merge.jpg)

# 使用blktrace导出原始数据 (3)

如前所述，运行blktrace可以导出内核组件trace到的IO events，并把信息写到当前目录下。常用的选项有：

- `-d`: 指定被trace的device。
- `-o`: 指定生成文件名`{device}.blktrace.{cpu}`的`{device}`部分。不指定时，默认为形如`sda`,`sdb`的设备名。这个通常用于和blkparse配合使用：1. **实时解析**：blktrace的`-o`选项为`-`，blkparse的`-i`选项也为`-`；2. **事后解析**：blktrace的`-o`选项为`x`，blkparse的`-i`选项也必须为`x`(若blktrace不指定`-o`选项，则使用默认的形如`sda`的设备名，此时blkparse的`-i`选项也必须是形如`sda`的设备名)。另外，可以使用多个`-d`选项来同时trace多个device，在这种情况下，`-o`无法使用（会报错）；然而，通常情况下，我们也不会同时trace多个device。
- `-a`: mask，也就是让blktrace只trace指定的IO。
- `-w`: 运行时间，单位是秒。若不指定，一直运行，直到被Ctrl-C停止。

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

例如，`-a write`过滤写事件；`-a sync`过滤sync事件。多个`-a`选项是**并**的关系。一般情况下，blktrace不用指定mask，而是把所有的事件都trace下来。blkparse有同样的`-a`选项，可以使用它来指定只解析某些事件，这可以提供更大的灵活性：trace时把全信息保存下来，然后通过mask按需解析不同事件。当然，影响性能的时候除外。所以，通常在**事后处理**模式下，只需`-d`和`-w`选项，例如，在一个32核的服务器上trace设备/dev/sde：

```
# blktrace -d /dev/sde -w 60

# ll -h
total 3.7M
-rw-r--r-- 1 root root    0 Aug 30 11:19 sde.blktrace.0
-rw-r--r-- 1 root root  472 Aug 30 11:19 sde.blktrace.1
-rw-r--r-- 1 root root    0 Aug 30 11:19 sde.blktrace.10
-rw-r--r-- 1 root root    0 Aug 30 11:19 sde.blktrace.11
...
-rw-r--r-- 1 root root    0 Aug 30 11:19 sde.blktrace.19
-rw-r--r-- 1 root root 620K Aug 30 11:19 sde.blktrace.2
-rw-r--r-- 1 root root 621K Aug 30 11:19 sde.blktrace.20
-rw-r--r-- 1 root root  472 Aug 30 11:19 sde.blktrace.21
...
-rw-r--r-- 1 root root    0 Aug 30 11:19 sde.blktrace.29
-rw-r--r-- 1 root root  360 Aug 30 11:19 sde.blktrace.3
-rw-r--r-- 1 root root    0 Aug 30 11:19 sde.blktrace.30
-rw-r--r-- 1 root root    0 Aug 30 11:19 sde.blktrace.31
-rw-r--r-- 1 root root 620K Aug 30 11:19 sde.blktrace.4
...
-rw-r--r-- 1 root root    0 Aug 30 11:19 sde.blktrace.9
```

# 使用blkparse分析数据 (4)

blktrace导出的信息是二进制的不可读的，需要blkparse来解析。

blkparse的输出包含两个部分：1. IO事件；2. summary

* **IO事件**

<div align=center>![IO事件](blkparse-output-1.png)

* 第1列：`8,0`是设备号，major和minor；
* 第2列：`3`是cpuid；
* 第3列：`11`是序列号；
* 第4列：`0.009507758`是时间偏移；
* 第5列：`697`是发起IO的进程的PID；
* 第6列：`C`是Event，见第2节；
* 第7列：R:Read; W:Write; D:Block; B:Barrier-Operation; S:Sync; N:Notify(比较新的内核才有，比如linux kernel 3.10)；
* 第8列：`223490 + 56`是IO的起始sector号和读写的sector数；即offset和size；
* 第9列：`kjournald`是进程名；

其实这是默认输出格式，我们也可以通过`-f`选项自定义其格式(类似于`date`命令自定义日期时间格式)：`%M`是major设备号；`%m`是minor设备号；`%c`表示cpuid；`%s`表示序列号；`%T`表示时间(秒)；`%t`表示时间(纳秒)；`%P`是进程PID；`%a`是事件；等等，详见`man blkparse`。

* **Summary**

Summary包括CPU和device两个维度:

<div align=center>![summary](blkparse-output-2.png)

**例1**：把/dev/sde(设备号:8,64)整个盘格式化成ext4，挂载到/mnt并使用fio写/mnt/fio-test-file (rw=randwrite ioengine=psync bs=4k fsync=1)，同时trace /dev/sde

```
# blktrace -d /dev/sde -w 60

# blkparse -i sde

......

  8,64  11     2944    33.779658276 90790  Q   W 7614344 + 24 [fio]
  8,64  11     2945    33.779659493 90790  G   W 7614344 + 24 [fio]
  8,64  11     2946    33.779660124 90790  P   N [fio]
  8,64  11     2947    33.779661287 90790  I   W 7614344 + 24 [fio]
  8,64  11     2948    33.779661791 90790  U   N [fio] 1
  8,64  11     2949    33.779662409 90790  D   W 7614344 + 24 [fio]
  8,64  11     2950    33.791580458     0  C   W 7614344 + 24 [0]
  8,64  11     2951    33.791608964 90790  Q  WS 7614368 + 8 [fio]
  8,64  11     2952    33.791609940 90790  G  WS 7614368 + 8 [fio]
  8,64  11     2953    33.791610381 90790  P   N [fio]
  8,64  11     2954    33.791611867 90790  I  WS 7614368 + 8 [fio]
  8,64  11     2955    33.791612372 90790  U   N [fio] 1
  8,64  11     2956    33.791612977 90790  D  WS 7614368 + 8 [fio]
  8,64  11     2957    33.799933777     0  C  WS 7614368 + 8 [0]
  8,64  13     5709    33.799963426 90708  Q  WS 3905206216 + 8 [jbd2/sde-8]
  8,64  13     5710    33.799964662 90708  G  WS 3905206216 + 8 [jbd2/sde-8]
  8,64  13     5711    33.799965100 90708  P   N [jbd2/sde-8]
  8,64  13     5712    33.799965942 90708  Q  WS 3905206224 + 8 [jbd2/sde-8]
  8,64  13     5713    33.799966423 90708  M  WS 3905206224 + 8 [jbd2/sde-8]
  8,64  13     5714    33.799967148 90708  Q  WS 3905206232 + 8 [jbd2/sde-8]

......

CPU8 (sde):
 Reads Queued:           0,        0KiB  Writes Queued:         195,   29,228KiB
 Read Dispatches:        0,        0KiB  Write Dispatches:      170,   29,228KiB
 Reads Requeued:         0               Writes Requeued:         0
 Reads Completed:        0,        0KiB  Writes Completed:      170,   29,228KiB
 Read Merges:            0,        0KiB  Write Merges:           25,      100KiB
 Read depth:             0               Write depth:            20
 IO unplugs:           104               Timer unplugs:           0
CPU9 (sde):
 Reads Queued:           0,        0KiB  Writes Queued:         991,   16,156KiB
 Read Dispatches:        0,        0KiB  Write Dispatches:      574,   16,156KiB
 Reads Requeued:         0               Writes Requeued:         0
 Reads Completed:        0,        0KiB  Writes Completed:      574,   16,156KiB
 Read Merges:            0,        0KiB  Write Merges:          417,    1,668KiB
 Read depth:             0               Write depth:            20
 IO unplugs:           346               Timer unplugs:           0
......
Total (sde):
 Reads Queued:           0,        0KiB  Writes Queued:       8,763,  352,192KiB
 Read Dispatches:        0,        0KiB  Write Dispatches:    5,380,  352,192KiB
 Reads Requeued:         0               Writes Requeued:         0
 Reads Completed:        0,        0KiB  Writes Completed:    5,380,  352,196KiB
 Read Merges:            0,        0KiB  Write Merges:        3,383,   13,532KiB
 IO unplugs:         3,092               Timer unplugs:           0

Throughput (R/W): 0KiB/s / 5,872KiB/s
Events (sde): 39,850 entries
Skips: 0 forward (0 -   0.0%)
```

我们尝试把一个特定request的事件给找出来(grep一个特定的起始sector号，而不是grep一个序列号):

```
# blkparse -i sde | grep -w 7614368
  8,64  11     2951    33.791608964 90790  Q  WS 7614368 + 8 [fio]
  8,64  11     2952    33.791609940 90790  G  WS 7614368 + 8 [fio]
  8,64  11     2954    33.791611867 90790  I  WS 7614368 + 8 [fio]
  8,64  11     2956    33.791612977 90790  D  WS 7614368 + 8 [fio]
  8,64  11     2957    33.799933777     0  C  WS 7614368 + 8 [0]
```

这是一个Write+Sync请求，起始sector是7614368，大小是8个secotr(即4k)。它是一个典型的request：经历了`Q->G->I->D-C`几个阶段，总耗时：33.799933777-33.791608964 = 8324813纳秒 = 8.3毫秒。

Summary部分，可以看到各个read指标都为0，因为fio全是写操作且大小和文件系统block对齐。可以对比还是全写操作但bs=5k(不再和文件系统block对齐)的情形，这时候read指标就不为0了，因为不对齐，文件系统需要把部分修改的block读出来，修改然后再写回去。

**例2**：把/dev/sde(设备号:8,64)分区并把/dev/sde2(设备号:8,66)格式化成ext4，挂载到/mnt并使用fio写/mnt/fio-test-file (rw=randwrite ioengine=psync bs=4k fsync=1)，同时还是trace /dev/sde

前面部分类似，我们还是看一个request:

```
# blkparse -i sde | grep -w 3927579616
  8,64  20     1160    21.225404128 98523  A  WS 3927579616 + 8 <- (8,66) 20560864
  8,64  20     1161    21.225405121 98523  Q  WS 3927579616 + 8 [fio]
  8,64  20     1162    21.225407092 98523  G  WS 3927579616 + 8 [fio]
  8,64  20     1164    21.225409225 98523  I  WS 3927579616 + 8 [fio]
  8,64  20     1166    21.225411107 98523  D  WS 3927579616 + 8 [fio]
  8,64   0     1176    21.238455782     0  C  WS 3927579616 + 8 [0]
```

和例1相比，这个request多了一个`A`事件，因为这个IO是从/dev/sde2(设备号:8,66)remap到/dev/sde(设备号:8,64)的。总耗时21.238455782-21.225404128 = 13051654纳秒 - 13毫秒。

**例3**：基于例2的数据，我们找一个merge的请求:

```
# blkparse -i sde | grep -w M
  ......
  8,64  13     4417    59.166019679 96519  M  WS 5857801416 + 8 [jbd2/sde2-8]
  8,64  13     4434    59.194332313 96519  M  WS 5857801440 + 8 [jbd2/sde2-8]
  8,64  13     4451    59.232388528 96519  M  WS 5857801464 + 8 [jbd2/sde2-8]
  ......

# blkparse -i sde | grep -w 5857801440
  8,64  13     4432    59.194331558 96519  A  WS 5857801440 + 8 <- (8,66) 1950782688
  8,64  13     4433    59.194331878 96519  Q  WS 5857801440 + 8 [jbd2/sde2-8]
  8,64  13     4434    59.194332313 96519  M  WS 5857801440 + 8 [jbd2/sde2-8]
```

这个请求从remap到queued，然后merge到一个已存在的请求中，之后便追踪不到了：因为它是其他某个请求的一部分，而不是一个独立的请求。假如知道它被merge到哪个请求了，去追踪那个请求，就能知道后续各个阶段。另外注意，被合并的情况下，没有get request (G)阶段，而直接被合并了（参考第2节结尾的IO流程图）。

通过这三个例子我们可以看出，blktrace可以追踪到一个特定request的各个阶段，及各个阶段的耗时。但这太详细了，我们无法逐一查看各个request。这时`btt`命令就派上用场了，它可以生成报表。blkparse还有一个功能：把blktrace生成的`{device}.blktrace.{cpu}`一堆文件dump成一个二进制文件，输出到`-d`指定的文件中（忽略标准输出）。这个功能正好方便`btt`的使用。

```
# blkparse -i sde -d sde.blktrace.bin
# ls sde.blktrace.bin
sde.blktrace.bin
```

# 使用btt生成报表 (5)

## 阶段定义 (5.1)

我们再用一个图表示各个阶段的时延：

<div align=center>![阶段图](blktrace-stags.jpg)

其中各个阶段的定义如下：

- Q2Q: time between requests sent to the block layer
- Q2G: time from a block I/O is queued to the time it gets a request allocated for it
- G2I: time from a request is allocated to the time it is Inserted into the device's queue
- Q2M: time from a block I/O is queued to the time it gets merged with an existing request
- I2D: time from a request is inserted into the device's queue to the time it is actually issued to the device
- M2D: time from a block I/O is merged with an exiting request until the request is issued to the device
- D2C: service time of the request by the device
- Q2C: total time spent in the block layer for a request

注意以下两点：
- Q2Q是表示两个reqeusts之间的间隔；而其他都是表示一个request的各个阶段的间隔。
- 上面绿色部分表示一个典型request（没有被merge的request）特有的阶段；下面红色表示一个被merge的request特有的阶段；黄色部分表示公共阶段。

## btt的默认输出 (5.2)

以第2节中例2的数据为例，btt最简单的使用方式如下：

```
# btt -i sde.blktrace.bin -o bttout
# ll
total 2388
-rw-r--r-- 1 root root     362 Sep  2 22:06 8,64_iops_fp.dat
-rw-r--r-- 1 root root     710 Sep  2 22:06 8,64_mbps_fp.dat
-rw-r--r-- 1 root root    3468 Sep  2 22:06 bttout.avg
-rw-r--r-- 1 root root   17023 Sep  2 22:06 bttout.dat
-rw-r--r-- 1 root root    7220 Sep  2 22:06 bttout_dhist.dat
-rw-r--r-- 1 root root       0 Sep  2 22:06 bttout.msg
-rw-r--r-- 1 root root    7236 Sep  2 22:06 bttout_qhist.dat
-rw-r--r-- 1 root root 2387128 Sep  2 22:06 sde.blktrace.bin
-rw-r--r-- 1 root root     362 Sep  2 22:06 sys_iops_fp.dat
-rw-r--r-- 1 root root     710 Sep  2 22:06 sys_mbps_fp.dat
```

btt的输入`-i`是blkparse生成（dump）的二进制文件（见第4节），默认输出包括：

- 标准输出：很多信息，下面细说。
- `{major,minor设备号}_iops_fp.dat`：包含两列，第一列为时间（以秒为单位，从第0秒开始）；第二列为IOPS；
- `{major,minor设备号}_mbps_fp.dat`：包含两列，第一列为时间（以秒为单位，从第0秒开始）；第二列为带宽，即MB/s；
- `sys_iops_fp.dat`：只trace一个设备的时候，和`{major,minor设备号}_iops_fp.dat`一样（两个文件的MD5相同）。
- `sys_mbps_fp.dat`：只trace一个设备的时候，和`{major,minor设备号}_mbps_fp.dat`一样（两个文件的MD5相同）

其中标准输出很多，把屏幕搞的很乱，看不清主要信息。为此，我们加一个`-o bttout`选项，这时屏幕上就没有任何信息，但生成以下四个文件：

- **bttout.avg**

这个输出文件最重要，里面包含各种统计信息，我们逐一解读（数据来自于第2节中的例2）：

```
==================== All Devices ====================

            ALL           MIN           AVG           MAX           N
--------------- ------------- ------------- ------------- -----------

Q2Q               0.000000915   0.007937919   0.103788139        7557
Q2G               0.000000410   0.000001214   0.000030594        5707
G2I               0.000000278   0.000001805   0.000033501        5707
Q2M               0.000000164   0.000000498   0.000014527        1851
I2D               0.000000308   0.000001078   0.000002753        5707
M2D               0.000001169   0.000002162   0.000032612        1851
D2C               0.001751673   0.013116498   0.103759327        7557
Q2C               0.001753524   0.013120242   0.103761266        7557


==================== Device Overhead ====================

       DEV |       Q2G       G2I       Q2M       I2D       D2C
---------- | --------- --------- --------- --------- ---------
 (  8, 64) |   0.0070%   0.0104%   0.0009%   0.0062%  99.9715%
---------- | --------- --------- --------- --------- ---------
   Overall |   0.0070%   0.0104%   0.0009%   0.0062%  99.9715%
```

All Devices部分显示了各个阶段的最小、平均及最大耗时，单位是秒。阶段的定义见第5.1节。Device Overhead部分显示了几个重要阶段在总耗时中的占比。Overall是所有被trace的设备的总体情况，我们只trace了一个设备（major,minor设备号为8,64），所以Overall这一行和（8, 64）这一样一样。在这个例子中，主要耗时是**D2C**阶段（正常情况就是这样）：


$$ \frac{AVG D2C}{AVG Q2C} = \frac{0.013116498}{0.013120242} = 99.9715\%$$


```
==================== Device Merge Information ====================

       DEV |       #Q       #D   Ratio |   BLKmin   BLKavg   BLKmax    Total
---------- | -------- -------- ------- | -------- -------- -------- --------
 (  8, 64) |     7558     5707     1.3 |        8      118     1024   678184
```


```
==================== Device Q2Q Seek Information ====================

       DEV |          NSEEKS            MEAN          MEDIAN | MODE
---------- | --------------- --------------- --------------- | ---------------
 (  8, 64) |            7558     894100249.2               0 | 0(3937)
---------- | --------------- --------------- --------------- | ---------------
   Overall |          NSEEKS            MEAN          MEDIAN | MODE
   Average |            7558     894100249.2               0 | 0(3937)



==================== Device D2D Seek Information ====================

       DEV |          NSEEKS            MEAN          MEDIAN | MODE
---------- | --------------- --------------- --------------- | ---------------
 (  8, 64) |            5707    1184091411.0               0 | 0(2086)
---------- | --------------- --------------- --------------- | ---------------
   Overall |          NSEEKS            MEAN          MEDIAN | MODE
   Average |            5707    1184091411.0               0 | 0(2086)
```


```
==================== Plug Information ====================

       DEV |    # Plugs # Timer Us  | % Time Q Plugged
---------- | ---------- ----------  | ----------------
 (  8, 64) |       3402(         0) |   0.014552920%

       DEV |    IOs/Unp   IOs/Unp(to)
---------- | ----------   ----------
 (  8, 64) |        1.0          0.0
---------- | ----------   ----------
   Overall |    IOs/Unp   IOs/Unp(to)
   Average |        1.0          0.0
```


```
==================== Active Requests At Q Information ====================

       DEV |  Avg Reqs @ Q
---------- | -------------
 (  8, 64) |           0.2
```


```
==================== I/O Active Period Information ====================

       DEV |     # Live      Avg. Act     Avg. !Act % Live
---------- | ---------- ------------- ------------- ------
 (  8, 64) |       4925   0.012143126   0.000036958  99.70
---------- | ---------- ------------- ------------- ------
 Total Sys |       4925   0.012143126   0.000036958  99.70
```

- **bttout.dat**

```
# Total System
#     Total System : q activity
  0.000042623   0.0
  0.000042623   0.4
 53.325061811   0.4
 ......

#     Total System : c activity
  0.013872226   0.5
  0.013872226   0.9
 59.986877686   0.9
 ......

# Per process
#  {user-process-name} : q activity

#  {user-process-name} : c activity
 23.098077380   1.5
 23.098077380   1.9
 ......

#     ext4lazyinit : q activity
  0.171033385   2.0
  0.171033385   2.4
  ......

#     ext4lazyinit : c activity

#              fio : q activity
  0.022288457   3.0
  0.022288457   3.4
  ......

#              fio : c activity

#             jbd2 : q activity
  0.000042623   5.0
  0.000042623   5.4
 ......

#             jbd2 : c activity

#           kernel : q activity

#           kernel : c activity
  0.013872226   6.5
  0.013872226   6.9
  ......

#          kworker : q activity
  4.219029550   7.0
  4.219029550   7.4
  4.219035624   7.4
  4.219035624   7.0
  ......

#          kworker : c activity
```

- **bttout_dhist.dat**

```
# D buckets
   1 0
   2 0
   3 0
   4 0
   ......
1022 0
1023 0
1024 608

# D bucket for > 1024
1024 0
```


- **bttout.msg**

- **bttout_qhist.dat**


