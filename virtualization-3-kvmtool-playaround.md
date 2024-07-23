---
title: 虚拟化入门笔记--kvmtool playaround
date: 2024-07-22 17:16:18
tags: [virtualization,kvm,kvmtool]
categories: kvm
---

Kvmtool可以认为是一个极简版的qemu (所处软件层、功能都和qemu一样)，只有几千行代码，便于学习虚拟化的原理。

<!-- more -->

# 前提 (0)


一台物理机，安装linux系统，并且安装了KVM.

```
uname -r
3.10.0-693.5.2.el7.x86_64

ls /dev/kvm
/dev/kvm
```

# 下载源码并编译 (1)


从github下载。当前git commit id是da4cfc3e540341b84c4bbad705b5a15865bc1f80.

```
git clone https://github.com/kvmtool/kvmtool.git
```

编译：

```
cd kvmtool
make -j8
```

编译完成之后，当前目录下生成可执行工具`lkvm`.


# 获取一个内核 (3)

## 编译 (3.1)

按kvmtool文档的要求，要设置编译选项:

```
- For the default console output:
   CONFIG_SERIAL_8250=y
   CONFIG_SERIAL_8250_CONSOLE=y

- For running 32bit images on 64bit hosts:
   CONFIG_IA32_EMULATION=y

- Proper FS options according to image FS (e.g. CONFIG_EXT2_FS, CONFIG_EXT4_FS).

- For all virtio devices listed below:
   CONFIG_VIRTIO=y
   CONFIG_VIRTIO_RING=y
   CONFIG_VIRTIO_PCI=y

- For virtio-blk devices (--disk, -d):
   CONFIG_VIRTIO_BLK=y

- For virtio-net devices ([--network, -n] virtio):
   CONFIG_VIRTIO_NET=y

- For virtio-9p devices (--virtio-9p):
   CONFIG_NET_9P=y
   CONFIG_NET_9P_VIRTIO=y
   CONFIG_9P_FS=y

- For virtio-balloon device (--balloon):
   CONFIG_VIRTIO_BALLOON=y

- For virtio-console device (--console virtio):
   CONFIG_VIRTIO_CONSOLE=y

- For virtio-rng device (--rng):
   CONFIG_HW_RANDOM_VIRTIO=y

- For vesa device (--sdl or --vnc):
   CONFIG_FB_VESA=y
```


## 直接使用虚拟机的内核 (3.2)

若购买过阿里云、腾讯云或者华为云的VM，可以直接使用VM的内核，厂家编译时做过配置。虽然配置和KVM文档要求不尽相同，但实测(阿里云)可以运行，可能有些特性不能使用。其配置是这样的：

```
CONFIG_SERIAL_8250=y
CONFIG_SERIAL_8250_CONSOLE=y

CONFIG_IA32_EMULATION=y

CONFIG_VIRTIO=m
# CONFIG_VIRTIO_RING is not set
CONFIG_VIRTIO_PCI=m

CONFIG_VIRTIO_BLK=m

CONFIG_VIRTIO_NET=m

# CONFIG_NET_9P is not set
# CONFIG_NET_9P_VIRTIO is not set
# CONFIG_9P_FS is not set

CONFIG_VIRTIO_BALLOON=m

CONFIG_VIRTIO_CONSOLE=m

CONFIG_HW_RANDOM_VIRTIO=m

CONFIG_FB_VESA=y
```

从虚拟机拷贝`/boot/vmlinuz-3.10.0-957.el7.x86_64`和`/boot/initramfs-3.10.0-957.el7.x86_64.img`到物理机kvmtool目录，待使用。

# 制作rootfs (4)

为了兼容，以3.2节中使用的VM当作模版来制作rootfs.

```
df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda1        20G  2.2G   17G  12% /
...
```

当前rootfs是2.2G，所以至少准备2.2G以上的空间，我这里分配3G。

```
truncate -s $(echo 1024*1024*1024*3|bc) rootfs-3.10.0-957.img
```

然后就是格式化，mount并拷贝数据：

```
mkfs.ext4 rootfs-3.10.0-957.img
mount -o loop rootfs-3.10.0-957.img /mnt/
cp -ax /{bin,dev,etc,lib,root,sbin,usr,var} /mnt
mkdir /mnt/{home,proc,opt,sys,tmp}
chmod 777 /mnt/tmp
```

修改/mnt/etc/fstab

```
cat /mnt/etc/fstab
/dev/vda  /  ext4  defaults  1  1
```

umount，就制作好了。也拷贝到物理机kvmtool目录。

```
umount /mnt/
```


# 运行 (5)

```
sudo ./lkvm run  \
        --dev /dev/kvm                           \
        --virtio-transport pci                   \
        --loglevel debug                         \
        --disk ./rootfs-3.10.0-957.img           \
        --kernel ./vmlinuz-3.10.0-957.el7.x86_64 \
        --initrd ./initramfs-3.10.0-957.el7.x86_64.img
```

不出错的话，就会显示登陆提示。

常见错误1: Warning: /dev/vda does not exist. 可能内核不对，编译时没有配置`CONFIG_VIRTIO=y|m`(所以其它VIRTIO相关的配置不起作用?)，所以也识别不到`--disk ./rootfs-3.10.0-957.img`指定的虚拟盘；它是通过virtio实现的。

常见错误2: systemctl/systemd相关的错误。可能rootfs的问题。因为systemd的配置在rootfs中，其中可能有些服务起不来。这也是第4节中我们以同一个VM为模版制作rootfs的原因。

# 小结 (6)

本文简单记录如何使用kvmtool启动虚拟机。下一篇研究PCI设备的虚拟化。
