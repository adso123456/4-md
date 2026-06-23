# Raft 算法详解

> 基于《Ch3 Crash Fault-Tolerant Consensus》文档 3.6 节整理，格式与 Paxos 详解对齐

---

## 一、Raft 解决什么问题？

**一句话：在部分同步网络、节点可能崩溃的环境下，让多个节点对一串命令的执行顺序达成一致，且保证每个状态机执行完全相同的命令序列。**

Raft 的核心目标是实现**复制日志**——集群中的每个服务器执行相同顺序的相同命令，从而在任意多数派存活时系统继续运行。

与 Paxos 的定位对比：

| | Paxos | Raft |
|------|------|------|
| 目标 | 对**一个值**达成共识 | 对**一串有序命令**达成共识（复制日志） |
| 网络模型 | 异步（asynchronous） | 部分同步（partial synchronous） |
| 设计理念 | 证明驱动（推导约束链 P1→P2-c） | 可理解性优先（拆成可独立理解的模块） |
| 活性保证 | 不保证（可能活锁） | 保证（Leader 机制 + 随机超时） |

约束条件：

| 类型 | 条件 |
|------|------|
| Safety — election safety | 每个 term 最多选出一个 Leader |
| Safety — leader append-only | Leader 只追加、不覆盖、不删除自己的日志 |
| Safety — log matching | 两个日志中相同 index + term 的条目，其命令相同，且之前所有条目都相同 |
| Safety — leader completeness | 某 term 已提交的日志条目，必然出现在所有更高 term 的 Leader 日志中 |
| Safety — state machine safety | 同一 index 处，不会有不同节点执行不同的命令 |
| Liveness | 只要多数派存活、网络最终同步，最终会选出 Leader 并推进日志 |

---

## 二、核心角色与多数派机制

| 角色 | 职责 |
|------|------|
| **Leader**（领导者） | 处理客户端请求、管理日志复制、发送心跳维持权威 |
| **Follower**（跟随者） | 完全被动，只响应 Leader 和 Candidate 的 RPC |
| **Candidate**（候选者） | 发起选举、争取成为新 Leader |

节点状态转换：

```
Follower
  │
  │ 选举超时 / 发现更高 term
  ▼
Candidate ─────────────────────────┐
  │                                │
  │ 收到多数派投票                   │ 选举超时 / 分裂投票（自循环重试）
  ▼                                │
Leader                              │
  │                                │
  │ 发现更高 term 的 Leader          │
  ▼                                │
Follower ◄─────────────────────────┘
  ▲
  │ 收到合法 Leader 心跳
  │
Candidate
```

简化记忆：**Follower → Candidate → Leader → Follower**，超时推动，投票晋升，心跳降级。

**根本机制**：一切决策（选举 + 提交）靠多数派（> n/2）批准。Leader 向多数派复制日志后即视为已提交。选举靠多数派投票。任意两个多数派必有交集——安全性由此保证。

---

## 三、为什么 Raft 简单？—— 拆解思路

### 3.1 Paxos 难在哪

Paxos 难 3 个原因：

1. 单值共识 → 多值共识之间需要 Multi-Paxos，概念跳跃大
2. 没有标准实现——论文的算法和工程系统差距巨大
3. 活锁问题——论文只说"选 Leader"，但没说怎么选

### 3.2 Raft 的拆解

Raft 把共识问题拆成 3 个**可独立理解**的子问题：

```
Raft = Leader 选举 + 日志复制 + 安全性约束
```

- **Leader 选举**：选出一个唯一 Leader，由它全权管理日志
- **日志复制**：Leader 接收命令、追加到自己的日志、同步给 Follower
- **安全性约束**：确保新 Leader 包含所有已提交条目，且只追加不覆盖

这种拆解是 Raft 的核心贡献——不是发明了新算法，而是把已知的正确思路组合成可理解的系统。

---

## 四、核心概念

### 4.1 Term（任期）

```
Term 1         Term 2          Term 3         Term 4          Term 5
|              |               |              |               |
选举           正常操作          选举           正常操作          选举
选出Leader     有Leader        分裂投票        有Leader         ...
               (持续心跳)      无人胜出        (持续心跳)
```

> 一个 Term 内要么有唯一的 Leader 在正常工作，要么没有任何 Leader（选举失败）。

| 属性 | 说明 |
|------|------|
| Term 是什么 | 整数编号，单调递增，作为逻辑时钟 |
| 何时递增 | 每次发起新选举时 +1 |
| 一个 Term 的结果 | 要么选出一个 Leader（正常操作），要么没选出（该 term 称为选举失败） |
| 核心作用 | **识别过时信息**——低 term 的消息一律被忽略 |

**类比**：Term 就是 Paxos 的 epoch。区别在于，Raft 的 Term 是有结构的（选举期 + 操作期），Paxos 的 epoch 只是提案编号。

