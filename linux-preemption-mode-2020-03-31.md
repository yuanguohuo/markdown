---
title: Linux Preemption模式
date: 2020-03-31 21:14:18
tags: [kernel, preemption]
categories: linux 
---

Linux中用户态程序总是preemptible的，内核使用clock tick中断用户态程序切换到别的线程，不用等待用户态程序主动放弃CPU。但在Kernel Preemption被引入之前，一个线程进入内核态，不放弃CPU也不返回用户态就能一直占用CPU。直到linux 2.6才引入Kernel Preemption。本文的主要目的是介绍Linux的三种Kernel Preemption模式。但介绍Voluntary Preemption的时候，也需要把`might_sleep`搞清楚。

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
- T2调用一个内核函数，这个内核函数是12秒的纯计算（没有IO）；之后返回用户态；

# No Forced Preemption (Server) (2)

这是引入"Kernel Preemption"之前的状态（linux 2.6之前），就是没有内核抢占。它提升throughput但交互性差，所以适合server系统。这种模式下kernel不允许在内核模式下进行线程切换；线程切换只能发生在以下两种情形之一：

- a. 从内核态返回用户态的时候；
- b. 在内核模式下主动放弃CPU（等待资源）；

所以，在前面假定的场景中：T1在内核函中不会被抢占（纯计算12秒），T2在12秒之后被唤醒（它只睡眠3秒，但12秒后才被唤醒，晚了9秒）。

实验验证一下！实验的代码包括两部分：一个内核模块，其中一个函数持续使用CPU12秒（使用`mdelay`来模拟）；一个用户态程序，调用这个内核函数。结构如下：

```
# tree preemption-demo/
preemption-demo/
├── demo-module
│   ├── Makefile
│   └── demo.c
└── userspace
    ├── Makefile
    ├── a.out
    └── read.c
```

preemption-demo/demo-module/demo.c:

```c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/fs.h>
#include <asm/uaccess.h>
#include <linux/delay.h>

#define DEVICE_NAME "demo"

static int major_num;
static int device_open_count = 0;
static char* msg_buffer = "this is a demo";
static char* msg_ptr;

static ssize_t device_read(struct file *flip, char __user *buffer, size_t len, loff_t *offset) {
  int bytes_read = 0;
  mdelay(12000);
  while(len) {
    //If we’re at the end, loop back to the beginning
    if (*msg_ptr == 0) {
      msg_ptr = msg_buffer;
    }
    //buffer is in user data, not kernel, so you can’t just reference
    //with a pointer. The function put_user handles this for us */
    put_user(*(msg_ptr++), buffer++);
    len--;
    bytes_read++;
  }
  return bytes_read;
}

static ssize_t device_write(struct file *flip, const char __user *buffer, size_t len, loff_t *offset) {
  printk(KERN_ALERT "This operation is not supported.\n");
  return -EINVAL;
}

//Called when a process opens our device
static int device_open(struct inode *inode, struct file *file) {
  //if device is open, return busy
  if (device_open_count) {
    return -EBUSY;
  }
  device_open_count++;
  try_module_get(THIS_MODULE);
  return 0;
}

//called when a process closes our device
static int device_release(struct inode *inode, struct file *file) {
  //decrement the open counter and usage count. Without this, the module would not unload
  device_open_count--;
  module_put(THIS_MODULE);
  return 0;
}

static struct file_operations file_ops = {
  .read = device_read,
  .write = device_write,
  .open = device_open,
  .release = device_release
};

static int __init demo_init(void) {
  msg_ptr = msg_buffer;
  //try to register character device
  major_num = register_chrdev(0, "demo", &file_ops);
  if (major_num < 0) {
    printk(KERN_ALERT "Could not register device: %d\n", major_num);
    return major_num;
  } else {
    printk(KERN_INFO "demo module loaded with device major number %d\n", major_num);
    return 0;
  }
}

static void __exit demo_exit(void) {
  //Remember — we have to clean up after ourselves. Unregister the character device. */
  unregister_chrdev(major_num, DEVICE_NAME);
  printk(KERN_INFO "Goodbye, World!\n");
}

//register module functions 
module_init(demo_init);
module_exit(demo_exit);
MODULE_LICENSE("GPL");
```

