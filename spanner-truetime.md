---
title: Spanner TrueTime 与并发控制
date: 2020-09-21 21:10:58
tags: [paxos, spanner, truetime]
categories: paxos
---

Google Spanner 基于 TrueTime 的并发控制机制支持跨多个 Paxos group 的事务。它使用两阶段提交协议（2PC: 2 Phase Commit）在 group 间分发。本文是一个简化版本，事务只涉及一个 Paxos group，因此也无需两阶段提交协议，旨在搞清楚 TrueTime 机制。

<!-- more -->

# 一、要解决的问题

## 1.1 一个简单的场景

假设有一个银行系统，用户 Alice 在上海，用户 Bob 在北京。数据分布在全球多个数据中心。
```
Alice 执行事务 T1：从账户 A 转 100 元到账户 B
  → T1 提交成功
  → Alice 打电话告诉 Bob："钱已经转了"

Bob 执行事务 T2：读取账户 B 的余额
  → Bob 应该能看到这 100 元
```

这就是**外部一致性**（External Consistency）的要求：如果 T1 在**现实世界中**先完成，T2 后开始，那么 T2 必须能看到 T1 的结果。

> 注：下文中，**物理时间**、**现实世界的时间**、**真实时间**、**t_abs()的返回值** 四者等价，指**物理世界中客观流逝的绝对时间**，与任何机器的本地时钟无关。（TODO：统一）

## 1.2 为什么这很难？

在单机数据库中，这很简单 — 用一把锁或一个递增的序列号就行。

但在分布式数据库中：
```
问题 1：数据在多台机器上
  → 需要分布式事务协议（如 2PC）

问题 2：不同机器的时钟不一样
  → 上海服务器的时钟可能比北京服务器快 5ms
  → 如果用本地时钟给事务打时间戳：
    T1 在上海提交，时间戳 = 100
    T2 在北京开始，时间戳 = 98  (北京时钟慢了)
    → 系统认为 T2 比 T1 早！违反外部一致性！

具体到 1.1 的例子：
  → Bob 的只读事务 T2 拿到 sread=98，去读账户 B 时，
    会查找 timestamp ≤ 98 的最新版本。
  → 而 Alice 的转账 T1 的写入会被跳过，因为 timestamp=100 大于 98
  → Bob 读到的是转账之前的旧余额，看不到 Alice 转的那 100 元。
  → "违反外部一致性"在用户层面的感知就是：
    Alice 说"钱转了"，Bob 查了说"没收到"。
```

**核心困难：分布式系统中没有统一的全局时钟。**

## 1.3 已有方案的问题

| 方案 | 做法 | 缺陷 |
|------|------|------|
| 逻辑时钟（Lamport Clock） | 用消息传递建立因果序 | 只保证因果序，不保证与物理时间一致 |
| 中心化时间戳服务 | 所有事务找一台机器要时间戳 | 单点瓶颈，跨洲延迟高 |
| NTP 同步时钟 | 定期同步各机器时钟 | 误差可达 ~100ms，且**不知道误差有多大** |

Spanner 的解决方案：**TrueTime — 不假装时钟精确，而是明确告诉你时钟的误差范围。**

---
> **关于逻辑时钟（Lamport Clock）的缺陷**   
>    
> **zhuhai**：   

对于 Alice 和 Bob 的例子，Lamport Clock 保证不了 Bob 能看到转账结果，因为转账和查看没有因果序。Alice 给 Bob 打电话，Bob 才去查看，看上去像因果关系，但这个因果关系不是分布式系统的，分布式系统不知道这个电话。   
>    
> **AI**：   

完全正确。Lamport Clock 的因果序定义：A → B（A 因果先于 B）当且仅当：   
>    
> - A 和 B 在同一个进程中，A 先执行 或   
> - A 是发送消息，B 是接收同一条消息 或   
> - 存在 C 使得 A → C 且 C → B（传递性）   
>    
> → Alice 转账和 Bob 查账是"并发"事件（concurrent）
> → Lamport Clock 不保证任何顺序
> → Bob 完全可能拿到比 Alice 更小的逻辑时间戳


**这就是外部一致性比因果一致性更强的地方**：外部一致性保证事务顺序与物理时间一致，不管事务之间有没有系统内的消息传递，只要 T1 在物理时间上先完成，T2 就能看到 T1 的结果。

用户的任何外部通信渠道（电话、微信、当面说）都不会导致不一致。这也是 Spanner 需要 TrueTime（物理时间）而不能只用逻辑时钟的根本原因 — 逻辑时钟无法捕获系统外部的因果关系。

---

