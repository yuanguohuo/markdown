---
title: block layer的plug
date: 2019-10-04 23:29:09
tags: [kernel,io,block,plug]
categories: linux 
---

Block层的请求在device的queue里会发生reorder与merge以提高效率，然而，在进入device的queue之前也会做同样的努力，这就是plug机制。

<!-- more -->

<script type="text/x-mathjax-config">
MathJax.Hub.Config({
tex2jax: {inlineMath: [['$','$'], ['\\(','\\)']]}
});
</script>

<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>

# 增大merge的机会 (1)

通常，减小per-request开销（例如HDD的寻道时间）的有效方式是merge：把几个小的request合并成一个连续的、大的request。每个device都会有一个（对于single-queue而言）queue；IO scheduler在这个queue上做各种reorder和merge，以提高效率。这里的merge有一些缺点：

- 当device比较快，或者device虽然不快但比较闲，那么merge的机会就会比较小。这不是大问题，虽然从整体上看device的效率不高，但是能满足请求；
- device的queue需要lock来保证一致性，这在多CPU的情况下会增大竞争开销；

Linux 2.6.39之前也有plug机制，但它是在device的queue上进行的，所以只能解决第一点。Linux 2.6.39引入per-process的plug机制。本文讨论的是后者，所引用的代码来自linux 3.19.8。

# per-process plug (2)

文件系统（或者block device的其它客户端）提交一批请求之前：

- 调用`blk_start_plug()`在`task_struct`上安装一个list；
- 多次调用`generic_make_request()`提交这一批请求；请求在list中reorder与merge；
- 调用`blk_finish_plug()`把list中的合并得到的大请求flush到device的queue；调用`schedule()`也会触发这样的flush；

下图简单的表示了这个过程，其中`mq_list`用于multi-queue（blk-mq），`cb_list`用于md，暂时忽略。

{% asset_img plug-unplug.jpg plug and unplug %}

在`task_struct`中维护这个list有一个好处：进程在调用`blk_start_plug()`和`blk_finish_plug()`之间若发生block，即调用`schedule()`，可以在block住之前**方便地**找到pending的请求（就在`task_struct`的list中），并flush它们。在进程block之前flush掉pending的请求非常重要：

- 提高性能：IO请求不会被delay到block之后；
- 避免死锁：比如，进程block就是为了等待pending的请求；再如，进程block是为了分配内存，进而需要reclaim，而reclaim又要等待pending的请求占用的内存页。这些情况下，block之前若不把pending的请求发下去就会死锁。


## 开始plug (2.1)

代码引自linux 3.19.8:

```c
void blk_start_plug(struct blk_plug *plug)
{
    struct task_struct *tsk = current;

    INIT_LIST_HEAD(&plug->list);
    INIT_LIST_HEAD(&plug->mq_list);
    INIT_LIST_HEAD(&plug->cb_list);

    /*
     * If this is a nested plug, don't actually assign it. It will be
     * flushed on its own.
     */
    if (!tsk->plug) {
        /*
         * Store ordering should not be needed here, since a potential
         * preempt will imply a full memory barrier
         */
        tsk->plug = plug;
    }
}
```

`blk_start_plug()`非常简单，把一个`struct blk_plug`对象安装到`struct task_struct`中。另外从代码上看，plug还可以嵌套的（注意只有最底层的`plug`才会安装到`struct task_struct`）：

- 调用`blk_start_plug(plug1)`
- 提交请求req1，req1被pending在plug1上；
- 调用`blk_start_plug(plug2)`
- 提交请求req2，req2被pending在plug2上；
- 调用`blk_finish_plug(plug2)` flush pending在plug2上的req2；
- 提交请求req3，req3被pending在plug1上；
- 调用`blk_finish_plug(plug1)` flush pending在plug1上的req1和req3；

## pending请求 (2.2)

```c
generic_make_request(bio)
    make_request_fn  (以blk_queue_bio为例)
        blk_queue_bio
            blk_attempt_plug_merge //尝试merge到plug中，成功则返回
            //尝试失败
            //尝试合并到device的queue，成功则返回
            //尝试失败
            //为bio生成一个struct request实例req
            plug = current->plug;
            if (plug) { //client安装了plug
                if (!request_count)  //plug中没有任何请求，此为第一个，blktrace中生成'P(plug)'事件
                  trace_block_plug(q);
                else {
                  if (request_count >= BLK_MAX_REQUEST_COUNT) { //plug中有足够多的请求，结束它，重新开始；
                    blk_flush_plug_list(plug, false);
                    trace_block_plug(q);
                  }
                }
                
                list_add_tail(&req->queuelist, &plug->list); //把req放入plug中；
            }
            //......
```