### 4.2 两种 RPC

| RPC | 发起者 | 用途 |
|-----|--------|------|
| **RequestVote** | Candidate | 请求投票以成为 Leader |
| **AppendEntries** | Leader | 日志复制 + 心跳（Entries 为空时即心跳） |
| InstallSnapshot | Leader | 日志压缩时传输快照（属于优化，非核心） |

心跳机制：Leader 周期性发送空的 AppendEntries（不带日志条目），Follower 收到后重置选举计时器。**如果 Follower 一段时间没收到心跳，就认为 Leader 挂了，转为 Candidate 发起选举。**

### 4.3 随机选举超时

| 机制 | 说明 |
|------|------|
| 随机范围 | 100~500ms（典型值） |
| 谁维护 | 每个 Follower 独立维护 |
| 目的 | **避免分裂投票**——最先超时的节点发起选举，通常能顺利当选 |

**什么是分裂投票？**

同一 term 内，多个 Candidate 几乎同时发起选举，每个 Follower 只能投一票（每 term），票数被分散，导致没有人获得多数派投票。结果：该 term 白白浪费，所有 Candidate 增大 term 各自重试。

```
term=2:  S1(Candidate) ──→ S3 投票给 S1
         S2(Candidate) ──→ S4 投票给 S2
         S5 还没投（或投给另一个）
结果: S1=2票, S2=2票, 都不到多数派3票 → 全部超时, term+1 重新选举
```

随机超时就是为了打破这种僵局——不同节点超时时间不同，最先超时的抢先发起选举，大概率独自拿到多数票。

约束：`broadcastTime << electionTimeout << MTBF`（平均 RPC 延迟 10~500ms，节点故障间隔几个月）

---

## 五、Leader 选举（Leader Election）

### 5.1 选举流程

Candidate 发起选举的完整处理逻辑：

```
on election timeout do
    currentTerm += 1                              // ① 进入新 term
    role = Candidate
    votedFor = self                                // ② 先投自己一票
    send RequestVote(currentTerm, self,            // ③ 并行向所有节点拉票
                     lastLogIndex, lastLogTerm)
    to all servers

    start election timer                           // ④ 等待结果
    while election timer not expired do
        if |votes| >= ⌊n/2⌋ + 1 then
            become Leader                          // ⑤ 胜出：立即发心跳
        end if
        if received AppendEntries
           from valid leader then
            role = Follower                        // ⑥ 发现已有合法 Leader，退选
            leaderId = sender
        end if
    end while
    // ⑦ 超时也没赢 → 回到开头，currentTerm++ 重试
end on
```

逐行说明：

| 行 | 逻辑 | 说明 |
|----|------|------|
| ① | `currentTerm += 1` | 每次选举都在新 term 中进行，term 单调递增 |
| ② | 投自己一票 | Candidate 先投自己，这是 Raft 保证进度的关键——即使只有一个 Candidate，它也能拿到至少 1 票 |
| ③ | 并行发 RequestVote | 向所有节点（包括自己）发送拉票请求，不会阻塞等单个节点响应 |
| ④ | 启动选举计时器 | 超时时间内没出结果就重试。注意这个计时器独立于心跳超时——心跳超时触发选举，选举超时限制选举本身 |
| ⑤ | 收到多数派投票 | 立即成为 Leader，发心跳确立权威，阻止其他节点发起新选举 |
| ⑥ | 发现已有 Leader | 如果收到合法的 AppendEntries（心跳），说明已经有节点成了 Leader，主动退选回到 Follower |
| ⑦ | 超时重试 | 选举超时未获胜（通常是分裂投票）→ `currentTerm++` 重试。随机超时保证重试时间错开 |

> **变量说明**：
> - `lastLogIndex` / `lastLogTerm`：Candidate 自己日志中最后一条 entry 的 index 和 term。发送给投票者，用于判断 Candidate 日志是否足够新
> - `votedFor`：当前 term 中本节点投给了哪个 Candidate（null 表示还没投）。RPC 重传时允许投给同一个人，防止因丢包导致误拒

### 5.2 投票规则（每个 Term 只能投一票）

Follower 收到 RequestVote 时的处理逻辑：

