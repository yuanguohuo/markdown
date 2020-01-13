---
title: 避免block device stack中的递归
date: 2019-10-04 13:17:11
tags: [kernel,io,block]
categories: linux 
---

本篇研究一个具体问题：block device stack中，如何避免递归。

<!-- more -->

<script type="text/x-mathjax-config">
MathJax.Hub.Config({
tex2jax: {inlineMath: [['$','$'], ['\\(','\\)']]}
});
</script>

<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>

# 问题的由来 (1)

在linux中，在block device上可以虚拟出新的block device（例如software RAID使用md，LVM2使用dm）；新的block device上还可以再虚拟出更上层的block device；这样，就形成一个block device stack。除最底层之外，每层block device不真正处理bio，而是split、合并或修改bio，然后发给下一层的block device。

逻辑上是这样没有问题，然而实现上是有讲究的。实现实现是层层调用，形成一个`generic_make_request`的递归：

{% asset_img layered-block-device.jpg layered-block-device %}

这样的实现比较直观，但当虚拟block层数过多时，内核调用栈就会很深。栈太深一方面会导致性能降低，另一方面可能导致栈溢出。当bio是文件系统提交的话，这个问题就更严重，因为文件系统本身就消耗了相当深的栈。Linux 2.6.22以前就存在这个问题。

# 避免递归 (2)

一句话，通过一个共享的数据结构（`current->bio_list`）来打破递归。具体说来是这样的：

- 假设进程初始状态是没有active的bio，`current->bio_list`为`NULL`；
- 一个bio到来，进入`generic_make_request`的时候，就在栈上分配一个`bio_list_on_stack`，并让`current->bio_list`指向它，然后把这个bio发往下层block device；
- 进入下层block device的`generic_make_request`时候，就会发现`current->bio_list`不为`NULL`，所以，把bio插入这个list，然后立即返回；

注意一点，如变量名所述，`bio_list_on_stack`在栈上，所以它的作用域是本层栈之上的函数运行，也就是说，它起作用的范围是当前这个bio在block device stack上的传递。当前进程`current`会在第0层`generic_make_request`返回之前，再发送bio吗？应该不会。如果会，那么就会产生另一个`generic_make_request`栈，但`current->bio_list`指向另外一个栈上的`bio_list_on_stack`。

看`generic_make_request`的代码（内核版本是3.19.8）：

```c
void generic_make_request(struct bio *bio)
{
	struct bio_list bio_list_on_stack;

	if (!generic_make_request_checks(bio))
		return;

	/*
	 * We only want one ->make_request_fn to be active at a time, else
	 * stack usage with stacked devices could be a problem.  So use
	 * current->bio_list to keep a list of requests submited by a
	 * make_request_fn function.  current->bio_list is also used as a
	 * flag to say if generic_make_request is currently active in this
	 * task or not.  If it is NULL, then no make_request is active.  If
	 * it is non-NULL, then a make_request is active, and new requests
	 * should be added at the tail
	 */
	if (current->bio_list) {     //point-1
		bio_list_add(current->bio_list, bio);
		return;
	}

	/* following loop may be a bit non-obvious, and so deserves some
	 * explanation.
	 * Before entering the loop, bio->bi_next is NULL (as all callers
	 * ensure that) so we have a list with a single bio.
	 * We pretend that we have just taken it off a longer list, so
	 * we assign bio_list to a pointer to the bio_list_on_stack,
	 * thus initialising the bio_list of new bios to be
	 * added.  ->make_request() may indeed add some more bios
	 * through a recursive call to generic_make_request.  If it
	 * did, we find a non-NULL value in bio_list and re-enter the loop
	 * from the top.  In this case we really did just take the bio
	 * of the top of the list (no pretending) and so remove it from
	 * bio_list, and call into ->make_request() again.
	 */
	BUG_ON(bio->bi_next);
	bio_list_init(&bio_list_on_stack);
	current->bio_list = &bio_list_on_stack;     //point-2
	do {
		struct request_queue *q = bdev_get_queue(bio->bi_bdev);
		q->make_request_fn(q, bio);
		bio = bio_list_pop(current->bio_list);  //point-3
	} while (bio);
	current->bio_list = NULL; /* deactivate */
}
```

