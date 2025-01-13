---
title: 虚拟化入门笔记--kvmtool virtio-blk设备
date: 2024-09-29 20:15:40
tags: [virtualization,kvm,kvmtool,virtio,blk]
categories: kvm
---

本文介绍kvmtool中的blk模块。其实所涉及的要点在前几篇文章中都已经讲过，所以本文相当于一个阶段性回顾。

<!-- more -->

# 总体流程 (0)

![figure1](virtio-blk-config-and-io-flow.png)
<div style="text-align: center;"><em>图1: 总体流程</em></div>

# PCI CAM/ECAM初始化 (1)

PCI CAM/ECAM是指PCI的Configuration Access Mechanism和Enhanced Configuration Access Mechanism。PCI是virtio的transport，所以，先看PCI的初始化。CAM/ECAM使得guest系统能够读写系统中所有PCI设备的configuration space；实现在pci.c:pci__init()中。具体见[kvmtool pci virtualization](https://www.yuanguohuo.com/2024/07/22/virtualization-4-kvmtool-pci/)第2节。

说明:

- PCI CAM/ECAM初始化是系统级别(guest系统)的，整个系统只做一次。
- CAM/ECAM使得guest系统能够读写系统中所有PCI设备的configuration space，但现在并没有做任何读写，因为还没有enumerate PCI设备呢！这里只是初始化这个机制。

# Virtio设备初始化：创建虚拟device实例 (2)

从`virtio_blk__init_one()`开始: 首先，构造一个`struct blk_dev`实例并加入`bdevs`列表。这就是虚拟device实例。用面向对象的角度看，自然还要模拟它的行为，即设置钩子函数：

```c
int virtio_init(struct kvm *kvm, void *dev, struct virtio_device *vdev,
        struct virtio_ops *ops, enum virtio_trans trans,
        int device_id, int subsys_id, int class)
{
    void *virtio;
    int r;

    switch (trans) {
    // ......

    //以PCI作为virtio的transport;
    case VIRTIO_PCI:
        virtio = calloc(sizeof(struct virtio_pci), 1);
        if (!virtio)
            return -ENOMEM;
        vdev->virtio            = virtio;
        vdev->ops           = ops;
        vdev->ops->signal_vq        = virtio_pci__signal_vq;     //common-queue触发中断的钩子函数
        vdev->ops->signal_config    = virtio_pci__signal_config; //configure-queue触发中断的钩子函数
        vdev->ops->init         = virtio_pci__init;
        vdev->ops->exit         = virtio_pci__exit;
        vdev->ops->reset        = virtio_pci__reset;
        //开始初始化
        r = vdev->ops->init(kvm, dev, vdev, device_id, subsys_id, class);
        break;

    // ......

    //以MMIO作为virtio的transport;
    case VIRTIO_MMIO:

        // ......

        break;
    default:
        r = -1;
    };

    return r;
}
```

# Virtio设备初始化：PCI configuration space (3)

接着开始初始化，即调用` vdev->ops->init`；我们只关注PCI作为transport的情况，对应函数就是`virtio_pci__init()`。顾名思义，这里进行PCI相关的初始化，以及virtio这类特殊PCI设备的初始化。主要就是configuration space，BAR, BAR的callback等。具体地：

- VendorID, DeviceID, BAR, Status(支持Capability-List), 第一个Capability (MSI-X)的位置等configuration space寄存器(内存模拟)。注意：kvmtool中都是使用3个BAR，其region的地址被kvmtool直接分配(这也符合PCI协议吗？物理环境下bar-region的base addr是BIOS或者OS分配的，然后写入pci设备的bar)。
- 为上述3个BAR的region注册callback：其中BAR-0和BAR-1的callback相同，都是`virtio_pci_modern__io_mmio_callback`；只不过BAR-0是port-map到guest的地址空间，而BAR-1是memory-map到guest的地址空间。BAR-2的callback是`virtio_pci__msix_mmio_callback`，专门用于配置MSI-X中断，见[kvmtool interrupt virtualization](https://www.yuanguohuo.com/2024/08/11/virtualization-5-kvmtool-interrupt/).
- MSI-X Capability相关的configuration space寄存器(内存模拟)：
    - msix.ctrl=32：有32个common-queue;
    - msix.table_offset = `0 << 29 | 2`；低3位表示哪个BAR region用于存放MSI-X table；高29位表示在region中的位置。这里是BAR-2，偏移0。
    - msix.pba_offset = `msix_table_size << 29 | 2`；低3位表示哪个BAR region用于存放PBA；高29位表示在region中的位置。这里是BAR-2，紧挨着MSI-X table。
- 添加到全局PCI设备注册表：`device_trees[DEVICE_BUS_PCI]`. 它代表一个PCI bus; kvmtool中只模拟一条PCI bus. 结构中有一个`dev_num`成员，用于递增地分配PCI device number. 所以，这里还进行device number的分配。如[kvmtool pci virtualization](https://www.yuanguohuo.com/2024/07/22/virtualization-4-kvmtool-pci/)中所述，PCI enumeration时，物理环境下依赖IDSEL选择设备进行回复；而kvmtool中，就是匹配这个device number。

接着`virtio_pci__init`调用`virtio_pci_modern_init`，进行virtio其它capability的初始化，它们也是configuration space中的寄存器(当然，这里是通过内存模拟的)：

- virtio common capability: 名字叫common，其实也是一个实实在在的capability；它支持的操作就是`virtio_pci_modern__io_mmio_callback` -> `virtio_pci_access` -> `virtio_pci__common_read/write`；
- virtio.notify capability: 支持的操作是`virtio_pci_modern__io_mmio_callback` -> `virtio_pci_access` -> `virtio_pci__notify_write`；
- virtio.isr capability: 支持的操作是`virtio_pci_modern__io_mmio_callback` -> `virtio_pci_access` -> `virtio_pci__isr_read`；
- virtio.device capability: 叫config更好。支持的操作是`virtio_pci_modern__io_mmio_callback` -> `virtio_pci_access` -> `virtio_pci__config_read/write`；读写设备类型相关的config，例如blk的capacity, cylinder/head/sector，virtio-net的mac, mtu等。

这些capabilities在configuration space中构成一个链表，guest会读取它们，见第6节。

# 设置blk设备的completion callback (4)

设置`disk->disk_req_cb`为`virtio_blk_complete`函数。可想而知，这是disk请求处理完成之后调用的callback，其中有两个重要的事情：

- 把used buffer放进vring;
- 触发中断；

```c
void virtio_blk_complete(void *param, long len)
{
    struct blk_dev_req *req = param;
    struct blk_dev *bdev = req->bdev;
    int queueid = req->vq - bdev->vqs;
    u8 *status;

    /* status */
    status = req->status;
    *status = (len < 0) ? VIRTIO_BLK_S_IOERR : VIRTIO_BLK_S_OK;

    mutex_lock(&bdev->mutex);
    //used-buffer放进vring；
    virt_queue__set_used_elem(req->vq, req->head, len);
    mutex_unlock(&bdev->mutex);

    //触发中断；对于PCI transport，signal_vq指向virtio_pci__signal_vq()函数；
    if (virtio_queue__should_signal(&bdev->vqs[queueid]))
        bdev->vdev.ops->signal_vq(req->kvm, &bdev->vdev, queueid);
}
```

# PCI enumeration (5)

- Guest通过CAM机制，往PCI_CONFIG_ADDRESS port写PCI设备的address=0x80000800(bdf=0:1.0, offset=0)；见`pci_config_address_mmio`函数；
- Guest通过CAM机制，从PCI_CONFIG_DATA port读数据。Kvmtool从全局设备注册表`device_trees[DEVICE_BUS_PCI]`找到blk设备，并返回它的configuration space(内存模拟)中的VendorID/DeviceID，0x10421af4，即VendorID=0x1af4(Red Hat, Inc.)，DeviceID=0x1042(Virtio block device). Guest通过VendorID/DeviceID知道加载哪个驱动。

# PCI配置: 读取Capability-List (6)

- Guest往PCI_CONFIG_ADDRESS port写0x80000804(bdf=0:1.0, offset=0x04=4)；offset=4-8在configuration space中是Command和Status；从上下文可知，guest是要查询Status，看是否支持Capability-List.
- Guest从PCI_CONFIG_DATA port读数据。Kvmtool返回0x0010，其中Status=0x10，即支持Capability-List.
- Guest往PCI_CONFIG_ADDRESS port写0x80000834(bdf=0:1.0, offset=0x34=52)；offset=52是指Cap.Pointer寄存器，即Capability-List的头在configuration space中的位置。也就是说，guest向device查讯Capability-List头在哪里。
- Guest从PCI_CONFIG_DATA port读数据。Kvmtool返回0x40，即64，也就是紧挨着configuration header(64字节)的位置。
- Guest往PCI_CONFIG_ADDRESS port写0x80000840(bdf=0:1.0, offset=0x40=64)，指向第一个Capability；
- Guest从PCI_CONFIG_DATA port读数据。Kvmtool返回0x7011：它表示msix_cap.cap=0x11=PCI_CAP_ID_MSIX; msix_cap.next=0x70=112(下一个Capability在112处);
- Guest往PCI_CONFIG_ADDRESS port写0x80000870(bdf=0:1.0, offset=0x70=112)，指向下一个Capability；
- Guest从PCI_CONFIG_DATA port读数据。Kvmtool返回0x8009：它表示msix_cap.cap=0x09=PCI_CAP_ID_VNDR; msix_cap.next=0x80=128(下一个Capability在128处);
- Guest往PCI_CONFIG_ADDRESS port写0x80000880(bdf=0:1.0, offset=0x80=128)，指向下一个Capability；
- Guest从PCI_CONFIG_DATA port读数据。Kvmtool返回0x9409：它表示msix_cap.cap=0x09=PCI_CAP_ID_VNDR; msix_cap.next=0x94=148(下一个Capability在148处);
- Guest往PCI_CONFIG_ADDRESS port写0x80000894(bdf=0:1.0, offset=0x94=148)，指向下一个Capability；
- Guest从PCI_CONFIG_DATA port读数据。Kvmtool返回0xa409：它表示msix_cap.cap=0x09=PCI_CAP_ID_VNDR; msix_cap.next=0xa4=164(下一个Capability在164处);
- Guest往PCI_CONFIG_ADDRESS port写0x800008a4(bdf=0:1.0, offset=0xa4=164)，指向下一个Capability；
- Guest从PCI_CONFIG_DATA port读数据。Kvmtool返回0xb409：它表示msix_cap.cap=0x09=PCI_CAP_ID_VNDR; msix_cap.next=0xb4=180(下一个Capability在180处);
- Guest往PCI_CONFIG_ADDRESS port写0x800008b4(bdf=0:1.0, offset=0xb4=180)，指向下一个Capability；
- Guest从PCI_CONFIG_DATA port读数据。Kvmtool返回0x0009：它表示msix_cap.cap=0x09=PCI_CAP_ID_VNDR; msix_cap.next=0x00(Capability-List结束)；

# PCI配置: 探测BAR region size (7)

上面看到了CAM机制如何工作的，分成两个步骤：先写入一个addr，然后在进行read/write操作；后文不再赘述，直接描述成读写某个寄存器(offset)。

探测一个BAR region size的过程见[kvmtool pci virtualization](https://www.yuanguohuo.com/2024/07/22/virtualization-4-kvmtool-pci/)第1.6节。如前所述，虚拟环境下region的起始地址是kvmtool分配的，guest需要先把region起始保存起来(save)；然后探测region size；最后再恢复(restore)。Command寄存器也需要同样的操作。所以，整个过程是这样的：

- 读Command寄存器原来的值，save；
- 设置Command寄存器=0x0000;
- 读BAR寄存器原来的值，save；
- 探测BAR region size：设置BAR寄存器=0xffffffff;
- 探测BAR region size：读回BAR寄存器;
- Restore BAR寄存器原来的值；
- Restore Command寄存器原来的值；

以BAR-0为例：

- 读0x80000804指向的寄存器(bdf=0:1.0,offset=0x04,Command寄存器)，kvmtool返回0x0003；
- 写0x80000804指向的寄存器(bdf=0:1.0,offset=0x04,Command寄存器)，值0x0000；
- 读0x80000810指向的寄存器(bdf=0:1.0,offset=0x10,BAR-0寄存器)；kvmtool返回0x00006301，即起始地址是0x6300(最后的0x1表示BAR region为IO port-mapped); 看`pci_get_io_port_block`可知，port-mapped region从PCI_IOPORT_START(0x6200)开始分配；应该是其它设备分配了0x6200，blk接着分配了0x6300；
- 写0x80000810指向的寄存器(bdf=0:1.0,offset=0x10,BAR-0寄存器)，值0xffffffff；
- 读0x80000810指向的寄存器(bdf=0:1.0,offset=0x10,BAR-0寄存器)；kvmtool返回0xffffff01；由此guest知道BAR-0-region是256字节，见[kvmtool pci virtualization](https://www.yuanguohuo.com/2024/07/22/virtualization-4-kvmtool-pci/)第1.6节。在virtio_pci__init中BAR-0分配的region大小是`PCI_IO_SIZE`，其值为0x100(256)，所以是吻合的。
- 写0x80000810指向的寄存器(bdf=0:1.0,offset=0x10,BAR-0寄存器)，值0x00006301；
- 写0x80000804指向的寄存器(bdf=0:1.0,offset=0x04,Command寄存器)，值0x0003；

BAR-1 (Command寄存器的save, set, restore不赘述)：

- 读0x80000814指向的寄存器(bdf=0:1.0,offset=0x14,BAR-1寄存器)；kvmtool返回0xd2000800，即起始地址是0xd2000800(最后的0x0表示BAR region为memory-mapped)；看`pci_get_mmio_block`可知，memory-mapped region是从KVM_PCI_MMIO_AREA(0xd2000000)开始分配；其它设备分配了一些空间，blk分配到0xd2000800；
- 写0x80000814指向的寄存器(bdf=0:1.0,offset=0x14,BAR-1寄存器)，值0xffffffff；
- 读0x80000814指向的寄存器(bdf=0:1.0,offset=0x14,BAR-1寄存器)；kvmtool返回0xffffff00；由此guest知道BAR-1-region也是256字节。在virtio_pci__init中BAR-1分配的region大小也是`PCI_IO_SIZE`，值为0x100(256)，所以是吻合的。
- 写0x80000814指向的寄存器(bdf=0:1.0,offset=0x14,BAR-1寄存器)，值0xd2000800；

BAR-2 (Command寄存器的save, set, restore不赘述)：

- 读0x80000818指向的寄存器(bdf=0:1.0,offset=0x18,BAR-2寄存器)；kvmtool返回0xd2000c00，即起始地址是0xd2000c00(最后的0x0表示BAR region为memory-mapped)；为什么不紧挨着BAR-1-region呢(0xd2000800+0x100=0xd2000900)？因为region的起始地址必须要是size的整数倍。下面可知BAR-2-region的大小是1KiB，而0xd2000900不是1KiB的整数倍。
- 写0x80000818指向的寄存器(bdf=0:1.0,offset=0x18,BAR-2寄存器)，值0xffffffff；
- 读0x80000818指向的寄存器(bdf=0:1.0,offset=0x18,BAR-2寄存器)；kvmtool返回0xfffffc00；由此guest知道BAR-2-region是1024字节。在virtio_pci__init中BAR-2分配的region大小是`VIRTIO_MSIX_BAR_SIZE`，值为1024，所以是吻合的。
- 写0x80000818指向的寄存器(bdf=0:1.0,offset=0x18,BAR-2寄存器)，值0xd2000c00；

# PCI配置: 协商feature set (8)

这是通过virtio common capability完成的，这个capability又是通过BAR-1实现的，所以行为上就是读写BAR-1-region内的地址，不同的偏移对应不同的操作。

- Guest写0xd2000800(长度为4)，对应VIRTIO_PCI_COMMON_DFSELECT操作；
- Guest读0xd2000804(长度为4)，对应VIRTIO_PCI_COMMON_DF操作；
- Guest写0xd2000808(长度为4)，对应VIRTIO_PCI_COMMON_GFSELECT操作；
- Guest读0xd200080c(长度为4)，对应VIRTIO_PCI_COMMON_GF操作；

通过"写————读"操作，完成各自支持的feature set的交换？

# MSI-X table填写 (9)

如[kvmtool interrupt virtualization](https://www.yuanguohuo.com/2024/08/11/virtualization-5-kvmtool-interrupt/)第4.2节所述，MSI-X table应该由系统OS或者BIOS来填写。这是通过BAR-2实现的，具体就是：guest写BAR-2-region内的某个地址(目标地址)，目标地址相对于region起始地址的offset，就是在MSI-X table中的位置————某个表项的某个字段。

- Guest写目标地址0xd2000c00，相对于region起始地址的offset是0，就是msix_table[0].msg.address_lo，写的内容是0xfee00000；
- Guest写目标地址0xd2000c04，相对于region起始地址的offset是4，就是msix_table[0].msg.address_hi，写的内容是0x00000000；
- Guest写目标地址0xd2000c08，相对于region起始地址的offset是8，就是msix_table[0].msg.data，写的内容是0x4041；

这就把`msix_table[0]`填写好了，将来要发这个中断，就往guest的地址0x00000000:0xfee00000处写0x4022。

同样的步骤，`msix_table[1]`被填写：

- msix_table[1].msg.address_lo = 0xfee1f000;
- msix_table[1].msg.address_hi = 0x00000000;
- msix_table[1].msg.data = 0x4021;

Virtio-blk只使用这2个中断。后面将会看到，第一个关联的是configure-queue，第二个关联的是common-queue(virtio-blk只使用一个common-queue)。

# Queue初始化 (10)

上一节配置了2个中断，它们在msix_table中的index分别是0和1，就是vector-number，现在给它关联queue，并填到中断路由表中。这也是通过virtio common capability完成的，这个capability又是通过BAR-1实现的，所以行为上就是读写BAR-1-region内的地址，不同的偏移对应不同的操作。

目的是构造这样一张表：

| gsi | type                     | u.irqchip.irqchip            | u.irqchip.pin |
|-----|--------------------------|------------------------------|---------------|
| 0   | KVM_IRQ_ROUTING_IRQCHIP  | IRQCHIP_MASTER(Master-8259A) | 0             |
| 1   | KVM_IRQ_ROUTING_IRQCHIP  | IRQCHIP_MASTER(Master-8259A) | 1             |
| ... | ...                      | ...                          | ...           |

| gsi | type                     | u.msi.address_hi | u.msi.address_lo | u.msi.data |
|-----|--------------------------|------------------|------------------|------------|
| 24  | KVM_IRQ_ROUTING_MSI      | 0x0              | 0xfee00000       | 0x4022     |
| 25  | KVM_IRQ_ROUTING_MSI      | 0x0              | 0xfee1f000       | 0x4021     |

首先是configure-queue：guest写BAR-1-region内的0xd2000810，对应VIRTIO_PCI_COMMON_MSIX操作，就是配置configure-queue的中断路由。写的数据长度是2字节，内容是vector-number(这里是0)。Kvmtool为configure-queue分配一个gsi号(24)，并在中断路由表中添加一项。然后把新的中断路由表同步给kvm内核模块，`ioctl(vm_fd, KVM_SET_GSI_ROUTING, 中断路由表地址)`。因为中断虚拟化主要是在kvm内核模块中完成的。

然后是common-queue：common-queue通过vring传递数据，vring是guest分配的，所以要把地址告诉kvmtool里的device。

- Guest写BAR-1-region内的0xd2000816，对应VIRTIO_PCI_COMMON_Q_SELECT操作；写的数据是2字节，即选择的common-queue number；kvmtool把它记下来，这里就是0，即选择common-queue-0。
- Guest写BAR-1-region内的0xd2000820，对应VIRTIO_PCI_COMMON_Q_DESCLO操作，即设置vring Descriptor-Area的低地址；
- Guest写BAR-1-region内的0xd2000824，对应VIRTIO_PCI_COMMON_Q_DESCHI操作，即设置vring Descriptor-Area的高地址；
- Guest写BAR-1-region内的0xd2000828，对应VIRTIO_PCI_COMMON_Q_AVAILLO操作，即设置vring Avail-Area的低地址；
- Guest写BAR-1-region内的0xd200082c，对应VIRTIO_PCI_COMMON_Q_AVAILHI操作，即设置vring Avail-Area的高地址；
- Guest写BAR-1-region内的0xd2000830，对应VIRTIO_PCI_COMMON_Q_USEDLO操作，即设置vring Used-Area的低地址；
- Guest写BAR-1-region内的0xd2000834，对应VIRTIO_PCI_COMMON_Q_USEDHI操作，即设置vring Used-Area的高地址；
- Guest写BAR-1-region内的0xd200081a，对应VIRTIO_PCI_COMMON_Q_MSIX操作，就是配置被选择的common-queue的中断路由。写的数据的长度是2字节，内容是vector-number(这里是1)。Kvmtool为common-queue-0分配一个gsi号(25)，并在中断路由表中添加一项。同样，把新的中断路由表同步给kvm内核模块，`ioctl(vm_fd, KVM_SET_GSI_ROUTING, 中断路由表地址)`。
- Guest写0xd200081c，对应VIRTIO_PCI_COMMON_Q_ENABLE操作，见下一节。

# Enable queue (11)

Enable queue的逻辑在`virtio_pci_init_vq`函数中。

首先就是启用异步通知。前面第3节说过`virtio_pci__init`调用`virtio_pci_modern_init`初始化了virtio.notify capability，它是一个同步通知机制：初始化之后，guest写BAR-1 region内的特定地址，就会触发kvmtool的`virtio_pci_modern__io_mmio_callback` -> `virtio_pci_access` -> `virtio_pci__notify_write`去同步处理。和同步通知相比，异步通知更高效：guest把通知写到ioeventfd中，就继续处理其它任务；kvmtool从ioeventfd poll通知，见[kvmtool virtio设备](https://www.yuanguohuo.com/2024/08/31/virtualization-6-kvmtool-virtio/)第4.1.2节。Poll到通知之后，如何处理呢？这其实和virtio无关了，是实现的事；kvmtool中，调用`virtio_pci__ioevent_callback`来处理，这个函数调用`ioeventfd->vdev->ops->notify_vq`钩子函数来处理。对于blk来说，这个钩子函数是virtio/blk.c:notify_vq()，即直接把通知写到另一个eventfd(`bdev->io_efd`)中。这完全是实现上的选择……

这里`bdev->io_efd`也要事先创建好，并且也应该创建一个线程来poll它。这就是`virtio_pci_init_vq` -> `vdev->ops->init_vq`做的事。对于blk来说，`init_vq`(virtio/blk.c中)就是创建`bdev->io_efd`，并创建一个线程来poll它。线程的body是`virtio_blk_thread`函数。

可想而知，`virtio_blk_thread`函数会poll `bdev->io_efd`中的通知:

```c
static void *virtio_blk_thread(void *dev)
{
    struct blk_dev *bdev = dev;
    u64 data;
    int r;

    kvm__set_thread_name("virtio-blk-io");

    while (1) {
        r = read(bdev->io_efd, &data, sizeof(u64));
        if (r < 0)
            continue;
        virtio_blk_do_io(bdev->kvm, &bdev->vqs[0], bdev);
    }

    pthread_exit(NULL);
    return NULL;
}
```

显然`virtio_blk_do_io`就是blk的IO逻辑。

# Blk的IO (12)

```c
static void virtio_blk_do_io(struct kvm *kvm, struct virt_queue *vq, struct blk_dev *bdev)
{
    struct blk_dev_req *req;
    u16 head;

    //检查avaiable-buffer queue是否不空
    while (virt_queue__available(vq)) {
        //获取avaiable-buffer queue的head；它是Descriptor Area表的index；
        head        = virt_queue__pop(vq);
        req     = &bdev->reqs[head];
        //从Descriptor Area的表项中得到buffer的地址和长度，放到req->iov中，并不拷贝buffer本身；
        req->head   = virt_queue__get_head_iov(vq, req->iov, &req->out,
                    &req->in, head, kvm);
        req->vq     = vq;

        virtio_blk_do_io_request(kvm, vq, req);
    }
}

static void virtio_blk_do_io_request(struct kvm *kvm, struct virt_queue *vq, struct blk_dev_req *req)
{
    struct virtio_blk_outhdr req_hdr;
    size_t iovcount, last_iov;
    struct blk_dev *bdev;
    struct iovec *iov;
    ssize_t len;
    u32 type;
    u64 sector;

    bdev        = req->bdev;
    iov     = req->iov;

    //  req既可能是read请求又可能是write请求；
    //  无论是read还是write，req->iov中都有一个struct virtio_blk_outhdr结构体；
    //  正是这个结构体告诉我们，本请求是read还是write；
    //      - 若是read： req->iov前面req->out个buffer(iovec结构)是virtio_blk_outhdr结构体；后面req->in个buffer(iovec结构)是空的，用于输入；
    //        注意空的buffer(iovec结构)也是有效的，它的长度(iovec::iov_len)也是大于0的；
    //      - 若是write：req->iov全部buffer(iovec结构)都是输出(个数是req->out; req->in应该等于0)；输出中，最开始是virtio_blk_outhdr结构体，后面是真实负载；

    //先读出struct virtio_blk_outhdr结构体；然后就知道是read还是write以及读写的起始sector；
    iovcount = req->out;
    len = memcpy_fromiovec_safe(&req_hdr, &iov, sizeof(req_hdr), &iovcount);
    if (len) {
        pr_warning("Failed to get header");
        return;
    }

    type = virtio_guest_to_host_u32(vq->endian, req_hdr.type);
    sector = virtio_guest_to_host_u64(vq->endian, req_hdr.sector);

    iovcount += req->in;
    if (!iov_size(iov, iovcount)) {
        pr_warning("Invalid IOV");
        return;
    }

    /* Extract status byte from iovec */
    last_iov = iovcount - 1;
    while (!iov[last_iov].iov_len)
        last_iov--;
    iov[last_iov].iov_len--;
    req->status = iov[last_iov].iov_base + iov[last_iov].iov_len;
    if (!iov[last_iov].iov_len)
        iovcount--;

    switch (type) {
    case VIRTIO_BLK_T_IN:
        disk_image__read(bdev->disk, sector, iov, iovcount, req);
        break;
    case VIRTIO_BLK_T_OUT:
        disk_image__write(bdev->disk, sector, iov, iovcount, req);
        break;
    case VIRTIO_BLK_T_FLUSH:
        len = disk_image__flush(bdev->disk);
        virtio_blk_complete(req, len);
        break;
    case VIRTIO_BLK_T_GET_ID:
        len = disk_image__get_serial(bdev->disk, iov, iovcount,
                         VIRTIO_BLK_ID_BYTES);
        virtio_blk_complete(req, len);
        break;
    default:
        pr_warning("request type %d", type);
        break;
    }
}
```

显然`disk_image__read`和`disk_image__write`处理读写，重点看`virtio_blk_complete`函数。

```c
void virtio_blk_complete(void *param, long len)
{
    struct blk_dev_req *req = param;
    struct blk_dev *bdev = req->bdev;
    int queueid = req->vq - bdev->vqs;
    u8 *status;

    /* status */
    status = req->status;
    *status = (len < 0) ? VIRTIO_BLK_S_IOERR : VIRTIO_BLK_S_OK;

    mutex_lock(&bdev->mutex);
    //used-buffer放进vring；
    virt_queue__set_used_elem(req->vq, req->head, len);
    mutex_unlock(&bdev->mutex);

    //触发中断；对于PCI transport，signal_vq指向virtio_pci__signal_vq()函数；
    if (virtio_queue__should_signal(&bdev->vqs[queueid]))
        bdev->vdev.ops->signal_vq(req->kvm, &bdev->vdev, queueid);
}
```

前面第4节也提到这个函数，那是用于同步通知的情况下。无论同步还是异步通知，complete逻辑是一样的。