# 二、TrueTime：一个暴露不确定性的时间 API

## 2.1 API 定义

TrueTime 只有三个方法：
```
TT.now()     → 返回一个区间 [earliest, latest]
TT.after(t)  → 返回 true/false
TT.before(t) → 返回 true/false
```

## 2.2 TT.now() 的含义
```
调用 TT.now() 返回 [earliest, latest]

保证：当前的真实绝对时间一定在这个区间内

即：earliest ≤ 真实时间 ≤ latest

例如：
  TT.now() 返回 [100, 108]
  → 真实时间可能是 100, 101, ..., 107, 108 中的任何一个
  → 但绝对不会是 99 或 109

误差半径 ε = (latest - earliest) / 2 = 4ms
```

**关键：这是一个硬性保证（bound），不是概率估计。**

如果这个保证被违反了（比如因为硬件故障），Spanner 的正确性就没了。所以 Google 在硬件上投入了大量成本来确保这个保证。

## 2.3 TT.after(t) 和 TT.before(t)
```
TT.after(t)：
  返回 true  当且仅当 now的真实时间 > t 是确定的
  实现：TT.now().earliest > t
  含义："时间点 t 已经确定过去了"

TT.before(t)：
  返回 true  当且仅当 now的真实时间 < t 是确定的
  实现：TT.now().latest < t
  含义："时间点 t 确定还没到"
```

**注意：**`**TT.after(t)**`**返回 false 不代表 t 还没过去，而是"我们无法确定 t 是否已经过去"。** 同理，`TT.before(t)` 返回 false 也不代表 t 已经到了，而是"无法确定 t 是否还没到"。TrueTime 是保守的 — 宁可说"不确定"，也不会给出错误的断言。

## 2.4 TrueTime 的硬件实现

**参与方**

```
1. Time Master（每个数据中心若干台）
   ├── GPS Master（大部分）
   │   └── 配备 GPS 接收器 + 专用天线
   │   └── 精度：微秒级
   │   └── 弱点：天线故障、无线干扰、GPS 系统故障
   │
   └── Armageddon Master（少量）
       └── 配备原子钟
       └── 精度：微秒级，但会缓慢漂移
       └── 弱点：频率误差导致长期漂移
       └── 优势：与 GPS 故障模式完全不同

   → GPS 和原子钟同时失效的概率极低

2. TimeSlave Daemon（每台服务器一个）
   └── 每 30 秒轮询多台 Time Master
   └── 使用 Marzullo 算法排除异常时钟
   └── 同步后 ε ≈ 1ms
   └── 随时间漂移：ε 按 200μs/s 增长
   └── 下次同步前 ε ≈ 7ms
   └── 平均 ε ≈ 4ms
```

**ε 的时间曲线（锯齿波）**

```
ε(ms)
 7 |    /|    /|    /|
 6 |   / |   / |   / |
 5 |  /  |  /  |  /  |
 4 | /   | /   | /   |
 3 |/    |/    |/    |
 2 |     |     |     |
 1 *     *     *     *   ← 每次同步后 ε 跌回 ~1ms
   |-----|-----|-----|
   0    30    60    90   秒
       每 30 秒同步一次
```

我们不关心硬件实现，**但要记住这个锯齿波**：软件每次调用 `TT.now()` 返回的区间宽度都是波动的。也正因为这个锯齿波，earliest 和 latest 都不是单调的。

- latest 没有单调性：硬件同步后，ε 会大幅变小
    - 真实时间 100，ε = 8，TT.now() = \[92,108\]   
    - 真实时间 101，ε = 2，TT.now() = \[99,103\]   

- earliest 没有单调性：ε 会慢慢变大
    - 真实时间 200，ε = 2，TT.now() = \[198,202\]
    - 真实时间 201，ε = 5，TT.now() = \[196,206\]   

---

# 三、简化场景：单 Paxos 组

## 3.1 什么是 Paxos 组

为了聚焦 TrueTime 的核心逻辑，我们只考虑**一个 Paxos 组**的情况。
```
一个 Paxos 组 = 一组副本（通常 3 或 5 个），
  其中一个是 Leader，其余是 Follower（Slave）。

Leader 负责：
  - 接受客户端的读写请求
  - 协调写操作（通过 Paxos 协议复制到多数副本）
  - 分配事务时间戳
  - 管理锁表（Follower没有锁表）

数据存储(每个副本都有Leader+Follower)：
  - 每条数据有多个版本，每个版本带一个时间戳
  - (key, timestamp) → value
  - 例如：("Alice_balance", 100) → 1000
          ("Alice_balance", 105) → 900   // 在时间戳 105 被修改
```

