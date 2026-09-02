---
title: NAND SSD TRIM and GC
date: 2020-10-08 18:18:20
tags: [disk,ssd]
categories: disk 
---

SSD 的写入性能并非一成不变。一块全新的 SSD 顺序写入可以跑到 3 GB/s，但当盘接近写满时，性能可能骤降至几百 MB/s 甚至更低，延迟也会出现剧烈抖动。这一切的根源在于 NAND 闪存的物理特性，以及 SSD 内部的**垃圾回收（Garbage Collection, GC）** 机制。而 **TRIM**（又称 Discard / Deallocate）正是操作系统与 SSD 之间沟通"哪些数据不再需要"的关键桥梁。

本文将从 NAND 闪存的物理结构出发，逐步深入到 GC 机制、TRIM 命令的工作原理，以及在不同存储栈中的工程实践。

<!-- more -->

<script type="text/x-mathjax-config">
MathJax.Hub.Config({
tex2jax: {inlineMath: [['$','$'], ['\\(','\\)']]}
});
</script>

<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>


# NAND 闪存的物理结构 (1)

NAND 闪存有两个核心概念：

| 概念      | 说明                                               | 典型大小                       |
|-----------|----------------------------------------------------|--------------------------------|
| **Page**  | 最小**读写**单位，只能整 page 读写                 | 4 KB / 8 KB / 16 KB            |
| **Block** | 最小**擦除**单位，包含多个 page，只能整 block 擦除 | 256 KB ~ 数 MB（如 64 × 4 KB） |

除了必须整 page 读写（假如不对齐，则RMW: Read-Modify-Write），还有一个关键约束：

**NAND 只能向已擦除（erased）的 page 写入数据，不能像 HDD 那样直接覆盖。**

> 例外：Intel Optane（3D XPoint）支持 write-in-place，不受此限制，但该技术已停产。

---

# SSD 的写入机制 (2)

由于不能覆盖写，SSD 内部维护了一张 **FTL（Flash Translation Layer）** 映射表，记录逻辑地址到物理地址的对应关系。

## 覆盖写流程 (2.1)

假设操作系统要覆盖逻辑 page-100（当前映射到物理 page-M）：

```
1. SSD 找到一个已擦除的物理 page-N；
2. 将新数据写入 page-N；
3. 更新 FTL 映射：逻辑 100 → 物理 N；
4. 物理 page-M 标记为 stale（无效但未擦除）。
```

注意：**stale page 既不在使用中，也未被擦除**（not in use but not erased），它占着空间却无法直接写入，只能等待 GC 处理。

---

# SSD 垃圾回收 (3)

当 SSD 内部的空闲（erased）block 不足时，必须触发 GC 来回收空间。

## GC 流程 (3.1)

假设 block P 包含 64 个 page，其中 61 个 stale，3 个 in use：

```
1. 从 block-P 中读出 3 个有效 page；
2. 在其他 block（如 Q、R）中找到 3 个 erased page，写入这 3 个有效 page；
3. 更新 FTL 映射；
4. 擦除整个 block-P → 64 个 page 全部变为可用。
```

## GC 的代价 (3.2)

- **写放大（Write Amplification）**：为了回收 61 个 stale page 的空间，需要额外搬运 3 个有效 page。
- **延迟抖动**：如果写入时恰好没有空闲 block，必须**同步等待 GC 完成**才能继续写入，导致尾延迟飙升。
- **寿命损耗**：每次擦除都消耗 block 的 P/E 寿命。

**核心矛盾：SSD 不知道哪些数据已经"逻辑上"不再需要，只能被动等待覆盖写时才发现 stale page。如果 SSD 能提前知道，就可以主动、从容地在空闲时做 GC，而不是在写入路径上被迫同步 GC。**

---

# 问题：文件系统删除 ≠ SSD 感知 (4)

对于**覆盖写**，SSD 没有问题——旧映射自然失效，SSD 知道哪些 page 是 stale 的，可以GC。

但对于**文件删除（delete）和截断（truncate）**，问题就来了：

假如删除或截断了一个文件：

| 视角     |  动作                      | 状态                              |
|----------|----------------------------|-----------------------------------|
| 文件系统 | 仅更新自己的元数据         | 逻辑 page 200~300 已释放，可复用  |
| SSD      | 毫不知情，无 stale 更无 GC | 逻辑 page 200~300 仍然有效        |


