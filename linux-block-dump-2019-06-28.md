---
title: 使用block_dump观察block io 
date: 2019-06-28 18:31:32
tags: [kernel,io,block]
categories: linux 
---

Linux提供了一种机制，可以用来dump出block io详情，即/proc/sys/vm/block_dump。

<!-- more -->

# 如何使用 (1)

```shell
# echo 1 > /proc/sys/vm/block_dump

read/write some files

# dmesg
...
[10489963.254731] kworker/u65:0(119773): WRITE block 3017556368 on sdm (136 sectors)
[10489963.254763] kworker/u65:0(119773): WRITE block 3017556360 on sdm (8 sectors)
[10489963.367301] prometheus(142509): dirtied inode 177997980 (00000050) on sdm
[10489963.367312] prometheus(142509): dirtied inode 177997980 (00000050) on sdm
...

# echo 0 > /proc/sys/vm/block_dump
```

# 原理 (2)

在当前的linux内核中(下面以kernel-3.19.8为例)有两个地方检察block_dump，若为true，则打印相应的block io信息。信息的格式如第1节所示。

block/blk-core.c

```c
void submit_bio(int rw, struct bio *bio)
{
    ...

    if (unlikely(block_dump)) {
      char b[BDEVNAME_SIZE];
      printk(KERN_DEBUG "%s(%d): %s block %Lu on %s (%u sectors)\n",
      current->comm, task_pid_nr(current),
        (rw & WRITE) ? "WRITE" : "READ",
        (unsigned long long)bio->bi_iter.bi_sector,
        bdevname(bio->bi_bdev, b),
        count);
    }

    ...
}
```

这段代码输入形如：

```
[10489963.254731] kworker/u65:0(119773): WRITE block 3017556368 on sdm (136 sectors)
```

`[10489963.254731] kworker/u65:0`是`current->comm`，即`struct task_struct`的comm字段，注释上说"executable name excluding path"；
`119773`是`task_pid_nr(current)`，即线程id；
`WRITE`表明当前block io是写操作；
`3017556368`是`bio->bi_iter.bi_sector`，这个值是`struct bio`的`struct bvec_iter bi_iter`中的`bi_sector`，表示当前block io的第一个sector号；
`sdm`是`bdevname(bio->bi_bdev, b)`，设备名；
`136`是count，当前block io的sector数；

也就是说，当前block io是写sdm从3017556368号sector开始的136个sector；发起线程是kworker/u65:0(119773)。



fs/fs-writeback.c

```c
void __mark_inode_dirty(struct inode *inode, int flags)
{
  ...

  if (unlikely(block_dump))
    block_dump___mark_inode_dirty(inode);

  ...
}

static noinline void block_dump___mark_inode_dirty(struct inode *inode)
{
  if (inode->i_ino || strcmp(inode->i_sb->s_id, "bdev")) {
    struct dentry *dentry;
    const char *name = "?";

    dentry = d_find_alias(inode);
    if (dentry) {
      spin_lock(&dentry->d_lock);
      name = (const char *) dentry->d_name.name;
    }
    printk(KERN_DEBUG
           "%s(%d): dirtied inode %lu (%s) on %s\n",
           current->comm, task_pid_nr(current), inode->i_ino,
           name, inode->i_sb->s_id);
    if (dentry) {
      spin_unlock(&dentry->d_lock);
      dput(dentry);
    }
  }
}
```

这段代码输入形如：

```
[10489963.367301] prometheus(142509): dirtied inode 177997980 (00000050) on sdm
```

`177997980`是`inode->i_ino`，即inode号；
`00000050`是文件名；
`sdm`是设备名；

# rsyslog (3)

打开block_dump会产生很多KERN_DEBUG级别的日志。若系统的rsyslog配置的级别包含KERN_DEBUG，这些log将会写入/var/log/messages，可能影响系统性能。
```shell
# vim /etc/rsyslog.conf
*.info;*.debug;mail.none;authpriv.none;cron.none                /var/log/messages
```

这时可以暂时停掉rsyslog:
```
systemctl stop rsyslog.service
```

# 小结 (4)

这个机制，使我们很容易的拿到generic block层的输入，以及writeback时的dirty inode。

