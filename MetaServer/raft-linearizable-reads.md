# Raft 线性一致性读的实现方案

## 问题背景

在 Raft 共识协议中，写入天然具有线性一致性——每次写入都经过 Leader → 多数派 Follower 的日志复制流程，committed 的写入在所有节点上顺序一致。

但**读取并非天然线性一致**。如果 Leader 直接从本地状态机读取数据，可能出现以下场景：

```
Leader A (被网络隔离)               Follower B, C
  |                                    |
  |  ~~~ 网络隔离 ~~~                  |
  |                                    | election_timeout 到期
  | (不知道自己被废黜)                 | B 成为新 Leader (term=2)
  |                                    |
  |                                    | Client 写 X=2 → Leader B → committed
  |                                    |
  | Client 读 X → Leader A            |
  | A 本地读: X=1 (旧值)              |
  | 返回 X=1                          |
  |                                    |
  |  ← 线性一致性被打破 →             |
```

**根本原因**：Leader 在被隔离后不知道自己已经不是 Leader，写入靠多数派 ACK 天然被阻塞，但读只访问本地数据，缺少天然的 fencing 机制。

---

## 方案一：Log Read（日志读）

### 原理

将读请求当作一条空日志（no-op），走完整的 Raft 共识流程。

### 流程

```
Client → Leader: read(X)
Leader: 将读请求封装为 no-op log entry
Leader → Followers: AppendEntries(no-op)
Followers → Leader: ACK
Leader: no-op committed (多数派确认)
Leader: 确认自己仍是 Leader (否则无法 commit)
Leader: 从状态机读取 X
Leader → Client: X=2
```

### 分析

| 维度 | 评价 |
|------|------|
| 线性一致性 | ✓ 严格保证（读被序列化到日志中） |
| 时钟依赖 | 无 |
| 延迟 | 高（一轮共识 = 多次网络往返 + 磁盘写入） |
| 吞吐 | 低（读写竞争同一条日志） |
| 适用场景 | 极端一致性要求，低读负载 |

---

## 方案二：ReadIndex

### 原理

Leader 不写日志，只发一轮心跳确认自己仍是 Leader，然后等状态机 apply 到足够新的位置后读取。

### 流程

```
Client → Leader: read(X)

Leader:
  ① 记录当前 commitIndex (readIndex = commitIndex)
  ② 发一轮心跳给所有 Followers
  ③ 等待多数派 ACK → 确认自己仍是 Leader
  ④ 等待状态机 apply 到 readIndex
  ⑤ 从状态机读取 X

Leader → Client: X=2
```

### 前提：新 Leader 上任时的 no-op

新当选的 Leader 日志中包含所有已 committed 条目（选举保证），但它的 **commitIndex 可能落后于真实的全局提交点**——因为 commit 通知是异步的，Follower 可能还没来得及收到前任 Leader 的最新 commitIndex。

```
Term 1: Leader A commit 了 entry 1~10

  Follower B:
    日志: [1] [2] ... [10]   ← 都收到了
    commitIndex = 8           ← 只被通知到 8
    状态机: apply 了 1~8

Term 2: A 宕机，B 当选 Leader
    B 的 commitIndex = 8，但 entry 9-10 实际已被前任 committed
    ∴ B 不知道 9-10 是 committed 的
```

Raft 要求新 Leader 上任后立即提交一条**当前 term 的 no-op 日志**。no-op 被多数派 ACK 后，之前所有条目的 commit 状态随之确定，commitIndex 推进到正确位置。**这是 ReadIndex 和 Leader Lease 正确工作的前提条件。**（Log Read 不需要额外 no-op，因为读请求本身作为日志条目走共识流程，天然起到 no-op 的作用。）

### 为什么需要步骤 ②③

即使 no-op 已提交、commitIndex 正确，Leader 后续仍可能被废黜而不自知：

```
Term 2: B 是 Leader，commitIndex 正确

  ~~~ B 被网络隔离 ~~~

Term 3: C 当选新 Leader，提交了新写入
  Client 已看到 C 提交的结果

  B（不知道自己被废黜）收到读请求
  B 从本地状态机读 → 返回过时数据 → 线性一致性违反
```

心跳确认解决这个问题：Leader 在每次处理读请求前发一轮心跳，收到多数派 ACK 后才确认自己仍是合法 Leader。如果已被废黜（其他节点已有更高 term），心跳会失败，读请求被拒绝。

