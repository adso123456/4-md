# Paxos 算法详解

> 基于《Ch3 Crash Fault-Tolerant Consensus》文档整理

---

## 一、Paxos 解决什么问题？

**一句话：在异步网络、节点可能崩溃的环境下，让多个节点对同一个值达成一致，且一旦达成绝不反悔。**

约束条件：

| 类型 | 条件 |
|------|------|
| Safety — validity | 只有被提议的值才能被选中 |
| Safety — agreement | 最多只有一个值被选中 |
| Liveness — termination | 只要少于半数节点故障，最终一定有值被选中 |

Paxos 优先保证安全性，必要时牺牲活性。

---

## 二、核心角色与多数派机制

| 角色 | 职责 |
|------|------|
| **Proposer**（提议者） | 发起提案，推动共识 |
| **Acceptor**（接受者） | 对提案投票，构成多数派（quorum） |
| **Learner**（学习者） | 被动获知最终结果 |

一个节点通常同时是 Proposer + Acceptor + Learner。

**根本机制**：值被**多数派**（> n/2）接受即为"选中（chosen）"。因为任意两个多数派必有交集（数学保证：`⌊n/2⌋+1 + ⌊n/2⌋+1 = n+2 > n`），后续发起者一定会从交集节点获知已被选中的值。

---

## 三、为什么需要两阶段？—— 推导过程

### 3.1 单阶段方案的问题

两个 Proposer 同时向不同 Acceptor 发不同的值，导致分裂——没有共识。

核心矛盾：**Proposer 在提议前不知道其他 Proposer 的提案状态**。

### 3.2 引入 Proposal Number 后的新问题

给提案分配全局唯一、单调递增的编号（epoch/ticket），编号大的优先级高。但新问题出现了：

> 一个新的 Proposer 发起提案时，是否已经有别的提案被部分 Acceptor 接受了？
> 如果已有，新 Proposer **必须沿用**旧提案的值，否则安全性被破坏。
> 但新 Proposer 怎么知道这些信息？—— **必须先问一圈**。

### 3.3 两阶段的分工

```
Phase 1（Prepare/Promise）：情报收集 — "有人已经接受了什么？"
Phase 2（Propose/Accept）：受约束的提议 — "根据情报决定提什么值"
```

---

## 四、核心约束链（P1 → P2 → P2-a → P2-b → P2-c）

这是 Lamport 推导 Paxos 安全性的逻辑链：

```
P1:  Acceptor 必须接受它收到的第一个 proposal
     ↓ （问题：两个 proposer 各发不同值 → 多数派分裂 → 没共识）
P2:  如果 v 被 chosen，所有更高编号的 proposal 必须携带 v
     ↓ （把责任从 acceptor 转到 proposer）
P2-a: 如果 v 被 chosen，acceptor 之后只能 accept 携带 v 的 proposal
     ↓ （但新 proposer 可能不知道 v，需要强制 proposer）
P2-b: 如果 v 被 chosen，proposer 之后只能 propose v
     ↓ （proposer 怎样才能知道 v？需要先去打探）
P2-c: proposer 发起编号 n 的 proposal 前，必须先向某个多数派发 Prepare(n)，
      收集他们的 lastAccepted，沿用其中 epoch 最大的那个值
```

**P2-c 就是 Phase 1 存在的根本原因。**

---

## 五、算法伪代码（逐行对照解释）

### 5.1 Proposer 算法

```
Proposer algorithm
  Initialization:
    n = 选择比之前用过的都大的唯一编号
    v = null                              // 将要提议的值（初始未知）

  Phase 1:
    send <prepare, n> to all acceptors    // 向全体 acceptor 发"准备请求"
    S = ∅                                  // S 记录已回复的 acceptor 集合
    n_max = null                           // n_max 记录收到的最大 lastAcceptedEpoch

    while |S| < ⌊n/2⌋ + 1 do              // 等待多数派回复（如 5 个节点需 ≥3）
      if <promise, n, n', v'>              // 收到 acceptor 的承诺回复
           received from acceptor α then   // n'=acceptor上次接受的epoch, v'=对应的值
        S = S ∪ {α}                        // 记录这个 acceptor 已回复
        if (n' ≠ null) ∧ (n' > n_max) then // 如果有已接受的提案，且 epoch 更大
          n_max = n', v = v'               // 记下这个值和它的 epoch
        end if
        if timeout then goto Initialization // 超时未凑齐多数派 → 增大 n 重试
      end if
    end while

    // ——— 多数派已回复，决定提议什么值 ———
    if v = null then                       // 所有 acceptor 都未接受过任何提案
      v = proposer's candidate value       // 自由选择：提自己的候选值
    end if                                 // 否则 v 沿用 n_max 对应的值（已在循环中赋值）

  Phase 2:
    send <accept, n, v> to all acceptors  // 向全体 acceptor 发"接受请求"
    S = ∅

    while |S| < ⌊n/2⌋ + 1 do              // 再次等待多数派确认
      if <accepted, n, v>
           received from acceptor α then
        S = S ∪ {α}
      end if
      if timeout then goto Initialization // 超时 → 增大 n 重试
    end while

    // v is chosen! 共识达成！
```

