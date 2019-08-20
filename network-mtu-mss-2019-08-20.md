---
title: MTU和TCP MSS 
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

平时接触的主要是以太网。由于以太网传输电气方面的限制，每个以太网帧最小64字节，最大不能超过1518字节。以太网的frame头是14字节(source MAC addr和destination MAC addr各6字节共12字节，加上2字节的type，共14字节)，frame尾是4字节(CRC)，所以还有剩1518-14-4=1500字节。这就是以太网的MTU，也就是Ethernet的payload(通常是IP报文)的最大长度。高层协议可能生成大于1500字节的报文，所以IP层(第三层)会进行分片(fragmentation)。

# TCP MSS (2)

分片会带来一些开销。所以，有的时候应该避免。TCP在建连的时候会协商一个MSS (maximum segment size)，这个MSS是TCP payload的最大长度。因为IP头是20字节，TCP头也是20字节，所以通过MSS是1500-20-20=1460字节。

![TCP MSS](tcp-mss.png)


TCP header的options字段只在SYN=1的时候出现。MSS是其中的一个option，它向对方声明：我可以接受的最大TCP segment是x(例如1460)。并且两个方向上的MSS是独立的。

![MSS 选项](mss-option.png)

对于GRE tunnel，假如MSS还为1460，那么最终IP报文的长度将大于1500。如下图所示：

![GRE MSS](gre-mss.png)

由于GRE header的长度是24字节，TCP的MSS应该是1460-24=1436。由于终端设备不是总能知道高层协议增加了报文的长度，就像这里的GRE，所以不会自动调整MSS的值。因此，网络设备提供覆盖MSS的选项，例如Cisco的路由器有一个`ip tcp mss-adjust 1436`命令，它会覆盖所以SYN的TCP报文的MSS选项。这样一来，通过该路由器建立的TCP连接的MSS就是1436了。

# TSO (3)

首先看一个从服务器上抓到的包

![TSO On](tso-on.png)

建连过程中协商的MSS是1460，但是发的包确慢慢增大，最后到5840（大于1460）了。这是为什么呢？这是由TSO (TCP Segmentation Offload)导致的。TSO把分片逻辑下放到(offload)网络接口卡(NIC)，以减少CPU的负载。当然，这需要NIC支持这个功能：把数据分片，然后给每个分片添加上TCP头，IP头以及Ethernet头。为了支持TSO，NIC还需要支持outbound (TX) checksumming 和scatter gather。TSO也叫做Large Segment Offload (LSO)。

`ethtool -k和ethtool -K`可以用来查看和修改TSO。

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

再抓包，发现每个TCP payload都是1460了：

![TSO Off](tso-off.png)
