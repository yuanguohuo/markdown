---
title: block设备stack中的递归避免 
date: 2019-10-04 13:17:11
tags: [linux,block]
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

<div align=center>![layered-block-device](layered-block-device.jpg)

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

首先，有一个设备/dev/sdb，把它分为两个分区：/dev/sdb1和/dev/sdb2，并在/dev/sdb2上创建ext4文件系统。

```
```

```
```
