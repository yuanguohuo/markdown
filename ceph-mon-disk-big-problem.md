---
title: Ceph-Mon Disk Big 报警的处理
date: 2024-08-22 18:30:12
tags: [ceph,mon]
categories: ceph
---

# 告警 (1)

告警：`ceph -s` 出现 `mons <a>,<b>,... are using a lot of disk space`。

触发条件：mon 数据目录大小超过 `mon_data_size_warn` （Nautilus 默认 15GiB，不同ceph版本可能不同）。

> 说明：
> - mon 是 Paxos 多副本，**每个副本各有一份独立的 RocksDB store**；
> - 内容经 Paxos 提交复制，每台 mon 的 db 大小通常一致；
> - PGMap 全量统计由 mgr 维护，不在 mon db 里。

# Mon db 空间分类 (2)

Ceph Monitor 分为3层（从下到上）：

- 存储层：MonitorDBStore (RocksDB)；按 prefix 组织的 KV
- 共识层：Paxos；不理解语义，只搬运/复制/持久化“state-machine 指令”
- 服务层：理解语义（知道 “osd out” 意味着什么）
    - OSDMonitor
    - MDSMonitor
    - LogMonitor
    - AuthMonitor
    - ConfigMonitor
    - ConfigKeyService (`ceph config-key get/set/rm`)
    - ...

按层级从下到上（注：非按“导致告警的可能性”排序），占 mon db 的地方有：

- 存储层：RocksDB 未回收；可通过`ceph tell 'mon.*' compact` 解决；
- 共识层：paxos prefix
    - 可以理解成 “paxos log” 或 “raft log”；
    - 每一轮共识（一个 paxos version）提交的聚合事务——这一轮里各服务（osdmap/auth/mdsmap...）各改了什么，打包成一个 blob 存下来；blob 本身就是完整 `MonitorDBStore::Transaction` 的编码，里面包含本轮对各服务 prefix 的全部写操作——提交时一条原子事务同时写“日志副本”（paxos prefix）和“状态”（服务 prefix）。这正是“两层各有独立裁剪参数”的原因（`paxos_trim_min/max` vs `paxos_service_trim_min/max`——前者裁共识流水，后者裁各服务状态）。
    - 另外，还会维护 `first_committed`/`last_committed` 标记（相当于 paxos log 的范围，raft也会维护）。
    - 注意：各服务的正式数据不在这个 prefix 里——osdmap 存 osdmap prefix、auth 存 auth prefix …… paxos prefix 只是“每轮提交了哪些变更”的流水；是consensus service 的范畴；consensus service “不理解” 每个聚合事务里是什么；对它来说，就是**修改 state-machine 的指令**。
    - 保留窗口：`paxos_min`（默认 500） ；
    - 裁剪：`paxos_trim_min=250` 意思是超出“保留窗口”250个以上才动手裁剪，例如日志窗口长700，则跳过裁剪；长800则裁剪到`paxos_min=500`；`paxos_trim_max=500` 意思是每次裁剪最多前进 500 个；若由于某种原因，窗口累积到1200，不会一下子裁剪到`paxos_min=500`，而是第一次先裁剪到700（裁剪`paxos_trim_max`个），第二次跳过，因为700 < 750（`paxos_min + paxos_trim_min`）；过一段时间，堆积到 750 以上，再裁剪到`paxos_min=500`。
- 服务层1：OSDMonitor 维护的 osdmap，这是最有可能导致告警的地方；下一节细讲；
- 服务层2：LogMonitor 维护的 cluster 级别的事件log，例如：“osd.123 marked down”、“pg 1.2 entered state active+clean”、health 告警、管理命令审计…… 就是`ceph -w`滚动刷屏的那份日志的持久副本；只保留最近 500 个 osdmap epoch 窗内的日志（`mon_max_log_epochs=500`），有天花板，撑不大。
- 服务层3：其它服务，例如 AuthMonitor/ConfigMonitor/ConfigKeyService 等，占用空间一般比较小，导致告警的可能性不大；

# OSDMap（最常见）详解 (3)

## 存储结构：每个 epoch 存两份 (3.1)

