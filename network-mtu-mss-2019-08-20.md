---
title: MTU,MSS和TSO
date: 2019-08-20 21:31:21
tags: [network,mtu,mss,tso]
categories: network 
---

简单介绍MTU，TCP MSS和TSO。

<!-- more -->

# 链路层MTU (1)

MTU(maximum transmission unit)，即最大传输单元，是指一种通信协议的某一层上面所能通过的最大数据包大小（以字节为单位）。通常所说的MTU是数据链路层的概念。一些常见链路层协议MTU的缺省数值如下：

- FDDI(基于Token Ring协议)：4352字节
- 以太网（Ethernet）协议：1500字节
- PPPoE（ADSL）协议：1492字节
- X.25协议（Dial Up/Modem）：576字节
- Point-to-Point：4470字节

平时接触的主要是以太网。由于以太网传输电气方面的限制，每个以太网帧最小64字节，最大不能超过1518字节。以太网的frame头是14字节(source MAC addr和destination MAC addr各6字节共12字节，加上2字节的type，共14字节)，frame尾是4字节(CRC)，所以还剩1518-14-4=1500字节。这就是以太网的MTU，也就是Ethernet的payload(通常是IP报文)的最大长度。高层协议可能生成大于1500字节的报文，所以IP层(第三层)会进行分片(fragmentation)。

# 网络层Fragmentation (2)

## Fragmentation的实现 (2.1)

以IPv4为例，虽然它的datagram最大长度是65535字节，但在链路层上传输要受到MTU的限制，所以要进行分片(fragmentation)。有分片当然就有合并：source-host把IPv4-datagram分片、发送，中间路由器接收、合并、再分片、发送，destination-host接收并合并成原始IPv4-datagram。为什么中间路由器要合并再重新分片呢？因为不同跳之间的链路层可能不同，比如某路由器上一跳是FDD，下一跳是以太网。要想让整个链路都不分片，见第5节PMTUD。

The IPv4 source, destination, identifier, total length, and fragment offset fields, along with "more fragments" (MF) and "do not fragment" (DF) flags in the IPv4 header, are used for IPv4 fragmentation and reassembly.

- identifier: 16bit, source-host分配的，帮助合并分片；
- fragment offset: 13bit, 当前分片在原IPv4-datagram中的偏移；单位是8B；
- do not fragment(DF): IPv4-datagram是否允许被分片：0 = "can fragment", 1 = "do not fragment"；
- more fragments(MF): 是否是最后一个分片：0 = "last fragment," 1 = "more fragments"；

{% asset_img ipv4-fragmentation.png %}

- 第一个分片(0-0)：fragment offset是0，长度是1500 (包含20字节的IP-header，这个Ip-heaer和原Ipv4-datagram的header很像，作了微小修改)；
- 第二个分片(0-1)：fragment offset是1480(185乘以8)，意思是：本分片的数据部分在原Ipv4-datagram数据中的偏移是1480(显然，第一个分片的长度是1500，减去20字节的IP-heaer就是1480)。第二个分片的长度也是1500(包含新建的20字节的分片IP-header)；
- 第三个分片(0-2)：fragment offset是2960(370乘以8)，意思是：本分片的数据部分在原Ipv4-datagram数据中的偏移是2960(前2个分片每个分片的数据是1480，所以第三个分片数据从2960开始)。第三个分片的长度也是1500(包含新建的20字节的分片IP-header)；
- 第四个分片(0-3)：fragment offset是4440(555乘以8)，意思是：本分片的数据部分在原Ipv4-datagram数据中的偏移是4440(前3个分片每个分片的数据是1480，所以第三个分片数据从4440开始)。第四个分片的长度是700(包含新建的20字节的分片IP-header)，所以数据长度是680。680加上偏移4440等于5120，也就是原IPv4-datagram的size(注意图中原Ipv4-datagram的Total Length 5140是包含20字节的IP-header的)；

