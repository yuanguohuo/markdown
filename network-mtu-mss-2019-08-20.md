---
title: MTU,MSS和TSO
date: 2019-08-20 21:31:21
tags: [network,mtu,mss,tso]
categories: network 
---

简单介绍MTU，TCP MSS和TSO。

<!-- more -->

# MTU (1)

MTU(maximum transmission unit)，即最大传输单元，是指一种通信协议的某一层上面所能通过的最大数据包大小（以字节为单位）。通常所说的MTU是数据链路层的概念。一些常见链路层协议MTU的缺省数值如下：

- FDDI协议：4352字节
- 以太网（Ethernet）协议：1500字节
- PPPoE（ADSL）协议：1492字节
- X.25协议（Dial Up/Modem）：576字节
- Point-to-Point：4470字节

平时接触的主要是以太网。由于以太网传输电气方面的限制，每个以太网帧最小64字节，最大不能超过1518字节。以太网的frame头是14字节(source MAC addr和destination MAC addr各6字节共12字节，加上2字节的type，共14字节)，frame尾是4字节(CRC)，所以还剩1518-14-4=1500字节。这就是以太网的MTU，也就是Ethernet的payload(通常是IP报文)的最大长度。高层协议可能生成大于1500字节的报文，所以IP层(第三层)会进行分片(fragmentation)。

# TCP MSS (2)

分片会带来一些开销。所以，有的时候应该避免。TCP在建连的时候会协商一个MSS (maximum segment size)，这个MSS是TCP payload的最大长度。因为IP头是20字节，TCP头也是20字节，所以通常MSS是1500-20-20=1460字节。

{% asset_img tcp-mss.png %}


TCP header的options字段只在SYN=1的时候出现。MSS是其中的一个option，它向对方声明：我可以接受的最大TCP segment是x(例如1460)。并且两个方向上的MSS是独立的。

{% asset_img mss-option.png %}

对于GRE tunnel，假如MSS还为1460，那么最终IP报文的长度将大于1500。如下图所示：

{% asset_img gre-mss.png %}

由于GRE header的长度是24字节，TCP的MSS应该是1460-24=1436。由于终端设备不是总能知道高层协议增加了报文的长度，就像这里的GRE，所以不会自动调整MSS的值。因此，网络设备提供覆盖MSS的选项，例如Cisco的路由器有一个`ip tcp mss-adjust 1436`命令，它会覆盖所以SYN的TCP报文的MSS选项。这样一来，通过该路由器建立的TCP连接的MSS就是1436了。

# TSO (3)

首先看一个从服务器上抓到的包

{% asset_img tso-on.png %}

建连过程中协商的MSS是1460，但是发的包却慢慢增大，最后到了5840（大于1460）。这是为什么呢？答案是TSO (TCP Segmentation Offload；也叫做LSO: Large Segment Offload)。TSO是一种利用网卡的少量处理能力，降低CPU发送数据包负载的技术。它把分片逻辑从CPU下放到(offload)网络接口卡(NIC)，以减少CPU的负载。当然，这需要NIC硬件及驱动的支持。

当不支持TSO或TSO被关闭时，TCP层向IP层发送数据会考虑MSS，使得TCP向下发送的数据可以包含在一个IP分组中而不会造成分片。而当网卡支持TSO且TSO被打开，TCP层会逐渐增大MSS（总是整数倍数增加），TCP层向下发送大块数据时，仅仅计算TCP头，网卡接到到了IP层传下的大数据包后自己重新分成若干个数据包，复制TCP头，添加IP头和链路层的frame头，并且重新计算校验和等相关数据，这样就把一部分CPU相关的处理工作转移到由网卡来处理。

注意MSS在建连的时候是遵从MTU的限制的，但当TSO打开时，MSS会逐渐增大，超出了MTU的限制。当然，IP的payload最大是64KiB，无论MSS增大到多大，TCP报文（包含TCP头的数据）的最大长度也就是64KiB。MSS增长到大于64KiB的时候，已经不再用于限制TCP报文大小了，而可能影响拥塞控制(???)。

为了支持TSO，NIC还需要支持outbound (TX) checksumming 和scatter gather。可以用命令`ethtool -k和ethtool -K`来查看和修改这些配置。

```
# ethtool -k eth0
Features for eth0:
rx-checksumming: on
tx-checksumming: on
        tx-checksum-ipv4: off [fixed]
        tx-checksum-ip-generic: on
        tx-checksum-ipv6: off [fixed]
        tx-checksum-fcoe-crc: on [fixed]
        tx-checksum-sctp: on
scatter-gather: on
        tx-scatter-gather: on
        tx-scatter-gather-fraglist: off [fixed]
tcp-segmentation-offload: on
        tx-tcp-segmentation: on
        tx-tcp-ecn-segmentation: off [fixed]
        tx-tcp6-segmentation: on
        tx-tcp-mangleid-segmentation: on

# ethtool -K eth0 tx off sg off tso off
Actual changes:
tx-checksumming: off
        tx-checksum-ip-generic: off
        tx-checksum-sctp: off
scatter-gather: off
        tx-scatter-gather: off
tcp-segmentation-offload: off
        tx-tcp-segmentation: off
        tx-tcp6-segmentation: off
        tx-tcp-mangleid-segmentation: off
generic-segmentation-offload: off [requested on]
```

上面关闭了TSO，再抓包，发现每个TCP payload都是1460了：

{% asset_img tso-off.png %}