两点值得注意一下：

- 当plug中pending的请求数到达`BLK_MAX_REQUEST_COUNT`（默认是16）时，就会触发flush；然后重新开始新一轮的plug；
- plug开始和结束（即flush）的时候，分别生成`P(plug)`和`U(unplug)`事件。见[blktrace](http://www.yuanguohuo.com/2019/08/28/linux-blktrace/).

## 结束plug (2.3)

结束plug时，会flush掉pending的请求。如前所述，大致有3个flush时机（没有找到timeout导致的flush，per-process的plug机制没有这种方式？）：

- 文件系统（或者block device的其它客户端）显式地调用`blk_finish_plug()`；
- pending的请求数到达`BLK_MAX_REQUEST_COUNT`；
- 进程block，即调用`schedule()`；

其中第2种前面已经展示。`blk_finish_plug()`代码如下（linux 3.19.8）：

```c
void blk_finish_plug(struct blk_plug *plug)
{
  blk_flush_plug_list(plug, false);

  if (plug == current->plug)
    current->plug = NULL;
}
```

`schedule()`在block之前（`__schedule()`）的调用关系是这样的：

```c
asmlinkage __visible void __sched schedule(void)
{
  struct task_struct *tsk = current;

  sched_submit_work(tsk);
      blk_schedule_flush_plug(tsk);    
          blk_flush_plug_list(plug, true);

  __schedule();
}
```

可见，flush是由`blk_flush_plug_list()`函数完成的，其中第2个参数表明flush是不是schedule触发的，若是，则以异步的方式处理pending的请求。

```c
void blk_flush_plug_list(struct blk_plug *plug, bool from_schedule)
{
    struct request_queue *q;
    unsigned long flags;
    struct request *rq;
    LIST_HEAD(list);
    unsigned int depth;

    flush_plug_callbacks(plug, from_schedule);

    if (!list_empty(&plug->mq_list))   //对于multi-queue, plug->mq_list不为空
        blk_mq_flush_plug_list(plug, from_schedule);

    //核心是flush plug->list中的请求

    //若plug->list中无请求，则返回
    if (list_empty(&plug->list))
        return;

    //把pending的请求拷贝到临时变量，并sort
    list_splice_init(&plug->list, &list);

    list_sort(NULL, &list, plug_rq_cmp);

    q = NULL;
    depth = 0;

    /*
     * Save and disable interrupts here, to avoid doing it for every
     * queue lock we have to take.
     */
    local_irq_save(flags);
    while (!list_empty(&list)) {
        rq = list_entry_rq(list.next);
        list_del_init(&rq->queuelist);
        BUG_ON(!rq->q);
        if (rq->q != q) {
            //不考虑第一次进入循环q为NULL的情况，这时，已经把一些请求
            //插入q(device的queue)中了，而下一个pending的请求不属于这
            //个device(因为queue不同)，就运行(见queue_unplugged)这个
            //device的queue；

            /*
             * This drops the queue lock
             */
            if (q)
                queue_unplugged(q, depth, from_schedule);
            q = rq->q;
            depth = 0;
            spin_lock(q->queue_lock);
        }

        /*
         * Short-circuit if @q is dead
         */
        if (unlikely(blk_queue_dying(q))) {
            __blk_end_request_all(rq, -ENODEV);
            continue;
        }

        //把请求插入device的queue
        /*
         * rq is already accounted, so use raw insert
         */
        if (rq->cmd_flags & (REQ_FLUSH | REQ_FUA))
            __elv_add_request(q, rq, ELEVATOR_INSERT_FLUSH);
        else
            __elv_add_request(q, rq, ELEVATOR_INSERT_SORT_MERGE);

        depth++;
    }

    //最后一个device的queue，在前面while循环中没有运行，这里运行它；
    /*
     * This drops the queue lock
     */
    if (q)
        queue_unplugged(q, depth, from_schedule);

    local_irq_restore(flags);
}
```

运行device的queue的代码如下：

```c
static void queue_unplugged(struct request_queue *q, unsigned int depth,
          bool from_schedule)
  __releases(q->queue_lock)
{
  //blktrace的'U(unplug)'事件
  trace_block_unplug(q, depth, !from_schedule);

  if (from_schedule)
    blk_run_queue_async(q);  //flush来自于schedule，异步运行
  else
    __blk_run_queue(q);      //flush不是来自于schedule，同步运行
  spin_unlock(q->queue_lock);
}
```

这里运行queue就是调用queue的`request_fn()`函数来处理queue中的请求。`request_fn`是一个函数指针，指向driver的请求处理函数，对scsi driver而言，就是`scsi_request_fn`。
