---
title: Linux Preemption模式
date: 2020-03-31 21:14:18
tags: [kernel, preemption]
categories: linux 
---

本文的主要目的是介绍Linux的三种Preemption模式。但介绍Voluntary Preemption的时候，就需要把`might_sleep`搞清楚。

<!-- more -->

<script type="text/x-mathjax-config">
MathJax.Hub.Config({
tex2jax: {inlineMath: [['$','$'], ['\\(','\\)']]}
});
</script>

<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>

# 三种Preemption模式 (1)

编译linux kernel 3.19.8，配置的时候（`make menuconfig`），关于preemption模式（Processor type and features ---> Preemption Model），有3个选项：

* No Forced Preemption (Server)  
* Voluntary Kernel Preemption (Desktop) 
* Preemptible Kernel (Low-Latency Desktop)  

选择"No Forced Preemption (Server)"产生的配置项是：

```
CONFIG_PREEMPT_NONE=y
# CONFIG_PREEMPT_VOLUNTARY is not set
# CONFIG_PREEMPT is not set
```

选择"Voluntary Kernel Preemption (Desktop)"产生的配置项是：

```
# CONFIG_PREEMPT_NONE is not set
CONFIG_PREEMPT_VOLUNTARY=y
# CONFIG_PREEMPT is not set
```

选择"Preemptible Kernel (Low-Latency Desktop)"产生的配置项是：

```
CONFIG_PREEMPT_RCU=y
# CONFIG_PREEMPT_NONE is not set
# CONFIG_PREEMPT_VOLUNTARY is not set
CONFIG_PREEMPT=y
CONFIG_PREEMPT_COUNT=y
CONFIG_DEBUG_PREEMPT=y
# CONFIG_PREEMPT_TRACER is not set
```

我们将在这样一个场景中对比三种模式：

- 两个线程T1和T2，T1优先级高（50），T2优先级低（30）；
- T1睡眠3秒；
- T2调用一个内核函数，这个内核函数是5秒的纯计算（没有IO）；之后返回用户态；

# No Forced Preemption (Server) (2)

这是三者之中最传统的preemption模式。它的主要目标是提升throughput，所以适合server系统。这种模式下kernel不允许在内核模式下进行线程切换；线程切换只能发生在以下两种情形之一：

- a. 在内核模式下主动放弃CPU；
- b. 从内核态返回用户态的时候；

所以，在前面假定的场景中：T1在内核函中不会被抢占（纯计算5秒），T2在5秒之后被唤醒（它只睡眠3秒，但5秒后才被唤醒，晚了2秒）。

# Preemptible Kernel (Low-Latency Desktop) (3)

这种模式和"No Forced Preemption (Server)"相反，它的目标是提升交互性（即减少内核的latency），牺牲throughput。它允许在**除临界区之外的所有内核code**中进行线程切换；**临界区也叫做atomic context，主要是**: 

a. spinlock保护的区间；
b. irq-handler，即中断上下文；

除了这两种区间的内核code，其他地方都允许线程切换，所以，低优先级的线程，即使在内核中且非自愿的情况下（没有IO不等待资源），也能被抢占，让CPU去处理交互性事件。用代价a. throughput降低；b. 线程切换更多（消耗CPU），换取更高的交互性，**适合于Desktop及嵌入式系统**。

对于前面假定的场景，T2将在睡眠3秒后“准时”地被唤醒。
