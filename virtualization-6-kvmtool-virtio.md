---
title: 虚拟化入门笔记--kvmtool virtio设备
date: 2024-08-31 21:23:47
tags: [virtualization,kvm,kvmtool,virtio]
categories: kvm
---

本文介绍kvmtool中对virtio的实现。[前文interrupt virtualization](https://www.yuanguohuo.com/2024/08/11/virtualization-5-kvmtool-interrupt/)留下一个问题，如何启用virtio device的queue，本文一并回答。

<!-- more -->

# virtio的组件 (1)

在本文中，我不打算从virtio specification出发去理解virtio，而是直接从kvmtool的代码出发，结合virtio-blk以及virtio-scsi，看virtio是如何工作的。当然，我也会尽可能去概括virtio设备的一般特征。

Virtio包括两个组件：guest中的driver和host中的device。VMM(kvmtool,qemu)将virtio device暴露给guest的transport有多种(如PCI, Memory Mapping I/O, S/390 Channel I/O)，PCI是最常见的选择，所以本文特指PCI transport实现的virtio。对于guest来讲，virtio设备就像真PCI设备一样：Real PCI hardware exposes its configuration space using a specific physical memory address range (i.e., the driver can read or write the device’s registers by accessing that memory range) and/or special processor instructions. In the VM world, the VMM (kvmtool,qemu) captures accesses to that memory range and performs device emulation, exposing the same memory layout that a real machine would have and offering the same responses. The virtio specification also defines the layout of its PCI Configuration space, so implementing it is straightforward. 参考[pci virtualization](https://www.yuanguohuo.com/2024/07/22/virtualization-4-kvmtool-pci/).

When the guest boots and uses the PCI/PCIe auto discovering mechanism, the virtio devices identify themselves wit the PCI vendor ID and their PCI Device ID. The guest’s kernel uses these identifiers to know which driver must handle the device。

也就是，通过PCI vendorID和deviceID找driver(这和物理设备一样)，只不过vendorID和deviceID是虚拟出来的。我观察到的所有virtio device的vendorID都是0x1af4 (Red Hat, Inc.)；deviceID 0x1003是virtio console, 0x1001是SCSI storage controller等等，可以在[pcilookup网站](https://www.pcilookup.com/)上查询。

因为有各种类型的virtio设备(blk, net, ...)，所以把guest中的server记作**virtio-x driver**，把host中的device记作**virtio-x device**。另外，为了衔接vhost和qemu，这里也把virtio-x device分成control plane和data plane两个部分，control plane和data plane。其实，在kvmtool的实现中这个两部分没有明显的模块化。Control plane指virtio设备的初始化和配置等管理功能，让guest感觉到那里有一个设备。Data plane是设备的真正的数据处理功能，例如virtio-blk设备能够读写，virtio-net设备能够接收和发送数据，让guest真的能使用那个设备的功能。Control plane都是在VMM(kvmtool,qemu)中，而为了提升性能，data plane常被offload到host的内核(vhost)或者host的用户态进程(vhost-user)。这也是我们把virtio-x device强行分成两部分的原因。

![figure1](virtio-components.png)
<div style="text-align: center;"><em>图1: virtio的组件</em></div>

# vring (2)

在[interrupt virtualization](https://www.yuanguohuo.com/2024/08/11/virtualization-5-kvmtool-interrupt/)中经常看到queue。现代PCI设备都是多queue的，每个queue有独立的中断；这样就可以让多个CPU来处理中断，实现性能提升。到虚拟化领域，virtio设备也有多个queue，叫virt-queue。每个virt-queue也有独立的中断。

当然，queue/virt-queue的作用是数据传输。就virt-queue而言，数据传输是vring完成。在很多文章中，virt-queue和vring经常混用，搞不清它们的关系。在kvmtool中可以明确的看到：一个virtio设备有多个virt-queue；每个virt-queue有一个vring；

```c
struct scsi_dev {
    struct virt_queue        vqs[NUM_VIRT_QUEUES];
    struct virtio_scsi_config    config;
    struct vhost_scsi_target    target;
    int                vhost_fd;
    struct virtio_device        vdev;
    struct list_head        list;
    struct kvm            *kvm;
};

struct virt_queue {
    struct vring    vring;
    struct vring_addr vring_addr;
    /* The last_avail_idx field is an index to ->ring of struct vring_avail.
       It's where we assume the next request index is at.  */
    u16        last_avail_idx;
    u16        last_used_signalled;
    u16        endian;
    bool        use_event_idx;
    bool        enabled;
    struct virtio_device *vdev;

    /* vhost IRQ handling */
    int        gsi;
    int        irqfd;
    int        index;
};
```

Vring传输数据的方式是共享内存(见下一节)。它在两个方向传输数据：

- virtio-x driver向virtio-x device发送available-buffer;
- virtio-x device向virtio-x driver发送used-buffer;

为了高效地完成这个任务，vring设计上分为3个区：Descriptor-Area, Driver-Area (avail-queue), Device-Area (used-queue). Descriptor-Area描述一些内存buffer，逻辑上没有顺序；Driver-Area是virtio-x driver维护的，它通过引用的方式，把Descriptor-Area内的一些buffer组织成queue，也就是available-buffer queue，传递给virtio-x device。类似地，Device-Area是virtio-x device维护的，通过引用的方式，把Descriptor-Area内的一些buffer组织成queue，也就是used-buffer queue，传递给virtio-x driver。[这篇文章](https://www.redhat.com/en/blog/virtqueues-and-virtio-ring-how-data-travels)细讲了数据如何传输。这里仅通过2张图说明。

注意available-buffer中的**available**这个用词，它以guest作为producer，device作为consumer。对于output操作比如符合直觉：available-buffer中存着待输出的数据，输出到device之后，就变成used/consumed；对于input操作有点反直觉：available-buffer是空的，写入数据之后变成used/consumed。

A buffer can be read-only or write-only from the device point of view, but never both.

![figure2](pass-available-buffer-to-dev.png)
<div style="text-align: center;"><em>图2: guest pass available-buffer to virtio device</em></div>

![figure3](pass-used-buffer-to-guest.png)
<div style="text-align: center;"><em>图3: virtio device pass used-buffer to guest </em></div>

# vring的内存共享 (3)

前面说过，vring传递数据的方式是共享内存(高效)。内存是virtio-x driver在guest的内核里分配的，VMM(qemu,kvmtool)能够访问的到，因为guest的内存在VMM进程的内存空间里。所以：

- 若virtio-x device整个是在VMM中模拟的，则可以直接访问(地址的translation还是必要的，从little-endian转换成host的cpu-endian；little-endian是virtio协议定义的吗？)；
- 若virtio-x device的data plane被offload，则需要地址映射：like POSIX shared memory; a file descriptor to that memory is shared through vhost protocol；

无论如何，VMM(qemu,kvmtool)需要拿到内存的地址(若offload，VMM再把地址共享给data plane，待确认)。Virtio的common capability可以实现这一点。关于PCI的Capability-List，参考[kvmtool pci](https://www.yuanguohuo.com/2024/07/22/virtualization-4-kvmtool-pci/)。

前一篇[interrupt virtualization](https://www.yuanguohuo.com/2024/08/11/virtualization-5-kvmtool-interrupt/)的第4.3.3节提到**guest发送被选择的common-queue的vring的地址给device**，就是现在说的这个事。首先，kvmtool实现了common capability(见virtio/pci-modern.c:virtio_pci_modern_init函数)：

```c
int virtio_pci_modern_init(struct virtio_device *vdev)
{
    int subsys_id;
    struct virtio_pci *vpci = vdev->virtio;
    struct pci_device_header *hdr = &vpci->pci_hdr;

    subsys_id = le16_to_cpu(hdr->subsys_id);

    hdr->device_id = cpu_to_le16(PCI_DEVICE_ID_VIRTIO_BASE + subsys_id);
    hdr->subsys_id = cpu_to_le16(PCI_SUBSYS_ID_VIRTIO_BASE + subsys_id);

    vpci->doorbell_offset = VPCI_CFG_NOTIFY_START;
    vdev->endian = VIRTIO_ENDIAN_LE;

    hdr->msix.next = PCI_CAP_OFF(hdr, virtio);

    hdr->virtio.common = (struct virtio_pci_cap) {
        .cap_vndr        = PCI_CAP_ID_VNDR,
        .cap_next        = PCI_CAP_OFF(hdr, virtio.notify),
        .cap_len        = sizeof(hdr->virtio.common),
        .cfg_type        = VIRTIO_PCI_CAP_COMMON_CFG,
        .bar            = 1,
        .offset            = cpu_to_le32(VPCI_CFG_COMMON_START),
        .length            = cpu_to_le32(VPCI_CFG_COMMON_SIZE),
    };
    BUILD_BUG_ON(VPCI_CFG_COMMON_START & 0x3);

    // ......
}
```

Guest从device的configuration space读到这个配置，知道device启用了这个capability，就写对应的BAR-region (偏移`VPCI_CFG_COMMON_START`处)，传递过来vring各个area的地址。在kvmtool中，BAR-0和BAR-1的region都支持这个操作，它们注册的callback都是`virtio_pci_modern__io_mmio_callback`，只是io-port map和memory map的不同。

```c
void virtio_pci_modern__io_mmio_callback(struct kvm_cpu *vcpu, u64 addr,
                     u8 *data, u32 len, u8 is_write,
                     void *ptr)
{
    struct virtio_device *vdev = ptr;
    struct virtio_pci *vpci = vdev->virtio;
    u32 mmio_addr = virtio_pci__mmio_addr(vpci);

    virtio_pci_access(vcpu, vdev, addr - mmio_addr, data, len, is_write);
}

static bool virtio_pci_access(struct kvm_cpu *vcpu, struct virtio_device *vdev,
                  unsigned long offset, void *data, int size,
                  bool write)
{
    access_handler_t handler = NULL;

    switch (offset) {
    case VPCI_CFG_COMMON_START...VPCI_CFG_COMMON_END:
        if (write)
            handler = virtio_pci__common_write;
        else
            handler = virtio_pci__common_read;
        break;

    // ...

    }

    if (!handler)
        return false;

    return handler(vdev, offset, data, size);
}

static bool virtio_pci__common_write(struct virtio_device *vdev,
                     unsigned long offset, void *data, int size)
{
    u64 features;
    u32 val, gsi, vec;
    struct virtio_pci *vpci = vdev->virtio;

    switch (offset - VPCI_CFG_COMMON_START) {

    // ...

    case VIRTIO_PCI_COMMON_Q_DESCLO:
        vpci_selected_vq(vpci)->vring_addr.desc_lo = ioport__read32(data);
        break;
    case VIRTIO_PCI_COMMON_Q_DESCHI:
        vpci_selected_vq(vpci)->vring_addr.desc_hi = ioport__read32(data);
        break;
    case VIRTIO_PCI_COMMON_Q_AVAILLO:
        vpci_selected_vq(vpci)->vring_addr.avail_lo = ioport__read32(data);
        break;
    case VIRTIO_PCI_COMMON_Q_AVAILHI:
        vpci_selected_vq(vpci)->vring_addr.avail_hi = ioport__read32(data);
        break;
    case VIRTIO_PCI_COMMON_Q_USEDLO:
        vpci_selected_vq(vpci)->vring_addr.used_lo = ioport__read32(data);
        break;
    case VIRTIO_PCI_COMMON_Q_USEDHI:
        vpci_selected_vq(vpci)->vring_addr.used_hi = ioport__read32(data);
        break;
    }

    return true;
}
```

代码中`desc_hi/desc_lo`，`avail_hi/avail_lo`，`used_hi/used_lo`分别是Descriptor-Area，Driver-Area和Device-Area的地址(高32位和低32位)。

# 通知机制 (4)

在第2节看到vring在两个方向上传输数据；与之对应，需要两个方向上的通知。The virtio specification defines bi-directional notifications:

- available-buffer notification: the virtio-x driver notifies the virtio-x device that there are buffers in the vring that are ready to be processed;
- used-buffer notification: the virtio-x device signals the virtio-x driver that it has finished processing some buffers;

In the PCI case, the guest sends the available-buffer notification by writing to a specific memory address, and the device uses a vCPU interrupt to send the used buffer notification.

名词约定：在kvmtool的代码中，available-buffer notification用的是**notify**一词；而used-buffer notification用的是**signal**一词(即interrupt)。

## Available-buffer notification (4.1)

The guest (virtio-x driver) sends the available buffer notification by writing to a specific memory address. 这是virtio的notify capability。和前面说的common capability一样，virtio/pci-modern.c:virtio_pci_modern_init函数中也启用了notify capability:

```c
int virtio_pci_modern_init(struct virtio_device *vdev)
{
    int subsys_id;
    struct virtio_pci *vpci = vdev->virtio;
    struct pci_device_header *hdr = &vpci->pci_hdr;

    subsys_id = le16_to_cpu(hdr->subsys_id);

    hdr->device_id = cpu_to_le16(PCI_DEVICE_ID_VIRTIO_BASE + subsys_id);
    hdr->subsys_id = cpu_to_le16(PCI_SUBSYS_ID_VIRTIO_BASE + subsys_id);

    vpci->doorbell_offset = VPCI_CFG_NOTIFY_START;
    vdev->endian = VIRTIO_ENDIAN_LE;

    // ...

    hdr->virtio.notify = (struct virtio_pci_notify_cap) {
        .cap.cap_vndr        = PCI_CAP_ID_VNDR,
        .cap.cap_next        = PCI_CAP_OFF(hdr, virtio.isr),
        .cap.cap_len        = sizeof(hdr->virtio.notify),
        .cap.cfg_type        = VIRTIO_PCI_CAP_NOTIFY_CFG,
        .cap.bar        = 1,
        .cap.offset        = cpu_to_le32(VPCI_CFG_NOTIFY_START),
        .cap.length        = cpu_to_le32(VPCI_CFG_NOTIFY_SIZE),
        /*
         * Notify multiplier is 0, meaning that notifications are all on
         * the same register
         */
    };

    // ...
}
```

Guest从device的configuration space读到这个配置，知道device启用了notify capability，就写对应的BAR-region (偏移`VPCI_CFG_NOTIFY_START`处)，实现通知功能。在kvmtool中，BAR-0和BAR-1的region都支持这个操作，它们注册的callback都是`virtio_pci_modern__io_mmio_callback`，只是io-port map和memory map的不同。

### 被动接收通知消息 (4.1.1)

如果不主动poll通知消息(见4.1.2节)的话，virtio-x driver写BAR-region-0或者BAR-region-1(偏移`VPCI_CFG_NOTIFY_START`)时，就触发`virtio_pci_modern__io_mmio_callback`函数：

```c
void virtio_pci_modern__io_mmio_callback(struct kvm_cpu *vcpu, u64 addr,
                     u8 *data, u32 len, u8 is_write,
                     void *ptr)
{
    struct virtio_device *vdev = ptr;
    struct virtio_pci *vpci = vdev->virtio;
    u32 mmio_addr = virtio_pci__mmio_addr(vpci);

    virtio_pci_access(vcpu, vdev, addr - mmio_addr, data, len, is_write);
}

static bool virtio_pci_access(struct kvm_cpu *vcpu, struct virtio_device *vdev,
                  unsigned long offset, void *data, int size,
                  bool write)
{
    access_handler_t handler = NULL;

    switch (offset) {

    // ...

    case VPCI_CFG_NOTIFY_START...VPCI_CFG_NOTIFY_END:
        if (write)
            handler = virtio_pci__notify_write;
        break;

    // ...

    }

    if (!handler)
        return false;

    return handler(vdev, offset, data, size);
}

static bool virtio_pci__notify_write(struct virtio_device *vdev,
                     unsigned long offset, void *data, int size)
{
    u16 vq = ioport__read16(data);
    struct virtio_pci *vpci = vdev->virtio;

    vdev->ops->notify_vq(vpci->kvm, vpci->dev, vq);

    return true;
}
```

通知消息中只包含virt queue number；因为buffer在vring中，这里只要告诉device vring里有数据就行了。

![figure4](passively-recv-notifications.png)
<div style="text-align: center;"><em>图4: 被动接收通知消息</em></div>

### 主动poll通知消息 (4.1.2)

主动poll通知消息具有更高的优先级，假如启用的话，4.1.1节中的方式就接收不到消息(我在kvmtool上实验过)。如何启用呢？这刚好填了前一篇[interrupt virtualization](https://www.yuanguohuo.com/2024/08/11/virtualization-5-kvmtool-interrupt/)中第4.3.3节留下的坑：启用被选择的common-queue。Guest选择一个common-queue，配置中断，共享vring地址(前面第3节)之后就启用它，触发如下callback：

```c
static bool virtio_pci__common_write(struct virtio_device *vdev,
                     unsigned long offset, void *data, int size)
{
    u64 features;
    u32 val, gsi, vec;
    struct virtio_pci *vpci = vdev->virtio;

    switch (offset - VPCI_CFG_COMMON_START) {

    // ...

    case VIRTIO_PCI_COMMON_Q_ENABLE:
        val = ioport__read16(data);
        if (val)
            virtio_pci_init_vq(vpci->kvm, vdev, vpci->queue_selector);
        else
            virtio_pci_exit_vq(vpci->kvm, vdev, vpci->queue_selector);
        break;

    // ...

    }

    return true;
}
```

我们只看启用`virtio_pci_init_vq`：启用queue的主要任务是`vdev->ops->init_vq`，让device处理vring上的buffer。但我们本小节的重点是`virtio_pci__init_ioeventfd`，它的任务是启用主动poll机制。如注释中所述，把它删掉也不影响功能，使用被动接收方式而已。

```c
int virtio_pci_init_vq(struct kvm *kvm, struct virtio_device *vdev, int vq)
{
    int ret;
    struct virtio_pci *vpci = vdev->virtio;

    //也可以跳过对virtio_pci__init_ioeventfd()的调用；
    //假如跳过，notification就会通过这个方式发送过来(就是4.1.1节中的被动接收通知消息)：
    //      virtio_pci_modern__io_mmio_callback() -->
    //      virtio_pci_access()
    //      {
    //        case VPCI_CFG_NOTIFY_START...VPCI_CFG_NOTIFY_END:
    //          virtio_pci__notify_write()
    //      }
    ret = virtio_pci__init_ioeventfd(kvm, vdev, vq);
    if (ret) {
        pr_err("couldn't add ioeventfd for vq %d: %d", vq, ret);
        return ret;
    }
    return vdev->ops->init_vq(kvm, vpci->dev, vq);
}
```

函数`virtio_pci__init_ioeventfd`主要做的事情是：

- 创建一个eventfd实例，叫做ioeventfd；
- 构造一个`struct kvm_ioeventfd`实例`kvm_ioevent`: {.addr=BAR-region起始地址+VIRTIO_PCI_QUEUE_NOTIFY; .fd=ioeventfd;}
- 通过`ioctl(vm_fd, KVM_IOEVENTFD, &kvm_ioevent)`，告诉kvm内核模块；

就是告诉内kvm模块：本该通过写BAR-region(偏移`地址+VIRTIO_PCI_QUEUE_NOTIFY`)发送的通知(第4.1.1节的方式)，现在通过ioeventfd发。看！它们的地址是相同的：`BAR-region起始地址+VIRTIO_PCI_QUEUE_NOTIFY`！

另外，在kvmtool中BAR-0和BAR-1的功能是相同的(只是一个io-port map另一个memory map)，所以上述过程对BAR-0和BAR-1分别执行一遍。

内核模块kvm如何和guest交互完成这一协定不是文本的内容；总之，**以后的通知会从ioeventfd上发送过来，谁实现data plane谁就要poll ioeventfd**；若不使用vhost，也就是VMM(qemu,kvmtool)实现data plane，完全模拟device，那么VMM就poll ioeventfd，得到通知之后去处理vring中的available-buffer；若使用vhost，则kvmtool不去poll它，交给host内核或者用户态进程的data plane(分别对应vhost和vhost-user模式)；

![figure5](actively-poll-notifications.png)
<div style="text-align: center;"><em>图5: 主动poll通知消息</em></div>

## Used-buffer notification (4.2)

The virtio-x device uses a vCPU interrupt to send the used buffer notification.

前一章[interrupt virtualization](https://www.yuanguohuo.com/2024/08/11/virtualization-5-kvmtool-interrupt/)已经说到，要给guest发中断需要请求kvm内核模块来完成：

- 对于irqchip方式：

```c
ioctl(kvm->vm_fd, KVM_IRQ_LINE, {.irq=gsi});
```

- 对于msi方式:

```c
ioctl(kvm->vm_fd, KVM_SIGNAL_MSI, {.address_lo=addr_lo .address_hi=addr_hi, .data=data});
```

以virtio-blk为例，应该是在处理完请求之后，发起中断。

```c
static void virtio_blk_do_io(struct kvm *kvm, struct virt_queue *vq, struct blk_dev *bdev)
{
    struct blk_dev_req *req;
    u16 head;

    //检查avaiable-buffer queue是否不空
    while (virt_queue__available(vq)) {
        //获取avaiable-buffer queue的head；它是Descriptor Area表的index；
        head        = virt_queue__pop(vq);
        //reqs和Descriptor Area表一一对应；
        req        = &bdev->reqs[head];
        //从Descriptor Area的表项中得到buffer的地址和长度，放到req->iov中，并不拷贝buffer本身；
        req->head    = virt_queue__get_head_iov(vq, req->iov, &req->out,
                    &req->in, head, kvm);
        req->vq        = vq;

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
    iov        = req->iov;

    // ...

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

void virtio_blk_complete(void *param, long len)
{
    struct blk_dev_req *req = param;
    struct blk_dev *bdev = req->bdev;
    int queueid = req->vq - bdev->vqs;
    u8 *status;

    /* status */
    status = req->status;
    *status    = (len < 0) ? VIRTIO_BLK_S_IOERR : VIRTIO_BLK_S_OK;

    mutex_lock(&bdev->mutex);
    //used-buffer放进queue
    virt_queue__set_used_elem(req->vq, req->head, len);
    mutex_unlock(&bdev->mutex);

    //触发中断；对于PCI transport，signal_vq指向virtio_pci__signal_vq()函数；
    if (virtio_queue__should_signal(&bdev->vqs[queueid]))
        bdev->vdev.ops->signal_vq(req->kvm, &bdev->vdev, queueid);
}

int virtio_pci__signal_vq(struct kvm *kvm, struct virtio_device *vdev, u32 vq)
{
    struct virtio_pci *vpci = vdev->virtio;
    int tbl = vpci->vq_vector[vq];

    if (virtio_pci__msix_enabled(vpci) && tbl != VIRTIO_MSI_NO_VECTOR) {
        if (vpci->pci_hdr.msix.ctrl & cpu_to_le16(PCI_MSIX_FLAGS_MASKALL) ||
            vpci->msix_table[tbl].ctrl & cpu_to_le16(PCI_MSIX_ENTRY_CTRL_MASKBIT)) {

            vpci->msix_pba |= 1 << tbl;
            return 0;
        }

        if (vpci->signal_msi)
            virtio_pci__signal_msi(kvm, vpci, vpci->vq_vector[vq]);
        else
            kvm__irq_trigger(kvm, vpci->gsis[vq]);
    } else {
        // ...
    }
    return 0;
}

//模拟边沿触发，所以发一个高电平加一个低电平
void kvm__irq_trigger(struct kvm *kvm, int irq)
{
    kvm__irq_line(kvm, irq, 1);
    kvm__irq_line(kvm, irq, 0);
}

static void virtio_pci__signal_msi(struct kvm *kvm, struct virtio_pci *vpci,
                   int vec)
{
    struct kvm_msi msi = {
        .address_lo = vpci->msix_table[vec].msg.address_lo,
        .address_hi = vpci->msix_table[vec].msg.address_hi,
        .data = vpci->msix_table[vec].msg.data,
    };

    if (kvm->msix_needs_devid) {
        msi.flags = KVM_MSI_VALID_DEVID;
        msi.devid = vpci->dev_hdr.dev_num << 3;
    }

    irq__signal_msi(kvm, &msi);
}
```

The virtio specification also allows the notifications to be enabled or disabled dynamically. That way, devices and drivers can batch buffer notifications or even actively poll for new buffers in virtqueues (busy polling). This approach is better suited for high traffic rates. 

也就是说，两端都能够使用polling模式：

- virtio-x driver端：guest里不使用内核态virtio-x driver，而是使用SPDK用户态virtio driver； 
- virtio-x device端：vhost-user；问题：在VMM(kvmtool,qemu)里poll ioeventfd，如第4.1.2节所示，算是polling模式吗？

# 低效问题 (5)

从前面图1,4,5可见，data plane在VMM(kvmtool,qemu)中，即用户态进程中，这带来一些效率问题：

- virtio-x driver发完available-buffer notification，vCPU停止运行，转到VMM(kvmtool,qemu)用户态进程。这有个内核态到用户态的context switch；
- 通常情况下，VMM(virtio-x device)还是要借助host kernel的能力来处理数据，这就有用户态到内核态的数据拷贝。例如，VMM中的虚拟网卡要把数据拷贝到host内核态的tap；VMM中的blk要把数据拷贝到内核态驱动。
- VMM(virtio-x device)给virtio-x driver发中断，要使用ioctl，它是一个系统调用，所以有个用户态到内核态再返回用户态的context switch；
- 因为vCPU停止运行了，还需要一个系统调用来resume vCPU，也是有用户态到内核态再返回用户态的context switch；

这所有的低效都因为data plane在VMM中。假如data plane在host的内核中，那么context switch就没有了，并且也不存在用户态到内核态的数据拷贝问题：通过一些内存映射，vring中的buffer可以直接共享给内核里的data plane。这就是下一章vhost的内容。

# 小结 (6)

结合kvmtool中virtio-blk的实现，粗略学习virtio机制。其中的重点是virt-queue和两个方向上的通知(notify和interrupt)。