```
on receiving RequestVote(term, candidateId, lastLogIndex, lastLogTerm) do
    if term > currentTerm then                     // ① 发现更高 term
        currentTerm = term                          //    立即更新自己的 term
        role = Follower                             //    若是 Candidate/Leader 则退位
    end if

    if term < currentTerm then                     // ② 过时请求
        reply (currentTerm, granted=false)          //    直接拒绝
        return
    end if

    if votedFor == null                            // ③ 还没投过票
       or votedFor == candidateId then              //    或已经投的就是它（重试）
        myLastTerm  = log[last].term
        myLastIndex = log[last].index
        if lastLogTerm > myLastTerm                 // ④ 选举限制：日志新旧比较
           or (lastLogTerm == myLastTerm
               and lastLogIndex >= myLastIndex) then
            votedFor = candidateId                  //    日志够新 → 投票
            reply (currentTerm, granted=true)
        else
            reply (currentTerm, granted=false)      //    日志不够新 → 拒绝
        end if
    else
        reply (currentTerm, granted=false)          // ⑤ 已经投给别人了
    end if
end on
```

逐行说明：

| 行 | 逻辑 | 说明 |
|----|------|------|
| ① | `term > currentTerm` | 发现更高 term 立刻更新自己的 term 并退位。这保证了整个集群的 term 单调推进，旧 term 的任何操作都被忽略 |
| ② | `term < currentTerm` | 来自旧 term 的请求一律拒绝。这是 Raft 用 term 实现"识别过时信息"的关键 |
| ③ | `votedFor == null or votedFor == candidateId` | 每 term 最多投一票。允许投给已投过的 candidate 是为了处理 RPC 重传场景 |
| ④ | 日志新旧比较 | **选举限制的核心**：Candidate 的日志不能比投票者旧。比较 `(lastLogTerm, lastLogIndex)`，term 大者优先，同 term 比 index。保证新 Leader 必然包含所有已提交条目 |
| ⑤ | 其他情况 | 当前 term 已经投给了别人，拒绝 |

**日志新旧判断规则**（`>` 表示"更新"）：

```
candidate.lastLogTerm > voter.lastLogTerm   → candidate 更新 ✓
candidate.lastLogTerm < voter.lastLogTerm   → candidate 更旧 ✗
同 term 时: candidate.lastLogIndex >= voter.lastLogIndex → candidate 更新 ✓
```

---

## 六、日志复制（Log Replication）

### 6.1 日志结构

```
log index:  1      2      3      4      5      6      7      8
term:       1      1      1      2      3      3      3      3
command:   add    cmp    ret    mov    jmp    div    shl    sub
                                       ↑
                              index 5 处已提交 (committed)
```

Leader 日志最全，Follower 可能落后。日志条目 = (term, index, command)，存储在持久化介质上，节点崩溃恢复后仍可读取。

### 6.2 日志复制流程

一次完整的日志复制涉及 Leader 和 Follower 两端协作。先看涉及的关键参数：

| 参数 | 含义 | 谁设置 |
|------|------|--------|
| `prevLogIndex` | 本次发送的 entries 之前那条日志的 index | Leader，= `nextIndex[follower] - 1` |
| `prevLogTerm` | 同上那条日志的 term | Leader，= `log[prevLogIndex].term` |
| `entries` | 要追加的日志条目列表（可为空，即心跳） | Leader |
| `leaderCommit` | Leader 当前已提交的最大 index | Leader，= `commitIndex` |
| `nextIndex[follower]` | 该 follower 下一条该发的 index | Leader 维护，初始 = `log.length` |
| `matchIndex[follower]` | 该 follower 已确认复制的最大 index | Leader 维护，初始 = 0 |

**Leader 端：**

```
on receiving client command do
    append (currentTerm, command) to log        // ① 先追加到本地日志
    for each follower do
        ReplicateLog(currentTerm, follower)      // ② 并行发送给所有 Follower
    end for
end on

function ReplicateLog(term, follower) do
    prevLogIndex = nextIndex[follower] - 1       // ③ 计算一致性检查锚点
    prevLogTerm = log[prevLogIndex].term
    entries = log[nextIndex[follower] .. -1]     // 发送该 follower 缺失的所有条目
    send AppendEntries(term, leaderId,
                       prevLogIndex, prevLogTerm,
                       entries, commitIndex)
    to follower
end function

on receiving AppendEntries ACK from majority do
    commitIndex = N                              // ④ 多数派确认后提交
    apply committed entries to state machine     // ⑤ 应用到状态机
    reply success to client                      // ⑥ 回复客户端
end on
```

**Follower 端：**

```
on receiving AppendEntries(term, leaderId,
                           prevLogIndex, prevLogTerm,
                           entries, leaderCommit) do
    if term < currentTerm then                   // ⑦ 拒绝旧 term 的请求
        reply false
        return
    end if

    if log[prevLogIndex].term != prevLogTerm then // ⑧ 一致性检查
        reply false                              //    失败 → Leader 会回退重试
        return
    end if

    // 追加新条目，自动覆盖冲突位置之后的内容
    log = log[0 .. prevLogIndex] + entries        // ⑨ 截断冲突部分 + 追加
    reply true

    if leaderCommit > commitIndex then           // ⑩ 跟随 Leader 的提交进度
        commitIndex = min(leaderCommit, log.length - 1)
        apply committed entries to state machine
    end if
end on
```

