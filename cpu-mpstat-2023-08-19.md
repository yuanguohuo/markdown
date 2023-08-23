---
title: CPU mpstat命令
date: 2023-08-19 20:03:12
tags: [cpu, mpstat, interrupt]
categories: cpu
---

本文简要介绍一下mpstat命令的使用，并补充一些CPU中断知识。

<!-- more -->

# mpstat命令 (1)

这个工具的主要功能有2个：CPU使用率(`-u`选项)和CPU的中断及软中断数(`-I SUM|CPU|SCPU|ALL`选项)。这两种模式下，都可以使用`-P`选项来选择CPU。

命令的格式是:
```
mpstat [选项] [interval [count]]

```
以interval为间隔，输出count次；假如不带interval和count，就输出1次；带interval不带count就持续输出。

## CPU使用率(`-u`选项，其实默认就是`-u`，可以不加) (1.1)

- 输出1次

```
$ mpstat -u
09:20:15 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
09:20:15 PM  all    2.98    0.01    2.02    6.78    0.00    0.59    0.00    0.00    0.00   87.62
```

- 以1秒为间隔输出3次

```
$ mpstat -u 1 3
09:22:23 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
09:22:24 PM  all    3.20    0.00    2.46    6.64    0.00    0.54    0.00    0.00    0.00   87.16
09:22:25 PM  all    2.52    0.00    1.62    6.16    0.00    0.33    0.00    0.00    0.00   89.37
09:22:26 PM  all    4.51    0.00    1.70    6.15    0.00    0.47    0.00    0.00    0.00   87.16
Average:     all    3.41    0.00    1.93    6.32    0.00    0.45    0.00    0.00    0.00   87.90
```

- 以1秒为间隔持续输出

```
$ mpstat -u 1
09:23:14 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
09:23:15 PM  all    2.82    0.00    2.31    5.22    0.00    0.64    0.00    0.00    0.00   89.01
09:23:16 PM  all    3.91    0.00    4.26    6.16    0.00    1.14    0.00    0.00    0.00   84.53
...
...
```

以上输出的信息和`top+1`基本一致，但它是所有processor的平均值，是整个系统的粗略状况，我们要看每个(或某个)processor就要加上`-P`选项；

- processor 0,1,2,3

```
$mpstat -u -P 0,1,2,3
09:27:02 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
09:27:02 PM    0    2.93    0.02    1.95    7.16    0.00    0.14    0.00    0.00    0.00   87.80
09:27:02 PM    1    3.03    0.01    2.06    6.37    0.00    0.86    0.00    0.00    0.00   87.67
09:27:02 PM    2    3.14    0.01    2.10    6.19    0.00    0.86    0.00    0.00    0.00   87.70
09:27:02 PM    3    3.10    0.01    2.04    6.49    0.00    0.62    0.00    0.00    0.00   87.74
```

- 所有processor

```
$ mpstat -u -P ALL
09:27:52 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
09:27:52 PM  all    2.98    0.01    2.02    6.78    0.00    0.59    0.00    0.00    0.00   87.62
09:27:52 PM    0    2.93    0.02    1.95    7.16    0.00    0.14    0.00    0.00    0.00   87.80
09:27:52 PM    1    3.03    0.01    2.06    6.37    0.00    0.86    0.00    0.00    0.00   87.67
09:27:52 PM    2    3.14    0.01    2.10    6.19    0.00    0.86    0.00    0.00    0.00   87.70
......
09:27:52 PM   63    2.88    0.01    2.03    7.00    0.00    0.53    0.00    0.00    0.00   87.54
```

- 所有online processor: 一般所有processor都是online的，所以等价于所有processor

```
$mpstat -u -P ON
09:28:53 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
09:28:53 PM  all    2.98    0.01    2.02    6.78    0.00    0.59    0.00    0.00    0.00   87.62
09:28:53 PM    0    2.93    0.02    1.95    7.16    0.00    0.14    0.00    0.00    0.00   87.80
09:28:53 PM    1    3.03    0.01    2.06    6.37    0.00    0.86    0.00    0.00    0.00   87.67
09:28:53 PM    2    3.14    0.01    2.10    6.19    0.00    0.86    0.00    0.00    0.00   87.70
......
09:28:53 PM   63    2.88    0.01    2.03    7.00    0.00    0.53    0.00    0.00    0.00   87.54
```

## CPU中断数(`-I`选项) (1.2)

和`-u`选项不同，`-I`选项需要一个参数：`SUM | CPU | SCPU | ALL`;

- 所有CPU的所有中断(以1秒为间隔输出3次)