## 3.2 各参与方维护的状态
```
┌─────────────────────────────────────────────────────┐
│                  Paxos Leader                       │
│                                                     │
│  维护的状态：                                       │
│                                                     │
│  1. Lock Table（锁表）                              │
│     key range → lock state                          │
│     用于两阶段锁（2PL）的并发控制                   │
│                                                     │
│  2. smax（最大已分配时间戳）                        │
│     Leader 已经分配给任何 Paxos 写的最大时间戳      │
│     用于保证时间戳单调递增                          │
│                                                     │
│  3. Leader Lease（领导者租约）                      │
│     lease_interval = [lease_start, lease_end]       │
│     Leader 只在 lease 有效期内才能分配时间戳        │
│                                                     │
│  4. MinNextTS(n)                                    │
│     Paxos 序号 n+1 的最小可能时间戳                 │
│     用于推进 Follower 的 safe time                  │
│                                                     │
│  5. LastTS()                                        │
│     最后一次 committed 写的时间戳                   │
│                                                     │
├─────────────────────────────────────────────────────┤
│                  每个副本（Leader + Follower）      │
│                                                     │
│  维护的状态：                                       │
│                                                     │
│  1. 多版本数据                                      │
│     (key, timestamp) → value                        │
│                                                     │
│  2. t_safe_Paxos                                    │
│     已 apply 的最高 Paxos 写的时间戳                │
│     保证不会再有 ≤ 此时间戳的写入                   │
│                                                     │
│  3. tsafe = t_safe_Paxos                            │
│     (单 Paxos 组无分布式事务，所以没有 t_safe_TM)   │
│     一个读请求 read@t 只有在 t ≤ tsafe 时才能执行   │
│                                                     │
│  4. Lease vote state                                │
│     (tend) 表示当前 lease vote 的过期时间           │
│     在 TT.after(tend) 之前不会给其他 Leader 投票    │
│                                                     │
├─────────────────────────────────────────────────────┤
│                  TimeSlave Daemon（每台机器一个）   │
│                                                     │
│  维护的状态：                                       │
│                                                     │
│  1. 本地时钟校准参数                                │
│  2. ε（当前时钟不确定性）                           │
│  3. 上次同步时间                                    │
│                                                     │
│  对外提供：TT.now() → [earliest, latest]            │
│                                                     │
├─────────────────────────────────────────────────────┤
│                  客户端                             │
│                                                     │
│  维护的状态：                                       │
│                                                     │
│  1. 事务缓冲区（写操作先缓存在客户端）              │
│  2. 已读数据的时间戳                                │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

说明：两阶段锁（2PL, Two-Phase Locking）：
-  阶段 1 — 增长阶段（Growing Phase）：
    - 只能加锁，不能释放任何锁
    - 可以加读锁，也可以加写锁，也可以升级
- 阶段 2 — 收缩阶段（Shrinking Phase）：
    - 只能释放锁，不能再加任何新锁

关键约束：一旦释放了任何一把锁，就不能再加新锁。

说明：Leader Lease

在 lease\_end 之前， 一定不可能出现新leade；即使当前leader 失去领导权，新leader也不可能被选举出来。原因：
1. Leader 的 lease 来自 quorum 的 lease vote
2. 每个副本投 lease vote 时记录：
tend = TT.now().latest \+ 10s
在 TT.after(tend) 返回 true 之前，该副本不会给任何其他候选人投票
3. 新 Leader 需要从 quorum 中获得 lease vote
4. 新旧 quorum 必有交集（多数派的性质）
5. 交集中的副本在 TT.after(tend) 之前不会投新票
→ 新 Leader 不可能在旧 lease 到期前凑齐 quorum
所以：
旧 Leader 的 lease\_end 之前 → 新 Leader 不可能产生
这就是 disjointness invariant（不重叠不变量）

## **3.3 tsafe** 

一个副本能否服务 read@t，取决于：t ≤ tsafe

- t ≤ tsafe → 可以读（数据完整，不会再变）；因为 apply 是顺序的（spanner限制），tsafe 确定（如何确定，见t_safe_Paxos）之后，不可能写入版本号（时间戳）小于等于 tsafe 的数据，换言之，tsafe以及以前的版本都是固定的，所以可以放心读取。
- t > tsafe → 不能读（可能还有未 apply 的写，数据不完整）。

在单 Paxos 组场景下，tsafe 等价于 t_safe_Paxos。对于跨 Paxos group 的事务，tsafe = min(t_safe_Paxos, t_safe_TM)。本文不考虑跨 Paxos group 的事务。

## **3.4 t_safe_Paxos**

t_safe_Paxos = 已 apply 的最高 Paxos 写的时间戳。

含义：这个副本上，所有 timestamp ≤ t_safe_Paxos 的数据都是完整的，不会再有 timestamp ≤ t_safe_Paxos 的新写入出现。也就是上面说的，因为 apply 是顺序的（spanner限制）。

> 注意：
> - slot 顺序 = 时间戳顺序，因为 Leader 按顺序分配时间戳，按顺序提交到 Paxos log。
> - Leader 切换，仍然一致，因为 Lease 不重叠 → 新 Leader 的时间戳一定 > 旧 Leader 的所有时间戳；新 Leader 的 slot 号也一定 > 旧 Leader 的所有 slot 号

t_safe_Paxos 何时推进呢？有三种方式：

- 有新写入 apply 时（自然推进，Leader和Follower）：
    - slot 42 apply, timestamp=95 → t_safe_Paxos = 95
    - slot 43 apply, timestamp=200 → t_safe_Paxos = 200

- 空闲时靠 MinNextTS 推进（Leader和Follower）：
    - 没有新写入 → t_safe_Paxos 卡住
    - Leader 推进 MinNextTS(42) = 8001
    - Leader 和 Follower 将 t_safe_Paxos 推进到 8000（= MinNextTS - 1）

- 主动查询MinNextTS（Follower）：
    - Follower 接收到一个读请求，发现自己无法服务（读取版本 > tsafe）
    - 向Leader发请求，查询最新MinNextTS（所以不必等Leader 8秒 周期更新，避免卡住请求8秒）
    - 推进 t_safe_Paxos，服务读请求；

问题：Leader 自己需要靠 MinNextTS 推进 t_safe_Paxos 吗？

- 如果读请求没有带版本号：Leader 用 sread = LastTS()，即最后一次 committed 写的时间戳（注意不是 TT.now().latest）。sread ≤ t_safe_Paxos 必然成立（因为那个slot/log-entry已经apply了，apply时更新LastTS）。
- 如果请求带了版本号 t 并且 t > t_safe_Paxos，Leader 确实需要推进 t_safe_Paxos，但 Leader 自己就是 MinNextTS 的维护者，它可以立即推进 MinNextTS 到 TT.now().latest + 1，然后推进 t_safe_Paxos。论文没有说，实现可以自由，不违反一致性约束。

## **3.5 MinNextTS(n)**

MinNextTS(n) 是 Leader 维护并同步给 Follower 的。

注意：虽然叫 MinNextTS(n)，leader并不需要维护 MinNextTS(1), MinNextTS(2), ... 多个值， 只需要维护一个值：当前最新 slot（slot 等价于raft 的 log-entry） 的 MinNextTS。

### **MinNextTS(n) 的含义**

"Paxos 序号 n\+1 的写操作，时间戳不会小于 MinNextTS(n)"

这是 Leader 对 Follower 做的一个承诺（promise）。

Follower 收到后，就可以安全地将 t_safe_Paxos 推进到 MinNextTS(n) - 1。

### **何时更新**

两种触发方式：
1. 定时推进（默认每 8 秒）
Leader 定期主动推进 MinNextTS：
每 8 秒,MinNextTS(n) = TT.now().latest \+ 1（承诺下一个写的时间戳至少在当前时间之后）
这是空闲时的行为 — 没有写操作时，Leader 通过定期推进 MinNextTS 让 Follower 的 tsafe 跟上当前时间
2. Follower 按需请求（on demand）
当 Follower 收到一个快照读 read@t，但 tsafe \< t 时：
Follower 可以主动向 Leader 请求推进 MinNextTS
Leader 收到请求后立即推进并回复
3. 事务commit或者apply时，**不需要更新** MinNextTS(n)，因为：Follower 在 apply 时直接更新  t_safe_Paxos；MinNextTS(n) 的作用就是推进 t_safe_Paxos，而 t_safe_Paxos 已直接更新。

### **更新约束**

Leader 更新 MinNextTS(n) 的约束：
1. MinNextTS(n) 必须在 Leader 当前的 lease 区间内
→ 如果要承诺一个超出 lease\_end 的值，
Leader 必须先续约 lease
2. MinNextTS(n) 只能增大，不能减小
→ 一旦承诺了"下一个写 ≥ 100"，
就不能改口说"下一个写 ≥ 90"
3. 更新后同步推进 smax：
smax = max(smax, max(MinNextTS(n)))
→ 保证 disjointness invariant

### **如何同步给 Follower**

论文没有详细说明同步机制，但有两种可能的方式：

方式 A：搭载在 Paxos 心跳/空操作上

Leader 定期发送的 Paxos 心跳或 no-op 消息中

附带最新的 MinNextTS 值

→ 零额外通信开销

方式 B：独立的推进消息

Leader 每 8 秒发送专门的 MinNextTS 推进消息

→ Follower 收到后更新本地的 t_safe_Paxos

实际效果：

Leader 推进 MinNextTS(42) = 8096

→ 同步给所有 Follower

→ Follower 将 t_safe_Paxos 推进到 8095

→ 快照读 read@8000 现在可以在 Follower 上执行了

### **完整的时间线示例：**

- t=0         
    - 写操作 commit, slot=42, timestamp=95
    - MinNextTS(42) = 96  （至少）
    - Follower: t_safe_Paxos = 95
- t=8s     Leader 定时推进
    - MinNextTS(42) = TT.now().latest \+ 1 ≈ 8001
    - 同步给 Follower，Follower: t_safe_Paxos = 8000
- t=16s    Leader 再次推进
    - MinNextTS(42) ≈ 16001
    - 同步给 Follower，Follower: t_safe_Paxos = 16000
- t=20s    新的写操作来了
    - Leader 分配 timestamp = max(TT.now().latest, smax\+1)
    - 这个 timestamp 一定 ≥ MinNextTS(42) ≈ 16001
    - 承诺没有被违反 ✓ slot=43, timestamp=20005，MinNextTS(43) = 20006

### **关键点**

| 问题         | 答案                                                     |
|--------------|----------------------------------------------------------|
| 谁维护       | Leader                                                   |
| 谁消费       | Follower，用于推进 t_safe_Paxos                          |
| 多久更新     | Leader：默认每8秒<br>Follower: Leader同步，或者按需请求  |
| 如何保证正确 | 承诺值必须在 lease 内，<br>且只增不减，<br>同步推进 smax |
| Leader 切换  | 新 Leader 的时间戳在新 lease 区间内，<br>自然 \> 旧承诺  |

---

# 四、读写事务的完整流程（单 Paxos 组）

## 4.1 流程总览
```
事务 T：读 key1，写 key1=newValue