**关键决策点（值选择规则）**：

| Phase 1 收集到的情况 | Phase 2 提议的值 |
|----------------------|------------------|
| 所有 Promise 中 n' 均为 null（无人接受过） | 提自己的候选值 |
| 只有一个 Promise 带了 n'（有人接受过） | 沿用那个提案的 v' |
| 多个 Promise 带了不同的 n'、v' | 沿用**n' 最大**的那个提案的 v' |

**为什么选 n' 最大的？** 因为 n' 最大的提案最"新"，它的值最可能已经被之前的多数派选定。沿用最大 n' 的值，就能保证"一旦有值被 chosen，后续所有提案都沿用同一个值"。

### 5.2 Acceptor 算法

```
Acceptor algorithm
  // 持久化变量（节点崩溃恢复后仍保留）
  lastPromisedEpoch = null              // 承诺过的最大 epoch
  lastAcceptedEpoch = null              // 已接受提案的 epoch
  lastAcceptedValue = null              // 已接受提案的值

  while true do
    msg = received from proposer

    // ——————— Phase 1 ———————
    if msg is <prepare, n> then          // 收到"准备请求"
      if n > lastPromisedEpoch then      // 只处理编号更大的请求（编号小的已过时）
        lastPromisedEpoch = n            // 承诺：不再接受 < n 的提案
        send <promise,                   // 回复三个信息：
              n,                          //   (1) 确认这个编号
              lastAcceptedEpoch,          //   (2) 我上次接受的 epoch（可能 null）
              lastAcceptedValue>          //   (3) 我上次接受的值（可能 null）
              to proposer
      end if
      // 如果 n ≤ lastPromisedEpoch，直接忽略（编号过时）
    end if

    // ——————— Phase 2 ———————
    if msg is <accept, n, v> then        // 收到"接受请求"
      if n ≥ lastPromisedEpoch then      // 承诺没被打破（n ≥ 已承诺的编号）
        lastPromisedEpoch = n            // 更新承诺
        lastAcceptedEpoch = n            // 记录本次接受的 epoch
        lastAcceptedValue = v            // 记录本次接受的值
        send <accepted, n, v> to proposer // 回复"已接受"
      end if
      // 如果 n < lastPromisedEpoch，说明已经答应过更大的编号，忽略
    end if
  end while
```

**Acceptor 的两条铁律**：

| 规则 | 含义 |
|------|------|
| Phase 1：`n > lastPromisedEpoch` 才响应 | 承诺了就不再反悔——不能给更小的编号发 Promise |
| Phase 2：`n ≥ lastPromisedEpoch` 才接受 | 如果 n=lastPromisedEpoch，说明正是承诺过的那个号；如果 n>lastPromisedEpoch，更新承诺后再接受 |

### 5.3 Proposer 属性清单（文档 P10 总结）

| # | 属性 |
|---|------|
| 1 | Proposer 使用唯一的 epoch（ticket）标识每个 proposal |
| 2 | epoch 单调递增 |
| 3 | 只有收到多数派 Promise 后，Proposer 才能提议值 |
| 4 | 只有收到多数派 Accepted 后，Proposer 才能返回"已选定" |
| 5 | Proposer 必须按值选择规则决定提议的值 |

### 5.4 Acceptor 属性清单（文档 P10 总结）

| # | 属性 |
|---|------|
| 6 | 只有 epoch ≥ lastPromisedEpoch 的消息才被处理 |
| 7 | 处理消息时，lastPromisedEpoch 更新为消息中的 epoch |
| 8 | 收到 Prepare(n) 后，回复 Promise(n, lastAcceptedEpoch, lastAcceptedValue) |
| 9 | 收到 Accept(n,v) 后，更新持久化状态，回复 Accepted(n,v) |
| 10 | lastPromisedEpoch 和 lastAccepted 是持久化存储，只在属性 7 和 9 时更新 |

---

## 六、运行示例（表格推演）

