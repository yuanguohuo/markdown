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
- T2调用一个内核函数，这个内核函数是12秒的纯计算（没有IO）；之后返回用户态；

# No Forced Preemption (Server) (2)

这是三者之中最传统的preemption模式。它的主要目标是提升throughput，所以适合server系统。这种模式下kernel不允许在内核模式下进行线程切换；线程切换只能发生在以下两种情形之一：

- a. 在内核模式下主动放弃CPU；
- b. 从内核态返回用户态的时候；

所以，在前面假定的场景中：T1在内核函中不会被抢占（纯计算12秒），T2在12秒之后被唤醒（它只睡眠3秒，但12秒后才被唤醒，晚了9秒）。

用实验验证一下。实验的代码包括两部分：一个内核模块，其中一个函数持续使用CPU12秒（使用`mdelay`来模拟）；一个用户态程序，调用这个内核函数。结构如下：

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

这种模式和"No Forced Preemption (Server)"相反，它的目标是提升交互性（即减少内核的latency），牺牲throughput。它允许在**除临界区之外的所有内核code**中进行线程切换；**临界区也叫做atomic context，主要是**: 

a. spinlock保护的区间；
b. irq-handler，即中断上下文；

除了这两种区间的内核code，其他地方都允许线程切换，所以，低优先级的线程，即使在内核中且非自愿的情况下（没有IO不等待资源），也能被抢占，让CPU去处理交互性事件。用代价a. throughput降低；b. 线程切换更多（消耗CPU），换取更高的交互性，**适合于Desktop及嵌入式系统**。

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