### 为什么需要步骤 ④

Leader 的状态机可能还没有 apply 到 commitIndex。如果直接读，可能读到过时的状态。必须等状态机 apply 到 readIndex 后才能读取。

### 分析

| 维度 | 评价 |
|------|------|
| 线性一致性 | ✓ 严格保证 |
| 时钟依赖 | 无 |
| 延迟 | 中（一轮心跳，无磁盘写入） |
| 吞吐 | 较好（不占用日志通道） |
| 适用场景 | 需要线性一致性但希望比 Log Read 快 |

---

## 方案三：Leader Lease（领导者租约）

### 原理

Leader 利用一个时间推理：如果 Follower 刚收到我的心跳，它们在 election_timeout 内不会发起选举，因此在这段时间内不可能有新 Leader，我可以安全地本地读。

> **前提**：与 ReadIndex 相同，新 Leader 上任后必须先提交一条当前 term 的 no-op 日志，确保 commitIndex 正确。Lease 只是替代了 ReadIndex 中"每次读前发心跳确认 Leadership"的步骤，不能替代 no-op 解决的 commitIndex 落后问题。（Log Read 不需要额外 no-op，因为读请求本身作为日志条目走共识，天然起到 no-op 的作用。）

### 流程

```
Leader 定期发送心跳:
  ① Leader 发送 Heartbeat 给所有 Followers
  ② Followers ACK
  ③ Leader 收到多数派 ACK
  ④ Leader 设置: lease_deadline = now + lease_duration
     其中 lease_duration < election_timeout

处理读请求:
  Client → Leader: read(X)
  Leader:
    if (mono_now < lease_deadline) {
      直接从状态机读取 X → 返回    // 零网络往返
    } else {
      降级为 ReadIndex 或拒绝
    }
```

### 安全性不变式

```
lease_duration < election_timeout

时间线:
  T=0: Leader 收到多数派 ACK, Follower 重置 election 计时器
       Leader: lease_deadline = T + lease_duration
       Follower: election_deadline = T + election_timeout

  T=lease_duration: Leader Lease 过期 → 停止本地读
  T=election_timeout: Follower 发起选举 → 可能产生新 Leader

  ∴ Leader 停止读 一定早于 新 Leader 产生
  ∴ 不存在两个 Leader 同时服务读的时刻
```

### 时钟假设

Leader 和 Follower 的单调时钟从**同一个事件**（心跳 ACK）开始计时，两者只是阈值不同。安全性依赖：

1. 两端的**单调时钟速率近似**（bounded clock drift）
2. `lease_duration` 严格小于 `election_timeout`

不依赖 NTP、不依赖系统时钟（wall clock），只依赖单调时钟的速率假设。

### 分析

| 维度 | 评价 |
|------|------|
| 线性一致性 | ✓ 在时钟假设下保证 |
| 时钟依赖 | 有（单调时钟速率需近似） |
| 延迟 | 低（零网络往返） |
| 吞吐 | 高 |
| 适用场景 | 高性能读，可接受时钟假设 |

---

## 三种 Leader 读方案对比

```
                  Log Read        ReadIndex       Leader Lease
              ┌──────────────┬──────────────┬──────────────────┐
一致性保证     │ 严格线性一致  │ 严格线性一致  │ 依赖时钟假设      │
时钟依赖       │ 无            │ 无            │ 单调时钟速率      │
读延迟         │ 一轮共识      │ 一轮心跳      │ 零网络往返        │
磁盘 I/O      │ 有 (写日志)   │ 无            │ 无                │
吞吐           │ 低            │ 中            │ 高                │
              └──────────────┴──────────────┴──────────────────┘
```

---

## Follower 读（从节点读）

### 动机

所有读都发往 Leader 会使 Leader 成为瓶颈。如果允许从 Follower 读，可以水平扩展读吞吐。但 Follower 的数据可能落后于 Leader。

### 方案：ReadIndex 转发

```
Client → Follower: read(X)
Follower → Leader: 请求当前 readIndex
Leader:
  ① 确认自己仍是 Leader（发心跳 / 检查 Lease）
  ② 返回 readIndex = commitIndex
Leader → Follower: readIndex
Follower:
  ① 等待本地状态机 apply 到 readIndex
  ② 从本地状态机读取 X
Follower → Client: X=2
```