注意：把所有分片的长度加起来大于原IPv4-datagram的长度，这是因为每个分片都有一个IP-header，所以IP-header多了。以上图为例，原IPv4-datagram被分成4个fragments，多出3个IP-header，故总长度多了60字节(1500+1500+1500+700-5140=60)。

只有收到最后一个分片才能知道原IPv4-datagram的size。最后一个分片的MF(more fragments)为0。

## Fragmentation的问题 (2.2)

引用[这篇文章](https://www.cisco.com/c/en/us/support/docs/ip/generic-routing-encapsulation-gre/25885-pmtud-ipfrag.html)，fragmentation有很多问题：

- IPv4 fragmentation results in a small increase in CPU and memory overhead to fragment an IPv4 datagram. This is true for the sender and for a router in the path between a sender and a receiver (如第2.1节所说，中间路由器会合并然后再分片，要想让整个链路都不分片，见第5节PMTUD).
- Fragmentation causes more overhead for the receiver when reassembling the fragments because the receiver must allocate memory for the arriving fragments and coalesce them back into one datagram after all of the fragments are received. Reassembly on a host is not considered a problem because the host has the time and memory resources to devote to this task. Reassembly, however, is inefficient on a router whose primary job is to forward packets as quickly as possible. A router is not designed to hold on to packets for any length of time. A router that does the reassembly chooses the largest buffer available (18K), because it has no way to determine the size of the original IPv4 packet until the last fragment is received.
- Another fragmentation issue involves how dropped fragments are handled. If one fragment of an IPv4 datagram is dropped, then the entire original IPv4 datagram must be present and it is also fragmented. This is seen with Network File System (NFS). NFS has a read and write block size of 8192. Therefore, a NFS IPv4/UDP datagram is approximately 8500 bytes (which includes NFS, UDP, and IPv4 headers). A sending station connected to an Ethernet (MTU 1500) has to fragment the 8500-byte datagram into six (6) pieces; Five (5) 1500 byte fragments and one (1) 1100 byte fragment. If any of the six fragments are dropped because of a congested link, the complete original datagram has to be retransmitted. This results in six more fragments to be created. If this link drops one in six packets, then the odds are low that any NFS data are transferred over this link, because at least one IPv4 fragment would be dropped from each NFS 8500-byte original IPv4 datagram.
- Firewalls that filter or manipulate packets based on Layer 4 (L4) through Layer 7 (L7) information have trouble processing IPv4 fragments correctly. If the IPv4 fragments are out of order, a firewall blocks the non-initial fragments because they do not carry the information that match the packet filter. This means that the original IPv4 datagram could not be reassembled by the receiving host. If the firewall is configured to allow non-initial fragments with insufficient information to properly match the filter, a non-initial fragment attack through the firewall is possible.
- Network devices such as Content Switch Engines direct packets based on L4 through L7 information, and if a packet spans multiple fragments, then the device has trouble enforcing its policies.

既然有这么问题，就要想办法避免它，所以TCP层引入MSS。(那为什么要设计它呢？)

# TCP层MSS (3)

## MSS的原意 (3.1)

还是引用[这篇文章](https://www.cisco.com/c/en/us/support/docs/ip/generic-routing-encapsulation-gre/25885-pmtud-ipfrag.html)：The Transmission Control Protocol (TCP) Maximum Segment Size (MSS) defines the maximum amount of data that a host accepts in a single TCP/IPv4 datagram. This TCP/IPv4 datagram is possibly fragmented at the IPv4 layer. The sending host is required to limit the size of data in a single TCP segment to a value less than or equal to the MSS reported by the receiving host. Originally, MSS meant how big a buffer (greater than or equal to 65496 bytes) was allocated on a receiving station to be able to store the TCP data contained within a single IPv4 datagram. MSS was the maximum segment of data that the TCP receiver was willing to accept. This TCP segment could be as large as 64K and fragmented at the IPv4 layer in order to be transmitted to the receiving host. The receiving host would reassemble the IPv4 datagram before it handed the complete TCP segment to the TCP layer.

就是说，最初TCP层MSS(Maximum Segment Size)就是定义一个host能够接收的最大TCP包的数据的长度，**不包括TCP头和IP头，说白了就是TCP的负载的长度**，和IP层的fragmentation没有什么关系：即设置了MSS，TCP包**也可能**在网络层进行分片。因为MSS的原意是定义接收方的buffer的大小，即接收方愿意接收的最大TCP包的数据的长度————数据要装在buffer里。所以TCP包可以大到64K，然后在IP层被分片。接收方在IP层合并，然后转给TCP层。如下图所示。

{% asset_img the-original-tcp-mss.png %}

Host A has a buffer of 16K and Host B a buffer of 8K. They send and receive their MSS values and adjust their send-mss for sending data to each other: Host A and Host B have to fragment the IPv4 datagrams that are larger than the interface MTU, yet less than the send-mss because the TCP stack passes 16K or 8K bytes of data down the stack to IPv4. In the case of Host B, packets are fragmented to get onto the Token Ring LAN and again to get onto the Ethernet LAN.

- Host A sends its MSS value of 16K to Host B.
- Host B receives the 16K MSS value from Host A.
- Host B sets its send-mss value to 16K.
- Host B sends its MSS value of 8K to Host A.
- Host A receives the 8K MSS value from Host B.
- Host A sets its send-mss value to 8K.

## MSS被用来避免IP层分片 (3.2)

后来，MSS被用来抑制IP层的分片(但它仍然是限制TCP包的负载的长度)。

To assist in avoiding IPv4 fragmentation at the endpoints of the TCP connection, the selection of the MSS value **was changed to** `min(buffer_size, mtu-40)`. MSS numbers are 40 bytes smaller than MTU numbers because MSS (the TCP data size) does not include the 20-byte IPv4 header and the 20-byte TCP header. MSS is based on default header sizes: the sender stack must subtract the appropriate values for the IPv4 header and the TCP header depends on what TCP or IPv4 options are used.

MSS **currently works** in a manner where each host first compares its outgoing interface MTU with its own buffer and chooses the lowest value as the MSS to send (HostA告诉HostB的自己的MSS是`min(HostA的buffer_size, HostA的outgoing_interface_mtu-40)`). The hosts then compare the MSS size received against their own interface MTU and again choose the lower of the two values (HostB最终选择`min(HostA发来的MSS, HostB的outgoing_interface_mtu-40)`作为send-mss，发给HostA TCP包时，包的数据长度不超过这个值).

{% asset_img the-current-tcp-mss.png %}

- Host A compares its MSS buffer (16K) and its MTU (1500 - 40 = 1460) and uses the lower value as the MSS (1460) to send to Host B.
- Host B receives the send mss (1460) from Host A and compares it to the value of its outbound interface MTU - 40 (4422).
- Host B sets the lower value (1460) as the MSS in order to send IPv4 datagrams to Host A.
- Host B compares its MSS buffer (8K) and its MTU (4462-40 = 4422) and uses 4422 as the MSS to send to Host A.
- Host A receives the send-mss (4422) from Host B and compares it to the value of its outbound interface MTU -40 (1460).
- Host A sets the lower value (1460) as the MSS for sending IPv4 datagrams to Host B.

Fragmentation does not occur at the endpoints of a TCP connection because both outgoing interface MTUs are taken into account by the hosts. Packets still become fragmented in the network between Router A and Router B if they encounter a link with a lower MTU than that of either hosts' outbound interface.

这样以来，**TCP连接的两个端点发送数据时，是保证不会进行IP分片的**，因为选择send-mss时考虑了MTU。**但是中间路由器仍然可能分片**，要想让整个链路都不分片，见第5节PMTUD。

In the example, 1460 is the value chosen by both hosts as the send-mss for each other. Often, **the send-mss value are the same on each end of a TCP connection**.

## MSS的实现 (3.3)

{% asset_img tcp-mss.png %}

The MSS value is sent as a TCP header option only in TCP SYN segments (SYN包和SYN+ACK包). Each side of a TCP connection reports its MSS value to the other side. The MSS value is not negotiated between hosts (不是针对2个hosts而是针对单个TCP连接). The sending host is required to limit the size of data in a single TCP segment to a value less than or equal to the MSS reported by the receiving host.

TCP header的options字段只在SYN=1的时候出现。MSS是其中的一个option，它向对方声明：我可以接受的最大TCP segment是x(例如1460)。并且两个方向上的MSS是独立的。

{% asset_img mss-option.png %}

对于GRE tunnel，假如MSS还为1460，那么最终IP报文的长度将大于1500。如下图所示：

{% asset_img gre-mss.png %}

由于GRE header的长度是24字节，TCP的MSS应该是1460-24=1436。由于终端设备不是总能知道高层协议增加了报文的长度，就像这里的GRE，所以不会自动调整MSS的值。因此，网络设备提供覆盖MSS的选项，例如Cisco的路由器有一个`ip tcp mss-adjust 1436`命令，它会覆盖所以SYN的TCP报文的MSS选项。这样一来，通过该路由器建立的TCP连接的MSS就是1436了。

## Send-mss的协商案例 (3.4)

第3.2节说过：the send-mss value are the same on each end of a TCP connection. 这也是不绝对的，看下面一个真实的案例：

- client和server在一个子网里；
- client和server的utgoing interface mtu都是8000；
- server端route设置advmss=1400；
- client端route没有设置advmss；

**TCP三次握手之SYN包:** client告诉server，自己的MSS是7960

{% asset_img tcp-syn-package.png %}

**TCP三次握手之SYN+ACK包:** server告诉client，自己的MSS是1400

{% asset_img tcp-syn-ack-package.png %}

通过上图我们知道双方发给对方的mss值，但双方真正设置的`send-mss`是什么呢？可以这样查看：

```bash
ss -aieon |grep -A1 {server-addr}:{server-listening-port}
```

**从client端看:**

{% asset_img send-mss-of-client.png %}

- client的send-mss(图中的mss)是1388；TCP握手的时候，server告诉client自己的mss是1400；这个1388应该是来自1400，又减了12(12来自哪里？待确认)；
- client的recv-mss(图中的rcvmss)是7948；
- client的advmss是7948；client端route没有设置advmss，所以这个7948是mtu(8000)减去40，然后又减去一个12(12来自哪里？待确认)；

**从server端看:**

{% asset_img send-mss-of-server.png %}

- server的send-mss(图中的mss)是7948；TCP握手的时候，server告诉client自己的mss是7960；这个7948应该是来自7960，又减了12(12来自哪里？待确认)；
- server的recv-mss(图中的rcvmss)是536；前面一直专注于send-mss的协商与选择，但这个recv-mss也是个疑问：536是默认MSS，但server在SYN+ACK包中告诉client自己的能够接收的大小是1400(1388)，自己却设置了默认的536；client方的send-mss是1388(以为server能够接收的大小是1388)，它若发来大小为1388的包，server能接收吗？
- server的advmss是1388；server端route设置了advmss=1400，这里又减去一个12(12来自哪里？待确认)；

所以,整体来讲是符合第3.2节描述的`send-mss`选择流程的(除了减了个12字节):

- client发送SYN包，mss = min(buffer_size, client_outgoing_interface_mtu-40-12) = 7948；不知道为啥多减了12，粗略的认为还是`mtu-40`;
- server设置send-mss = min(client在SYN包发来的7948, server_outgoing_interface_mtu-40-12) = min(7948, 7948) = 7948;
- server发送SYN+ACK包，mss=1388(advmss-12)；同样不知道为啥减了12；
- client设置send-mss = min(server在ACK+SYN包发来的1388, client_outgoing_interface_mtu-40-12) = 1388;

所以，client发给server的TCP包的最大数据长度是1388字节(不包括IP-header和TCP-header)；而server发给client的TCP包的最大数据长度是7948字节(同样不包括IP-header和TCP-header)。

双方都知道对方能够接收的最大包的长度了，看看双方是否遵守！

{% asset_img tcp-data-transfer.png %}

刚开始server发送的TCP数据包的最大长度还是遵守7948的约定，**但过了一段时间，居然翻倍了，变成15896！**这个问题留到第4节。

# TSO (4)

前面例子中server端发送的TCP数据负载超出了send-mss。这里再看一个真实案例：

{% asset_img tso-on.png %}

建连过程中协商的MSS是1460，但是发的包却慢慢增大，最后到了5840（大于1460）。这是为什么呢？答案是TSO (TCP Segmentation Offload；也叫做LSO: Large Segment Offload)。TSO是一种利用网卡的少量处理能力，降低CPU发送数据包负载的技术。它把分片逻辑从CPU下放到(offload)网络接口卡(NIC)，以减少CPU的负载。当然，这需要NIC硬件及驱动的支持。

当不支持TSO或TSO被关闭时，TCP层向IP层发送数据会考虑MSS，使得TCP向下发送的数据可以包含在一个IP分组中而不会造成分片。

而当网卡支持TSO且TSO被打开，TCP层会逐渐增大MSS（总是整数倍数增加），TCP层向下发送大块数据时，仅仅计算TCP头，网卡接到到了IP层传下的大数据包后自己重新分成若干个数据包，复制TCP头，添加IP头和链路层的frame头，并且重新计算校验和等相关数据，这样就把一部分CPU相关的处理工作转移到由网卡来处理。

注意MSS在建连的时候是遵从MTU的限制的，但当TSO打开时，MSS会逐渐增大，超出了MTU的限制。当然，IP的payload最大是64KiB，无论MSS增大到多大，TCP报文（包含TCP头的数据）的最大长度也就是64KiB。MSS增长到大于64KiB的时候，已经不再用于限制TCP报文大小了，而可能影响拥塞控制(???)。

可以这样理解：**硬件网卡的分片工作还是属于TCP层；TCP转给IP层的数据包还是满足MSS的限制。只不过，在主机上抓的TCP包，不是最终的TCP包，网卡分片之后的才是！**所以，前面2个例子中看到的，TCP负载超过MSS的现象，是抓包导致的。可以验证：把TSO功能关闭，再抓包就会满足MSS的限制，下面有例子。

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

# PMTUD (5)

设置了MTU和DF(do not fragment)以后，TCP连接的两个端点发送数据时，是保证不会进行IP分片的，**但是中间路由器仍然可能分片**，要想让整个链路都不分片，看PMTUD。

PMTUD was developed in order to avoid fragmentation in the path between the endpoints. It is used to dynamically determine the lowest MTU along the path from a packet source to its destination.

If a router attempts to forward an IPv4 datagram (with the DF bit set) onto a link that has a lower MTU than the size of the packet, the router drops the packet and returns an Internet Control Message Protocol (ICMP) "Destination Unreachable" message to the IPv4 datagram source with the code that indicates "fragmentation needed and DF set" (type 3, code 4).

例子：
```
ICMP: dst (10.10.10.10) frag. needed and DF set
unreachable sent to 10.1.1.1
```

当一个host发了一个full MSS data packet，但收到了一个ICMP不可达错误"fragmentation needed and DF set"，PMTUD就知道要减小它的send-mss；

# 下一步 (6)

前面留下2个问题：

- TCP握手时，发给对方的MSS分别时7960和1400，但双方选择的send-mss却分别是7948和1388，相差12字节。这12字节来自哪里？
- 第3.4节中，server端选择的recv-mss是536，为什么？

这2个问题不影响对MSS的理解(可能不同系统的实现不同)，留作下次调查。
