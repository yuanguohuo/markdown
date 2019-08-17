---
title: linux iostat详解
date: 2019-08-17 08:40:28
tags: [linux,iostat]
categories: linux 
---

首先介绍`struct disk_stats`的字段，接着介绍如何基于这些字段生成/proc/diskstats，然后介绍如何基于/proc/diskstats生成iostat的输出。本文基于linux kernel 3.19.8。

<!-- more -->

# struct disk_stats (1)

这个结构体定义在include/linux/genhd.h中。它针对一个part(它可能代表一个分区也可能代表一整个disk)，统计从系统启动到当前时刻的所有read/write请求。

```c
struct disk_stats {
  unsigned long sectors[2]; /* READs and WRITEs */
  unsigned long ios[2];
  unsigned long merges[2];
  unsigned long ticks[2];
  unsigned long io_ticks;
  unsigned long time_in_queue;
};
```

* `sectors[2]`    : read扇区的数量和write扇区的数量；
* `ios[2]`        : read请求数和write请求数；
* `merges[2]`     : read请求的合并次数和write请求的合并次数；
* `ticks[2]`      : read请求从初始化到完成消耗的jiffies累计，和write请求从初始化到完成消耗的jiffies累计；
* `io_ticks`      : 该分区上存在请求(不管是read还是write)的jiffies累计；
* `time_in_queue` : 该分区上存在的请求数量(不管是read还是write)与逝去jiffies的加权累计；


## sectors字段 (1.1)

`sectors`字段是在请求结束阶段统计的：

```c
void blk_account_io_completion(struct request *req, unsigned int bytes)
{
  if (blk_do_io_stat(req)) {
    const int rw = rq_data_dir(req);
    struct hd_struct *part;
    int cpu;

    cpu = part_stat_lock();
    part = req->part;
    part_stat_add(cpu, part, sectors[rw], bytes >> 9);
    part_stat_unlock();
  }
}
```

`part_stat_add`是一个macro，它累加part的`struct disk_stats`的某个字段。注意，如果`part->partno`为0，那么这个part其实代表的是一整个disk；否则`part->partno`不为0，这个part代表的是disk的一个分区，在这种情况下，还要累加整个disk的同一字段(`part_to_disk((part))`得到disk，其`part0`字段就是代表整个disk的part)。其定义如下：

```c
#define part_stat_add(cpu, part, field, addnd)  do {      \
  __part_stat_add((cpu), (part), field, addnd);     \
  if ((part)->partno)           \
    __part_stat_add((cpu), &part_to_disk((part))->part0,  \
        field, addnd);        \
} while (0)
```

`rq_data_dir(req)`拿到req的方向(direction)，即read(0)还是write(1)。`bytes >> 9`是根据字节数计算扇区数。`part_stat_add(cpu, part, sectors[rw], bytes >> 9)`就是做相应的累加。


## ios字段 (1.2)

`ios`字段也是在请求结束阶段统计的。请求结束时的调用是这样的：

```c
blk_end_request
    ----> blk_end_bidi_request	
              ----> blk_update_bidi_request
                        ----> blk_update_request
                                  ----> blk_account_io_completion
              ----> blk_finish_request
                        ----> blk_account_io_done
```

如1.1节所述，`sectors`的统计是在`blk_account_io_completion`中完成的。而`ios`的统计是在`blk_account_io_done`中完成的：

```c
void blk_account_io_done(struct request *req)
{
  /*
   * Account IO completion.  flush_rq isn't accounted as a
   * normal IO on queueing nor completion.  Accounting the
   * containing request is enough.
   */
  if (blk_do_io_stat(req) && !(req->cmd_flags & REQ_FLUSH_SEQ)) {
    unsigned long duration = jiffies - req->start_time;
    const int rw = rq_data_dir(req);
    struct hd_struct *part;
    int cpu;

    cpu = part_stat_lock();
    part = req->part;

    part_stat_inc(cpu, part, ios[rw]);
    part_stat_add(cpu, part, ticks[rw], duration);
    part_round_stats(cpu, part);
    part_dec_in_flight(part, rw);

    hd_struct_put(part);
    part_stat_unlock();
  }
}
```