每次 osdmap 提交（leader 的 `OSDMonitor::encode_pending`），store 里写入：

| 存储项           | 写入点                                                        | 单个体积                           |
|------------------|---------------------------------------------------------------|------------------------------------|
| **full map**     | `put_version_full(t, epoch, fullbl)`（无条件、每 epoch 一份） | 与集群规模相关，千盘级集群约 2 MB+ |
| **incremental**  | `put_version(t, epoch, inc_bl)`                               | 通常几 KB，crush 变更时可达几百 KB |

体积大头永远是 full。例如一次实际案例：2,000+ OSD 集群，单个 full = 2,226,238 B ≈ 2.12 MB。

## trim 机制：决定“留多久” (3.2)

Mon leader 周期性尝试 trim，由`Monitor::tick()`触发；tick周期默认是 5 秒（见配置项`mon_tick_interval`）；trim 时，`OSDMonitor::get_trim_to()` 计算 trim 终点（floor），trim 机制分批（`paxos_service_trim_max`）删除 `first_committed` 到 floor 之间的历史；删除区间`[first_committed, floor)`，floor 保留，成为新的`first_committed`。

```cpp
// floor 是 保留的起点，也就是 trim 的终点；
//   1. 主要是 min(各 pool 的 last_epoch_clean)——哪个池最不干净，整个下限就被它拉到哪；
//   2. 所有已 osd out（未 rm）OSD 上报过的最老 epoch，如果比 1 算出的值还老，继续把下限往下拉（代码里 osd_epochs 那个循环，注释写着 "don't trim past the oldest reported osd epoch"）。
epoch_t floor = get_min_last_epoch_clean();

// last_committed 是最新提交的 osdmap；
// mon_min_osdmap_epochs 默认 500，即至少保留最近 500 个；
if (floor + mon_min_osdmap_epochs > last_committed)
    floor = last_committed - mon_min_osdmap_epochs;
```

规则拆解：

1. `min_last_epoch_clean`：各 pool “最后一次全部 PG clean” 的 epoch 的最小值
    - Nautilus 版本按 pool 跟踪，`ceph report` 的 `osdmap_clean_epochs.last_epoch_clean.per_pool` 可查；
    - **只要集群有降级/恢复中的 PG，或存在长期 down 的 OSD，该值就冻结**；
    - trim 一路裁到它为止，之后一步都裁不动；
2. **out OSD 的次级钉扎**：已 `osd out` 但未 `osd rm` 的 OSD，其上报的最老 epoch 同样拉低 floor，直到被 `osd rm` 才解除；
3. **保底区间**：`mon_min_osdmap_epochs`，和前文共识层的“paxos_min”完全对应，默认值也是500（只是巧合；它俩是不同层里同一个东西，默认值相同不是必须的）。看前文共识层的保底区间`paxos_min`即可。唯一不同：某些情况下，floor 被拉低，保留区间会超过 `mon_min_osdmap_epochs`，甚至超出很多（而共识层不会：其区间受容忍带上限约束，`paxos_min + paxos_trim_min`，约 750）。
4. **裁剪**：和前文共识层对应；配置`paxos_service_trim_min/max`和共识层的`paxos_trim_min/max`也对应，`floor − first_committed < paxos_service_trim_min` 则跳过裁剪；每批最多裁剪 `paxos_service_trim_max` 个。
5. **trim 时机是持续的**：每个周期（如前所说，默认 5 秒）都尝试裁到 floor，**与 mon_osdmap_full_prune_min 这个数无关**（那是 prune 的，见下文）；
6. 例外：`creating_pgs` 非空时 trim 完全暂停；
7. `mon_osd_force_trim_to` 可显式指定 floor（会丢弃更早历史，谨慎使用）。

## prune 机制：决定“留的东西怎么存” (3.3)

背景：trim 保留区间内每个 epoch 都带一份 2.12 MB 的 full，区间一长（如上万 epoch）体积爆炸。prune 对**保留区间内部**做压缩，不动区间边界。

**激活条件**

激活条件是`should_prune()`，即同时满足如下2个条件（另隐含：总历史须 > `mon_min_osdmap_epochs` 且 `first_committed` 尚未达 B）：

