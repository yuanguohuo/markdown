---
title: Tikv的BatchSystem
date: 2020-02-14 08:57:12
tags: [tikv, BatchSystem]
categories: tikv 
---

BatchSystem是Tikv实现multi-raft的基石，本文介绍BatchSystem的实现。BatchSystem本身是一个抽象出来的通用的模块，不牵涉业务逻辑（multi-raft），方便单独介绍。

<!-- more -->

<script type="text/x-mathjax-config">
MathJax.Hub.Config({
tex2jax: {inlineMath: [['$','$'], ['\\(','\\)']]}
});
</script>

<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>

# 总体结构 (1)

{% asset_img batch_system.001.jpeg Batch System %}

# `Fsm`、`BasicMailbox`与`Scheduler` (2)

`Fsm`是一个trait，由业务逻辑来实现。`Fsm`的运转是靠`Message`来驱动的，所以就有了`BasicMailbox`。一个`Fsm`和一个`BasicMailbox`绑定：`BasicMailbox`用于接收驱动`Fsm`的`Message`；`Fsm`是发到`BasicMailbox`的`Message`的owner。

```
pub struct BasicMailbox<Owner: Fsm> {
    sender: mpsc::LooseBoundedSender<Owner::Message>,
    state: Arc<FsmState<Owner>>,
}
```

其中`sender`提供发送`Message`的功能，`state`里面实际上包裹了一个`Fsm`（除了`Fsm`本身还包含`Fsm`的状态）。可以从`BasicMailbox`中把`Fsm`拿出来（`take_fsm`函数），也可以还回去（`release`函数）。

当需要驱动`Fsm`的时候，就通过以下两个函数向关联的`BasicMailbox`发一个`Message`：

```
impl<Owner: Fsm> BasicMailbox<Owner> {
    pub fn force_send<S: FsmScheduler<Fsm = Owner>>(
        &self,
        msg: Owner::Message,
        scheduler: &S,
    ) -> Result<(), SendError<Owner::Message>> {...}

    pub fn try_send<S: FsmScheduler<Fsm = Owner>>(
        &self,
        msg: Owner::Message,
        scheduler: &S,
    ) -> Result<(), TrySendError<Owner::Message>> {...}
}
```

发送到`BasicMailbox`的`Message`立即驱动`Fsm`吗？不是的，`Fsm`经过调度，才能最终被`Message`驱动。这也是`force_send`和`try_send`的第二个参数的用途。这两个函数是这样工作的：把`msg`放入sender；把`Fsm`从`BasicMailbox`里拿出来，扔给`Scheduler`，这时`Fsm`处于`NOTIFYSTATE_NOTIFIED`状态（调度之前，`Fsm`在`BasicMailbox`中的时候处于`NOTIFYSTATE_IDLE`状态。将来，`Fsm`被`Message`之后就会被还回`BasicMailbox`，那时又回到`NOTIFYSTATE_IDLE`状态。

`Scheduler`里面包含一个`channel`的发送端。当`BasicMailbox`接收到一个`Message`的时候，就把对应的`Fsm`扔到这个`channel`里。下文再说`channel`的接收端以及如何消费里面的`Fsm`。

# `Router` (3)

`Router`包含上文介绍过的3个组件：

- `BasicMailbox`: 一个`Control BasicMailbox`和若干个`Normal BasicMailbox`。`Router`提供的接口是：向`Control BasicMailbox`或某个（由参数`addr`指定）`Normal BasicMailbox`发送一个`Message`；除此之外，还有广播，即向把`Message`发给所有`BasicMailbox`；
- `Fsm`: 每个`BasicMailbox`都关联一个`Fsm`，即有一个`Control Fsm`和若干个`Normal Fsm`；
- `Scheduler`: 一个`Control Scheduler`和一个`Normal Scheduler`；当一个`Message`发到一个`BasicMailbox`的时候，就把对应的`Fsm`拿出来，让`Scheduler`调度（即放进`Scheduler`的`channel`里）；


# `Poller`与`PollHandler` (4)

前文说过，每个`Scheduler`有一个`channel`，有`Message`发给一个`Fsm`（发送到其关联的`BasicMailbox`）的时候，`Fsm`就被放进`channel`。`Poller`就是这个`channel`的消费者：

```
struct Poller<N: Fsm, C: Fsm, Handler> {
    router: Router<N, C, NormalScheduler<N, C>, ControlScheduler<N, C>>,
    fsm_receiver: channel::Receiver<FsmTypes<N, C>>,
    handler: Handler,
    max_batch_size: usize,
}
```

其中`fsm_receiver`就是`channel`的接收端。一个`Poller`就是一个线程，线程中运行`Poller::poll()`函数。这个函数是一个大循环，每一轮都是从`fsm_receiver`中接收一批`Fsm`（可能是`Control Fsm`也可能是`Normal Fsm`，这些`Fsm`上有`Message`），由结构体`Batch`表示，然后处理它们。BatcySystem的名字就来源于此：一次接收并处理一批`Fsm`。详细点说：

* 上面有$M$个`Fsm/BasicMailbox`（在Tikv中，一个region对应一个`Normal Fsm/BasicMailbox`，此外还有一个`Control Fsm/BasicMailbox`）；
* 下面有$N$个`Poller`，构成线程池；
* 当一个`Fsm`（无论`Normal Fsm`还是`Control Fsm`）的`BasicMailbox`接收到`Message`的时候，这个`Fsm`就被`Scheduler`发给线程池去处理；
* 每个线程（`Poller::pool()`）一次拿一批`Fsm`来处理，处理完之后再还给`BasicMailbox`（等`BasicMailbox`再接收到`Message`的时候，再处理）；

注意，这里面有两个消息通道：

- 1. `Fsm`被发往`Poller`的消息通道：发送端在`Scheduler`里，接收端在`Poller`里； 
- 2. `Message`被发往`Fsm`的消息通道：发送端在`BasicMailbox`里，接收端在`Fsm`里；

如上所述，`Poller::poll()`的主要工作是从`fsm_receiver`中接收一批`Fsm`并处理它们。但，处理的逻辑是业务层的事，所以这里由一个trait（`PollHandler`）表示处理逻辑。由于`fsm_receiver`接收到既有`Control Fsm`也有`Normal Fsm`，所以`PollHandler`定义了`handle_control`和`handle_normal`接口。可以想象，业务成实现`PollHandler`的时候，应该是从`Fsm`接收`Message`，处理它，驱动`Fsm`。

# `BatchSystem` (5)

`BatchSystem`就是把上述东西组合起来，其`spawn`函数的工作就是创建$N$个`Poller`实例，并启动线程。

# 小结 (6)

本文梳理一下BatchSystem的逻辑，后文就不用再陷入这些细节，聚焦于Tikv的业务逻辑：一个消息发给一个`PeerFsm`或`StoreFsm`的时候，直接去看对应的`PollHandler`实现即可。