文件系统只在自己的元数据（inode、bitmap）中标记空间可用，**并不会通知 SSD**。等到文件系统真正复用 200~300 进行覆盖写时，SSD 才知道旧数据已失效。

这意味着：
- SSD 无法提前对 200~300 进行 GC；
- 当 SSD 中 erased block 耗尽时，写入必须先触发同步 GC → **性能断崖**。

---

# TRIM：告知 SSD "这些空间不再需要" (5)

## 术语统一 (5.1)

不同层级和协议对同一操作有不同的命名，但**语义完全相同**：

| 层级/协议      | 术语           | 说明                             |
|----------------|----------------|----------------------------------|
| SATA           | **TRIM**       | SATA 协议中的命令名              |
| NVMe           | **Deallocate** | 通过 Dataset Management 命令实现 |
| SCSI           | **UNMAP**      | SCSI 标准术语                    |
| Linux 块设备层 | **Discard**    | `BLKDISCARD` ioctl               |
| Linux 文件系统 | **Discard**    | `mount -o discard` / `fstrim`    |
| SPDK BlobStore | **Unmap**      | `spdk_blob_io_unmap()`           |

> 一句话总结：**TRIM = Discard = Deallocate = UNMAP**，都是"通知 SSD 某些 LBA 不再使用，可以回收"。

## TRIM 的作用 (5.2)

操作系统通过 TRIM 命令告诉 SSD："逻辑地址 X~Y 的数据已无效。"SSD 收到后：

1. 在 FTL 中将对应 page 标记为 stale；
2. 在空闲时主动进行 GC（defragmentation），提前擦除 block；
3. 后续写入时大概率已有充足的 erased block，**避免同步 GC**。

## Linux 内核中的实现 (5.3)

```c
// 块设备层下发 discard
blkdev_issue_discard(bdev, sector, nr_sects, gfp_mask);

// 用户态直接 ioctl
ioctl(fd, BLKDISCARD, &range);   // range = {start, length}
```

判断设备是否支持 discard：

```bash
$ cat /sys/block/sdc/queue/discard_granularity
0   # HDD，不支持

$ cat /sys/block/nvme0n1/queue/discard_granularity
512 # NVMe SSD，支持，最小粒度 512 字节
```

---

# TRIM 的层级透传 (6)

TRIM 必须在**所有 I/O 抽象层**上逐层启用，否则命令无法到达 SSD。

以 `ext4 → LVM → dm-crypt → NVMe SSD` 为例：

```
┌─────────────────────────────────┐
│  ext4 (mount -o discard)        │  ← 文件系统层
├─────────────────────────────────┤
│  LVM (issue_discards = 1)       │  ← 逻辑卷层
├─────────────────────────────────┤
│  dm-crypt (--allow-discards)    │  ← 加密层
├─────────────────────────────────┤
│  NVMe SSD                       │  ← 硬件
└─────────────────────────────────┘
```

任何一层未启用，TRIM 命令就会在该层被丢弃，SSD 永远收不到。

---

# TRIM 策略：Continuous vs Periodic (7)

Linux 文件系统提供两种 TRIM 策略：

## Continuous TRIM (7.1)

Continuous TRIM 即 实时 TRIM。

```bash
mount -o discard /dev/nvme0n1p1 /mnt/data
```

- **行为**：每次删除文件时，立即向 SSD 发送 TRIM 命令。
- **优点**：空间回收最及时。
- **缺点**：
  - 频繁触发 SSD 内部 GC，可能影响前台 I/O 性能；
  - 若中间某层未正确启用 TRIM，**不会报错**，静默失败；
  - 误删文件后，数据立即被标记无效，**无法恢复**。

## Periodic TRIM (7.2)

```bash
systemctl enable fstrim.timer    # 默认每周执行一次
```

- **行为**：定期（默认每周）扫描整个文件系统，一次性 TRIM 所有空闲空间。
- **优点**：
  - 效率高，批量下发；
  - 若中间层未启用 TRIM，`fstrim` **会报错**，便于排查；
  - 一周内误删文件仍有恢复机会。
- **缺点**：空间回收不及时，两次 TRIM 之间 SSD 可能积累大量 stale 数据。

## 如何选择 (7.3)

| 场景                    | 推荐策略                                          |
|-------------------------|---------------------------------------------------|
| 桌面 / 普通服务器       | **Periodic TRIM**（`fstrim.timer`）               |
| 分布式存储（Ceph、SPDK）| **异步实时 Discard**（见下文）                    |
| 数据库（高写入负载）    | 视 SSD OP 空间而定，通常 Periodic + 配置充足的 OP |