```
$mpstat -I SUM 1 3
09:34:22 PM  CPU    intr/s
09:34:24 PM  all 114224.75
09:34:25 PM  all 113701.00
09:34:26 PM  all 123838.00
Average:     all 117244.52
```

以上输出的是所有processor的中断数之和：既不区分各个processor，也不区分各个中断号；要显示各个CPU对各个中断的处理数，使用`-I CPU`选项:

- 各processor的中断：列是中断号，行是processor号

```
$ mpstat -I CPU

time   CPU    中断1    中断2 ... 中断M
time    0      x        x         x
time    1      x        x         x
time    2      x        x         x
time    ...
time    63     x        x         x
```

这里输出的列非常多(所以非常难看)，我现在看的服务器上是584列(除去前几列的时间、CPU号等)，每列代表一个中断：`0/s`, `4/s`, `8/s`, `9/s`, ..., `598/s`, `NMI/s`, `LOC/s`, ..., `PIW/s`。

这些中断属于谁呢？可以对照`/proc/interrupts`；这个文件有584行(除去第1行表头)，每行代表一个中断；每列代表一个CPU。所以，`mpstat -I CPU`的输出是`/proc/interrupts`的转置。CPU列之后还有几列，标明中断对应的设备:

```
$ cat /proc/interrupts | tr -s ' ' | cut -d ' ' -f 2,67-

0: IR-IO-APIC 2-edge timer
4: IR-IO-APIC 4-edge ttyS0
8: IR-IO-APIC 8-edge rtc0
9: IR-IO-APIC 9-fasteoi acpi

......

注释: 112-126是hdd盘中断，15个 (和下面有什么区别?)
112: IR-PCI-MSI 23068721-edge mpt3sas0-msix49
113: IR-PCI-MSI 23068722-edge mpt3sas0-msix50
...
126: IR-PCI-MSI 23068735-edge mpt3sas0-msix63

......

注释: 127-382是nvme盘中断，共256个(4个nvme盘，每个nvme盘64队列)
127: IR-PCI-MSI 13631489-edge nvme1q1
128: IR-PCI-MSI 13631490-edge nvme1q2
129: IR-PCI-MSI 13631491-edge nvme1q3
130: IR-PCI-MSI 13631492-edge nvme1q4
131: IR-PCI-MSI 13631493-edge nvme1q5
132: IR-PCI-MSI 13631494-edge nvme1q6
133: IR-PCI-MSI 13631495-edge nvme1q7
134: IR-PCI-MSI 13631497-edge nvme1q9
135: IR-PCI-MSI 13631498-edge nvme1q10
136: IR-PCI-MSI 13631499-edge nvme1q11
137: IR-PCI-MSI 13631500-edge nvme1q12
138: IR-PCI-MSI 13107200-edge nvme0q0
139: IR-PCI-MSI 13631501-edge nvme1q13
140: IR-PCI-MSI 13107201-edge nvme0q1
141: IR-PCI-MSI 13631502-edge nvme1q14
142: IR-PCI-MSI 13107202-edge nvme0q2
143: IR-PCI-MSI 13631503-edge nvme1q15
144: IR-PCI-MSI 13107203-edge nvme0q3
...
380: IR-PCI-MSI 71303230-edge nvme3q62
381: IR-PCI-MSI 71303231-edge nvme3q63
382: IR-PCI-MSI 71303232-edge nvme3q64

......

注释：384-447是网卡eth0的中断，64队列，64个
384: IR-PCI-MSI 12582912-edge mlx5_async@pci:0000:18:00.0
385: IR-PCI-MSI 12582913-edge eth0-0
386: IR-PCI-MSI 12582914-edge eth0-1
387: IR-PCI-MSI 12582915-edge eth0-2
388: IR-PCI-MSI 12582916-edge eth0-3
...
447: IR-PCI-MSI 12582975-edge eth0-62

注释：449-512是网卡eth1的中断，64队列，64个
449: IR-PCI-MSI 12584960-edge mlx5_async@pci:0000:18:00.1
450: IR-PCI-MSI 12584961-edge eth1-0
451: IR-PCI-MSI 12584962-edge eth1-1
452: IR-PCI-MSI 12584963-edge eth1-2
...
512: IR-PCI-MSI 12585023-edge eth1-62

注释: 514-577是hdd盘中断，64个 (和上面有什么区别?)
514: IR-PCI-MSI 70254592-edge mpt3sas1-msix0
515: IR-PCI-MSI 70254593-edge mpt3sas1-msix1
516: IR-PCI-MSI 70254594-edge mpt3sas1-msix2
517: IR-PCI-MSI 70254595-edge mpt3sas1-msix3
518: IR-PCI-MSI 70254596-edge mpt3sas1-msix4
519: IR-PCI-MSI 70254597-edge mpt3sas1-msix5
...
577: IR-PCI-MSI 70254655-edge mpt3sas1-msix63


578: IR-PCI-MSI 360448-edge mei_me

注释：Intel I/OAT中断?
580: IR-PCI-MSI 65536-edge ioat-msix
582: IR-PCI-MSI 67584-edge ioat-msix
583: IR-PCI-MSI 69632-edge ioat-msix
584: IR-PCI-MSI 71680-edge ioat-msix
585: IR-PCI-MSI 73728-edge ioat-msix
586: IR-PCI-MSI 75776-edge ioat-msix
587: IR-PCI-MSI 77824-edge ioat-msix
588: IR-PCI-MSI 79872-edge ioat-msix
590: IR-PCI-MSI 67174400-edge ioat-msix
592: IR-PCI-MSI 67176448-edge ioat-msix
593: IR-PCI-MSI 67178496-edge ioat-msix
594: IR-PCI-MSI 67180544-edge ioat-msix
595: IR-PCI-MSI 67182592-edge ioat-msix
596: IR-PCI-MSI 67184640-edge ioat-msix
597: IR-PCI-MSI 67186688-edge ioat-msix
598: IR-PCI-MSI 67188736-edge ioat-msix

......

注释：看说明
NMI: Non-maskable interrupts
LOC: Local timer interrupts
SPU: Spurious interrupts
PMI: Performance monitoring interrupts
IWI: IRQ work interrupts
RTR: APIC ICR read retries
RES: Rescheduling interrupts
CAL: Function call interrupts
TLB: TLB shootdowns
TRM: Thermal event interrupts
THR: Threshold APIC interrupts
DFR: Deferred Error APIC interrupts
MCE: Machine check exceptions
MCP: Machine check polls
HYP: Hypervisor callback interrupts
HRE: Hyper-V reenlightenment interrupts
HVS: Hyper-V stimer0 interrupts
ERR:
MIS:
PIN: Posted-interrupt notification event
NPI: Nested posted-interrupt event
PIW: Posted-interrupt wakeup event
```