逐行说明：

| 行 | 逻辑 | 说明 |
|----|------|------|
| ① | Leader 先追加到本地 | Leader 的日志是最权威的副本，必须先写入自己再复制 |
| ② | 并行发送 | 向所有 Follower 同时发 AppendEntries，不等单个响应。心跳也会触发同一个 `ReplicateLog`，只不过 entries 为空 |
| ③ | 计算锚点 | `prevLogIndex` = 该 Follower 已确认的前一条 index。从这里开始的一致性检查保证了日志匹配属性 |
| ④ | 多数派确认 | Leader 统计每个 Follower 的 `matchIndex`，当某 index N 被多数派达到且 `log[N].term == currentTerm` 时，N 被提交 |
| ⑤ | 应用到状态机 | 状态机按 index 顺序执行已提交的命令，保证所有节点执行相同序列 |
| ⑥ | 回复客户端 | 只有 Leader 能回复——Follower 不知道提交进度，会把读请求重定向到 Leader |
| ⑦ | 拒绝旧 term | Follower 的第一道防线：来自旧 Leader 的残留消息直接忽略 |
| ⑧ | 一致性检查 | Follower 的第二道防线：`myLog[prevLogIndex].term` 必须和 Leader 说的一致，否则说明日志有分歧 |
| ⑨ | 截断 + 追加 | 匹配成功后，Follower 丢弃 `prevLogIndex` 之后的所有条目，用 Leader 的覆盖。这就是"Leader 强制同步"的核心 |
| ⑩ | 跟随提交 | Follower 不自己决定提交——它从 `leaderCommit` 字段得知 Leader 的提交进度，同步应用到自己的状态机 |

一致性检查 + 逐步回退保证了**日志匹配属性**：相同 index+term 的条目，前面所有条目都相同。

### 6.3 提交规则

**只有当前 Term 的条目被多数派复制后，才能提交。** 同时，提交当前 term 的条目会间接提交之前所有未提交的条目。

```
entry committed = (entry.term == currentTerm) ∧ (存储于多数派节点)
```

为什么必须是当前 term？考虑：旧 term=T 的某条目已存储在多数派节点上，但还没被提交（term=T 的 Leader 在提交前崩溃了）。新 Leader（term=U > T）当选后，如果直接按多数派存储来提交该旧条目，可能出错——因为新 Leader 的日志可能在同样 index 处有一个不同 term 的条目，提交旧条目会导致状态机分歧。这也就是 Raft 论文 Figure 8 所示的经典安全性问题（论文 Section 5.4.3）。

---

## 七、安全性分析（Safety Properties）

### 7.1 五个安全性属性

| # | 属性 | 含义 |
|---|------|------|
| 1 | Election Safety | 每个 term 最多一个 Leader（因为每个节点每 term 只投一票 + 多数派规则） |
| 2 | Leader Append-Only | Leader 只追加、不覆盖、不删除日志 |
| 3 | Log Matching | 相同 index+term → 相同命令 + 之前完全一致（由 AppendEntries 一致性检查保证） |
| 4 | Leader Completeness | 已提交的条目必出现在所有后续 Leader 的日志中（由选举限制保证） |
| 5 | State Machine Safety | 同一 index 不会执行不同命令（由属性 3 + 4 推导） |

### 7.2 选举限制（Election Restriction）的证明

这是 Raft 安全性最核心的约束。

**规则**：Candidate 的日志必须至少和多数派中任意一个节点一样"新"，才能当选。

**为什么必要？** 考虑一个已提交的日志条目 E（在 term T 被提交）。E 必然存储在某个多数派 S_commit 中。后续选举时，Candidate 必须获得某个多数派 S_vote 的投票。因为任意两个多数派必有交集，S_vote 中至少有一个节点包含 E。

**选举限制**要求 Candidate 的日志不能比这个节点旧 → Candidate 一定包含 E → 新 Leader 一定包含 E。证毕。

### 7.3 与 Paxos 安全性约束的对应

| Paxos | Raft |
|-------|------|
| P2-c：新提案必须先打听旧提案的值 | 选举限制：新 Leader 的日志必须包含所有已提交条目 |
| 通过 Phase 1 的 Promise 发现已接受的值 | 通过 AppendEntries 一致性检查发现并修复日志差异 |
| Acceptor 的 lastPromisedEpoch 阻止旧提案 | Term 机制：低 term 的消息一律被忽略 |

---

## 八、算法伪代码（完整参考实现）

> 以下为 Raft 的完整伪代码，按事件驱动的方式组织。第五、六节已对选举和日志复制做了详细讲解，本节集中呈现完整实现，并在每个小节后补充关键设计考量。