**一致性**：严格线性一致。Follower 每次读都向 Leader 获取实时的 commitIndex，保证读到的数据至少和 Leader 在处理时刻一样新。

**延迟**：一轮 Follower → Leader 的网络往返（获取 readIndex），加上可能的 apply 等待。

### 为什么 Follower 不能用 Lease 省掉这轮转发

一个自然的想法是：Leader 通过心跳把 readIndex 和 Lease 信息发给 Follower，Follower 在 Lease 有效期内直接本地读，省掉每次向 Leader 的转发。但这**不满足线性一致性**：

```
T=0: Leader 发心跳，携带 readIndex=100 给 Follower
T=1: Client 向 Leader 写入 X=2 → committed → commitIndex=101 → Client 收到 ACK
T=2: Client 向 Follower 读 X
     Follower 的 readIndex 仍是 100（下一次心跳还没到）
     → 返回 X=1（旧值）
     → 写在 T=1 完成，读在 T=2 发起，但读没看到写 → 线性一致性违反
```

根本原因：Follower 的 readIndex 是上一次心跳时 Leader 的 commitIndex 快照，两次心跳之间 Leader 提交的新写入对 Follower 不可见。Lease 只保证了"没有新 Leader"，但没有解决"当前 Leader 又提交了新写入"的问题。因此 Follower 要实现线性一致性读，每次都必须向 Leader 获取实时的 readIndex。

---

## Leader 主动退出机制（Step Down / CheckQuorum）

### 标准 Raft 的问题

标准 Raft 中，Leader **没有规划的退出时间**。Leader 只在收到更高 term 的消息时被动降级。如果被网络隔离，Leader 完全不知道自己已经被废黜，会一直认为自己是 Leader。

- **写入**：安全（凑不齐多数派 ACK，写入无法 commit）
- **读取**：不安全（本地读可能返回旧数据）

### braft / etcd 的 Step Down 机制

工业级 Raft 实现增加了**主动退出**机制：

```
Leader 定期执行 check_dead_nodes():
  alive_count = 1  // 自己
  for each follower:
    if (mono_now - follower.last_response_time < election_timeout):
      alive_count++
  
  if (alive_count < majority):
    step_down()  // 主动降为 Follower，停止一切服务
```

### Step Down 与 Lease 的关系

在 braft 等实现中，Step Down 和 Lease 读**共享同一个锚定事件和时钟**：

```
            同一个事件: last_quorum_ack (最后一次多数派 ACK)
            同一个时钟: Leader 本地 mono_clock
                    │
        ┌───────────┼──────────────┐
        │           │              │
        ▼           │              ▼
  Lease 读判断:     │      Step down 判断:
  now < last_ack    │      now > last_ack
   + lease_duration │       + election_timeout
        │           │              │
        ▼           │              ▼
  "可以本地读"      │      "主动降为 Follower"

  ─────────────────────────────────────→ 时间
  last_ack  lease到期    step_down
       │←  lease  →│←  gap  →│
```

这构成了一个**单机闭环**：
- 一台机器（Leader）
- 一个时钟（本地 mono_clock）
- 一个事件源（多数派 ACK）
- 两个阈值（lease_duration < election_timeout）

安全性完全由本地算术关系保证，不涉及跨机器时钟比较。

---

## 总结

### 读一致性的本质权衡

```
延迟 ←───────────────────────────────→ 一致性强度
                                      
  Leader Lease   ReadIndex   Log Read
  (零往返)       (一轮心跳)  (一轮共识)
  依赖时钟       不依赖      不依赖
```

### 各方案适用场景

| 方案 | 适用场景 |
|------|---------|
| Log Read | 极端一致性要求，读负载低 |
| ReadIndex | 需要线性一致性，不信任时钟 |
| Leader Lease | 高性能读，可接受时钟假设 |
| Follower ReadIndex | 需要线性一致性 + 读扩展 |

### 核心原则

- **写入安全性**由多数派 ACK 天然保证，不需要额外机制
- **读取安全性**需要额外机制，因为读只访问本地数据，缺少天然 fencing
- **Lease 的本质**是用时钟假设换取读性能——将"每次读都确认身份"变为"一段时间内免确认"
- **Lease 的安全性**取决于 Lease 过期和故障检测是否锚定在同一事件和同一时钟上