想看nvme0的中断情况，先找出它的中断号：

```
$ cat /proc/interrupts | tr -s ' ' | grep nvme0
 138: 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 379 IR-PCI-MSI 13107200-edge nvme0q0
 140: 2311717 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 IR-PCI-MSI 13107201-edge nvme0q1
 142: 0 2187723 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 IR-PCI-MSI 13107202-edge nvme0q2
 144: 0 0 2072374 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 IR-PCI-MSI 13107203-edge nvme0q3
 146: 0 0 0 2134044 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 IR-PCI-MSI 13107204-edge nvme0q4
 148: 0 0 0 0 2129282 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 IR-PCI-MSI 13107205-edge nvme0q5
 150: 0 0 0 0 0 2152753 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 IR-PCI-MSI 13107206-edge nvme0q6
 ......
 254: 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 2004184 IR-PCI-MSI 13107264-edge nvme0q64
```

然后cut出对应的列(注意，列号并不连续，真是恶心):

```
$ cat /tmp/int | head -n 1  | tr -s ' ' | cut -d ' ' -f 3,112,114,116,省略,228
```

- 只看一个processor的中断(通过上面的方法抽取特定中断)

```
$ mpstat -I CPU -P 3

time   CPU    中断1    中断2 ... 中断M
time    3      x        x         x
```

- 各processor的软中断：列是中断号，行是processor号

```
$ mpstat -I SCPU
09:47:59 PM  CPU       HI/s    TIMER/s   NET_TX/s   NET_RX/s    BLOCK/s IRQ_POLL/s  TASKLET/s    SCHED/s  HRTIMER/s      RCU/s
09:47:59 PM    0       0.00     364.86       1.53       2.14      24.81       0.00       0.58     296.28       0.00     219.59
09:47:59 PM    1       0.00     408.30      18.05      66.36      22.45       0.00      68.60     254.46       0.03     241.84
09:47:59 PM    2       0.00     410.69      23.20     299.20      21.80       0.00     314.33     192.84       0.03     244.09
......
09:47:59 PM   63       0.00     357.20      17.42     278.88      24.23       0.00     300.86     134.34       0.02     231.53
```

- 只看一个processor的软中断

