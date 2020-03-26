---
title: Linux file read 
date: 2020-03-26 21:38:21
tags: [io]
categories: linux 
---

本文基于linux kernel 3.19.8版本梳理一下读文件的过程，从vfs开始，到发送请求给block层结束。

<!-- more -->

<script type="text/x-mathjax-config">
MathJax.Hub.Config({
tex2jax: {inlineMath: [['$','$'], ['\\(','\\)']]}
});
</script>

<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>

# 普通模式 (1)

这里的普通是指没有设置`O_DIRECT`（也没设置`O_SYNC`，但它只对write起作用，read肯定是同步的，所以不提）。

## `vfs_read()`

这个函数接收4个参数：

* `struct file *file`：读取的目标文件；
* `char __user *buf`：用户态buffer的地址；
* `size_t count`：用户态buffer的大小；
* `loff_t *pos`：目标文件中读取的起始位置；

首先做一些检查，包括：

* 读权限检查，`FMODE_READ`和`FMODE_CAN_READ`；
* 用户态指针检查，即检查应用程序提供的buffer是否合法。注意`access_ok(VERIFY_WRITE, buf, count)`里面的`VERIFY_WRITE`是从buffer的角度出发的，读完数据要往buffer里写嘛。
* 调用`rw_verify_area`检查：1. 参数检查，`*pos`是否小于0，`*pos+count`是否小于0；2. 是否有mandatory lock冲突；3. security permission；

若这些检查都通过，则调用`__vfs_read()`，把自己的4个参数传过去。

## `__vfs_read()`

这个函数从`vfs_read()`接收那4个参数，直接传入filesystem相关的函数`file->f_op->read`。这是一个函数指针，对于ext4和xfs来说它都指向`new_sync_read`。

## `new_sync_read()`

到现在，我们手上还是那4个参数（只是参数名略不同）:

* `struct file *filp`;
* `char __user *buf`;
* `size_t len`;
* `loff_t *ppos`;

这个函数用这4个参数，初始化3个结构体：

* `struct iovec iov = { .iov_base = buf, .iov_len = len };`：即用户态buffer；
* `struct iov_iter iter;`：指向用户态buffer的迭代器，用于fill用户态buffer；
* `struct kiocb kiocb;`：这个结构体对于同步read和libaio有不同用法，我们当前讨论的是同步read，主要用到里面的这5个字段：`ki_ctx=NULL`; `ki_filp=filp`；`ki_obj.tsk=current`; `ki_pos=*ppos`; `ki_nbytes=len`；其中`ki_ctx=NULL`表明这是一个同步read而不是libaio；

初始化完成之后，再次调用filesystem相关的函数`filp->f_op->read_iter`，传入两个参数：`&kiocb`和`&iter`，前者代表从哪里读；后者代表读到的输入放到哪里。`filp->f_op->read_iter`对于ext4指向的是`generic_file_read_iter`；对于xfs指向的是`xfs_file_read_iter`。`xfs_file_read_iter`做一些处理之后也最终调用`generic_file_read_iter`，所以我们看这个函数。

## `generic_file_read_iter()`

如前所述，这个函数两个参数是`struct kiocb *iocb`和`struct iov_iter *iter`。它从`iocb`中取出:

*	`struct file *file = iocb->ki_filp;`
*	`loff_t *ppos = &iocb->ki_pos;`

如果`O_DIRECT`没有设置（正是我们讨论的case），直接调用`do_generic_file_read`，传入`file`, `ppos`, `iter`和`retval`（值为0）。

## `do_generic_file_read()`

这个函数就比较复杂了。它从前面`generic_file_read_iter()`得到的参数是：

* `struct file *filp`：目标文件；
* `loff_t *ppos`：目标文件中读取的起始位置；
* `struct iov_iter *iter`：指向用户态buffer；
* `ssize_t written`：值为0；

首先是初始化一些变量：

* `struct address_space *mapping = filp->f_mapping;`：`mapping`里面的`page_tree`就是pagache；
* `struct inode *inode = mapping->host;`： `mapping`的owner inode。1. 对于普通文件来说，就是`filp->f_dentry->d_inode`；2. 对于block device file来说，就是bdev special filesystem里面的inode；`filp`就是/dev/sdc这样的文件，所属文件系统是devtmpfs；这个文件只是bdev special filesystem里面的文件的"指针"；bdev special filesystem没有mount，所以不可见。"指针"是这样实现的：devtmpfs文件系统中的inode有两个字段：`i_bdev`和`i_rdev`；其中`i_bdev`是一个指向`struct block_device`的指针，而`struct block_device`嵌套在`struct bdev_inode`之内（可以通过`struct block_device`的地址得到外面`struct bdev_inode`的地址）。`struct bdev_inode`是什么呢？它就是bdev special filesystem内的inode。所以说，可以认为devtmpfs文件系统中的inode的`i_bdev`指向bdev special filesystem内的inode。当这个`i_bdev`为NULL的时候，就用到`i_rdev`，它是主设备号和从设备号，可以通过它去打开`struct block_device`，然后把返回的地址存入`i_bdev`。见`blkdev_open()`函数。所以说，**devtmpfs文件系统中的文件是bdev special filesystem里面的文件的"指针"或"快捷方式"**。这里拿到的`inode`就是bdev special filesystem里的inode，它是pagacache的owner，也是服务读请求的主体。
* `index`：把文件看做是一个连续的page（通常是4KiB）数组，`index`就是read请求的第1字节所在的page号，也就是这个page在pagecache中的位置；
* `last_index`：read请求的最后1字节所在page的后面一个page的号（注意不是最后1字节所在的那个page的号），也就是那个page在pagecache中的位置。这样形成一个前闭后开的区间`[index, last_index)`；
* `offset`：read请求的第1字节在它所在的page内的偏移。虽然内核是按page从底层读数据的，但返回给用户必须精确到字节，`offset`就是用来标记从第1个page的哪个位置开始往iter（指向用户态buffer）拷贝数据。当read请求区间跨page时，对于后续的page，`offset`被更新为0，表示从page的开头开始拷贝。
* 还有read ahead相关的`prev_index`和`prev_offset`，暂且不看它们。

然后就开始一个大循环。假如read ahead完全没有起到作用，一次循环从block层读取一个page并拷贝到iter（用户态buffer），即page-by-page的读；否则，read ahead一下获取了很多page，一次循环只是拷贝一个page到iter（用户态buffer）。值得一提的是，**read ahead完全起不到作用的情况极少发生**，即使用户通过`posix_fadvise`设置了`POSIX_FADV_RANDOM`，比较新的内核也会走`force_page_cache_readahead`流程，详见[read ahead](https://www.yuanguohuo.com/2020/03/24/linux-read-ahead/)；

首先还是先看一下page-by-page的读吧，虽然它极少发生。

- 调用`find_get_page(mapping, index)`在pagecache中找index指向的page；
- 没有找到，从而调用`page_cache_sync_readahead`（对于read ahead被完全禁止的情况，`ra->ra_pages=0`，所以这个函数什么都不做直接返回），然后再调用`find_get_page`；
- 当然还是找不到，从而goto no_cached_page；在这里分配一个内存page，添加到pagacache中，然后goto readpage；
- 在`readpage`处，调用`mapping->a_ops->readpage`；它是一个文件系统相关的函数指针，对于ext4来说，指向`ext4_readpage`。`ext4_readpage`再调用`mpage_readpage`，构造一个bio，发到block层。返回之后，goto page_ok；
- 在`page_ok`处，把page的数据拷贝到`iter（用户态buffer）`
- continue进行下一轮循环；