> OP: Over-Provisioning，预留空间；即对用户不可见、仅供控制器内部使用的额外 NAND 空间。

---

# 高性能存储系统中的实践 (8)

## Ceph BlueStore：异步 Discard 线程 (8.1)

Periodic TRIM 对分布式存储来说回收太慢，而 Continuous TRIM 如果同步执行又会阻塞主 I/O 路径。Ceph BlueStore 的解决方案是**独立的异步 discard 线程**：

```
WAL / DB / 数据删除
    → wal_discard_cb / db_discard_cb / slow_discard_cb
        → KernelDevice::_discard_thread（独立线程）
            → 积攒一批 interval_set
                → BlkDev::discard()
                    → ioctl(fd, BLKDISCARD, range)
```

- 主 I/O 路径只负责将待 discard 的范围加入队列，**不阻塞**；
- 后台线程批量下发，兼顾及时性与性能；
- 仅在设备支持 discard 时启用（通过 `discard_granularity` 判断）。

## SPDK 用户态：绕过内核 (8.2)

SPDK 使用用户态 NVMe 驱动，**完全绕过 Linux 内核块设备层**，因此：

- 没有 `/sys/block/...`；
- 没有 `BLKDISCARD` ioctl；
- 直接通过 PCIe 下发 **NVMe Dataset Management（Deallocate）** 命令；
- BlobStore 层对应 API 为 `spdk_blob_io_unmap()`；

```
spdk_blob_io_unmap()          [BlobStore 层，异步]
    → 更新 cluster 元数据映射
    → spdk_nvme_ns_cmd_dataset_management()   [NVMe 层]
        → SSD 控制器标记 LBA 无效
            → SSD 后台 GC 择机擦除
```

**注意**：无论是内核态的 `BLKDISCARD` 还是用户态的 NVMe Deallocate，都只是**通知 SSD 标记无效**，物理擦除仍然由 SSD 固件在后台异步完成，无法强制立即擦除。

---

# 安全注意事项：加密层的 TRIM (9)

在 dm-crypt / LUKS 加密卷上启用 TRIM（`--allow-discards`）存在**信息泄露风险**：

- 攻击者可以观察哪些 block 被 discard，推断出加密卷中哪些区域有数据、哪些是空闲的；
- 结合文件系统结构知识，可能进一步推断文件分配模式。

**建议**：在高安全要求场景下，加密层**不要**启用 discard 透传。

---

# 总结 (10)

```
┌──────────────────────────────────────────────────────────────┐
│                      应用 / 文件系统                         │
│         删除文件 → 释放逻辑空间 → 发送 TRIM/Discard          │
└────────────────────────┬─────────────────────────────────────┘
                         │  TRIM / Discard / Deallocate
                         ▼
┌──────────────────────────────────────────────────────────────┐
│                      SSD 控制器 (FTL)                        │
│         标记对应 page 为 stale → 空闲时后台 GC               │
│         搬运有效 page → 擦除整个 block → 释放空间            │
└──────────────────────────────────────────────────────────────┘
```

核心要点：

1. **NAND 不能覆盖写**，只能写 erased page，擦除以 block 为单位 → 必须有 GC。
2. **文件系统删除不通知 SSD** → SSD 无法提前 GC → 写入时可能被迫同步 GC → 性能断崖。
3. **TRIM 是桥梁**：告知 SSD 哪些 LBA 已无效，让 GC 从容进行。
4. **TRIM ≠ 立即擦除**：它只是通知，物理擦除由 SSD 固件异步完成。
5. **工程实践**：通用场景用 Periodic TRIM；高性能存储用异步 discard 线程；SPDK 用户态直接下发 NVMe Deallocate。
6. **保障写入性能的关键**：充足的 OP 空间 + 及时的 TRIM + 企业级 SSD 固件。

# 参考 (11)

- [How To Configure Periodic TRIM for SSD Storage on Linux Servers](https://www.digitalocean.com/community/tutorials/how-to-configure-periodic-trim-for-ssd-storage-on-linux-servers)
- [How to properly activate trim for your ssd on linux fstrim lvm and dmcrypt](http://blog.neutrino.es/2013/howto-properly-activate-trim-for-your-ssd-on-linux-fstrim-lvm-and-dmcrypt/)
- [Block layer discard requests](https://lwn.net/Articles/293658/)
