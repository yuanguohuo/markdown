---
title: 虚拟化入门笔记--kvmtool pci virtualization 
date: 2024-07-22 18:03:55
tags: [virtualization,kvm,kvmtool,pci]
categories: kvm
---

Kvmtool可以认为是一个极简版的qemu (所处软件层、功能都和qemu一样)，只有几千行代码，便于学习虚拟化的原理。本文先简单回顾物理环境下PCI/PCIe基础，然后看kvmtool中是如何模拟PCI/PCIe的。

<!-- more -->

# PCI/PCIe回顾 (1)

## PCI (1.1)

![figure1](pci.png)
<div style="text-align: center;"><em>图1: PCI 拓扑</em></div>

- A tree structure of interconnected I/O buses is supported through a series of PCI bus bridges.
- Every PCI device has a unique vendor ID and device ID.
- Multiple devices of the same kind are further identified by their unique device numbers on the bus where they reside.
- Each PCI peripheral is identified by a bus number, a device number, and a function number. The PCI specification permits a single system to host up to 256 buses, but because 256 buses are not sufficient for many large systems, Linux now supports PCI domains. Each PCI domain can host up to 256 buses. Each bus hosts up to 32 devices, and each device can be a multifunction board (with a maximum of eight functions), such as an audio device with an accompanying CD-ROM drive (这可能是function一词的由来), a single PCI network card could have two logically separate NICs (这里2个function相同).  Therefore, each function can be identified at hardware level by a 16-bit address, or key. 
    - domain   : 16-bit，一般为0，好多地方都省略，起码我见过的服务器上全是0000
    - bus      : 8-bit
    - device   : 5-bit
    - function : 3-bit
- 对操作系统来说，function就是独立的PCI硬件设备，由domain:bus:device.function唯一定位。可以理解为一张物理卡上包含多个逻辑PCI设备。
- Typical PCI devices:
    - Bridges: Host/North Bridge (class=0x060000), PCI-to-PCI bridge (class=0x060400)
    - HBA: SATA controller (class=0x010601), SAS controller (class=0x010700) 
    - Network Adapter (class=0x020000) 
    - and so on.
- 引用维基百科PCIe：Similar to a host bridge in a PCI system, the root complex generates transaction requests on behalf of the CPU ... 所以在PCI系统中，host bridge (north bridge) 负责生成transaction requests (TLP: Transaction Layer Packet).

## PCIe (1.2)

![figure2](pcie-topology.svg)
<div style="text-align: center;"><em>图2: PCIe 拓扑</em></div>

和PCI相比，PCIe的系统结构发生了很大变化。摘自维基百科：Conceptually, the PCI-Express bus is a high-speed serial replacement of the older PCI/PCI-X bus. One of the key differences between the PCI-Express bus and the older PCI is the bus topology; PCI uses a shared parallel bus architecture, in which the PCI host and all devices share a common set of address, data, and control lines. In contrast, PCI Express is based on point-to-point topology, with separate serial links connecting every device to the root complex (host). 就是说，PCIe是点对点的。每个PCI device和root complex之间是独立的serial link，而不是共享的bus。

PCI device是通过switch连到root complex的，所以switch内部各个port到root complex的连线是独立的，如下图所示：

![figure3](pcie-switch.jpeg)
<div style="text-align: center;"><em>图3: PCIe Switch</em></div>

可见，switch就是若干internel bus和若干PCI-to-PCI bridge的组合。所谓point-to-point topology，是不是使用更多的internal bus和PCI-to-PCI bridge让每个PCI设备都有单独的link？另外，在真实服务器上使用`lspci -vv`命令查看发现bus number并不连续，是不是因为"internal bus"的number看不到？

![figure4](pcie-detailed.png)
<div style="text-align: center;"><em>图4: PCIe 展开</em></div>


从这个图可以看出:

- root complex和switch的内部结构一样：也是internel bus和PCI-to-PCI bridge的组合；
- 只不过**upstream bridge就是host bridge (north bridge)**；
另外，摘自维基百科：

Similar to a host bridge in a PCI system, the root complex generates transaction requests on behalf of the CPU, which is interconnected through a local bus. Root complex functionality may be integrated in the chipset and/or the CPU. A root complex may contain more than one PCI Express port and multiple switch devices can be connected to ports on the root complex or cascaded.

- root complex可能在主板上或者CPU内部;
- root complex负责生成transaction requests (TLP: Transaction Layer Packet); 每个PCI-to-PCI bridge和PCI endpoint设备通过memory-mapping和(或)port-mapping对应一些地址空间。这些地址空间不是main memory的地址空间(可能掩盖了对应main memory空间，也可能和main memory的空间不重合)。当CPU访问这些地址时，root complex生成transaction requests，发给对应设备处理。对配置IO Port(0xCF8和0xCFC)的读写应该也由root complex处理生成配置请求。

