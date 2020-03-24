---
title: Linux read ahead 
date: 2020-03-24 20:32:38
tags: [io,read]
categories: linux 
---

应用程序可以通过`posix_fadvise()`来告诉内核访问文件的模式，**建议**内核如何进行IO以达到最优性能（如名字所示，它仅仅是一个建议或期望，内核不承诺遵守）。可用的模式有：`POSIX_FADV_NORMAL`，`POSIX_FADV_SEQUENTIAL`，`POSIX_FADV_RANDOM`，`POSIX_FADV_NOREUSE`，`POSIX_FADV_WILLNEED`，`POSIX_FADV_DONTNEED`。本文看一看它们的行为。

<!-- more -->

<script type="text/x-mathjax-config">
MathJax.Hub.Config({
tex2jax: {inlineMath: [['$','$'], ['\\(','\\)']]}
});
</script>

<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>

# `POSIX_FADV_RANDOM` (1)

先做个测试：

|`bs`(KiB)     |`read_ahead_kb`(KiB)    |`max_sectors_kb`(KiB)     |`avgrq-sz`(KiB)      |
|--------------|------------------------|--------------------------|---------------------|
|16            |8                       |4                         |4                    |
|16            |8                       |64                        |8                    |
|16            |32                      |64                        |16                   |

说明：

* 测试是fio发起的，顺序读但设置`fadvise_hint=random`（这样是为了突出`POSIX_FADV_RANDOM`的作用）；文件系统是ext4；`bs`如表中所示；
* `read_ahead_kb`：/sys/block/sda/queue/read_ahead_kb；
* `max_sectors_kb`：/sys/block/sda/queue/max_sectors_kb；
* `avgrq-sz`：iostat观察到的请求大小；

结果是：

$$ avgrq-sz = min{bs, read_ahead_kb, max_sectors_kb} $$

一些文档说`POSIX_FADV_RANDOM`用于禁止read ahead。可是，要是read ahead被禁止的话，请求应该是page-by-page的（见linux 3.19.8 `mm/filemap.c:do_generic_file_read()`），也就是`avgrq-sz=4KiB`（通常page size是4KiB），而实际上并不是如此，这是为什么呢？

找到[patch-187024](https://lore.kernel.org/patchwork/patch/187024/)才发现，在过去的版本中的确是page-by-page的，这很低效，现在已经不是这样了：

```
This fixes inefficient page-by-page reads on POSIX_FADV_RANDOM.

POSIX_FADV_RANDOM used to set ra_pages=0, which leads to poor
performance: a 16K read will be carried out in 4 _sync_ 1-page reads.
```

现在`ra_pages=0`（完全禁止read ahead，page-by-page的读）只在multi-page读没有帮助或者应该被避免的地方：

```
- it's ramfs/tmpfs/hugetlbfs/sysfs/configfs
- some IO error happened
```

现在`POSIX_FADV_RANDOM`的语义不是禁止read ahead，而是禁止基因算法：

```
POSIX_FADV_RANDOM actually want a different semantics: to disable the
*heuristic* readahead algorithm, and to use a dumb one which faithfully
submit read IO for whatever application requests.

So introduce a flag FMODE_RANDOM for POSIX_FADV_RANDOM.
```

直接使用请求大小(`bs`)，当然，也受限于`read_ahead_kb`和`max_sectors_kb`（三者中取最小）。 

看这个patch的diff:

linux 3.19.8 `mm/fadvise.c:fadvise64_64`:

```
 	switch (advice) {
 	case POSIX_FADV_NORMAL:
 		file->f_ra.ra_pages = bdi->ra_pages;
+		spin_lock(&file->f_lock);
+		file->f_flags &= ~FMODE_RANDOM;
+		spin_unlock(&file->f_lock);
 		break;
 	case POSIX_FADV_RANDOM:
-		file->f_ra.ra_pages = 0;
+		spin_lock(&file->f_lock);
+		file->f_flags |= FMODE_RANDOM;
+		spin_unlock(&file->f_lock);
 		break;
 	case POSIX_FADV_SEQUENTIAL:
 		file->f_ra.ra_pages = bdi->ra_pages * 2;
+		spin_lock(&file->f_lock);
+		file->f_flags &= ~FMODE_RANDOM;
+		spin_unlock(&file->f_lock);
 		break;
 	case POSIX_FADV_WILLNEED:
```

linux 3.19.8 `mm/readahead.c:page_cache_sync_readahead()`:

```
 	if (!ra->ra_pages)
 		return;
 
+	/* be dumb */
+	if (filp->f_mode & FMODE_RANDOM) {
+		force_page_cache_readahead(mapping, filp, offset, req_size);
+		return;
+	}
+
 	/* do read-ahead */
 	ondemand_readahead(mapping, ra, filp, false, offset, req_size);
```

就是`POSIX_FADV_RANDOM`不再设置`ra_pages=0`，而是设置`FMODE_RANDOM`（`FMODE_RANDOM`就是这个patch新引入的）；在`page_cache_sync_readahead()`中就不会立即返回（立即返回就会page-by-page的读），而是检查`FMODE_RANDOM`，从而进行强制read ahead。


# `POSIX_FADV_SEQUENTIAL` (2)