这段代码有很多注释，结合上面我们的介绍，就比较容易理解了。若不了解block device stack和`generic_make_request`的递归，就会有这样的疑惑：`point-2`处把`current->bio_list`初始化为一个空的list，`point-3`处`bio_list_pop`必然会失败，又何须一个`do-while`循环呢？

我们用systeamtap抓一个栈看看。

首先，有一个设备/dev/sdb，把它分为3个分区，然后通过lvm创建逻辑卷。这么做的目的在于构造block device stack。

```
# parted /dev/sdb mklabel gpt

# parted /dev/sdb mkpart primary 0% 33%
# parted /dev/sdb mkpart primary 33% 66%
# parted /dev/sdb mkpart primary 66% 100%            

# parted /dev/sdb toggle 1 lvm
# parted /dev/sdb toggle 2 lvm                          
# parted /dev/sdb toggle 3 lvm                          

# parted /dev/sdb p
Number  Start   End     Size    File system  Name     Flags
 1      1049kB  14.2GB  14.2GB               primary  lvm
 2      14.2GB  28.3GB  14.2GB               primary  lvm
 3      28.3GB  42.9GB  14.6GB               primary  lvm

# pvcreate /dev/sdb1 /dev/sdb2 /dev/sdb3
# vgcreate vg-bar /dev/sdb1 /dev/sdb2 /dev/sdb3

# lvcreate --size 10G -n lv-bar-1 vg-bar
# lvcreate --size 10G -n lv-bar-2 vg-bar

# lvs
  LV       VG     Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv-bar-1 vg-bar -wi-a----- 10.00g                                                    
  lv-bar-2 vg-bar -wi-a----- 10.00g      

# lsblk /dev/sdb
NAME                   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sdb                      8:16   0   40G  0 disk 
├─sdb1                   8:17   0 13.2G  0 part 
│ └─vg--bar-lv--bar--1 253:2    0   10G  0 lvm  
├─sdb2                   8:18   0 13.2G  0 part 
│ └─vg--bar-lv--bar--2 253:3    0   10G  0 lvm  
└─sdb3                   8:19   0 13.6G  0 part 
```

写一个systemtap脚本：

```
# cat generic_make_request.stp 
probe begin {
    printf("trace generic_make_request begin\n");
}

probe kernel.function("generic_make_request") {
    printf("print_backtrace dev_t=%x current->bio_list=%p current->pid=%d\n", $bio->bi_bdev->bd_dev, task_current()->bio_list, task_current()->pid);
    print_backtrace();
}

probe end {
    printf("trace generic_make_request end\n");
}
```

然后，打开两个窗口，一个运行systemtap脚本，另一个往`lv-bar-2`写数据：

```
# stap -v -d kernel -d dm_mod generic_make_request.stp > log
```

```
# dd of=/dev/vg-bar/lv-bar-2 if=/dev/urandom bs=1024 count=1 &
[1] 3158    //dd进程的pid；
```

最后，看systemtap的log：