```
$ mpstat -I SCPU -P 3
09:49:02 PM  CPU       HI/s    TIMER/s   NET_TX/s   NET_RX/s    BLOCK/s IRQ_POLL/s  TASKLET/s    SCHED/s  HRTIMER/s      RCU/s
09:49:02 PM    3       0.00     396.12      19.99      74.02      22.63       0.00      99.65     167.30       0.02     240.19
```

# 补充知识interrupt (2)

## disable interrupt vs mask interrupt

这两个词在同一本书里经常被混用，在不同的书里甚至是相反的。这里统一成：

- disable: 通过编程告诉PIC暂时禁止一个特定中断(IRQ号)；disabled的中断并没有丢失，enable的时候，PIC会发给CPU；注意：disable是通PIC完成的，所以对所有processor都生效；
- mask: 告诉processor忽略所有maskable中断。注意：它不是针对一个特定中断IRQ号，而是针对所有maskable中断，即IO设备的中断。Mask不是PIC的行为，而是CPU的。与maskable中断对应的是non-maskable interrupt (NMI)；mask实现方式是clear processor的eflags的`IF(Interrupt Flag)`标志，即`cli`指令；当`IF`标志被clear时(即mask中断时)，PIC还是会发来中断，但CPU会忽略(因为外部设备会保持它们的interrupt line asserted，直到被处理，所以也不会丢失)。取消mask的操作是unmask，即置位`IF`标志，`sti`指令；注意：mask是通过clear一个processor的`IF`标志完成的，所以是针对当前processor;

简而言之: disable是在PIC上针对一个IRQ和所有processor; mask是在一个processor上针对所有maskable中断；

## maskable interrupt vs. non-maskable interrupt (NMI)

如上，前者可以被`cli`指令mask；后者不受影响。所有IO设备的中断都是maskable interrupt；

## interrupt gate vs. trap gate

中断描述符表(IDT)中的两种描述符(好吧，共三种，忽略第三种)：interrupt gate和trap gate。中断描述符是: `irqno(异常0-31,中断IRQ+32)` => `segment and offset of interrupt/exception handler`的映射。这两种描述符的区别在于：对于interrupt gate，CPU(硬件)跳到对应segment时会自动clear `IF`标志，即自动mask中断(即忽略PIC发来的信号)，注意这是硬件行为，而trap gate不会。注意：**Linux uses interrupt gates to handle interrupts and trap gates to handle exceptions**.

当我们讨论中断(非异常)时，总是interrupt gate，即**进入handler时，中断总是被mask的**。那么，是否所有的handler都有这个必要呢？不是的。根据需要：必须mask中断的，注册handler时要带上`SA_INTERRUPT`标志，否则不带。触发时，发现没有带`SA_INTERRUPT`标志，就把中断unmask了(`sti`指令)。 引用Understanding the Linux Kernel 3rd Editin，一个handler要执行一些ISR(多个device可能共用一个IRQ一个handler，每个device一个ISR，所以有多个)，就调用`handle_IRQ_event()`，这个函数：

- 如果当前handler注册时没带`SA_INTERRUPT`，就调用`sti`，即不需要mask中断；
- 逐个执行ISR;
- 调用`cli`；为什么又mask中断呢？因为通过interrupt gate进来的时候就是mask中断的，返回是要unmask，所以保持进来时的样子；

另外，handler执行时，总是要disable当前IRQ。

## bottom-half vs. tasklet

如上所述，interrupt handler执行时，
- 当前IRQ总是disabled(注意，disable是通过PIC完成的，所以是对所有processor生效，虽然只是一个IRQ)；
- 所有maskable中断可能被mask(取决于注册handler时是否带`SA_INTERRUPT`)，在当前processor上；

所以，interrupt handler应该尽快返回，否则其他中断(相同或不同IRQ)可能被耽搁。然而，有时候中断中又有很多事情要做，难以做到尽快返回，因此bottom-half被引入:
- 中断处理被分层两部分：top-half和bottom-half；
- top-half就是interrupt handler(通过`request_irq`注册的)；它处理一些简单的任务，调度一个bottom-half将来执行，然后就响应中断(中断返回)；
- bottom-half处理大部分的工作: 唤醒进程、开始另一个IO操作等；
- 当bottom-half处理前一个中断时，top-half可以处理后一个中断；

两者最大的不同是：如前所述，top-half执行时，总是需要当前IRQ被disable，且可能需要maskable中断被mask(取决于是否带`SA_INTERRUPT`)，而bottom-half不要求。

但其他限制，bottom-haf和top-half是一样的：cannot sleep, cannot access user space, and cannot invoke the scheduler.

Linux没有明确要求driver开发者如何划分这两者，但开发者应该只把时间敏感的、硬件相关的、不能被另一个interrupt中断的事情放在top-half中，其他的都放在bottom-half。典型情况是：top-half copy data from/to hardware, 数据的处理由bottom-half完成。网卡中断就是如此。