阶段 1：读
         → 获取读锁
         → 并返回当前值
阶段 2：客户端缓冲写
阶段 3：提交：
         → 获取写锁 
         → 分配时间戳
         → Paxos 写 
         → Commit Wait 
         → 释放锁 
         → 回复客户端
```

## 4.2 详细步骤
```
Client                         Paxos Leader                    Follower 1, 2
  |                                |                               |
  |  ① Read(key1) --------------->|                               |
  |                                | 获取 key1 的读锁              |
  |                                | 读取 key1 最新版本的值         |
  |<-- value1 --------------------|                               |
  |                                |                               |
  | [客户端本地：计算 newValue]      |                               |
  | [客户端本地：缓冲写操作]         |                               |
  |                                |                               |
  |  ② Commit(key1=newValue) ---->|                               |
  |                                |                               |
  |                                | ③ 获取 key1 的写锁            |
  |                                |    (升级读锁为写锁)            |
  |                                |                               |
  |                                | ④ 分配 commit timestamp s：   |
  |                                |    s = max(                   |
  |                                |      TT.now().latest,        |
  |                                |      smax + 1                |
  |                                |    )                          |
  |                                |    更新 smax = s              |
  |                                |                               |
  |                                | ⑤ Paxos 写：将 (key1, s,     |
  |                                |    newValue) 复制到多数副本    |
  |                                |    --Paxos Propose---------->|
  |                                |    <--Paxos Accept------------|
  |                                |    (多数副本确认后 commit)      |
  |                                |                               |
  |                                | ⑥ Commit Wait：               |
  |                                |    while (!TT.after(s)) {    |
  |                                |      等待...                  |
  |                                |    }                          |
  |                                |    // 现在确定：真实时间 > s   |
  |                                |                               |
  |                                | ⑦ Apply：写入本地存储          |
  |                                |    (key1, s) → newValue       |
  |                                |    更新 t_safe_Paxos = s      |
  |                                |                               |
  |                                | ⑧ 释放所有锁                  |
  |                                |                               |
  |<-- Committed(s) --------------|                               |
  |                                |                               |