### 8.1 初始化

```
on initialization do
    currentTerm = 0
    votedFor = null
    log = []
    commitIndex = 0
    role = Follower
    leaderId = null
end on

on recovery from crash do
    role = Follower
    votedFor = null
    leaderId = null
    // currentTerm, log 从持久化存储恢复
end on
```

变量说明：

| 变量 | 持久化 | 含义 |
|------|--------|------|
| `currentTerm` | 是 | 节点已知的最新 term，重启后保留 |
| `votedFor` | 是 | 当前 term 投给了谁（null = 还没投）。恢复时清空，因为旧 term 的投票已无效 |
| `log[]` | 是 | 日志条目数组，按 index 递增。崩溃恢复后必须可读 |
| `commitIndex` | 否 | 已提交的最大 index，重启后通过 Leader 心跳重新获知 |
| `role` | 否 | 当前角色：Follower / Candidate / Leader |
| `leaderId` | 否 | 当前已知的 Leader ID（Follower 记录，Leader 记录自己，Candidate 为 null） |
| `nextIndex[f]` | 否 | （Leader 专用）该 follower 下一条要发的 index，初始 = `log.length` |
| `matchIndex[f]` | 否 | （Leader 专用）该 follower 已确认复制的最大 index，初始 = 0 |

> **持久化 vs 易失**：`currentTerm`、`votedFor`、`log` 必须持久化——节点崩溃重启后不能丢失，否则可能破坏安全性。其余变量可从已有状态推导。

### 8.2 Leader 选举（发起方）

```
on election timeout do
    currentTerm += 1
    role = Candidate
    votedFor = self
    send RequestVote(currentTerm, self, lastLogIndex, lastLogTerm) to all servers

    start election timer
    while election timer not expired do
        if |votes| >= ⌊n/2⌋ + 1 then
            become Leader                     // → 跳转到 8.4
        end if
        if received AppendEntries from valid leader then
            role = Follower                   // 发现有合法 Leader，退选
            leaderId = sender                 // → 跳转到 8.6
        end if
    end while
    // 超时后自动回到本函数开头，currentTerm++ 重试
end on
```

> **设计考量**：选举超时计时器独立于心跳计时器。心跳超时触发选举（"Leader 可能挂了"），选举超时限制选举本身（"这次选举可能没结果"）。两者分离避免了选举卡死。`while` 循环内同时监听投票结果和外来心跳——收到心跳说明别人已当选，立即退选，防止集群被拖入不必要的选举。

### 8.3 收到投票请求（接收方）

```
on receiving RequestVote(term, candidateId, lastLogIndex, lastLogTerm) do
    if term > currentTerm then
        currentTerm = term
        role = Follower
    end if

    if term < currentTerm then
        reply (currentTerm, granted=false)
        return
    end if

    if votedFor == null or votedFor == candidateId then
        // 选举限制：检查候选人日志是否足够新
        myLastTerm  = log[last].term
        myLastIndex = log[last].index
        if lastLogTerm > myLastTerm
           or (lastLogTerm == myLastTerm and lastLogIndex >= myLastIndex) then
            votedFor = candidateId
            reply (currentTerm, granted=true)
        else
            reply (currentTerm, granted=false)
        end if
    else
        reply (currentTerm, granted=false)  // 已经投过别人了
    end if
end on
```

> **设计考量**：8.3 的核心是选举限制（日志新旧比较），详见 5.2 节。此处伪代码展示了完整的投票逻辑——三层 if 分别处理"发现更高 term 退位"、"拒绝旧 term"、"根据日志新旧决定是否投票"。注意 `votedFor` 必须持久化，否则节点崩溃重启后可能在同一个 term 投两次票。

### 8.4 Leader 收集投票

```
on receiving VoteResponse(term, granted) do
    if term > currentTerm then
        currentTerm = term
        role = Follower
        return
    end if

    if role != Candidate or term != currentTerm then
        return                              // 旧 term 的回复，忽略
    end if

    if granted then
        votes.add(sender)
        if |votes| >= ⌊n/2⌋ + 1 then
            role = Leader
            leaderId = self
            cancel election timer

            for each follower do
                nextIndex[follower] = log.length
                matchIndex[follower] = 0
            end for

            send AppendEntries(currentTerm, leaderId,
                               prevLogIndex, prevLogTerm,
                               entries=[],
                               leaderCommit=commitIndex)
            to all followers                // 立即发心跳确立权威
        end if
    end if
end on
```

> **设计考量**：成为 Leader 后第一件事是发空 AppendEntries（心跳），目的有二：(1) 通知所有节点新 Leader 已产生，阻止新选举；(2) 在心跳的 ACK 中收集 `matchIndex`，为后续提交提供基准。`nextIndex[follower]` 初始化为 `log.length` 是乐观假设——认为 Follower 日志与 Leader 完全一致，不一致时再回退修正（见 8.8）。

