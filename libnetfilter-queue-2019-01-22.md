---
title: libnetfilter_queue的使用 
date: 2019-01-22 21:09:18
tags: [libnetfilter_queue,libmnl,netlink]
categories: linux 
---

<!-- more -->

# nfnetlink_queue, libnfnetlink以及libmnl之间的关系 (1)

说明：对这几者没作深入研究，只是简单的浏览一下代码的调用关系。

## linux编程接口 (1.1)

/usr/include/linux/netlink.h中定义了一种地址类型(sockaddr_nl，类比sockaddr_in或者sockaddr_in6)，这种地址类型和domain AF_NETLINK(类比AF_INET或AF_INET6)配合，用来和内核里的一些subsystems进行通信。这个domain里支持很多种协议：NETLINK_ROUTE， NETLINK_USERSOCK，NETLINK_NETFILTER……每种协议对应一个kernel subsystem，例如NETLINK_NETFILTER对应的是nfnetlink_queue；这是linux提供的编程接口。
	
## libnfnetlink (1.2)

一个用户态库，使用socket编程接口(地址类型是sockaddr_nl, domain是AF_NETLINK，协议是NETLINK_NETFILTER)与内核里的nfnetlink_queue subsystem通信。和socket网络编程类似，使用的还是socket, bind, sendto/sendmsg, recvfrom/recvmsg等函数，只是domain和协议不同；例如打开一个socket：
 
```
fd = socket(AF_NETLINK, SOCK_RAW, NETLINK_NETFILTER);
```

## libmnl (1.3)

一个用户态库，是在domain AF_NETLINK里的socket编程接口的一个封装。libmnl和libnfnetlink的不同点在于，前者不针对某种协议，只是对socket编程接口进行封装；后者针对NETLINK_NETFILTER协议。例如打开一个socket：

```
fd = socket(AF_NETLINK, SOCK_RAW | flags, bus);
```

# 依赖 (2)
- nfnetlink_queue：这是一个kernel subsystem，linux kernel 2.6.14及以后的版本默认自带，可以检查编译内核的配置文件(对于CentOS，通常是/usr/src/kernels/3.10.0-327.el7.x86_64/.config或者/boot/config-3.10.0-862.el7.x86_64)，确认以下选项被设置：
* CONFIG_NETFILTER_NETLINK_QUEUE 
* CONFIG_NETFILTER_ADVANCED

- libnfnetlink：如1.2节所述。不过1.2节没有提到的是，libnfnetlink依赖内核的nfnetlink subsystem。检查配置文件(对于CentOS，通常是/usr/src/kernels/3.10.0-327.el7.x86_64/.config或者/boot/config-3.10.0-862.el7.x86_64)，确认以下选项被设置：
* CONFIG_NETFILTER_NETLINK
	
- libmnl：如1.2节所述。

# 安装 (2)

```
wget https://netfilter.org/projects/libnetfilter_queue/files/libnetfilter_queue-1.0.3.tar.bz2
tar xjvf libnetfilter_queue-1.0.3.tar.bz2
cd libnetfilter_queue-1.0.3/
./configure --prefix=/usr/local/libnetfilter_queue-1.0.3
make && make install

export LD_LIBRARY_PATH=/usr/local/libnetfilter_queue-1.0.3/lib:$LD_LIBRARY_PATH
export C_INCLUDE_PATH=/usr/local/libnetfilter_queue-1.0.3/include:$C_INCLUDE_PATH
export PKG_CONFIG_PATH=/usr/local/libnetfilter_queue-1.0.3/lib/pkgconfig
```