```

## 4.3 每一步的详细解释

**① Read(key1)**

客户端发送读请求到 Leader。Leader 在锁表中获取 key1 的读锁（共享锁），然后读取 key1 的最新版本数据返回给客户端。读锁的作用：防止其他事务在我读之后、提交之前修改这个 key。这是标准的两阶段锁（2PL）协议。

**② Commit(key1=newValue)**

客户端将所有写操作一次性发送给 Leader。写操作在事务执行期间一直缓冲在客户端，不会提前发送。好处：读操作不会看到自己事务未提交的写（简化逻辑）。

**③ 获取写锁**

Leader 将 key1 的读锁升级为写锁（排它锁）。如果有其他事务持有 key1 的锁 → 等待（wound-wait 避免死锁）。此时所有涉及的 key 都已上锁 → 满足 2PL 的"锁点"。

**④ 分配 commit timestamp s — 这是核心！**

规则：s = max(TT.now().latest, smax \+ 1)。TT.now().latest 是真实时间的上界，保证 s ≥ 真实时间（Start 规则）。smax \+ 1 保证时间戳严格单调递增。

> zhuhai：latest 没有单调性，见第2.4节（ε 锯齿波），当 latest 回退时，smax \+ 1 保证时间戳严格单调递增。   

**⑤ Paxos 写**

Leader 通过 Paxos 协议将写操作复制到多数副本，确保即使 Leader 挂了数据也不会丢失。Paxos 保证写按时间戳顺序 apply。

**⑥ Commit Wait — TrueTime 的核心机制！**

Leader 等待直到 TT.after(s) 返回 true，即确定真实时间大于 s。最坏情况需要等 2ε ≈ 8ms，但通常和 Paxos 通信并行。

**⑦ Apply & ⑧ 释放锁**

写入本地多版本存储，更新 t_safe_Paxos，释放所有锁，向客户端回复 Committed(s)。

---

# 五、只读事务的完整流程

读事务完全不需要锁，这是 Spanner 的一大优势。Spanner 支持两种读事务：快照读和只读事务，

两者都利用多版本存储读取固定时间戳的快照，不会和任何写操作冲突，因此完全无锁。

## **5.1 快照读**

Client 指定目标版本 t
```