`part_stat_inc`是一个macro，调用1.1节中介绍过的`part_stat_add`。它完成的工作是给part的`struct disk_stats`某个字段(这里是`ios`字段)加1；当然，如果part代表的是一个分区，它还会给disk的同一字段加1。

```c
#define part_stat_inc(cpu, gendiskp, field)       \
  part_stat_add(cpu, gendiskp, field, 1)
```

## ticks字段 (1.3)

`ticks`字段也是在前述`blk_account_io_done`函数中统计的。首先通过`duration = jiffies - req->start_time`计算请求从初始化到完成的jiffies，然后通过`part_stat_add`累加到part的`ticks`(若该part代表的是分区，还会累加到disk的`ticks`)。`req->start_time`是在`blk_rq_init`中设定的：

```c
void blk_rq_init(struct request_queue *q, struct request *rq)
{
  memset(rq, 0, sizeof(*rq));

  INIT_LIST_HEAD(&rq->queuelist);
  INIT_LIST_HEAD(&rq->timeout_list);
  rq->cpu = -1;
  rq->q = q;
  rq->__sector = (sector_t) -1;
  INIT_HLIST_NODE(&rq->hash);
  RB_CLEAR_NODE(&rq->rb_node);
  rq->cmd = rq->__cmd;
  rq->cmd_len = BLK_MAX_CDB;
  rq->tag = -1;
  rq->start_time = jiffies;
  set_start_time_ns(rq);
  rq->part = NULL;
}
```

## io_ticks字段和time_in_queue字段 (1.4)

`io_ticks`和`ticks`关系不大：如1.3节所述，后者是各个read/write请求从初始化(`blk_rq_init`)到完成经历的jiffies的累计；前者是part上存在请求(不管是read还write)的时间(jiffies)的累计。怎么理解呢？在part上的请求的数量发生变化的地方(如请求开始、结束和merge的地方)，去看刚刚逝去的这一段时间(jiffies)里part上是不是存在请求。若存在，这段jiffies就累加到`io_ticks`上；若不存在，则不累加。

反而`io_ticks`和`time_in_queue`的关系更大：`time_in_queue`是分区上存在的请求数量与jiffies的加权累计。什么意思呢？和统计`io_ticks`一样，还是在part上的请求数量发生变化的地方，去看刚刚逝去的这一段时间(jiffies)里part上是不是存在请求。不同的是，若存在请求，则把(存在的请求数量 \* jiffies)累加到`time_in_queue`，否则不累加。

大致可以这么理解：`io_ticks`更注重server忙的时间；`time_in_queue`更注重client等待的时间。比如超市收银员，我们想看他的繁忙程度：从他早晨上班开始，`io_ticks`统计的是结账队列不空的时间总和；`time_in_queue`统计的是所有顾客排队结账消耗的时间总和。

这两者的统计都是在`part_round_stats`中完成的。这个函数在part上的请求数量发生变化的时候被调用(如前面的`blk_account_io_done`函数)。

```c
void part_round_stats(int cpu, struct hd_struct *part)
{
  unsigned long now = jiffies;

  if (part->partno)
    part_round_stats_single(cpu, &part_to_disk(part)->part0, now);
  part_round_stats_single(cpu, part, now);
}
```

和前述`part_stat_add`一样，若part代表的是一个分区，则不但要为分区作统计，而且还要为它所在的disk作统计。具体统计的过程是这样的：

```c
static void part_round_stats_single(int cpu, struct hd_struct *part,
            unsigned long now)
{
  int inflight;

  if (now == part->stamp)
    return;

  inflight = part_in_flight(part);
  if (inflight) {
    __part_stat_add(cpu, part, time_in_queue,
        inflight * (now - part->stamp));
    __part_stat_add(cpu, part, io_ticks, (now - part->stamp));
  }
  part->stamp = now;
}
```