需要澄清：bottom-half这个词是有歧义的，一个意思就是上面说的“中断后半段”；另一个意思是实现“中断后半段”的一种机制。Linux Device Drivers, Second Edition中使用bottom-haf表示“中断后半段”；使用简写BH表示实现bottom-half的机制。实现bottom-half的机制有两种，除了BH，还有tasklet；现在tasklet更流行。

Tasklets of the same type are always serialized: in other words, the same type of tasklet cannot be executed by two CPUs at the same time. However, tasklets of different types can be executed concurrently on several CPUs. Serializing the tasklet simplifies the life of device driver developers, because the tasklet function needs not be reentrant.

## 软中断(softirq)

Tasklet是通过softirq实现的(但softirq不只用于softirq)。也就是说，bottom-half是通过tasklet实现的，tasklet又是基于softirq实现的，所以softirq和bottom-half有很多相同特征：高优先级、运行时打开中断。

Softirq处理几乎和硬件中断一样重要的事情。softirq在高优先级下运行（有一个例外，是什么?），但运行时没有mask/disable硬件中断。因此，它们通常会抢占除硬件中断以外的其他所有工作。

从前，有32个软件中断向量(不是32个异常)，每个向量分配给每个设备驱动程序或相关任务。现在驱动程序已经和softirq分离，但仍然使用softirq：通过中间API(像tasklet和timer)进行访问，而不是直接调用。在当前的内核中，定义了十个softirq向量; 两个用于tasklet处理，两个用于网络(the source of the softirq mechanism and its most important application)，两个用于块层，两个用于计时器，一个用于调度程序，一个用于读-复制-更新（RCU）处理。对应`mpstat -I SCPU`的十列:

- HI/s(`HI_SOFTIRQ`): high priority tasklets;
- TIMER/s(`TIMER_SOFTIRQ`):
- NET_TX/s(`NET_TX_SOFTIRQ`): send operations in networks;
- NET_RX/s(`NET_RX_SOFTIRQ`): receive operations in networks;
- BLOCK/s(`BLOCK_SOFTIRQ`): used by the block layer to implement asynchronous request completions (libaio的completion?);
- IRQ_POLL/s
- TASKLET/s(`TASKLET_SOFTIRQ`): regular tasklets;
- SCHED/s(`SCHED_SOFTIRQ`): used by the scheduler to implement periodic load balancing on SMP systems;
- HRTIMER/s(`HRTIMER_SOFTIRQ`): required when high-resolution timers are enabled;
- RCU/s

实现方式：The kernel maintains a per-CPU bitmask indicating which softirqs need processing at any given time. So, for example, when a kernel subsystem calls tasklet_schedule(), the TASKLET_SOFTIRQ bit is set on the corresponding CPU and, when softirqs are processed, the tasklet will be run. There are two places where software interrupts can "fire" and preempt the current thread. One of them is at the end of the processing for a hardware interrupt; it is common for interrupt handlers to raise softirqs, so it makes sense (for latency and optimal cache use) to process them as soon as hardware interrupts can be re-enabled (硬件interrupt handler经常发起tasklet, 它是一种softirq，所以interrupt handler结束就处理softirq). The other possibility is anytime that kernel code re-enables softirq processing (via a call to functions like `local_bh_enable()` or `spin_unlock_bh()`).

每个processor对应一个`ksoftirqd`内核线程，例如在64核系统上：

```
$ ps -ef | grep ksoftirqd
root         9     2  0 Aug03 ?        00:02:31 [ksoftirqd/0]
root        16     2  0 Aug03 ?        00:02:37 [ksoftirqd/1]
root        21     2  0 Aug03 ?        00:02:29 [ksoftirqd/2]
root        26     2  0 Aug03 ?        00:01:42 [ksoftirqd/3]
......
root       326     2  0 Aug03 ?        00:01:29 [ksoftirqd/63]
```

注意：不是说softirq都是通过这些内核线程处理的，而是当正常流程处理不完的时候，才把过剩的交给它们：these processes exist to offload softirq processing when the load gets too heavy. If the regular, inline softirq processing code loops ten times and still finds more softirqs to process (because they continue to be raised), it will wake the appropriate ksoftirqd process (there is one per CPU) and exit; that process will eventually be scheduled and pick up running softirq handlers. Ksoftirqd will also be poked if a softirq is raised outside of (hardware or software) interrupt context; that is necessary because, otherwise, an arbitrary amount of time might pass before softirqs are processed again.