Client                                                     任意副本
  |                                                          |
  | ----Read key1, key2 @ t (请求Leader或Follower)----------> |
  ｜                                 检查 t ≤ tsafe           ｜
  ｜                                 如果不满足，等待            ｜
  ｜                                 或向Leader查询MinNextTS(n) |
  |<-- value1, value2 ----------------------------------------|

```

## 5.2 只读事务

Client 不指定目标版本，需要向 leader 获取；
```

Client                         Paxos Leader               任意副本
  |                                |                          |
  |  ① ReadOnly(key1, key2) ----->|                          |
  |                                |                         |
  |                                | ② 选择 sread：           |
  |                                |    无 prepared 事务时：   |
  |                                |    sread = LastTS()     |
  |                                |                         |
  ｜ <------- ③返回 sread----------|                         |
  |                                                          |
  |                                                          |
  |                                                          |
  |  ④ Read key1, key2 @ t=sread (请求Leader或Follower)-----> |
  ｜                                 检查 t ≤ tsafe            ｜
  ｜                                 如果不满足，等待            ｜
  ｜                                 或向Leader查询MinNextTS(n) |
  |<-- value1, value2 ----------------------------------------|

```

## 5.2 sread 的选择

单 Paxos 组：sread = LastTS()（最后一次 committed 写的时间戳）。保证能看到所有已提交的写，不会看到未提交的写，满足外部一致性。

## 5.3 为什么不需要锁？

只读事务读的是一个固定时间戳 sread 的快照。由于数据是多版本的，读 sread 时刻的快照不会和任何写冲突：sread 之前的写已经 committed，sread 之后的写不会被读到。所以只读事务永远不会被写事务阻塞。

> zhuhai：对于同一个key来说，读事务完全不需要锁；而读写事务又会升级为读写锁（排他锁）。假设两个读写事务都获取读锁成功，将来升级为排他锁，必然有一个失败。读写事务为什么不直接加排他锁呢？   
> 关键点：**读写事务读的 key 和写的 key 可能不同**。例如读写事务TX1读key1写key2， TX2读key1写key3，则两个事务只对key1加读锁（共享锁），都能成功。   

## 5.4 效率问题

从 leader 上获取 sread，再去读 follower，不是很低效吗？既然访问了 leader，leader 从 LastTS() 得到 sread，何不直接返回数据？

- 场景 1：读多个 key，分摊 Leader 负载
- 场景 2：地理就近读：从地理上很远的 leader 获取一次sread，多次从很近的follower上读数据。
- 快照读（Snapshot Read）不需要访问 Leader：client 直接指定版本号。

此外，对于单 Paxos 组 + 少量 key 的只读事务，Leader 拿到 sread 后直接返回数据也是合理的（论文实现也支持）。

---

# 六、一致性约束：精确定义

## 6.1 外部一致性（External Consistency）

如果事务 T1 的提交在现实世界中先于事务 T2 的开始，那么 T1 的 commit timestamp 必须小于 T2 的 commit timestamp。

形式化：t_abs(e\_commit\_T1) \< t_abs(e\_start\_T2) → s1 \< s2

外部一致性 = 事务的时间戳顺序必须和真实世界中事务发生的顺序一致。这比可序列化（serializability）更强：可序列化只要求存在某个合法的序列化顺序，外部一致性要求这个顺序和物理时间一致。

## 6.2 单调性约束

在同一个 Paxos 组内，Spanner 分配的时间戳严格单调递增。保证方式：同一 Leader 用 smax；跨 Leader 用 lease 不重叠。

## 6.3 Leader Lease 不重叠约束（Disjointness Invariant）

对于同一个 Paxos 组，任意两个 Leader 的 lease 区间不重叠。副本授予 lease vote 时记录 tend = TT.now().latest \+ 10s，在 TT.after(tend) 之前不会给其他 Leader 投票，保证同一时间只有一个 Leader。

---

# 七、如何保证外部一致性：分步证明

## 7.1 两条规则

规则 A — Start 规则：分配 s ≥ TT.now().latest，保证 s ≥ 真实时间。见第4.3 小节。

规则 B — Commit Wait 规则：等待 TT.after(s) 为 true，保证提交时真实时间 \> s。见第4.3 小节。

## 7.2 证明过程
```
(1) s1 < t_abs(e_commit_T1)        ← Commit Wait 保证
(2) t_abs(e_commit_T1) < t_abs(e_start_T2)   ← 前提假设
(3) t_abs(e_start_T2) ≤ t_abs(e_server_T2)   ← 因果关系
(4) t_abs(e_server_T2) ≤ s2                 ← Start 规则