总结：

- PCIe是一个point-to-point的网络，类似于以太网；switch也可类比以太网交换机；
- Switch有多个port，其中一个是upstream port，其它是downstream port。每个port对应一个PCI-to-PCI bridge。PCI-to-PCI bridge有自己的type-1 configuration space；必须对所有upstream/downstream bridge进行program（设置它的configuration space），然后switch才能工作，为连接在它上面的下游PCI设备转发TLP memory packets。下游PCI设备可能是endpoint device，也可能是其它switch.
- Switch有内部bus，连接多个port；
- 逻辑上看，和PCI总线结构是等价的；但point-to-point结构性能更好，因为bus是独占的，不需要arbitration.

## Configuration Space (1.3)

每个BDF (bus:device.function)定位的PCI function都有一组寄存器，叫做configuration space. 后文也把PCI function叫做PCI设备，因为它是操作系统OS眼中的PCI设备。要使用PCI设备，BIOS或OS必须首先enumerate PCI设备(包括bridge和endpoint)并对它们进行编程(即设置这些寄存器)。PCI的configuration space是256字节，PCIe是4096字节，它们的header是相同的。但Non-bridge设备(即endpoint设备)和bridge设备的configuration space的header不同，见图5和图6。

![figure5](type-0-config-space.png)
<div style="text-align: center;"><em>图5: Type-0 (Non-Bridge) Configuration Space Header</em></div>

Configuration space header是64字节；PCI设备有192字节的额外空间，存储capabilities列表。

PCI Express introduced an extended configuration space, up to 4096 bytes. The only standardized part of extended configuration space is the first four bytes at 0x100 which are the start of an extended capability list. 也就是说前256字节和PCI一样。从第256字节(0x100)开始的4字节是标准化的，后面的部分(4096-256-4=3836B)都是扩展的。

Configuration space header中有一类重要寄存器叫做BAR: Base Address Register. Type-0 (Non-Bridge) configuration space中有6个BAR。

前面说root complex的时候提到，每个PCI设备(包括bridge和endpoint)都对应一些地址空间，当CPU访问这些地址时root complex就生成transaction requests发给对应设备处理。其中地址空间address region就是由BAR寄存器指定的，一个BAR指定一个address region。所以non-bridge (endpoint)设备至多可以使用6个address region。

BAR还描述address region的大小。在物理机环境下，BIOS或OS探测到region的大小之后(探测方法见下文)，就分配空间，再把空间的base address写入BAR中。注意：在kvmtool中是由kvmtool(VMM: VM Monitor)直接分配的，而不是guest BIOS/OS分配的。

若这些地址空间与main memory的地址空间重合，就掩盖了main memory空间(CPU访问不到这些main memory；相当于失去了这块main memory)；也可能和main memory的地址空间不重合。

另外，每个address region可以是memory-mapped，也可以是port-mapped。对于前者，CPU像访问main memory一样使用`mov`这样的指令去读写；对于后者，CPU使用单独的`in`, `out`指令去访问。

![figure6](type-1-config-space.png)
<div style="text-align: center;"><em>图6: Type-1 (Bridge) Configuration Space Header</em></div>

Type-1 (Bridge) Configuration Space中只有2个BAR寄存器。

## 读写Configuration Space (1.4)

BIOS/OS有两种方式读写Configuration Space:

### Configuration Access Mechanism (CAM)

BIOS/OS通过`PCI_CONFIG_ADDRESS`和`PCI_CONFIG_DATA`这两个port读写所有PCI function的所有configuration space寄存器，包括endpoint设备、PCI-to-PCI bridge。`PCI_CONFIG_ADDRESS`和`PCI_CONFIG_DATA`是PCI specification定义的，所以值是固定的，分别是`0xCF8`和`0xCFC`。具体地，

- 先往`PCI_CONFIG_ADDRESS`写入目标寄存器的地址。因为目标寄存器可能是任何PCI function的任何寄存器，所以地址必须包含Bus#, Device#, Function#以及offset. 其中offset是在configuration space内的偏移，以byte为单位；若舍弃offset的最后2bit，则刚好得到Register#，因为每个Register是4字节。

![figure7](pci-config-address.png)
<div style="text-align: center;"><em>图7: PCI_CONFIG_ADDRESS格式</em></div>

- 然后读写`PCI_CONFIG_DATA`，就是读写目标寄存器。

### Enhanced Configuration Access Mechanism (ECAM)

摘自维基百科：The second method was created for PCI Express. It is called Enhanced Configuration Access Mechanism (ECAM). It extends device's configuration space to 4 KB, with the bottom 256 bytes overlapping the original (legacy) configuration space in PCI. The section of the addressable space is "stolen" so that the accesses from the CPU don't go to memory but rather reach a given device. During system initialization, BIOS determines the base address for this "stolen" address region and communicates it to the root complex and to the operating system. 
就是说，把PCIe设备的4KB configuration space直接memory map到main memory地址空间；main memory的这段空间被掩盖(stolen)：CPU对这段空间的访问被root complex转换成对PCI configuration space的读写。 