- 条件1：`(last_committed - mon_min_osdmap_epochs) - first_committed >= mon_osdmap_full_prune_min`；即 “可 prune 区间 >= mon_osdmap_full_prune_min”:

```
                                                                  此区间不许动
     |<---------------- 可 prune 区间 ---------------->|<---- mon_min_osdmap_epochs (500)----->|
     |                                                 |                                       |
     |-------------------------------------------------|---------------------------------------|
A:first_committed                 B:last_committed - mon_min_osdmap_epochs               C:last_committed
  (floor)
```

其实 `first_committed` 就是 `floor`，因为，假如 `floor > first_committed`，则会被 trim 机制裁剪到 floor 处。通常 `mon_osdmap_full_prune_min` 比较大（默认是 10,000），所以，满足此条件，意味着 floor 被某种原因（见上一节）拉的很低（即值很小）。也就是说，**prune 机制只是在 trim 机制失效时才起作用**；假如 floor 能及时更新，trim 机制会保持“可 prune 区间”很短（甚至为0，即 `first_committed` 被裁剪到 B），不会触发大于`mon_osdmap_full_prune_min`的条件。

- 条件 2：最后一个 pinned full 之后还凑得满一个 `mon_osdmap_full_prune_interval`（默认 10）——不足一个间隔的零头本轮不压（逐轮推进的进度闸门）。

**压缩方式**

每 `mon_osdmap_full_prune_interval`（默认 10）个 epoch 保留 1 个 full（pinned）osdmap ，**删除中间冗余的 full；增量全部保留**；注：最近 500 个 epoch 原样保留（“此区间不许动”）；

所以，prune 只是压缩（删除冗余的 full osdmap），**旧 osdmap 仍可精确取回**：请求落在 pinned 区间的任意 epoch，由 `get_full_from_pinned_map` 取最近的 pinned full，然后逐个 apply 保留的增量（≤10 个）就精确得到请求的 osdmap。即现场重构——**以少量 CPU 时间换 10 倍空间**；

所以说，**prune 机制是安全的，它没有删除“信息”，只是“压缩”信息**。任何需要旧 osdmap 的场景，例如落后很久的 OSD，需要某个 osdmap 时，先拿到最近的 pinned full 再 apply 增量，就会精确得到所需。

注意：prune 是 leader 执行、paxos 全集群复制的动作，无法单机灰度；

## 故障发生 (3.4)

正常情况下，trim 会把保留区间稳在大约 500 个，**prune 永远不会被激活**；告警是因为两种机制同时没起作用：

- trim 失效：floor 被 `min_last_epoch_clean` 或者 out OSD 钉死，导致保留区间拉长到数千 epoch；
- prune 还未激活：保留区间被拉长到数千，但仍小于 `mon_osdmap_full_prune_min` （默认是 10,000）
    - 每个 epoch 仍存一份 full osdmap；
    - 占空间 $=$ epoch数 $\times$ full-osdmap-size，线性持续增长；

一个案例：
- full osdmap 大小 2.12MB
- 每天新增约166个epoch

| 集群状态                        | 哪个机制工作                           | db 体积走势                                     |
|---------------------------------|----------------------------------------|-------------------------------------------------|
| 正常（持续 clean）              | 只有 trim，区间约 500                  | 稳定在约 1 GiB（500 × 2.12 MB ≈ 1.04 GiB）      |
| 轻度异常（floor 钉住 < ~60 天） | trim 停住，区间线性增长，prune 未激活  | 线性上涨（约 0.34 GiB/天），可能撞告警线 15GiB  |
| 重度异常（floor 钉住 > ~60 天） | prune 接力，每 10 留 1                 | 增速降为约 1/10                                 |

经常遇到的场景是：floor 被钉住较久，足以撑爆告警线，但又钉得不够久、未触发 prune 机制。

## 排查步骤（按序执行）(3.5)

1. Mon db 体积 与 RocksDB 健康：选择一个 非leader mon 实例（因为所有 mon 实例通常一致）

```bash
du --max-depth=1 -h /var/lib/ceph/mon

# 停止 mon 进程

# 看 compaction 是否健康
ceph-kvstore-tool rocksdb {mon-home-dir}/store.db stats

# 启动 mon 进程
```