### 8.5 日志复制（Leader 端）

```
on receiving client command do
    append (currentTerm, command) to log
    for each follower do
        ReplicateLog(currentTerm, follower)
    end for
end on

function ReplicateLog(term, follower) do
    prevLogIndex = nextIndex[follower] - 1
    prevLogTerm = log[prevLogIndex].term
    entries = log[nextIndex[follower] .. log.length-1]
    send LogRequest(term, leaderId, prevLogIndex, prevLogTerm,
                    entries, commitIndex) to follower
end function
```

> Leader 在收到客户端命令时立即尝试复制（心跳周期内也会触发），心跳是 AppendEntries(entries=[]) 的特殊情况。

### 8.6 Follower 接收日志

```
on receiving LogRequest(term, leaderId, prevLogIndex, prevLogTerm, entries, leaderCommit) do
    if term > currentTerm then
        currentTerm = term
        role = Follower
    end if

    if term < currentTerm then
        reply (currentTerm, success=false)       // 拒绝旧 term
        return
    end if

    // term == currentTerm：接受此 Leader
    role = Follower                               // Candidate 发现合法 Leader，退选
    leaderId = msg.leaderId                       // 记录当前 Leader

    // 一致性检查
    if prevLogIndex >= 0
       and (log.length <= prevLogIndex or log[prevLogIndex].term != prevLogTerm) then
        reply (currentTerm, nextIndex=0, success=false)
    else
        AppendEntries(prevLogIndex, entries)
        reply (currentTerm, nextIndex=log.length, success=true)
    end if
end on
```

> **设计考量**：Follower 在收到 `LogRequest` 时做了两件关键的事：(1) 如果 `term > currentTerm`，说明自己落后了，更新 term 并退为 Follower——这是节点间 term 同步的唯一途径；(2) 一致性检查失败时返回 false，触发 Leader 的回退机制。注意 Follower 不会主动 apply 日志——它必须等 `leaderCommit > commitIndex` 才提交（见 ⑩），这保证了提交安全性。

### 8.7 AppendEntries 内部操作

```
function AppendEntries(prevLogIndex, entries) do
    // 从 prevLogIndex+1 开始，逐条检查
    for i = 0 to entries.length - 1 do
        targetIndex = prevLogIndex + 1 + i
        if targetIndex < log.length then
            if log[targetIndex].term != entries[i].term then
                // 冲突！删除此位置及之后的所有条目
                log = log[0 .. targetIndex-1]
            end if
        end if
        if targetIndex >= log.length then
            log.append(entries[i])   // 追加
        end if
    end for
end function
```

**关键点**：如果 Follower 在某个 index 处 term 不匹配，它删除该位置之后的所有条目，然后用 Leader 的条目覆盖。这就是"Leader 强制 Follower 的日志与自己一致"的机制。注意删除后该 index 之后的已提交条目也会被丢弃——但这是安全的，因为已提交的条目必然也存储在 Leader 及其他多数派节点上，下次心跳会补回来。

### 8.8 Leader 接收日志确认

```
on receiving LogResponse(term, follower, nextIndex, success) do
    if term > currentTerm then
        currentTerm = term
        role = Follower
        return
    end if

    if role != Leader or term != currentTerm then
        return                              // 旧 term 或已不是 Leader
    end if

    if success then
        matchIndex[follower] = nextIndex - 1
        nextIndex[follower] = nextIndex

        // 检查是否可以提交：找多数派都达到的最大 index
        for N = commitIndex+1 to log.length-1 do
            count = 1                       // 算上自己
            for each follower do
                if matchIndex[follower] >= N then
                    count += 1
                end if
            end for
            if count >= ⌊n/2⌋ + 1 and log[N].term == currentTerm then
                commitIndex = N
            end if
        end for
    else
        nextIndex[follower] -= 1            // 日志不一致，回退一步，下次心跳重发
    end if
end on
```

> **设计考量**：提交条件 `log[N].term == currentTerm` 是最容易被忽视的安全约束。它防止了 Raft 论文 Figure 8 的场景——旧 term 的条目即使已被多数派存储，也不能在新 term 提交，必须通过提交当前 term 的条目来间接提交。失败分支的 `nextIndex[follower] -= 1` 实现了"逐步回退"——每次只退一步，最坏情况退到 index 0，但实践中通常在常数步内找到一致的锚点。

### 8.9 Raft 属性清单