## Bus Enumeration (1.5)

Bus enumeration是通过CAM(`PCI_CONFIG_ADDRESS`和`PCI_CONFIG_DATA`)完成的。

引用维基百科：

When the computer is powered on, the PCI bus(es) and device(s) must be enumerated by BIOS or operating system. Bus enumeration is performed by attempting to access the PCI configuration space registers for each buses, devices and functions. Note that device number, different from VID and DID, is merely a device's sequential number on that bus. Moreover, after a new bridge is detected, a new bus number is defined, and device enumeration restarts at device number zero.

If no response is received from the device's function #0, the bus master performs an abort and returns an all-bits-on value (FFFFFFFF in hexadecimal), which is an invalid VID/DID value, thus the BIOS or operating system can tell that the specified combination bus/device_number/function (B/D/F) is not present. In this case, reads to the remaining functions numbers (1–7) are not necessary as they also will not exist.

BIOS/OS遍历各个bus以及bus上的slot；同时顺序分配bus#和device# (即从bus0开始，bus1，bus2，...；对于每个bus，从device0开始，以此...)。对于一个bus上的一个slot，当前分配到busX, deviceY:

- 往`PCI_CONFIG_ADDRESS`写入`0x80000000 | busX << 16 | deviceY << 11 | function#=0 | offset=0`;
- 读`PCI_CONFIG_DATA`;

若slot上没有设备，读到将是0xFFFFFFFF(非法VendorID/DeviceID)，继续下一个slot ... 若slot上有设备(设备必须有function0，PCI规范要求的)，它就会响应，返回自己的VendorID/DeviceID，表示扫描到一个PCI设备，这个PCI设备也就被分配到busX:deviceY；它若有多个function, function#分别是0, 1, 2, ... 它也可能是一个PCI-to-PCI bridge，这样就产生一个新的bus，再对新bus进行enumerate ...

这里有一个问题：slot上的设备如何决定自己要不要响应呢？这时设备还不知道自己将要分配到busX:deviceY。这不是"鸡生蛋-蛋生鸡"的问题吗？答：这是硬件实现的。设备决定是否响应，不是看busX:deviceY是否指向自己，而是靠物理信号Initialization Device Select (IDSEL)。应该是此时硬件保证只有这个slot的IDSEL被点亮。事实上，设备只解析0-10bit，看目标是哪个function的哪个register，根本不会解析bus#和device#，更不会依靠bus#和device#判断是否是给自己的请求。不止Bus enumeration时，以后任何对configuration space register的访问，设备都不是看bus#:device#是否指向自己，都是靠IDSEL信号。

但是在kvmtool中，确实是靠bus#:device#来判断如何响应guest BIOS/OS的，并且只模拟了一条bus，所以只靠device#判断：模拟PCI设备时，直接给它分配了device#；guest BIOS/OS进行bus enumeration时，判断device#是否存在，决定是回复对应设备的VendorID/DeviceID还是0xFFFFFFFF。

## 配置BAR (1.6)

Bus Enumeration之后就知道系统上连接了哪些PCI设备(PCI function)，然后就要对它们进行配置，即使用CAM或者ECAM(见第1.4节)对configuration space的寄存器读写。这里简单说一下对BAR寄存器的配置。前面说过，一个BAR定义一个address region；CPU对这个region的读写被root complex转换成对PCI设备的请求(TLP)。所以配置BAR很重要。

第一个工作就是探测address region的size。规范要求region size必须是2的幂次，并且base address必须对齐到size的整数倍，所以base address的最低`log2(size)`位一定为0。BAR就是存base address的，所以最低N位应该为0(N>=4), region的size就是`2^N`。实际上，最低4位是reserved，前面的`N-4`位为0，这N位都不可写。探测时，先往BAR中写`0xFFFFFFFF`。因为最低N位不可写，所以只有前面`32-N`位被写成`1`。然后读回BAR看后面有多少位不是`1`(reserved最低4位无论是什么值都视为0)，就得到N，也就知道了region的size。

有了region size，BIOS/OS分配对齐的地址空间，写到BAR。

注意：在上述过程中，假如使用CAM方式，一次寄存器读写都是通过两次IO完成的：先写`PCI_CONFIG_ADDRESS`再读写`PCI_CONFIG_DATA`。

在kvmtool中，BAR region空间不是guest BIOS/OS分配的，而是kvmtool分配的(也符合协议吗?)。所以，探测size之前，需要把原来的BAR保存起来；探测完之后再恢复：

- read BAR (保存原值);
- write 0xFFFFFFFF to BAR;
- read BAR;
- write BAR (恢复原值);

# kvmtool中的PCI (2)