```
print_backtrace dev_t=800012 current->bio_list=0xffff880092077aa0 current->pid=3158
 0xffffffff812e6660 : generic_make_request+0x0/0x130 [kernel]
 0xffffffffa0002173 : __split_and_process_bio+0x253/0x410 [dm_mod]
 0xffffffffa00023b1 : dm_request+0x81/0xf0 [dm_mod]
 0xffffffff812e6740 : generic_make_request+0xe0/0x130 [kernel]
 0xffffffff812e6807 : submit_bio+0x77/0x150 [kernel]
 0xffffffff81230689 : _submit_bh+0x119/0x170 [kernel]
 0xffffffff81230ba2 : ll_rw_block+0x72/0xc0 [kernel]
 0xffffffff81230f16 : __block_write_begin+0x246/0x4b0 [kernel]
 0xffffffff812311c6 : block_write_begin+0x46/0x90 [kernel]
 0xffffffff812332d3 : blkdev_write_begin+0x23/0x30 [kernel]
 0xffffffff81180004 : generic_perform_write+0xc4/0x1d0 [kernel]
 0xffffffff8118230f : __generic_file_write_iter+0x15f/0x350 [kernel]
 0xffffffff8123383e : blkdev_write_iter+0x3e/0xc0 [kernel]
 0xffffffff811f9a0e : new_sync_write+0x8e/0xd0 [kernel] (inexact)
 0xffffffff811fa217 : vfs_write+0xb7/0x1f0 [kernel] (inexact)
 0xffffffff81022dfc : do_audit_syscall_entry+0x6c/0x70 [kernel] (inexact)
 0xffffffff811fae95 : sys_write+0x55/0xd0 [kernel] (inexact)
 0xffffffff81122b06 : __audit_syscall_exit+0x1f6/0x2a0 [kernel] (inexact)
 0xffffffff8168b209 : system_call_fastpath+0x12/0x17 [kernel] (inexact)
```

看第一行：
- dev_t=800012，说明本请求发给设备8:18的，这是/dev/sdb2；
- current->bio_list=0xffff880092077aa0，表明这是递归调用`generic_make_request`；
- current->pid=3158，请求进程的pid；即dd进程的pid；

# 死锁及解决 (3)

这种避免递归的方式在绝大多数情况下可以正常工作，但是在少数情况下可能导致死锁。问题在于：`generic_make_request`调用的`make_request_fn`是一个函数指针，它指向的函数是驱动开发者（例如md_mod的开发者）开发的，并且内核允许这个函数发生等待，即等待之前提交的bio完成。考虑第M层虚拟设备（`0<M<N`），它的`make_request_fn`向M-1层虚拟设备提交了一些bio，然后在提交一下一个bio之前，它去等待那些bio完成。由前面的递归避免策略可知，那些bio进入了`current->bio_list`。这就发生了死锁。

所以，死锁发生的关键是**一个bio等待之前提交的bio完成**。驱动开发者开发的`make_request_fn`通常不会等待之前提交的bio完成，但还是有一些情况比较微妙。一个例子是，某个虚拟设备由于请求size或者alignment的限制，`make_request_fn`必须把一个bio分裂成多个bio，所以它需要从一些mempool中分配额外的bio对象。假如，之前提交了一下bio，`make_request_fn`从mempool中分配了一些bio对象，现在又要提交一个bio，由于分裂，又要再分配。而这时候发生内存紧张，linux需要把脏页write-out到磁盘以便回收它们，而write-out也需要从mempool分配内存，这可能需要等待之前分配的bio对象被归还，即等待之前提交的bio完成。简言之：一个bio等待writ-out，write-out等待之前提交的bio完成，就构成了一个bio间接等待之前提交的bio完成的情况。这些情况通常难以通过代码审查发现，而需要测试暴露。

解决这种死锁的尝试之一是bioset：

```
# ps -ef |grep bioset
root         30      2  0 20:30 ?        00:00:00 [bioset]
root        382      2  0 20:30 ?        00:00:00 [bioset]
root        394      2  0 20:30 ?        00:00:00 [bioset]
root       2737      2  0 20:34 ?        00:00:00 [bioset]
root       2750      2  0 20:34 ?        00:00:00 [bioset]
```

即内核有一些bioset线程，一个线程对应一个mempool。当一个mempool已满而不能再分配bio对象的时候，之前从这个mempool中分配出来的bio对象（现在在`current->bio_list`中）就转交给这个mempool对应的bioset线程来处理。这就打破了死锁：bioset线程处理完bio，bio对象就能归还到mempool中。

这种方式比较重，需要创建一些内核线程，而这些内核线程又很少被使用（死锁很少发生），所以，linux 4.11又作了一些优化（以上部分是基于linux 3.19.8的），不在本文的范围之内。

参考：
https://lwn.net/Articles/736534/
