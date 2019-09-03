---
title: Advanced Format Disks 
date: 2019-09-03 20:20:13
tags: [disk]
categories: disk 
---

简单介绍advanced format disks（4K扇区磁盘）的产生与标准，并对比512e磁盘分区对齐与不对齐时的性能。

<!-- more -->

<script type="text/x-mathjax-config">
MathJax.Hub.Config({
tex2jax: {inlineMath: [['$','$'], ['\\(','\\)']]}
});
</script>

<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>

# 4K扇区 (1)

磁盘的高级格式（Advanced Format）使用4K作为扇区的大小（至少第一代Advanced Format就是指4K扇区）。传统磁盘的扇区大小是512B。有什么问题呢？这得从扇区的结构说起：磁盘上的扇区不是一个接一个紧密排列的，相邻的扇区之间有一个间隙（gap），并且每个扇区前面有一个sync字段和一个address mark字段，接着才是我们所知的512B数据（data），后面还有一个纠错码ECC（error correcting code）。并且：

$$ gap + sync + address mark = 15B $$
$$ data = 512B $$
$$ ECC = 50B $$

这样，磁盘空间的利用率是：

$$ \frac{512}{15+512+50}=88.7\% $$

显然，增大扇区可以提高空间的利用率。另外，由于纠错码ECC只有50B，纠错算法也受限，无法使用更高级的纠错算法来提高磁盘的可靠性。于是，2010年的时候，International Disk Drive Equipment and Materials Association (IDEMA)完成了一个新的标准：采用4096B作为扇区大小，并且把ECC增大到100B。虽然ECC增大了，但现在一个扇区相当于原来的8个扇区，空间利用率还是提高了：

$$ gap + sync + address mark = 15B $$
$$ data = 4096B $$
$$ ECC = 100B $$

那么，磁盘空间利用率是：

$$ \frac{4096}{15+4096+100} = 97.3\% $$