preemption-demo/demo-module/Makefile:

```
CONFIG_MODULE_SIG=n

obj-m := demo.o

KERN_PATH = /home/workspace/code-reading/linux-3.19.8

all:
	make -C $(KERN_PATH) M=$(PWD) modules


clean:
	make -C $(KERN_PATH) M=$(PWD) clean
```

preemption-demo/userspace/read.c:

```c
#include<stdio.h>
#include<unistd.h>
#include<pthread.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <errno.h>
#include <string.h>

void *hi_prio_t1(void *p)
{
  long startAt, stopAt;

  startAt = time(NULL);
	printf("T1 start at %ld\n",startAt);

	sleep(3);
  stopAt = time(NULL);

	printf("T1 stop at %ld. elapse: %ld seconds\n", stopAt, stopAt-startAt);
	return NULL;
}

void *low_prio_t2(void *p)
{
	char buf[20];
  char errmsg[256];
  int fd, len;
  long startAt, stopAt;

  startAt = time(NULL);
	printf("T2 start at %ld\n",startAt);

	sleep(1);

	if((fd=open("/dev/demo", O_RDONLY))<0) {
    strerror_r(errno, errmsg, 256);
    printf("T2 open /dev/demo failed. error:%s\n", errmsg);
    return NULL;
  }

	if((len=read(fd, buf, 19))<0) {
    strerror_r(errno, errmsg, 256);
    printf("T2 read failed. error:%s\n", errmsg);
    return NULL;
  }
  buf[len]='\0';

  stopAt = time(NULL);
	printf("T2 stop at %ld. elapse: %ld seconds. buf=[%d:\"%s\"]\n", stopAt, stopAt-startAt, len, buf);
	return NULL;
}

int main()
{
	pthread_t t1, t2;
	pthread_attr_t attr;
	struct sched_param param;
	  
	pthread_attr_init(&attr);
	pthread_attr_setschedpolicy(&attr, SCHED_RR);
	
	param.sched_priority = 50;
	pthread_attr_setschedparam(&attr, &param);
	pthread_create(&t1, &attr, hi_prio_t1, NULL);
	
	param.sched_priority = 30;
	pthread_attr_setschedparam(&attr, &param);
	pthread_create(&t2, &attr, low_prio_t2, NULL);

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
	return 0;
}
```

preemption-demo/userspace/Makefile:

```
.PHONY: all clean test

all: clean build test

build:
	gcc read.c -lpthread

clean:
	rm -f a.out

test:
	taskset --cpu-list 1 ./a.out
```

编译安装内核（本例中使用linux 3.19.8）, 选择"No Forced Preemption (Server)"，并且禁止模块签名验证（"Enable loadable module support" --> "Module signature verification"）。然后编译内核模块和用户态程序，Makefile如上，不赘述。


Load内核模块，并且创建设备节点。
```
# insmod demo.ko

# tail -n 4 /var/log/messages
......
Apr  2 23:12:46 devbuild kernel: demo module loaded with device major number 249

# mknod /dev/demo c 249 0
```

运行用户态程序：

```
# uname -a 
Linux devbuild 3.19.8.hyg.preemption.none+ #4 SMP Thu Apr 2 01:17:08 CST 2020 x86_64 x86_64 x86_64 GNU/Linux

# make test
taskset --cpu-list 1 ./a.out
T1 start at 1585840558
T2 start at 1585840558
T1 stop at 1585840571. elapse: 13 seconds
T2 stop at 1585840571. elapse: 13 seconds. buf=[19:"this is a demothis "]
```

有两点需要说明：

- 1. Makefile中的`taskset --cpu-list 1`是为了让进程只使用一个CPU，这很明显，我们就是验证2个线程在一个CPU上的调度；去掉它，若是多核系统，T1在3秒后可以获得其他CPU运行；
- 2. 前面分析T2会在12秒后被唤醒，但这里是13秒。这是因为T1在用户态睡眠了1秒（确保T1先执行T2后执行），然后又占有内核态12秒；

# Preemptible Kernel (Low-Latency Desktop) (3)