| # | 属性 |
|---|------|
| 1 | Term 单调递增，作为逻辑时钟 |
| 2 | 每个 Term 最多一个 Leader（election safety） |
| 3 | 每个 Follower 每 Term 只投一票 |
| 4 | Candidate 日志必须足够新才能当选（election restriction） |
| 5 | Leader 只追加不覆盖（append-only） |
| 6 | AppendEntries 携带 prevLogIndex/prevLogTerm 做一致性检查 |
| 7 | 一致性检查失败时 Leader 递减 nextIndex 重试 |
| 8 | 只有当前 term 的条目被多数派确认后才能提交 |
| 9 | 提交当前 term 条目会间接提交之前所有未提交的条目 |

---

## 九、运行示例（表格推演）

以下使用 5 个节点（S1~S5），多数派 = `⌊5/2⌋+1 = 3`。

### 9.1 正常 Leader 选举

**初始状态**：5 个节点，term=0，都是 Follower。

| 步骤 | 谁 | 动作 | 结果 |
|------|-----|------|------|
| 1 | S1 | 选举超时（随机值最小），term=1，变 Candidate，投自己 | S1: term=1, votedFor=S1 |
| 2 | S1 | 向 S2~S5 发 RequestVote(1, S1, ...) | — |
| 3 | S2 | term 1 > 0 ✓，还没投过票，S1 日志够新 ✓，投票 | S2: term=1, votedFor=S1 |
| 4 | S3 | 同上 | S3: term=1, votedFor=S1 |
| 5 | S4 | 同上 | S4: term=1, votedFor=S1 |
| 6 | S5 | 同上 | S5: term=1, votedFor=S1 |
| 7 | S1 | 收到 5 票（含自己），> 3 ✓ | **S1 成为 Leader** |
| 8 | S1 | 立即发 AppendEntries 心跳给 S2~S5 | Follower 重置选举计时器 |

### 9.2 日志复制正常流程

**初始状态**：S1 是 Leader，所有节点日志为空，commitIndex=0。

| 步骤 | 谁 | 动作 | 结果 |
|------|-----|------|------|
| 1 | Client | 向 S1 发 `add x,5` | — |
| 2 | S1 | 追加到本地 log[1] = (term=1, add x,5) | S1 log: [1: add] |
| 3 | S1 | 发 AppendEntries 给 S2~S5 | prevLogIndex=0 |
| 4 | S2~S5 | 一致性检查通过，追加到本地 log[1] | 所有节点 log: [1: add] |
| 5 | S2~S5 | 回复 ACK(true) | — |
| 6 | S1 | 收到 4 个 ACK，多数派（含自己 5/5）✓，term=currentTerm ✓ | commitIndex=1 |
| 7 | S1 | 应用到状态机，回复 Client | **写入完成** |
| 8 | S1 | 下次心跳携带 leaderCommit=1 | Follower 也 commit，应用到状态机 |

### 9.3 日志不一致的修复

**场景**：S1(Leader, term=4) 的日志比 S2(Follower) 更全。

```
S1 (Leader):   1:add  2:cmp  3:ret  4:mov  5:jmp  6:div  7:shl (term: 1,1,1,2,3,3,3)
S2 (Follower): 1:add  2:cmp  3:ret  4:mov  5:jmp (term: 1,1,1,2,2)
```

| 步骤 | 谁 | 动作 | 结果 |
|------|-----|------|------|
| 1 | S1 | 对 S2 发 AppendEntries(prevLogIndex=5, prevLogTerm=3, entries=[div,shl]) | — |
| 2 | S2 | 检查 log[5].term: 自己的是 2，Leader 说是 3 → **不一致** | 回复 false |
| 3 | S1 | nextIndex[S2] 从 7 减到 6 → 重发(prevLogIndex=4, prevLogTerm=2, entries=[jmp,div,shl]) | — |
| 4 | S2 | 检查 log[4].term: 2=2 ✓ → 通过 | — |
| 5 | S2 | 追加 entries: 覆盖 index 5(jmp, term=3 覆盖旧 term=2)，追加 6(div), 7(shl) | S2 与 S1 一致 |
| 6 | S2 | 回复 true | 复制完成 |

### 9.4 分裂投票与重试

**场景**：S1 和 S2 几乎同时选举超时，同时以 term=2 发起选举。

| 步骤 | 谁 | 动作 | 结果 |
|------|-----|------|------|
| 1 | S1 | 选举超时，term=2，变 Candidate，投自己 | S1: term=2, votedFor=S1 |
| 2 | S2 | 几乎同时超时，term=2，变 Candidate，投自己 | S2: term=2, votedFor=S2 |
| 3 | S3 | 收到 S1 的 RequestVote → 还没投 → 投 S1 | S3: term=2, votedFor=S1 |
| 4 | S4 | 收到 S2 的 RequestVote → 还没投 → 投 S2 | S4: term=2, votedFor=S2 |
| 5 | S5 | 收到 S1 的 RequestVote → 还没投 → 投 S1 | S5: term=2, votedFor=S1 |
| 6 | S1 | 只收到 S1+S3+S5 = 3 票 | 等等！3 票 = ⌊5/2⌋+1 = 3，按理应该赢？ |