我们看这段代码，每次被调用时：

* 1. 计算距离上次被调用逝去的时间(`now - part->stamp`)；
* 2. 看是否**存在**请求(if (inflight))；
* 3. 若**存在**，则 `time_in_queue`累加上(请求数\*逝去时间)；`io_ticks`累加上逝去时间；
* 4. 更新被调用时间，为下次被调用做准备(`part->stamp = now`)；


**存在**的定义是：part上的`in_flight`大于0。`in_flight`是在这两个函数中完成的：

```c
static inline void part_inc_in_flight(struct hd_struct *part, int rw)
{
  atomic_inc(&part->in_flight[rw]);
  if (part->partno)
    atomic_inc(&part_to_disk(part)->part0.in_flight[rw]);
}

static inline void part_dec_in_flight(struct hd_struct *part, int rw)
{
  atomic_dec(&part->in_flight[rw]);
  if (part->partno)
    atomic_dec(&part_to_disk(part)->part0.in_flight[rw]);
}
```

很明显，这两个函数分别是增加和减小part上的`in_flight`值(当part代表分区时，也会对disk做同样的统计)。`part_dec_in_flight`在前面的`blk_account_io_done`中被调用。`part_inc_in_flight`在`blk_account_io_start`中被调用。`blk_account_io_start`和`blk_account_io_done`是对称的，一个在请求开始阶段，一个在请求结束阶段。

在`blk_account_io_start`中，我们只看`new_io`为true的情况(为false的情况见下文1.5节)，除去异常分成相当直观：

```c
void blk_account_io_start(struct request *rq, bool new_io)
{
  struct hd_struct *part;
  int rw = rq_data_dir(rq);
  int cpu;

  if (!blk_do_io_stat(rq))
    return;

  cpu = part_stat_lock();

  if (!new_io) {
    part = rq->part;
    part_stat_inc(cpu, part, merges[rw]);
  } else {
    part = disk_map_sector_rcu(rq->rq_disk, blk_rq_pos(rq));
    if (!hd_struct_try_get(part)) {
      /*
       * The partition is already being removed,
       * the request will be accounted on the disk only
       *
       * We take a reference on disk->part0 although that
       * partition will never be deleted, so we can treat
       * it as any other partition.
       */
      part = &rq->rq_disk->part0;
      hd_struct_get(part);
    }
    part_round_stats(cpu, part);
    part_inc_in_flight(part, rw);
    rq->part = part;
  }

  part_stat_unlock();
}
```

总结来说：`io_ticks`和`time_in_queue`的统计是这样的：
- 在一个请求产生的时候(part的请求数量发生变化)：1.调用`part_round_stats`(是否把最近这一段时间累计到`io_ticks`和`time_in_queue`)；2. `in_flight++`；
- 在一个请求结束的时候(part的请求数量发生变化)：1.调用`part_round_stats`(是否把最近这一段时间累计到`io_ticks`和`time_in_queue`)；2. `in_flight--`；

这里需要强调一点：`in_flight`是进入elevator的数量。只要elevator不空，`io_ticks`就累计。所以`io_ticks`代表的是elevator不空的时间。也就是说，基于它得到的磁盘繁忙程度(iostat的util，见下文)其实不能精确代表物理硬件的繁忙程度，若把elevator往下(包含elevator)看成一个黑盒的话，它代表的是这个黑盒的繁忙程度。

## merges字段 (1.5)

`merges`字段在`blk_account_io_start`函数中统计(见1.4节)：`new_io`为false，表示本请求不是一个新请求，而是一个合并的请求，所以累加merges，不难理解。`bio_attempt_back_merge`和`bio_attempt_front_merge`在合并成功的时候，以`new_io`为false来调用本函数。

# /proc/diskstats (2)

# iostat (3)
