---
title: libnetfilter_queue的使用 
date: 2019-01-22 21:09:18
tags: [libnetfilter_queue,libmnl,netlink]
categories: linux 
---

最近看了下[namazu](https://github.com/osrg/namazu)，试图使用它来模拟filesystem以及ethernet的抖动以及错误，发现ethernet inspector使用的是netfilter queue；这里对它做一个简单的调研。

<!-- more -->

# netlink与netfilter (1)

linux的netlink用于在内核和用户态进程之间的通信。它包含两个方面：1.用户态进程的接口：基于socket的，类似于unix domain的接口；2.内核模块的接口：内部kernel API。它们就像一根管子的两端，内核程序员基于后者开发内核模块，用户态程序员基于前者开发用户态程序。本文不涉及后者。对于前者，/usr/include/linux/netlink.h中定义了一种地址类型(sockaddr_nl，类比sockaddr_un，sockaddr_in或者sockaddr_in6)，这种地址类型和domain AF_NETLINK(类比AF_UNIX，AF_INET或AF_INET6)配合，提供基于socket的接口供用户态进程使用。AF_NETLINK里支持很多种协议：NETLINK_ROUTE， NETLINK_USERSOCK，NETLINK_NETFILTER……协议用于选择与之通信的内核模块或者netlink族。本文只关注NETLINK_NETFILTER协议。

NETLINK_NETFILTER协议对应的是netfilter，netfilter包含3个功能：

* netfilter log
* netfilter queue
* netfilter conntrack。

这3个功能都包含**内核模块**和**用户态库**两部分。

**内核模块:**

* nfnetlink_log
* nfnetlink_queue 
* nfnetlink_conntrack
* nfnetlink

前3个模块对应netfilter的3个功能。最后一个是nfnetlink，它是一个基于netlink的传输层(内核态)，为前3个模块提供一个统一的kernel/userspace接口。

**用户态库:**

* libnetfilter_log：与nfnetlink_log对应；
* libnetfilter_queue：与nfnetlink_queue对应；
* libnetfilter_conntrack：与nfnetlink_conntrack对应；
* libnfnetlink：与内核模块nfnetlink对应，提供一些底层的处理函数，是上面3个库的基础。

前3个库对应netfilter的3个功能。最后一个是libnfnetlink，它是一个基于netlink的传输层(用户态)，为前3个模块提供一个统一的kernel/userspace接口。

总之:
- netlink是一个内核和用户态进程的通信机制(一种socket)，支持多种协议；
- 其中一种协议是NETLINK_NETFILTER，与之对应的是netfilter；
- netfilter包含3个功能，每个功能都包含内核模块和用户态库两部分；

用一张图表示它们之间的关系：

{% asset_img relationship.jpg module %}

# netfilter queue (2)

netfilter queue是netfilter的3个功能之一，它涉及到以下内核模块和用户态库：

* nfnetlink_queue：内核模块，linux 2.6版本引入的，用于代替ip_queue。ip_queue用于将数据包从内核空间传递到用户空间，缺点是只能有一个应用程序接收内核数据。nfnetlink_queue兼容ip_queue(从linux 3.5开始就被移除了)，但支持多个应用程序(最大65535)。需要注意的是，从内核到用户空间的通道还是只有一个，这里说的65536个子队列的实现就象802.1Q实现VLAN一样是在数据包中设置ID号来区分的，不同ID的包都通过相同的接口传输，只是在两端根据ID号进行了分类处理。
* nfnetlink：内核模块，其地位和作用见1节。
* libnfnetlink：用户态库，其地位和作用见1节。
* libnetfilter_queue：用户态库，它是本文的主题，见下文。

# libnetfilter_queue的依赖 (3)

简单的讲，netfilter queue涉及到的内核模块和用户态库有4个，libnetfilter_queue依赖于其他3个。

## nfnetlink_queue (3.1)

检查内核配置文件(对于CentOS，通常是/usr/src/kernels/3.10.0-327.el7.x86_64/.config或者/boot/config-3.10.0-862.el7.x86_64)，确认以下选项被设置：
* CONFIG_NETFILTER_NETLINK_QUEUE 
* CONFIG_NETFILTER_ADVANCED

## nfnetlink (3.2)

检查配置文件(对于CentOS，通常是/usr/src/kernels/3.10.0-327.el7.x86_64/.config或者/boot/config-3.10.0-862.el7.x86_64)，确认以下选项被设置：
* CONFIG_NETFILTER_NETLINK

## libnfnetlink (3.3)

如前所述，它是一个用户态库，提供一些底层的nfnetlink处理函数，是libnetfilter_log， libnetfilter_queue和libnetfilter_conntrack的基础。

## libmnl (3.4)

一个用户态库，面向netlink开发者。因为在netlink开发中，有很多重复性的通用的工作，例如解析，验证，构造netlink头和TLV等等，很容易出错。这个库提供了这些封装，避免每个项目重复造轮子。所以，虽然libnfnetlink和libmnl都是基于netlink socket接口的库，但容易看出它们的不同点在于，libmnl提供netlink开发所需的基本封装，不针对哪一个内核模块和协议，而libnfnetlink针对netfilter和NETLINK_NETFILTER协议。

libnetfilter_queue有两套接口，新的基于libmnl，而旧的基于libnfnetlink；

# 安装libnetfilter_queue(3)

```
wget https://netfilter.org/projects/libnetfilter_queue/files/libnetfilter_queue-1.0.2.tar.bz2
tar xjvf libnetfilter_queue-1.0.2.tar.bz2
cd libnetfilter_queue-1.0.2/
./configure --prefix=/usr/local/libnetfilter_queue-1.0.2
make && make install

export LD_LIBRARY_PATH=/usr/local/libnetfilter_queue-1.0.2/lib:$LD_LIBRARY_PATH
export C_INCLUDE_PATH=/usr/local/libnetfilter_queue-1.0.2/include:$C_INCLUDE_PATH
export PKG_CONFIG_PATH=/usr/local/libnetfilter_queue-1.0.2/lib/pkgconfig
```

# libnetfilter_queue的编程接口 (4)

如前所述，有两套编程接口。前者不依赖libmnl但已经标记为deprecated；后者依赖于libmnl。源代码路径下，examples/nf-queue.c就是后者的一个例子。我们基于它稍作修改，丢弃50%的包；

```
42c42,46
<       nfq_nlmsg_verdict_put(nlh, id, NF_ACCEPT);
---
>
>   if (id%2==0)
>     nfq_nlmsg_verdict_put(nlh, id, NF_ACCEPT);
>   else
>     nfq_nlmsg_verdict_put(nlh, id, NF_DROP);
```

编译：
```
# cat Makefile
.PHONY: all

SRC=nf-queue.c
INCL=-I/usr/local/libnetfilter_queue-1.0.2/include
LIB=-L/usr/local/libnetfilter_queue-1.0.2/lib -lnetfilter_queue -lmnl

build:
        gcc $(INCL) $(LIB) $(SRC) -o nfqueue

run:
        export LD_LIBRARY_PATH=/usr/local/libnetfilter_queue-1.0.2/lib
        ./nfqueue 3

clean:
        rm -f nfqueue

# make
```

如何测试这段程序呢？这需要iptables的配合。iptables rule有一个特殊的target叫做`NFQUEUE`，匹配rule的数据包被放到一个queue中发到用户态，即我们这段程序(注意这段程序需要一个参数，即iptables -j NFQUEUE的queue号)。

```
# iptables -t filter -A OUTPUT -m owner --uid-owner $(id -u ${testuser}) -j NFQUEUE --queue-num 3
# make run 
```

然后打开一个窗口，使用${testuser}登录，随意ping一个IP：

```
# ping 192.168.30.100
PING 192.168.30.100 (192.168.30.100) 56(84) bytes of data.
64 bytes from 192.168.30.100: icmp_seq=2 ttl=64 time=0.128 ms
64 bytes from 192.168.30.100: icmp_seq=4 ttl=64 time=0.080 ms
64 bytes from 192.168.30.100: icmp_seq=6 ttl=64 time=0.156 ms
64 bytes from 192.168.30.100: icmp_seq=8 ttl=64 time=0.134 ms
^C
--- 192.168.30.100 ping statistics ---
9 packets transmitted, 4 received, 55% packet loss, time 8000ms
rtt min/avg/max/mdev = 0.080/0.124/0.156/0.029 ms
#
# ping 192.168.30.100
PING 192.168.30.100 (192.168.30.100) 56(84) bytes of data.
64 bytes from 192.168.30.100: icmp_seq=1 ttl=64 time=0.163 ms
64 bytes from 192.168.30.100: icmp_seq=3 ttl=64 time=0.148 ms
64 bytes from 192.168.30.100: icmp_seq=5 ttl=64 time=0.152 ms
64 bytes from 192.168.30.100: icmp_seq=7 ttl=64 time=0.174 ms
^C
--- 192.168.30.100 ping statistics ---
7 packets transmitted, 4 received, 42% packet loss, time 6002ms
rtt min/avg/max/mdev = 0.148/0.159/0.174/0.013 ms
```

# libnetfilter_queue的性能问题 (5)

就我在CentOS-7(3.10.0-327.36.4.el7.x86_64)的测试而言，libnetfilter_queue有很大的性能问题。[namazu](https://github.com/osrg/namazu)就是一个使用libnetfilter_queue的案例。我在使用namazu的时候，设置延迟100ms丢包率为0，但在网络负载高的时候，延迟会涨到几秒甚至十几秒，丢包率百分之几十。我尝试过如下优化，但都只能在一定程度上解决问题，没有完全达到理想的效果。

## 使用多个子队列代替单个队列 (5.1)

前文iptables rule是这样设置的：

```
iptables -t filter -A OUTPUT -m owner --uid-owner $(id -u ${testuser}) -j NFQUEUE --queue-num 3
```

这样的话，所有匹配rule的包都被放进同一个队列，namazu也无法并行处理。可以这样，使用128个子队列：

```
iptables -t filter -A OUTPUT -m owner --uid-owner $(id -u ${testuser}) -j NFQUEUE --queue-balance 0:127
```

然后修改namazu，在其中启用多个routine，每个routine处理一个子queue。这样做还是有很大的效果的。但还是有两个问题：

* 如前文所述，这里并不是真正的多个队列，而是虚拟的；
* Linux(也可能只是这边版本的linux)并没有使用全部的128个子队列。通过`cat /proc/net/netfilter/nfnetlink_queue`发现，只有3-4个子队列上有数据包；

这些因素(我猜主要是后者)导致性能无法大幅度提升；

## 减小package的拷贝range (5.2)

/proc/net/netfilter/nfnetlink_queue的结构是这样的：

* queue_number
* peer_portid: good chance it is process ID of software listening to the queue
* queue_total: current number of packets waiting in the queue
* copy_mode: 0 and 1 only message only provide meta data. If 2, the message provides a part of packet of size copy range.
* copy_range: length of packet data to put in message
* queue_dropped: number of packets dropped because queue was full
* user_dropped: number of packets dropped because netlink message could not be sent to userspace. If this counter is not zero, try to increase netlink buffer size. On the application side, you will see gap in packet id if netlink message are lost.
* id_sequence: packet id of last packet
* The last field is always ‘1’ and is ignored.

其中copy_mode和copy_range可以设置，

```
nfq_set_mode(queue, copy_mode, copy_range);              //老接口
nfq_nlmsg_cfg_put_params(queue, copy_mode, copy_range);  //新接口
```

尝试把copy_mode设置为0或者1，避免拷贝package，但无法工作(原因?)；
尝试减小copy_range，也没有非常明显的效果；

## 其他尝试 (5.3)

如[libnetfilter_queue的文档](https://netfilter.org/projects/libnetfilter_queue/doxygen/html/)所述:

* 把nice设置为-20
* 增大socket的buffer 
```
#include <libnfnetlink/libnfnetlink.h>
nfnl_rcvbufsiz(nfq_nfnlh(h), 4 * 1024 * 1024);
```
* 增大队列长度
```
nfq_set_queue_maxlen(queue, 1024 * 1024);
```

都没有明显的效果；

# 结论 (6)

libnetfilter_queue的性能需要进一步研究，目前是用tc来替代namazu的ethernet inspector。