从下面一图一表([来自维基百科](https://en.wikipedia.org/wiki/Advanced_Format#cite_note-27%20%E7%BB%B4%E5%9F%BA%E7%99%BE%E7%A7%91))我们可以看的更直观。

|Description            |512B section  |4096B sector   |
|-----------------------|--------------|---------------|
|gap,sync,address mark  |15B           |15B            |
|user data              |512B          |4096B          |
|error correcting code  |50B           |100B           |
|total                  |577B          |4211B          |
|efficiency             |88.7%         |97.3%          |

<div align=center>![Advanced Format](advfmt.jpg)

总结来说：引入4K扇区有两个好处：

 - 提高空间利用率；
 - 可以引入更高级的纠错算法，来提高磁盘的可靠性；

# 物理扇区和逻辑扇区 (2)

历史原因，给这个看上去很好的优化带来很多麻烦。因为直到现在，还有很多硬件、软件系统是基于512B的扇区设计的，包括芯片组，操作系统，数据库引擎、磁盘分区工具、文件系统等。为了和现有系统兼容，就引入物理扇区和逻辑扇区的概念。

 - **物理扇区**是磁盘驱动器读写的最小单元，也就是说，操作系统读写100B，磁盘驱动器也必须读写一整个扇区。
 - **逻辑扇区**是磁盘的firmware模拟出来的操作系统看到的扇区。比如，为了兼容老的软硬件系统，很多4K扇区的磁盘(512e类型，见下文4K扇区磁盘的分类)，都模拟512B扇区。

# 4K扇区磁盘的分类 (3)

## 4K native (3.1)

简写为4Kn，这类磁盘firmware里不会把4K的物理扇区模拟成512B的逻辑扇区，也就是说，外部可见的逻辑扇区直接映射到内部的物理扇区。自从2014年开始，企业级的4K native磁盘已经在市场上流通了。这类磁盘的logo如下：

<div align=center>![4Knative](4knative.jpg)

但，由于4Kn不能兼容512B扇区的软硬件系统，它的使用并不广泛。虽然Windows(Windows8和Windows Server 2012)和Linux(自从kernel 2.6.31)都支持4Kn的磁盘，然而很多操作系统不能从4Kn磁盘引导([见这里](https://askubuntu.com/questions/337693/how-to-format-a-4k-sector-hard-drive)和[这里](https://lwn.net/Articles/377895/))，这些磁盘通常作为外部存储使用。
## 512e (3.2)

为了向下兼容，磁盘提供商在firmware里模拟512B扇区，目前这也是事实上的标准([见这里](http://idema.org/?page_id=2900))。这类磁盘被叫做Advanced Format 512e, 或者512 emulation磁盘。它的logo如下：

<div align=center>![512e](512e.jpg)

4096B扇区和512B扇区之间的转换是磁盘的firmware完成的，对外界透明，也就是说，可以像读写传统磁盘一样来读写512e磁盘，如下图所示：

<div align=center>![512e-read-write](512e-rw.jpg)

比如，操作系统读取512B数据，磁盘驱动器会：

 1. 读取目标扇区4096B；
 2. 从中抽取被读取的512B返回给操作系统；

对于读操作来说，这个过程带来的性能损失很小，可以忽略不计。但是对于写操作，问题就麻烦了。操作系统写512B数据，磁盘驱动器必须采用“读-修改-写”(RMW: Read-Modify-Write)的方式：

 1. 读取目标扇区4096B；
 2. 修改其中的512B；
 3. 把目标扇区写回；

可以预见，这种方式有明显的性能影响。但是，假如操作系统写入的要是整个4K，并且和一个扇区对齐，那么就不必采用RMW的方式了，性能的影响可以规避。

# 分区对齐 (4)

为了避免RMW带来性能损耗，最好能做到如下两点：

 - 写入大小是4K的整数倍；
 - 写入边界和物理扇区边界对齐； 

Linux系统已经作了优化：

 - 尽量写入4K的整数倍；
 - 尽量和分区对齐。

注意第二点，**这要求分区和物理扇区是对齐的**。假如分区和物理扇区没有对齐，而每个写操作和分区又是对齐的，导致的结果是，每个写操作恰恰和物理扇区没有对齐。如下图所示：

<div align=center>![512e-alignment](512e-align.jpg)

可见，系统IO-1和系统IO-2和分区是对齐的。但是，分区和物理扇区没有对齐，导致磁盘IO-1和磁盘IO-2都没对齐，并且每个磁盘IO需要两次RMW，共4个RMW才能完成。所以，**分区和物理扇区的对齐至关重要**。parted可以检查已有分区是否对齐；并且在做新分区时，若没对齐，也会得到警告。下面我们做一个测试。


- **磁盘信息**

如下，有一块size为6T转速为7200 rpm的512e的磁盘(512 bytes logical, 4096 bytes physical)：

```
# smartctl -a /dev/sde
smartctl 6.2 2013-07-26 r3841 [x86_64-linux-3.10.0-327.36.4.el7.x86_64] (local build)
Copyright (C) 2002-13, Bruce Allen, Christian Franke, www.smartmontools.org

=== START OF INFORMATION SECTION ===
......
Add. Product Id:  DELL(tm)
Firmware Version: DA25
User Capacity:    6,001,175,126,016 bytes [6.00 TB]
Sector Sizes:     512 bytes logical, 4096 bytes physical
Rotation Rate:    7200 rpm
Device is:        Not in smartctl database [for details use: -P showall]
ATA Version is:   ACS-3 (unknown minor revision code: 0x006d)
SATA Version is:  SATA 3.1, 6.0 Gb/s (current: 6.0 Gb/s)
Local Time is:    Tue Sep  3 21:04:43 2019 CST
SMART support is: Available - device has SMART capability.
SMART support is: Enabled

```

- **fio jobs**

我们将分别用这2个job来测试对齐和不对齐情况下，分区和Filesystem的性能：

```
# cat job.write.part.4k
[global]
    filename=/dev/sde1
    runtime=60

[seqwrite_job]
    rw=write
    bs=4k
    size=100G
    ioengine=psync
    fsync=1
    iodepth=1
    numjobs=1
    new_group=1


# cat job.write.fs.4k
[global]
    filename=/mnt/fio-test-file
    runtime=60

[seqwrite_job]
    rw=write
    bs=4k
    size=100G
    ioengine=psync
    fsync=1
    iodepth=1
    numjobs=1
    new_group=1
```

- **分区对齐时的性能**

```
# parted /dev/sde
GNU Parted 3.1
Using /dev/sde
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) mklabel gpt
Warning: The existing disk label on /dev/sde will be destroyed and all data on this disk will be lost. Do you want to continue?
Yes/No? Yes
(parted) mkpart primary 131072k 50%
(parted) align-check opt 1
1 aligned
(parted) quit


# ./fio job.write.part.4k
seqwrite_job: (g=0): rw=write, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=psync, iodepth=1
fio-3.14
Starting 1 process
Jobs: 1 (f=1): [W(1)][100.0%][w=480KiB/s][w=120 IOPS][eta 00m:00s]
seqwrite_job: (groupid=0, jobs=1): err= 0: pid=170558: Tue Sep  3 21:53:00 2019
  write: IOPS=118, BW=474KiB/s (486kB/s)(27.8MiB/60001msec)
    clat (nsec): min=4848, max=58867, avg=10749.07, stdev=5038.56
     lat (nsec): min=4971, max=59524, avg=11261.16, stdev=5232.99
    clat percentiles (nsec):
     |  1.00th=[ 6368],  5.00th=[ 6624], 10.00th=[ 6688], 20.00th=[ 6752],
     | 30.00th=[ 6816], 40.00th=[ 7072], 50.00th=[ 7456], 60.00th=[13632],
     | 70.00th=[13760], 80.00th=[13888], 90.00th=[15424], 95.00th=[16064],
     | 99.00th=[27520], 99.50th=[34560], 99.90th=[47360], 99.95th=[49408],
     | 99.99th=[58624]
   bw (  KiB/s): min=  415, max=  488, per=100.00%, avg=474.07, stdev=12.52, samples=120
   iops        : min=  103, max=  122, avg=118.47, stdev= 3.17, samples=120
  lat (usec)   : 10=53.59%, 20=42.28%, 50=4.09%, 100=0.04%
  fsync/fdatasync/sync_file_range:
    sync (usec): min=2145, max=50031, avg=8421.18, stdev=1587.52
    sync percentiles (usec):
     |  1.00th=[ 8291],  5.00th=[ 8291], 10.00th=[ 8291], 20.00th=[ 8291],
     | 30.00th=[ 8291], 40.00th=[ 8356], 50.00th=[ 8356], 60.00th=[ 8356],
     | 70.00th=[ 8356], 80.00th=[ 8356], 90.00th=[ 8356], 95.00th=[ 8356],
     | 99.00th=[ 8455], 99.50th=[ 8717], 99.90th=[42206], 99.95th=[48497],
     | 99.99th=[50070]
  cpu          : usr=0.13%, sys=0.88%, ctx=14225, majf=0, minf=33
  IO depths    : 1=200.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,7112,0,7112 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=474KiB/s (486kB/s), 474KiB/s-474KiB/s (486kB/s-486kB/s), io=27.8MiB (29.1MB), run=60001-60001msec

Disk stats (read/write):
  sde: ios=46/14199, merge=0/0, ticks=21/59247, in_queue=59263, util=98.85%


# mkfs.ext4 /dev/sde1
# mount /dev/sde1 /mnt/
# ./fio job.write.fs.4k
seqwrite_job: (g=0): rw=write, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=psync, iodepth=1
fio-3.14
Starting 1 process
seqwrite_job: Laying out IO file (1 file / 102400MiB)
Jobs: 1 (f=1): [W(1)][100.0%][w=132KiB/s][w=33 IOPS][eta 00m:00s]
seqwrite_job: (groupid=0, jobs=1): err= 0: pid=170723: Tue Sep  3 21:55:59 2019
  write: IOPS=29, BW=118KiB/s (121kB/s)(7096KiB/60017msec)
    clat (nsec): min=10482, max=63573, avg=18145.23, stdev=7382.36
     lat (nsec): min=10781, max=64236, avg=18640.49, stdev=7572.48
    clat percentiles (nsec):
     |  1.00th=[11200],  5.00th=[11456], 10.00th=[11968], 20.00th=[12224],
     | 30.00th=[12352], 40.00th=[12480], 50.00th=[12608], 60.00th=[22912],
     | 70.00th=[24448], 80.00th=[24704], 90.00th=[24960], 95.00th=[28288],
     | 99.00th=[39168], 99.50th=[46848], 99.90th=[62208], 99.95th=[63744],
     | 99.99th=[63744]
   bw (  KiB/s): min=   56, max=  152, per=100.00%, avg=118.24, stdev=13.52, samples=120
   iops        : min=   14, max=   38, avg=29.54, stdev= 3.38, samples=120
  lat (usec)   : 20=57.05%, 50=42.56%, 100=0.39%
  fsync/fdatasync/sync_file_range:
    sync (msec): min=24, max=249, avg=33.81, stdev= 8.94
    sync percentiles (msec):
     |  1.00th=[   26],  5.00th=[   26], 10.00th=[   26], 20.00th=[   34],
     | 30.00th=[   34], 40.00th=[   34], 50.00th=[   34], 60.00th=[   34],
     | 70.00th=[   34], 80.00th=[   34], 90.00th=[   42], 95.00th=[   51],
     | 99.00th=[   59], 99.50th=[   84], 99.90th=[  117], 99.95th=[  251],
     | 99.99th=[  251]
  cpu          : usr=0.03%, sys=0.26%, ctx=3550, majf=0, minf=31
  IO depths    : 1=200.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,1774,0,1774 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=118KiB/s (121kB/s), 118KiB/s-118KiB/s (121kB/s-121kB/s), io=7096KiB (7266kB), run=60017-60017msec

Disk stats (read/write):
  sde: ios=0/6772, merge=0/5288, ticks=0/79680, in_queue=79702, util=99.30%
```

- **分区不对齐时的性能**

```
# parted /dev/sde
GNU Parted 3.1
Using /dev/sde
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) mklabel gpt
Warning: The existing disk label on /dev/sde will be destroyed and all data on this disk will be lost. Do you want to continue?
Yes/No? Yes
(parted) mkpart primary 131074k 50%
Warning: The resulting partition is not properly aligned for best performance.
Ignore/Cancel? Ignore
(parted) align-check opt 1
1 not aligned
(parted) quit

# ./fio job.write.part.4k
seqwrite_job: (g=0): rw=write, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=psync, iodepth=1
fio-3.14
Starting 1 process
Jobs: 1 (f=1): [W(1)][100.0%][w=160KiB/s][w=40 IOPS][eta 00m:00s]
seqwrite_job: (groupid=0, jobs=1): err= 0: pid=170864: Tue Sep  3 21:58:13 2019
  write: IOPS=38, BW=155KiB/s (158kB/s)(9280KiB/60006msec)
    clat (nsec): min=6149, max=59737, avg=13039.31, stdev=5751.16
     lat (nsec): min=6287, max=60617, avg=13552.40, stdev=5942.20
    clat percentiles (nsec):
     |  1.00th=[ 7136],  5.00th=[ 7584], 10.00th=[ 7648], 20.00th=[ 7776],
     | 30.00th=[ 8096], 40.00th=[ 8512], 50.00th=[11200], 60.00th=[16192],
     | 70.00th=[16320], 80.00th=[17280], 90.00th=[19328], 95.00th=[22912],
     | 99.00th=[29312], 99.50th=[33024], 99.90th=[53504], 99.95th=[59648],
     | 99.99th=[59648]
   bw (  KiB/s): min=   48, max=  160, per=100.00%, avg=154.61, stdev=18.17, samples=120
   iops        : min=   12, max=   40, avg=38.61, stdev= 4.53, samples=120
  lat (usec)   : 10=45.43%, 20=44.83%, 50=9.61%, 100=0.13%
  fsync/fdatasync/sync_file_range:
    sync (msec): min=8, max=141, avg=25.85, stdev= 6.73
    sync percentiles (msec):
     |  1.00th=[   26],  5.00th=[   26], 10.00th=[   26], 20.00th=[   26],
     | 30.00th=[   26], 40.00th=[   26], 50.00th=[   26], 60.00th=[   26],
     | 70.00th=[   26], 80.00th=[   26], 90.00th=[   26], 95.00th=[   26],
     | 99.00th=[   75], 99.50th=[   75], 99.90th=[  101], 99.95th=[  101],
     | 99.99th=[  142]
  cpu          : usr=0.03%, sys=0.33%, ctx=4640, majf=0, minf=34
  IO depths    : 1=200.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,2320,0,2320 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=155KiB/s (158kB/s), 155KiB/s-155KiB/s (158kB/s-158kB/s), io=9280KiB (9503kB), run=60006-60006msec

Disk stats (read/write):
  sde: ios=52/4631, merge=0/16212, ticks=34/59687, in_queue=59734, util=99.62%


# mkfs.ext4 /dev/sde1
# mount /dev/sde1 /mnt/
# ./fio job.write.fs.4k
seqwrite_job: (g=0): rw=write, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=psync, iodepth=1
fio-3.14
Starting 1 process
seqwrite_job: Laying out IO file (1 file / 102400MiB)
Jobs: 1 (f=1): [W(1)][100.0%][w=44KiB/s][w=11 IOPS][eta 00m:00s]
seqwrite_job: (groupid=0, jobs=1): err= 0: pid=171321: Tue Sep  3 22:00:11 2019
  write: IOPS=11, BW=44.9KiB/s (45.9kB/s)(2692KiB/60019msec)
    clat (nsec): min=10276, max=68360, avg=18524.63, stdev=8076.31
     lat (nsec): min=10564, max=69026, avg=19006.17, stdev=8237.92
    clat percentiles (nsec):
     |  1.00th=[10816],  5.00th=[11328], 10.00th=[11712], 20.00th=[12096],
     | 30.00th=[12224], 40.00th=[12352], 50.00th=[12736], 60.00th=[23424],
     | 70.00th=[24448], 80.00th=[24704], 90.00th=[24960], 95.00th=[30336],
     | 99.00th=[49408], 99.50th=[60672], 99.90th=[68096], 99.95th=[68096],
     | 99.99th=[68096]
   bw (  KiB/s): min=   16, max=   56, per=100.00%, avg=44.77, stdev= 5.01, samples=120
   iops        : min=    4, max=   14, avg=11.12, stdev= 1.28, samples=120
  lat (usec)   : 20=54.83%, 50=44.43%, 100=0.74%
  fsync/fdatasync/sync_file_range:
    sync (msec): min=74, max=450, avg=89.16, stdev=20.00
    sync percentiles (msec):
     |  1.00th=[   75],  5.00th=[   75], 10.00th=[   84], 20.00th=[   84],
     | 30.00th=[   84], 40.00th=[   84], 50.00th=[   84], 60.00th=[   84],
     | 70.00th=[   84], 80.00th=[   84], 90.00th=[  117], 95.00th=[  117],
     | 99.00th=[  142], 99.50th=[  159], 99.90th=[  451], 99.95th=[  451],
     | 99.99th=[  451]
  cpu          : usr=0.00%, sys=0.11%, ctx=1347, majf=0, minf=30
  IO depths    : 1=200.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,673,0,673 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=44.9KiB/s (45.9kB/s), 44.9KiB/s-44.9KiB/s (45.9kB/s-45.9kB/s), io=2692KiB (2757kB), run=60019-60019msec

Disk stats (read/write):
  sde: ios=0/2894, merge=0/3017, ticks=0/89696, in_queue=89741, util=99.68%
```

- **对比**

|Aligned  |RawPartition or Filesystem  |BandWidth (KiB/s)  |IOPS     |Avg SyncLatency (ms)  |
|---------|----------------------------|-------------------|---------|----------------------|
|Yes      |RawPartition                |480                |120      |8.4                   |
|No       |RawPartition                |160                |40       |25.85                 |
|Yes      |Filesystem                  |132                |33       |33.81                 |
|No       |Filesystem                  |44                 |11       |89.16                 |

很明显，不对齐的时候，无论是裸分区还好文件系统，性能都降低至对齐的1/3。值得一提的是，测裸分区时，若使用ioengine=libaio, direct=1，则对齐与不对齐差别不大，原因是没强制sync，磁盘firmware可能对齐后再写到盘片。

# 小结 (5)

在了解advanced format disks标准的时候，我们猜测512e磁盘在分区不对齐的情况下会有严重的性能损耗。然后，我们通过实验证实了这一猜想。