这种模式和"No Forced Preemption (Server)"相反，它的目标是提升交互性（即减少内核的latency）而牺牲throughput。它允许在**除临界区之外的所有内核code**中进行线程切换；**临界区也叫做atomic context，主要是**: 

* a. spinlock保护的区间；
* b. irq-handler，即中断上下文；

除了这两种区间的内核code，其他地方都允许线程切换，所以，低优先级的线程，即使在内核中且非自愿的情况下（例如纯CPU计算）也能被抢占，让CPU去服务优先级更高的线程（例如交互性事件）。用代价a. throughput降低；b. 线程切换更多（消耗CPU），换取更高的交互性，**适合于Desktop及嵌入式系统**。

对于前面假定的场景，T2将在睡眠3秒后“准时”地被唤醒。

试验（步骤和第2节一样，除了编译安装内核时选择"Preemptible Kernel (Low-Latency Desktop)"）：

运行用户态程序：

```
# uname -a
Linux devbuild 3.19.8.hyg.preemption.preemptible+ #5 SMP PREEMPT Thu Apr 2 13:51:19 CST 2020 x86_64 x86_64 x86_64 GNU/Linux

# make test 
taskset --cpu-list 1 ./a.out
T1 start at 1585841585
T2 start at 1585841585
T1 stop at 1585841588. elapse: 3 seconds
T2 stop at 1585841598. elapse: 13 seconds. buf=[19:"this is a demothis "]
```

如期望的那样，T1在3秒后被唤醒。

# Voluntary Kernel Preemption (4)

只考虑前述两种preemption模式的时候，内核抢占(Kernel Preemption)的特征是：

- a. 发生在内核态（不是从内核态返回用户态的时候）；
- b. 不是主动放弃CPU（等待资源）；

同时满足这两点的线程切换是内核抢占。后来，添加了第三种模式，即"Voluntary Kernel Preemption"。这种和"No Forced Preemption"很像，内核不会强制切换线程，而是靠内核函数的开发者主动去放弃CPU：在复杂的代码中，时不时的检查rescheduling是否必要，这些检查点（"explicit preemption point"）就是`might_sleep()`函数。可以理解为在"No Forced Preemption"的基础上增加了“不等待资源的主动放弃”：

- a. 从内核态返回用户态的时候； 和"No Forced Preemption"一样；
- b. 在内核模式下主动放弃CPU（等待资源）；和"No Forced Preemption"一样；
- c. 在内核模式下主动放弃CPU（不等待资源，`might_sleep()`主动放弃CPU）；增加；