2. 查看 osdmap 区间，per-pool floors 等信息：在 mon leader 上执行

```bash
ceph report > /tmp/report.json
python3 - <<'EOF'
import json
r = json.load(open('/tmp/report.json'))
print("osdmap_first_committed:", r["osdmap_first_committed"])
print("osdmap_last_committed :", r["osdmap_last_committed"])
print("min_last_epoch_clean  :", r["osdmap_clean_epochs"]["min_last_epoch_clean"])
m = r.get("osdmap_manifest")
print("pinned:", (m["first_pinned"], m["last_pinned"], len(m["pinned_maps"])) if m else "absent(prune未激活)")
print("per-pool floors:", r["osdmap_clean_epochs"]["last_epoch_clean"])  # 本版本为按 pool 列表
EOF
```

3. 查看 full osdmap 大小

```bash
ceph osd getmap -o /tmp/osdmap.current && ls -l /tmp/osdmap.current
```

4. 估算体积

机制 prune 未激活时：

`(osdmap_last_committed - osdmap_first_committed + 1) * full_osdmap_size`

若大约等于 mon db size （第一步 `du` 获取），则确认根因！

## 处理方案 (3.6)

- 激活 prune 机制（安全，简单）：
    - **虽然安全，但也建议先备份 mon db；备份一台就够，但注意备份时要停 mon 进程**，因为运行中直接拷 RocksDB 可能拿到不一致快照；
    - 步骤：a. `ceph tell 'mon.*' injectargs --mon_osdmap_full_prune_min=8000`（**注意：只要能激活 prune 机制，设置 8000 还是 2000 并无区别，不会影响 prune 后 mon db 的体积；影响体积的是** `mon_osdmap_full_prune_interval`，**它是压缩密度**）；b. 用`ceph report`观察`osdmap_manifest`推进（`first_pinned/last_pinned`），等待压缩完成（完成标志是`last_pinned`逼近`last_committed - 500`；prune 每 tick 5 秒推进 1 个间隔，8,000 区间需约 1 小时）；c. `ceph tell 'mon.*' compact`；
    - 注：如前所述，**保留区间没变，信息没有任何丢失，只是压缩了存储**，删除冗余 full osdmap，可由最近 pinned full osdmap 和增量原样重构（**时间换空间**）。
- 找出 floor 被钉住的根因（彻底）：找出垫底项（per-pool floor / 长期 down OSD / 未 rm 的 out OSD）:
    - 降级 pg：尽快恢复clean
    - down OSD: 尽快 out，迁数据后 `osd rm`
- 强制裁剪（风险）：设置 `mon_osd_force_trim_to`；更早的 osdmap 永久不可取，落后 OSD 只能全量同步，非紧急不用！
- 调高告警线（消音）：设置 `mon_data_size_warn` 到更大值，例如 30GiB；

# 小结 (4)

1. 存储层：`ceph tell mon.{id} compact` 只回收“删了没缩”的部分（RocksDB内部机制），对真实存量数据无效；
2. 共识层：通过 `paxos_min`，`paxos_trim_min/max` 控制 paxos log 的保留区间长度，占很多空间的可能性不大；
3. 服务层：最可能导致 mon db 体积增大的是 osdmap：相关机制有两套，trim 和 prune；
4. trim 缩短保留区间：类似于共识层的 paxos log 的保留区间裁剪，删除 osdmap `first_committed` 到 floor 之间的 epoch，然后 floor 成为新的 `first_committed`（由此缩短保留区间）；
5. 当 floor 被钉死时，区间持续增长，导致 mon db 体积超限；而 floor 被钉死的原因通常是集群不健康：floor = min(所有 pool 的 last_clean, down/out osd 的依赖)；
6. 所以，治本的办法是尽快让集群恢复到健康状态：所有 pool 的 pg 都恢复到 clean；down/out osd 迁移完后 remove；这些都依赖数据迁移；
7. 线上通常要控制数据迁移速度，所以，有的时候不能尽快“治本”；此时，可以通过 prune 机制降低存储密度；