∴ s1 < s2  ∎
```

> zhuhai   
- t_abs(e\_commit\_T1) : 事务T1的commit的真实时间（现实世界时间）；
- s1: 事务T1提交的key的版本号；
- t_abs(e\_start\_T2)：事务T2开始的真实时间（现实世界时间）；Client端：
> Transaction Begin T2     \<--   t_abs(e\_start\_T2)   
> read some keys   
> update some keys   
> Transaction Commit T2    \<-- 向Leader发 Commit 请求   
- t_abs(e\_server\_T2):  leader 接收到 client 发来的事务T2的Commit 请求的真实时间（现实世界时间）；
- s2: 事务T2提交的key的版本号；


## 7.3 图解
```
真实时间轴 →→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→
                    s1     |           |        |                s2
  earliest        latest   |           |        |  earliest     latest
     [---ε---|---ε---]     |           |        |  [---ε---|---ε---] 
                      T1真实提交时间  T2-start  T2-Commit请求到达时间

Commit Wait 让 s1 回退到比 “T1真实提交时间” 更早。
Start 规则让 s2 推迟到比 “T2-Commit请求到达时间” 更晚。
间隔恰好覆盖时钟不确定性。
```

---

# 八、如果不遵守事务模型，会怎样违反一致性？

## 8.1 去掉 Commit Wait：违反外部一致性
```
场景：真实时间 = 100，ε = 4ms

T1 在机器 A 上：s1 = TT.now().latest = 104
不做 Commit Wait，立即提交，真实提交时间 = 100

T2 在机器 B 上（时钟偏慢），真实时间 = 101：
  TT.now() = [97, 103]
  s2 = 103 < s1 = 104 ✗ 违反外部一致性！

T2 读 T1 写的 key @103：timestamp=104 > 103 → 跳过 → 读到旧值
```



zhuhai: 具体到 1.1 的例子：
- Alice 的转账事务 T1，TT.now=\[96,104\]，真实提交时间是100（现实世界时间），值是 balance=900@104
- Bob的只读事务T2开始时间是101（现实世界时间）；由于Bob的机器时钟稍晚，它的TT.now = \[97,103\]
- 即 T2 的 sread=103，去读账户 B 时，会查找 timestamp ≤ 103 的最新版本；读不到 balance=900@104，只能读到旧余额。

注意：Bob的读开始时间是101（现实世界时间）比 T1 提交时间100（现实世界时间）要晚，但却读不到T1的更新，违反了外部一致性。

## 8.2 去掉 Start 规则：违反外部一致性

如果用 TT.now().earliest 分配时间戳：s 可能小于真实时间，导致后来的事务拿到更小的时间戳。
```
场景：ε = 4ms