第3节的"Preemptible Kernel"是真正的内核抢占，它允许在**除临界区之外的所有内核code**中进行线程切换；而"Voluntary Kernel Preemption"本质上还是非抢占内核。实现真正的内核抢占肯定是有代价的：代码的复杂性增加，运行时开销也增加。[引入"Voluntary Kernel Preemption"的patch](https://lwn.net/Articles/137259/)统计了抢占内核vmlinux.preempt的text段size增加了3.5%，而vmlinux.voluntary只增加了0.2%。

由于这种模式比"Preemptible Kernel"轻量级，同时也能提供较好的交互性，所以也适合于Desktop，并且是现在内核的默认选择（kernel 3.19.8）。重复之前的实验：

```
# uname -a
Linux devbuild 3.19.8.hyg.preemption.voluntary+ #7 SMP Sat Apr 4 08:16:39 CST 2020 x86_64 x86_64 x86_64 GNU/Linux

# make test
taskset --cpu-list 1 ./a.out
T1 start at 1585968037
T2 start at 1585968037
T1 stop at 1585968050. elapse: 13 seconds
T2 stop at 1585968050. elapse: 13 seconds. buf=[19:"is a demothis is a "]
```

可见和"No Forced Preemption"行为一致，T1睡眠了13秒（T2在用户态睡眠1秒在内核态占用CPU 12秒）。现在修改一下内核态的代码，增加一个"explicit preemption point"（`might_sleep()`）：

```
static ssize_t device_read(struct file *flip, char __user *buffer, size_t len, loff_t *offset) {
  int bytes_read = 0;
  mdelay(6000);
  might_sleep();
  mdelay(6000);
  while(len) {
    //If we’re at the end, loop back to the beginning
    if (*msg_ptr == 0) {
      msg_ptr = msg_buffer;
    }
    //buffer is in user data, not kernel, so you can’t just reference
    //with a pointer. The function put_user handles this for us */
    put_user(*(msg_ptr++), buffer++);
    len--;
    bytes_read++;
  }
  return bytes_read;
}
```

把占用CPU 12秒改成了两个占用CPU 6秒，中间插入一个`might_sleep()`检查点。在第一个6秒结束的时候，检查发现有更高优先级的线程T1，所以唤醒T1。按这样的分析，T1应该睡眠7秒（T2在用户态睡眠1秒，然后占用CPU 6秒）。验证一下：

```
# make test
taskset --cpu-list 1 ./a.out
T1 start at 1585968353
T2 start at 1585968353
T1 stop at 1585968360. elapse: 7 seconds
T2 stop at 1585968366. elapse: 13 seconds. buf=[19:"this is a demothis "]
```

可见是符合预期的。然而，如前所述，这种模式不是真正的内核抢占，它依赖于内核程序员的“主动性”。换言之，需要内核代码中存在很多很多的"explicit preemption point"。[引入"Voluntary Kernel Preemption"的patch](https://lwn.net/Articles/137259/)不可能在一个patch中去往内核代码中插入很多"explicit preemption point"吧？这就需要介绍一下`might_sleep()`函数。

函数`might_sleep()`就是我们的"explicit preemption point"，但它不是这个patch中引入的：它之前就存在，并且它广泛存在于内核代码中。它是干什么的呢？在内核编程中（跟本文的主题"Kernel Preemption"无关，或者说无论"Kernel Preemption"出现之前还是之后）有一个共识，那就是在"atomic context"里不能进行上下文切换，即当前execution path不能进入sleep状态。"Preemptible Kernel"也是一样，不能在"atomic context"中进行抢占（见第3节）。临界区是：

* a. spinlock保护的区间；
* b. irq-handler，即中断上下文；

如何保证这个共识呢？比如我写个函数里面会sleep，我可能在注释中写上"this function may sleep"，但这也不能保证别人不在临界区里调用它啊。函数`might_sleep()`就是用来解决这个问题的：它会检查当前上下文是不是在临界区里，假如在临界区，就会panic。假如我写一个会sleep的函数，我就在函数开头调用一下`might_sleep()`，别人在临界区调用我的函数，也就panic了。当然这是在debug（`CONFIG_DEBUG_ATOMIC_SLEEP`被定义）模式下的行为；假如`CONFIG_DEBUG_ATOMIC_SLEEP`没被定义，`might_sleep()`的作用就相当于一个annotation。所以，现在的内核代码中，可能sleep的函数，大都有对`might_sleep()`的调用。

```
#ifdef CONFIG_DEBUG_ATOMIC_SLEEP
# define might_sleep() \
	do { __might_sleep(__FILE__, __LINE__, 0); } while (0)
#else
# define might_sleep() do { } while (0)
#endif

void __might_sleep(const char *file, int line, int preempt_offset)
{
	/*
	 * Blocking primitives will set (and therefore destroy) current->state,
	 * since we will exit with TASK_RUNNING make sure we enter with it,
	 * otherwise we will destroy state.
	 */
	WARN_ONCE(current->state != TASK_RUNNING && current->task_state_change,
			"do not call blocking ops when !TASK_RUNNING; "
			"state=%lx set at [<%p>] %pS\n",
			current->state,
			(void *)current->task_state_change,
			(void *)current->task_state_change);

	___might_sleep(file, line, preempt_offset);
}

void ___might_sleep(const char *file, int line, int preempt_offset)
{
	static unsigned long prev_jiffy;	/* ratelimiting */

	rcu_sleep_check(); /* WARN_ON_ONCE() by default, no rate limit reqd. */
	if ((preempt_count_equals(preempt_offset) && !irqs_disabled() &&
	     !is_idle_task(current)) ||
	    system_state != SYSTEM_RUNNING || oops_in_progress)
		return;
	if (time_before(jiffies, prev_jiffy + HZ) && prev_jiffy)
		return;
	prev_jiffy = jiffies;

	printk(KERN_ERR
		"BUG: sleeping function called from invalid context at %s:%d\n",
			file, line);
	printk(KERN_ERR
		"in_atomic(): %d, irqs_disabled(): %d, pid: %d, name: %s\n",
			in_atomic(), irqs_disabled(),
			current->pid, current->comm);

	debug_show_held_locks(current);
	if (irqs_disabled())
		print_irqtrace_events(current);
#ifdef CONFIG_DEBUG_PREEMPT
	if (!preempt_count_equals(preempt_offset)) {
		pr_err("Preemption disabled at:");
		print_ip_sym(current->preempt_disable_ip);
		pr_cont("\n");
	}
#endif
	dump_stack();
}
```

最主要的工作是在第一个if语句那里，尤其是`preempt_count_equals`和`irqs_disabled`，都是用来判断当前的上下文是否是一个"atomic context":

- 只要进程获得了`spin_lock`的任一个变种形式的lock，那么无论是单处理器系统还是多处理器系统，都会导致`preempt_count`发生变化。所以`preempt_count_equals`用来判断是否在spinlock保护的区间里；
- `irqs_disabled`判断当前中断是否开启，即是否在中断上下文；

顺便提一下，在Linux 2.4（Kernel Preemption引入以前）的单核系统上，spinlock是no-op，因为不需要互斥。这很好理解，因为对同一个变量的竞争访问发生在：1. 多核系统，多个线程在不同的核上并行执行；2. 单核系统，但由于抢占，多个线程并发（交替）执行。

回到"Voluntary Kernel Preemption"，由于`might_sleep()`已经广泛存在于内核代码中，且是允许sleep的地方，所以，正好可以在这些地方植入"explicit preemption point"。这个patch就是这么做的：

```
#ifdef CONFIG_DEBUG_ATOMIC_SLEEP
# define might_sleep() \
	do { __might_sleep(__FILE__, __LINE__, 0); might_resched(); } while (0)
#else
# define might_sleep() do { might_resched(); } while (0)
#endif
```

可见，无论`CONFIG_DEBUG_ATOMIC_SLEEP`是否定义，`might_sleep()`都会调用`might_resched()`，这个函数定义是：

```
#ifdef CONFIG_PREEMPT_VOLUNTARY
# define might_resched() _cond_resched()
#else
# define might_resched() do { } while (0)
#endif
```

当抢占模式是"Voluntary Kernel Preemption"的时候（`CONFIG_PREEMPT_VOLUNTARY`被定义），`might_resched`被定义为`_cond_resched()`；否则定义为空。

# Real Time 内核

前面所说的一切，都是基于RR调度（见前文用户态代码`pthread_attr_setschedpolicy(&attr, SCHED_RR)`）或FIFO调度（把`SCHED_RR`改成`SCHED_FIFO`）；`SCHED_OTHER`, `SCHED_IDLE`, `SCHED_BATCH`没有priority。除了这些调度模式之外，linux还有一个RT patch（Real Time），对应内核配置项是`CONFIG_PREEMPT_RT`。如果使用这种模式，所有的代码都可能block别的代码（高优先级一定block低优先级），即使是中断处理程序（ISR）也能够被block。这个patch做了如下修改：

* Converting hardware Interrupts to threads with RT priority 50;
* Converting SoftIRQs to threads with RT 49 priority;
* Converting all spinlocks to mutexes;
* Configuring and using Hi resolution timers;
* some more minor features;

如果创建一个priority大于50的线程，就会block中断处理。这种模式下，系统将做更多的调度所以浪费更多的CPU，但可以满足毫秒级（1ms）的deadline。

# 参考 (6)

https://github.com/torvalds/linux/blob/master/kernel/Kconfig.preempt
https://kernelnewbies.org/FAQ/Preemption
https://lwn.net/Articles/137259/
https://blog.csdn.net/dayancn/article/details/50844985
https://devarea.com/understanding-linux-kernel-preemption/#.XoNhBtMzbVq