以下使用 3 个 Acceptor（A1、A2、A3），多数派 = `⌊3/2⌋+1 = 2`。

### 6.1 正常流程（无竞争）

**初始状态**：所有 Acceptor 的 `lastPromisedEpoch = null`，`lastAccepted = null`

| 步骤 | 角色 | 动作 | 结果 |
|------|------|------|------|
| 1 | P1 | 选 epoch=0，向 A1/A2/A3 发 `Prepare(0)` | — |
| 2 | A1 | 0 > null ✓，`lastPromisedEpoch=0`，回复 `Promise(0, null, null)` | A1 状态: promised=0 |
| 3 | A2 | 同上 | A2 状态: promised=0 |
| 4 | A3 | 同上 | A3 状态: promised=0 |
| 5 | P1 | 收到 3 个 Promise，全部为 null，多数派 ✓ → 用自己的值 v | v = v（候选值） |
| 6 | P1 | 向 A1/A2/A3 发 `Accept(0, v)` | — |
| 7 | A1 | 0 ≥ 0 ✓，`lastPromisedEpoch=0, lastAccepted=(0,v)`，回复 `Accepted(0,v)` | A1 状态: accepted=(0,v) |
| 8 | A2 | 同上 | A2 状态: accepted=(0,v) |
| 9 | A3 | 同上 | A3 状态: accepted=(0,v) |
| 10 | P1 | 收到 3 个 Accepted，多数派 ✓ | **v 被 chosen！** |

**结论**：无人竞争时，一轮两阶段即可达成共识。

---

### 6.2 两 Proposer 竞争，后发者沿用前者的值

这是最能体现 Paxos 精妙之处的例子。

**初始状态**：全部为 null。P1 候选值 = A，P2 候选值 = B。

| # | 谁 | 动作 | 谁的状态变化 | 关键说明 |
|---|----|------|-------------|----------|
| 1 | P2 | 发 `Prepare(1)` → A1、A2、A3 | A2: `promised=1` | P2 先行动，epoch=1 |
| 2 | A2 | `1 > null` ✓，回复 `Promise(1, null, null)` | — | A2 承诺不再接受 <1 的 Prepare |
| 3 | P1 | 发 `Prepare(0)` → A1、A2、A3 | A1: `promised=0` | P1 不知情，epoch=0 |
| 4 | A1 | `0 > null` ✓，回复 `Promise(0, null, null)` | — | A1 向 P1 承诺 |
| 5 | A2 | `0 < 1` ✗ → **忽略** | — | A2 已经承诺了更大的 epoch=1 |
| 6 | A3 | `0 > null` ✓，回复 `Promise(0, null, null)` | A3: `promised=0` | — |
| 7 | P2 | 发 `Accept(1, B)` → A1、A2、A3 | — | P2 进入 Phase 2 |
| 8 | A2 | `1 ≥ 1` ✓，`accepted=(1,B)`，回复 `Accepted(1,B)` | A2: `accepted=(1,B)` | **A2 接受了 B** |
| 9 | A3 | `1 ≥ 0` ✓，`promised=1, accepted=(1,B)`，回复 `Accepted(1,B)` | A3: `accepted=(1,B)` | **A3 也接受了 B** |
| 10 | P2 | 收到 A2、A3 的 Accepted → 多数派 ✓ | — | P2 可能认为 B 已 chosen（实际 A1 还没接受） |
| — | — | — | — | — |
| 11 | P1 | 之前只收到 A1、A3 的 Promise(0)，凑足多数派 → 以为没人接受过 | v=A | **关键！P1 不知 B 已被部分接受** |
| 12 | P1 | 发 `Accept(0, A)` → A1、A2、A3 | — | P1 用 A 进入 Phase 2 |
| 13 | A1 | `0 ≥ 0` ✓，`accepted=(0,A)`，回复 `Accepted(0,A)` | A1: `accepted=(0,A)` | **A1 接受了 A！** |
| 14 | A2 | `0 < 1` ✗ → **忽略** | — | A2 的 promised=1，拒绝 0 |
| 15 | A3 | `0 < 1` ✗ → **忽略** | — | A3 的 promised=1，拒绝 0 |
| 16 | P1 | 只收到 A1 一票 Ack，未达多数派 → **超时** | — | P1 的 A 未能被 chosen |

**此时状态**：A1 认为 (0,A) 被选定，但 A2 和 A3 认为 B 才是。系统处于可能不一致的危险状态。

**Paxos 如何拯救？** — P2 发起新一轮提案：