T1 在机器 A 上提交：
  真实时间 = 100，TT.now() = [96, 104]
  s1 = TT.now().earliest = 96
  Commit Wait：TT.after(96) → earliest(96) ≤ 96 → 还不确定
  等一小会儿... TT.now() = [97, 105] → earliest(97) > 96 → 通过
  T1 真实提交时间 ≈ 101（等了约 1ms）
  数据写入：balance=900@96
  
T2 在机器 B 上（时钟偏慢 3ms），真实时间 = 102：
  机器 B 认为的时间比真实时间慢 3ms
  TT.now() = [95, 103]
  s2 = TT.now().earliest = 95
  s2 = 95 < s1 = 96 ✗ 违反外部一致性！
  T2 读 balance @95：
  → 96 > 95 → 跳过 T1 的写入
  → 读到旧值

```

zhuhai: 即使在同一台机器上也会出问题
```
场景：ε = 4ms

T1 在机器 A 上提交：
  真实时间 = 100，TT.now() = [96, 104]
  s1 = TT.now().earliest = 96
  Commit Wait：TT.after(96) → earliest(96) ≤ 96 → 还不确定
  等一小会儿... TT.now() = [97, 105] → earliest(97) > 96 → 通过
  T1 真实提交时间 ≈ 101（等了约 1ms）
  数据写入：balance=900@96
  
T2 还在机器 A 上，但 ε 变大了（见 ε 的锯齿波）
  T1在 真实时间101 提交后；T2在真实时间103开始。由于 ε 变大了，
  TT.now() = [95, 111]
  s2 = 95 还是会读到旧值
```

## 8.3 去掉 Leader Lease 不重叠：违反单调性

两个 Leader 的 lease 重叠时，可能出现时间戳回退（200 → 195）。

## 8.4 去掉 smax 追踪：违反单调性

不维护 smax 时，两个事务可能拿到相同的时间戳。

## 8.5 如果 TrueTime 的保证被违反（ε 不够大）

真实时间跑出 \[earliest, latest\] 区间 → 所有一致性保证崩塌。

zhuhai: 这是硬件故障。本文是在硬件正确的基础上进行分析，故忽略。plus：GPS 和原子钟故障模式不相关 → 同时失效概率极低。

---

# 九、各机制的协作关系
```
              TrueTime 保证 ε 可靠
                    |
                    v
        ┌───────────┴────────────┐
        |                        |
        v                        v
   Start 规则              Commit Wait
   s ≥ TT.now().latest     等待 TT.after(s)
   保证：s ≥ 真实时间       保证：提交时真实时间 > s
        |                        |
        v                        v
   后来事务的 s                先前事务的 s
   一定足够大                   一定足够小
        |                        |
        └───────────┬────────────┘
                    |
                    v
         外部一致性：s1 < s2
```
> zhuhai:   
>    s ≥ 真实时间是成立的，因为   
> s = max(TT.now().latest, smax\+1) \>= TT.now().latest \>= 真实时间   
>    虽然 latest 不单调（ε 波动可能导致latest回退），但它必定 \>= 真实时间   

---

# 十、总结

## 10.1 一句话概括

TrueTime 用硬件（GPS \+ 原子钟）提供可靠的时钟误差上界，Spanner 用两条简单规则（Start 规则 \+ Commit Wait）把这个误差上界转化为外部一致性保证。

## 10.2 各组件的职责

| 组件 | 职责 | 如果缺失会怎样 |
|------|------|---------------------|
| TrueTime (GPS \+ 原子钟) | 提供时钟误差上界 ε | 所有一致性保证崩塌 |
| Start 规则 (s ≥ TT.now().latest) | 保证时间戳 ≥ 真实时间 | 后来的事务可能拿到更小的时间戳 |
| Commit Wait (等待 TT.after(s)) | 保证提交时真实时间 \> s | 先提交的事务 s 可能大于后来事务的 s |
| smax 追踪 | 保证同 Leader 内单调性 | 相同时间戳 |
| Leader Lease 不重叠 | 保证跨 Leader 切换时单调性 | 时间戳回退 |
| Safe Time (tsafe) | 控制快照读安全性 | 读到不一致的快照 |
| 多版本存储 | 支持无锁快照读 | 只读事务需要加锁 |

## 10.3 代价与收益

代价：硬件成本（GPS \+ 原子钟）、写延迟增加 ~2ε ≈ 8ms（大部分被 Paxos 掩盖）。

收益：外部一致性（最强一致性）、无锁只读事务、全局一致快照、任意副本可读。