别急——这里 S5 的情况更微妙。S5 可能因为网络延迟，先收到了 S2 的 RequestVote 并投给了 S2。RPC 到达顺序不可控。真正的分裂投票场景是：

| 步骤 | 谁 | 动作 | 结果 |
|------|-----|------|------|
| 1 | S1 | 超时，term=2，变 Candidate，投自己 | S1: term=2 |
| 2 | S2 | 几乎同时超时，term=2，变 Candidate，投自己 | S2: term=2 |
| 3 | S3 | 收到 S1 的 RequestVote，还没投 → 投 S1 | S3: votedFor=S1 |
| 4 | S4 | 收到 S2 的 RequestVote，还没投 → 投 S2 | S4: votedFor=S2 |
| 5 | S5 | 先收到 S2 的 RequestVote → 投 S2 | S5: votedFor=S2 |
| 6 | S1 | 票数：S1+S3 = 2 票 → 未达多数派 3 | S1 选举超时 |
| 7 | S2 | 票数：S2+S4+S5 = 3 票 → 达到多数派 | **S2 成为 Leader** |

--- 另一种情况：S5 两边都不投（网络延迟，RequestVote 都丢了）——

| 步骤 | 谁 | 动作 | 结果 |
|------|-----|------|------|
| 1 | S1 | 超时，term=2，变 Candidate | S1 票数: 1（自己） |
| 2 | S2 | 几乎同时超时，term=2，变 Candidate | S2 票数: 1（自己） |
| 3 | S3 | 投 S1 | S1: 2 票 |
| 4 | S4 | 投 S2 | S2: 2 票 |
| 5 | S5 | RequestVote 都丢了/超时了，没投任何人 | S5 保持 Follower |
| 6 | S1 | 2 票 < 3 → 超时 | **选举失败** |
| 7 | S2 | 2 票 < 3 → 超时 | **选举失败** |
| 8 | S1 | 随机超时先到，term=3，重新选举 | S1: term=3 |
| 9 | S2 | S1 超时更短先发起，S2 收到 S1 的 RequestVote(term=3) | term 3 > 2 → 退为 Follower，投 S1 |
| 10 | S1 | S1+S2+S3+S4+S5 全部投票 → 5 票 | **S1 成为 Leader** |

**关键**：这就是分裂投票的全貌——term=2 两人都没拿到多数，term 白白浪费。随机超时的作用是让 S1 和 S2 下次超时时间几乎不可能再次相同，term=3 时大概率只有一个人抢先发起选举。

---

## 十、Raft vs Paxos 对照总结

| | Basic Paxos | Raft |
|---|-------------|------|
| 共识目标 | 单个值 | 有序日志（多值） |
| 发起者 | 任何 Proposer | 只有 Leader |
| 协议阶段 | Prepare + Accept | Leader 选举 + 日志复制 |
| 优先级机制 | epoch 大者优先 | term 大者优先 |
| 角色投票 | Acceptor 不投票选 Leader | Follower 投票选 Leader |
| 活锁处理 | 依赖外部选 Leader | 随机超时内置解决 |
| 实践特点 | 理论简洁，工程复杂 | 工程直接可落地 |

**Raft 的实质**：Raft ≈ Multi-Paxos + Leader 选举 + 选举限制（对应 P2-c 约束）+ 标准工程实践。它没有发明新共识算法，而是把 Paxos 家族中已被广泛采用的实践（选 Leader、日志复制、多数派提交）打包成一个完整、可理解的协议。

```
核心机制：
  1. Term 作为逻辑时钟，单调递增，低 term 消息一律忽略
  2. Leader 选举：随机超时 + 多数派投票 + 选举限制（日志必须足够新）
  3. 日志复制：Leader 驱动，AppendEntries + 一致性检查 + 逐步回退
  4. 提交规则：只有当前 term 的条目被多数派确认后才提交
  5. 安全性靠选举限制保证：有已提交条目的节点才能当选 Leader

保证：
  ✓ 至多选出一个 Leader（election safety）
  ✓ 已提交条目永不丢失（leader completeness）
  ✓ 所有状态机执行相同命令序列（state machine safety）
  ✓ 多数派存活时最终选出 Leader 并推进（liveness）
```

### 关键参考文献

- Ongaro, Diego, and John Ousterhout. "In search of an understandable consensus algorithm." 2014 USENIX ATC.
- Ongaro, Diego. *Consensus: Bridging theory and practice.* Stanford University, 2014.
- Raft Visualization: https://raft.github.io/
- Howard, Heidi. *Distributed consensus revised.* Doctoral dissertation, University of Cambridge, 2019.