| # | 谁 | 动作 | 谁的状态变化 | 关键说明 |
|---|----|------|-------------|----------|
| 17 | P2 | 发 `Prepare(2)` → A1、A2、A3 | — | P2 继续推进 |
| 18 | A1 | `2 > 0` ✓，`promised=2`，回复 `Promise(2, 0, A)` | A1: `promised=2` | A1 透露：已接受 (0,A) |
| 19 | A2 | `2 > 1` ✓，`promised=2`，回复 `Promise(2, 1, B)` | A2: `promised=2` | A2 透露：已接受 (1,B) |
| 20 | A3 | `2 > 1` ✓，`promised=2`，回复 `Promise(2, 1, B)` | A3: `promised=2` | A3 透露：已接受 (1,B) |
| 21 | P2 | 收到 3 个 Promise。有已接受提案：(0,A) 和 (1,B) → **选 n' 最大的** → epoch 1 > 0 → 沿用 B | v=B | **P2-c 规则生效！** |
| 22 | P2 | 发 `Accept(2, B)` → A1、A2、A3 | — | 虽然 P2 沿用 B，但编号升级为 2 |
| 23 | A1/A2/A3 | `2 ≥ 2` ✓，全部接受 (2,B) | 全部: `accepted=(2,B)` | 全体一致 |
| 24 | P2 | 收到全部 Ack，多数派 ✓ | — | **(2,B) 被 chosen！** |

**最终结果**：A1 曾经认为自己选了 A，但经过新一轮 Paxos，最终全体一致选 B。**Paxos 保证：不论中间有多混乱，最终只有一个值被选中。**

---

### 6.3 活锁（Livelock）

| # | 谁 | 动作 | 结果 |
|---|----|------|------|
| 1 | P1 | `Prepare(0)` → 收到多数派 Promise | P1 准备发 Accept |
| 2 | P2 | `Prepare(1)` → Acceptor 的 `promised` 被更新为 1 | P1 的 Accept(0) 将被多数派拒绝 |
| 3 | P1 | `Accept(0, v1)` → 被拒（概率大）→ **超时** | P1 增大 epoch 重试 |
| 4 | P1 | `Prepare(2)` → 收到多数派 Promise | 把 P2 的 Accept(1) 的路堵上了 |
| 5 | P2 | `Accept(1, v2)` → 被拒 → **超时** | P2 增大 epoch 重试 |
| 6 | P2 | `Prepare(3)` → ... | ...循环，无人成功 |

**原因**：两个 Proposer 轮流把对方的 Accept 阶段堵死。这是 Paxos 牺牲活性的直接体现。

**实践中的解决方案**：
- **随机退避**：超时后在随机时间后重试
- **选 Leader**（Multi-Paxos / Raft 的做法）：只有一个 Proposer，根本不会竞争

---

## 七、关于 S1 可以和 S2 不同

Phase 1 收集 Promise 的 acceptor 集合（S1）和 Phase 2 收集 Accepted 的 acceptor 集合（S2），**不需要是同一组多数派**。

文档明确指出这是常见误解。S1 = S2 是特例，不是必须条件。只要 S1 和 S2 都是多数派，它们必然有交集。那个交集节点会在 Phase 1 中透露它的 `lastAccepted`，Proposer 在 Phase 2 中受 P2-c 约束沿用该值。安全性成立。

```
S1: {A1, A2, A3} ← Phase 1 的多数派
S2: {A3, A4, A5} ← Phase 2 的多数派，可以不同
交集: {A3}       ← A3 保证了信息的传递
```

---

## 八、Paxos 总结

```
问题：N 个节点，异步网络，允许崩溃，对一个值达成共识

核心机制：
  1. 提案有递增编号（epoch），编号大者优先
  2. 两阶段协议：
     Phase 1 — 向多数派发 Prepare(n)，收集已有的 accepted 信息
     Phase 2 — 受 P2-c 约束决定值（沿用最大 epoch 的已接受值，或自己的候选值）
  3. 被多数派 Accept 后即为 chosen
  4. 任意两个多数派必有交集 → 后续 proposer 必然发现已选中的值 → 安全

保证：
  ✓ 至多选出一个值（safety）
  ✗ 不保证一定能选出值——可能活锁（liveness），实践中靠 Leader 解决
```

### 关键参考文献

- Lamport, Leslie. "The part-time parliament." ACM TOCS, 1998.
- Lamport, Leslie. "Paxos made simple." ACM Sigact News, 2001.
- De Prisco, Roberto, Butler Lampson, and Nancy Lynch. "Revisiting the Paxos algorithm." WDAG, 1997.
