# Zyzzyva 协议详解

> 基于《Ch4. Byzantine Fault Tolerant Consensus》PPT 整理

---

## 一、Zyzzyva 解决什么问题？

**一句话：在部分异步网络、存在拜占庭恶意节点的环境下，让多个节点对一系列命令的执行顺序达成一致，且在无故障时几乎无需共识开销。**

| 类型 | 条件 |
|------|------|
| Safety — agreement | 所有正确节点执行相同的命令序列（线性一致性） |
| Safety — validity | 客户端的命令最终会被执行 |
| Liveness — termination | 只要不超过 f 个拜占庭节点，客户端请求最终完成 |

Zyzzyva 的核心创新：**推测执行（Speculative Execution）**——正常情况下，客户端直接收集响应而无需走过重的共识协议。只有出问题时才"补票"。

---

## 二、背景：拜占庭共识的基本设定

### 2.1 崩溃故障 vs 拜占庭故障

| 维度 | Crash Fault（崩溃故障） | Byzantine Fault（拜占庭故障） |
|------|------------------------|------------------------------|
| 行为 | 节点停止工作（crash-stop 或 crash-recovery） | 任意行为：发送矛盾消息、选择性不响应、撒谎、共谋 |
| 性质 | 被动的、可预测的 | 主动的、恶意的 |
| 典型场景 | 服务器宕机、进程崩溃、网络断开 | 被入侵的节点、恶意参与者 |
| 模型简化 | "节点可能不说话，但说的都是真话" | "节点可能说任何话，包括假话" |
| 代表性协议 | Paxos、Raft | PBFT、Zyzzyva、Tendermint、HotStuff |
| 节点数要求 | n ≥ 2f+1 | n ≥ 3f+1 |
| 认证要求 | 不需要 | 需要数字签名防伪造 |

### 2.2 拜占庭共识的三个属性

| 属性 | 含义 |
|------|------|
| **Agreement** | 所有正确节点决定相同的值 |
| **Termination** | 所有正确节点在有限时间内终止 |
| **Validity** | 决定值必须是某个节点的输入值 |

**三种有效性定义**（一个重要但容易被忽略的细节）：

| 定义 | 含义 | 用途 |
|------|------|------|
| any-input validity | 决定值可以是**任意节点**（包括拜占庭）的输入 | 理论完备，但工程难处理 |
| correct-input validity | 决定值必须是**某个正确节点**的输入 | 实际 BFT 协议的默认选择 |
| all-same validity | 如果所有正确节点输入相同，决定值必须是该值 | 弱定义，容易满足 |

### 2.3 为什么 n > 3f？

经典的三将军问题：3 个将军（1 个指挥官 + 2 个副官），其中 1 个是叛徒（f=1），忠诚者能否对进攻/撤退达成一致？

**设定**：

| 角色 | 职责 |
|------|------|
| Commander | 发出命令 "attack" 或 "retreat" |
| General 1 | 收到命令后，与 General 2 交换各自听到的内容 |
| General 2 | 收到命令后，与 General 1 交换各自听到的内容 |

**场景 A：Commander 忠诚，General 2 是叛徒**

| 步骤 | 谁 | 做什么 |
|------|----|--------|
| 1 | Commander | 向 G1、G2 发出 **"attack"** |
| 2 | G1 | 听到 "attack"，如实告诉 G2："Commander 说的是 attack" |
| 3 | G2（叛徒） | 听到 "attack"，却对 G1 撒谎："Commander 说的是 **retreat**" |
| 4 | G1 的视角 | 收到两条消息：(1) Commander 直接说 attack，(2) G2 转述说 Commander 说的是 retreat |
| **结果** | — | **G1 无法判断**：是 Commander 发了矛盾命令，还是 G2 在撒谎？ |

**场景 B：Commander 是叛徒，G1、G2 都忠诚**

| 步骤 | 谁 | 做什么 |
|------|----|--------|
| 1 | Commander（叛徒） | 对 G1 说 **"attack"**，对 G2 说 **"retreat"** |
| 2 | G1 | 听到 "attack"，如实告诉 G2："Commander 说的是 attack" |
| 3 | G2 | 听到 "retreat"，如实告诉 G1："Commander 说的是 retreat" |
| 4 | G1 的视角 | 收到两条消息：(1) Commander 直接说 attack，(2) G2 转述说 Commander 说的是 retreat |
| **结果** | — | **G1 无法判断**：和场景 A 看到的信息**完全一样**！ |

**两场景对比**：

| 维度 | 场景 A（Commander 忠诚） | 场景 B（Commander 叛徒） |
|------|------------------------|-------------------------|
| 谁叛变 | General 2 | Commander |
| Commander 发出的命令 | attack, attack（一致） | attack, retreat（矛盾） |
| G1 收到 Commander 说的 | attack | attack |
| G1 收到 G2 转述的 | retreat | retreat |
| G1 看到的全部信息 | {attack, retreat} | {attack, retreat} |
| G1 能判断谁叛变吗？ | **不能** | **不能** |

G1 在两个场景中看到的信息**完全不可区分**——这就是 n=3, f=1 时共识不可能的根因。

**推广到一般情况**：

| 条件 | 说明 |
|------|------|
| 总节点数 | n，其中 f 个是拜占庭 |
| Phase 1 | 每个节点广播自己的值 |
| Phase 2 | 每个节点广播自己收到的一切 |
| 正确消息数 | 每个 correct 节点收到 n 条消息，其中 f 条可能是假的 → 至少 n-f 条正确 |
| 多数派交集约束 | 两个 correct 节点的多数派必须相交：(n-f) + (n-f) - n > f |
| 化简 | 2n - 2f - n > f → n > 3f |
| 结论 | **n ≥ 3f+1** |

| f（可容忍拜占庭数） | 最少节点数 n |
|---------------------|-------------|
| 1 | 4 |
| 2 | 7 |
| 3 | 10 |
| f | 3f+1 |

**结论**：要容忍 f 个拜占庭节点，需要总节点数 n ≥ 3f+1。Zyzzyva 采用标准配置 **n = 3f+1**。

---

## 三、模型与角色

### 3.1 Zyzzyva 的模型设定

| 维度 | 设定 |
|------|------|
| 节点数 | n = 3f+1（f 个拜占庭，2f+1 个正确） |
| 网络模型 | 部分异步（异步，加上一个已知上限 δ） |
| 网络轮次 | Q1 等待多数派消息；Q2 轮广播 |
| 认证 | 节点可认证身份（数字签名），拜占庭节点**不能**破解签名 |

### 3.2 角色

| 角色 | 职责 | 发送的消息 | 接收的消息 |
|------|------|-----------|-----------|
| **Client** | 发起命令、收集响应、检测不一致、触发恢复 | REQUEST, COMMIT | SPEC-RESPONSE, LOCAL-COMMIT |
| **Primary** | 分配序列号、广播命令（"半个独裁者"） | ORDER-REQ | REQUEST, CONFIRM-REQ |
| **Replica** | 执行命令、维护历史、发起 view change | SPEC-RESPONSE, LOCAL-COMMIT, VIEW-CHANGE | ORDER-REQ, COMMIT |

**消息流向**：

| 步骤 | 方向 | 消息 |
|------|------|------|
| 1 | Client → Primary | REQUEST |
| 2 | Primary → 所有 Replica | ORDER-REQ |
| 3 | 所有 Replica → Client | SPEC-RESPONSE |
| 4 | Client → 所有 Replica（需要时） | COMMIT |
| 5 | Replica → Client（需要时） | LOCAL-COMMIT |

**注意**：与 Paxos 不同，Paxos 中 Proposer 和 Acceptor 共同构成共识；Zyzzyva 中 Primary 是"半个独裁者"——它单方面决定顺序，replica 负责验证和执行。

---

## 四、推测执行：Zyzzyva 的核心洞察

### 4.1 视角变迁

| 维度 | PBFT (1999) | Zyzzyva (2007) |
|------|------------|---------------|
| 策略 | 每一笔操作都走完整共识流程 | 乐观执行，出问题再"补办手续" |
| 正常情况通信轮数 | 3 轮 | 1 轮 |
| 核心假设 | 不信任 primary | 大多数时候 primary 诚实 |
| 故障处理 | 每步都验证 | 先斩后奏，不一致时补救 |

**核心假设**：大多数时候，primary 是诚实的。既然诚实，为什么每次都要跑完整的共识？

**Zyzzyva 的做法**：
1. Primary 直接下令执行 → replica 执行并返回结果
2. 客户端收集响应——如果 3f+1 个全部一致 → 命令完成，零共识开销
3. 只有不一致时，才补充 **commit certificate** 或触发 **view change**

### 4.2 三种完成路径

| 响应数 S | 走哪条路径 | 处理方式 | 开销 |
|----------|-----------|---------|------|
| S = 3f+1 | Case 1（快路径） | 直接完成 | 1 轮 |
| 2f+1 ≤ S < 3f+1 | Case 2（慢路径） | 补充 commit certificate | 2 轮 |
| S < 2f+1 | Case 3（故障路径） | 重发或 view change | 4 轮+ |

---

## 五、协议消息定义

### 5.1 数据结构

```
命令/请求:
  REQUEST    = <REQUEST,    o, t, c>_σc
  ORDER-REQ  = <ORDER-REQ,  v, n, h, o, t, c, ND>_σp
  SPEC-RESP  = <SPEC-RESP,  v, n, h, r, c, d>_σr
  COMMIT     = <COMMIT,     v, n, d>_σr
  LOCAL-COMMIT = <LOCAL-COMMIT, h, r, c>_σr
```

| 符号 | 含义 | 说明 |
|------|------|------|
| **o** | Operation / Command | 客户端想要执行的操作 |
| **t** | timestamp | 客户端时间戳，用于排序 |
| **c** | client id | 客户端标识 |
| **v** | view number | 逻辑时钟，每个 view 有一个 primary |
| **n** | sequence number | 命令序号，单调递增 |
| **h** | history digest | 历史摘要 hash，H(h_prev, o, t, c, ND) |
| **r** | replica id | 副本标识 |
| **d** | result | 命令执行结果 |
| **ND** | non-deterministic | 非确定性变量，用于处理不确定性操作 |
| **_σx** | signature by x | 节点 x 的数字签名 |

### 5.2 消息说明

```
  <m>_σx  表示：消息 m 附带节点 x 的签名
  H(m)    表示：消息 m 的 hash 值
```

签名保证了两件事：(1) 消息来源可验证，(2) 拜占庭节点不能伪造消息。这是 BFT 协议区别于 CFT 协议（如 Paxos/Raft）的关键保障。

---

## 六、基本协议（Agreement 子协议）—— 三种情况详解

### 6.1 Case 1：快路径（无故障，primary 诚实）

**条件**：客户端收到 3f+1 个一致的 SPEC-RESPONSE

**执行流程**：

| 步骤 | 发起方 | 消息 | 接收方处理 |
|------|--------|------|-----------|
| 1 | Client | `REQUEST(o, t)`_σc → Primary | — |
| 2 | Primary | — | (a) 检查 t 是否为最新时间戳 (b) 分配 v、n (c) 计算 h = H(h_prev, o, t, c, ND) |
| 3 | Primary | `ORDER-REQ(v, n, h, o, t, c, ND)`_σp → 所有 Replica | — |
| 4 | Replica | — | (a) 验证签名 (b) 检查 v、n、h (c) 追加到本地历史 (d) 执行命令 (e) 生成结果 d |
| 5 | Replica | `SPEC-RESP(v, n, h, r, c, d)`_σr → Client | — |
| 6 | Client | — | 收集 3f+1 个响应，验证所有 h 一致 → 命令完成 |

**效率对比**：

| 协议 | 正常路径轮数 | 说明 |
|------|------------|------|
| **Zyzzyva Case 1** | **1 轮** | Client → Primary → Replica → Client |
| PBFT | 3 轮 | Pre-prepare → Prepare → Commit → Reply |
| Paxos | 2 轮 | Prepare/Promise → Accept/Accepted |

**为什么 3f+1 一致就安全？**

因为 n=3f+1，3f+1 个一致意味着所有 correct replica（2f+1 个）都收到了相同的 order，任何后续命令都会以这份历史为基础。即使 primary 想推翻也来不及了。

### 6.2 Case 2：慢路径（有拜占庭 replica）

**条件**：客户端收到 2f+1 ~ 3f 个一致响应，不够 3f+1

**执行流程**：

| 步骤 | 发起方 | 消息 | 接收方处理 |
|------|--------|------|-----------|
| 1 | Client | — | 检测到响应数在 [2f+1, 3f] 区间 |
| 2 | Client | `COMMIT` + CC (2f+1 个 SPEC-RESP 签名打包) → 所有 Replica | — |
| 3 | Replica | — | 验证 CC 与本地历史一致 → 发送 LOCAL-COMMIT |
| 4 | Replica | `LOCAL-COMMIT(h, r, c)`_σr → Client | — |
| 5 | Client | — | 收集 2f+1 个 LOCAL-COMMIT → 命令完成 |

**对比 Case 1**：

| 维度 | Case 1（快路径） | Case 2（慢路径） |
|------|-----------------|-----------------|
| 响应数 | 3f+1 | 2f+1 ~ 3f |
| 额外步骤 | 无 | 需 COMMIT + LOCAL-COMMIT |
| 通信轮数 | 1 轮 | 2 轮（多了一轮提交确认） |
| 原因 | 所有节点正常 | 个别 replica 未响应（可能是拜占庭） |
| 安全性保证 | 全票通过 | Commit Certificate（2f+1 签名） |

**Commit Certificate (CC)** = 2f+1 个来自不同 replica 的相同 SPEC-RESPONSE 签名

CC 的意义：**证明有 2f+1 个节点执行了该命令**。因为至多 f 个拜占庭，所以至少有 f+1 个 correct 节点执行过。这个事实不可撤销。

**Lemma 1**：两个不同的完成命令必然有不同的 sequence number。

| 步骤 | 推理 |
|------|------|
| 已知 | command α 完成 → 至少 2f+1 个节点为 α 发送了 SPEC-RESPONSE/COMMIT |
| 已知 | command β 完成 → 至少 2f+1 个节点为 β 发送了 SPEC-RESPONSE/COMMIT |
| 鸽巢原理 | 总节点 n=3f+1，两集合交集 ≥ (2f+1)+(2f+1)−(3f+1) = f+1 |
| 关键 | f+1 个交集节点中**至少 1 个是 correct**（因至多 f 个拜占庭） |
| 结论 | 该 correct 节点为 α 和 β 各发了一个 SPEC-RESPONSE，一个 correct 节点不会为同一个 n 发两次 → **α.n ≠ β.n** |

**Lemma 2**：如果 command α.n < command β.n，则 α 的 history 是 β 的 history 的前缀。

| 步骤 | 推理 |
|------|------|
| 已知 | Lemma 1 → 两个命令有不同的 n，且至少有一个 correct replica 同时持有 α 和 β |
| 时间线 | 该 replica 为 α 发 SPEC-RESPONSE 时，已确认了到 α 为止的历史 |
| 一致性检查 | replica 接受 β 时要求 β.h 与本地历史一致 |
| 结论 | β 的历史必然包含 α 及其之前的所有命令 → α 的 history 是 β 的 history 的前缀 |

### 6.3 Case 3：primary 故障

**条件**：客户端收不到足够响应（不足 2f+1 个）

| 子情况 | 可能原因 | 处理方式 | 结果 |
|--------|---------|---------|------|
| **Case 3.1** | primary 太慢（非拜占庭，只是网络延迟） | Client 重发 REQUEST 给所有 replica → replica 发 CONFIRM-REQ 给 primary → primary 收到后补发 ORDER-REQ | 流程回到 Case 1 或 Case 2 |
| **Case 3.2** | primary 是拜占庭（故意不发 ORDER-REQ） | 重发后仍然收不到足够响应 | 触发 View Change（见第八章） |

**Case 3.1 详细步骤**：

| 步骤 | 发起方 | 动作 |
|------|--------|------|
| 1 | Client | 将 REQUEST 直接发给所有 replica |
| 2 | Replica | 发 CONFIRM-REQ 给 primary；如果 t 是最新时间戳，重发缓存的 SPEC-RESPONSE 给 client |
| 3 | Primary | 收到 CONFIRM-REQ 后发 ORDER-REQ |
| 4 | — | 流程回到基本协议的步骤 2-3 |

---

## 七、Proof of Misbehavior — 如何抓"现行犯"

### 7.1 问题

客户端收到了不一致的 ORDER-REQ 消息——同一 primary 对同一 sequence number 或同一客户端请求签发了矛盾的排序。例如：

```
  ORDER-REQ(v, n=5, h="A", ...)_σp    ← primary 对 replica 1 说"第5号命令是A"
  ORDER-REQ(v, n=5, h="B", ...)_σp    ← primary 对 replica 2 说"第5号命令是B"
```

这两条消息由同一个 primary 签名，内容矛盾 → **铁证如山**。

### 7.2 PoM 的定义和用途

```
  PoM (Proof of Misbehavior) =
      { ORDER-REQ₁_σp, ORDER-REQ₂_σp }
      两个由同一 primary 签名但内容矛盾的消息
```

**流程**：

| 步骤 | 发起方 | 动作 | 结果 |
|------|--------|------|------|
| 1 | Client | 收到两个矛盾的 ORDER-REQ，都由同一 primary 签名 | 发现 primary 作恶 |
| 2 | Client | 构造 PoM = {REQ₁_σp, REQ₂_σp}，向 primary 广播 | primary 丢弃（不处理） |
| 3 | Client | 将 PoM 广播给所有 replica | — |
| 4 | Replica | 验证签名：确实是 primary 的矛盾签名 | **立刻发起 View Change** |

**关键安全性质**：PoM 的存在证明 primary 一定是拜占庭节点。因为 correct primary 绝不会对同一个请求签发矛盾的 ORDER-REQ。

---

## 八、View Change — 替换作恶的 Primary

### 8.1 触发条件

View Change 在以下情况被触发：

| 触发条件 | 说明 |
|----------|------|
| 收到有效的 PoM | primary 被证明是拜占庭 |
| 超时未收到 ORDER-REQ | Case 3.2：primary 不响应 |

### 8.2 协议流程

**概述**：当前 View v，primary P 被怀疑是拜占庭，需切换到 View v+1，新 primary = replica[(v+1) mod n]。

**Step 1 — 发起（I-HATE-PRIMARY → VIEW-CHANGE）**

| 发起方 | 消息 | 内容 |
|--------|------|------|
| Replica r | `I-HATE-PRIMARY(v)`_σr | 广播给所有副本，表达"当前 primary 有问题" |
| Replica r | `VIEW-CHANGE(v, h_r, cc_r, C_r, r)`_σr | h_r: 在 view v 中的有序请求历史；cc_r: 最新 commit certificate；C_r: 收到的 I-HATE-PRIMARY 集合 |

**Step 2 — 新 primary 收集并广播 NEW-VIEW**

| 步骤 | 动作 |
|------|------|
| 1 | 新 primary 收集 VIEW-CHANGE 消息，凑齐 2f+1 个 |
| 2 | 从 V 中恢复新历史 H_new（见 History Recovery） |
| 3 | 构造 `NEW-VIEW(v+1, V, H_new, CC_new)`_σ_newPrimary 并广播 |

**Step 3 — 确认新视图（VIEW-CONFIRM）**

| 发起方 | 动作 | 条件 |
|--------|------|------|
| 各 Replica | 验证 V 包含 2f+1 个有效 VIEW-CHANGE，恢复本地历史为 H_new | 收到 NEW-VIEW |
| 各 Replica | 广播 `VIEW-CONFIRM(v+1, h_r, r)`_σr | 验证通过 |
| 各 Replica | 进入 view v+1，开始接受新 primary 命令 | 收到 2f+1 个匹配的 VIEW-CONFIRM |

### 8.3 正确性关键点

| 问题 | 答案 |
|------|------|
| 为什么 2f+1 个 VIEW-CHANGE 足够？ | V 有 2f+1 个 → 至多 f 个拜占庭 → ≥ f+1 个来自 correct replica |
| I-HATE-PRIMARY 需要多少个？ | 2f+1 个中有 ≥ f+1 个 correct，足以推动 view change |
| 如果只有 1 个 correct replica 发现问题？ | 它广播的 I-HATE-PRIMARY 会被所有 correct replica 收到并转发，最终全体进入 view change |

---

## 九、History Recovery — 视图切换的历史恢复

### 9.1 问题的本质

View Change 之后，新 primary 面对一堆 VIEW-CHANGE 消息，每个包含不同的本地历史。它需要从中**重建一份统一的、没有缺口、没有丢失已完成命令的历史**。

**VIEW-CHANGE 集合示例**：

| Replica | 本地历史 | 持有的 CC |
|---------|---------|-----------|
| Replica 1 | [cmd_0, cmd_1, cmd_2, cmd_3] | cmd_1 |
| Replica 2 | [cmd_0, cmd_1, cmd_2, cmd_3] | cmd_1 |
| Replica 3 | [cmd_0, cmd_1, cmd_2] | (none) |
| Replica 4 | [cmd_0, cmd_1, cmd_2, cmd_3, ???] | cmd_1 |

**关键问题**：

| 问题 | 分析 |
|------|------|
| cmd_3 是否已完成？ | 3f+1 不一致，也没有 CC —— 处于灰色地带 |
| Replica 4 的第 5 个命令 ??? | 仅 replica 4 有 —— 可能是拜占庭节点伪造，应丢弃 |
| 新 primary 如何判断？ | 依赖 History Recovery 算法（9.3 节） |

### 9.2 核心引理

**Lemma 3**：全局最新的 commit certificate 一定被包含在新 primary 收集的 VIEW-CHANGE 集合中。

| 步骤 | 推理 |
|------|------|
| 已知 | 最新 CC 由 2f+1 个 replica 持有 |
| 已知 | VIEW-CHANGE 集合 V 来自 2f+1 个 replica |
| 鸽巢原理 | 2f+1 + 2f+1 − (3f+1) = f+1 → 两集合至少共享 f+1 个节点 |
| 关键 | f+1 个交集节点中至少 1 个是 correct（至多 f 个拜占庭） |
| 结论 | 该 correct 节点既持有 CC 也发送了 VIEW-CHANGE → CC 被包含在 V 中 |

**Lemma 4**：任何在最新 CC 之后完成的命令，必须在至少 f+1 个 VIEW-CHANGE 中包含的 replica 历史中出现。

| 步骤 | 推理 |
|------|------|
| 已知 | 命令在最新 CC 之后完成 → 至少 2f+1 个 replica 发送了 SPEC-RESPONSE |
| 已知 | V 包含 2f+1 个 replica 的历史 |
| 鸽巢原理 | 2f+1 + 2f+1 − (3f+1) = f+1 |
| 结论 | 至少 f+1 个 replica 的历史中包含该命令 |

**Lemma 5**：一旦命令被客户端认为完成，它在 view change 后不会丢失或被重排。

### 9.3 恢复算法

```
01  C = { 收到的 VIEW-CHANGE 消息集合 }
02  cc* = C 中序列号最大的 commit certificate     // Lemma 3 保证 cc* 一定在 C 中
03  R = { C 中携带了 cc* 的 replica }
04  history = R 中某个 replica 的本地历史          // 取到 cc* 为止的前缀
05  n = cc*.seq + 1
06  while C 中存在序号 >= n 的命令 do
07      if <cmd, n> 被 C 中至少 f+1 个 replica 报告 then   // Lemma 4
08          history.append(<cmd, n>)
09          C = C \ { 支持 <cmd, n> 的 replica }           // 收紧集合
10          n = n + 1
11      else
12          break                                          // 不足 f+1 -> 未完成，丢弃
13      end if
14  end while
15  return history
```

**逐行解释**：

| 行 | 做什么 | 为什么 |
|----|--------|--------|
| 01 | 收集所有 VIEW-CHANGE 消息 | 2f+1 个消息中包含所有正确节点的本地历史 |
| 02 | 找到全局最新 CC | Lemma 3：两个 2f+1 交集 ≥ f+1，保证 CC 被带入新视图 |
| 03 | 确定可信 replica 集合 R | R 中的 replica 承认最新 CC，以它们为基准重建 |
| 04 | 以 cc* 为锚点截取历史前缀 | cc* 之前的所有命令确认已完成，不容丢失 |
| 05 | 从 cc* 的下一个序号开始检查 | cc* 之后的命令处于灰色地带，需逐个验证 |
| 07 | 检查是否 ≥ f+1 个 replica 支持 | Lemma 4：≥ f+1 支持 → 至少有 1 个 correct 确认过 |
| 09 | 排除不支持的 replica | 收紧信任集，不支持的 replica 可能是拜占庭 |
| 12 | 不满足则停止 | 剩余命令无法确认完成，安全丢弃 |

---

## 十、Zyzzyva Gap —— 最危险的陷阱

### 10.1 什么是 Zyzzyva Gap？

命令在系统中的三种"存在状态"：

| 路径 | 持有者数量 | 恢复保证 | 安全性 |
|------|-----------|---------|--------|
| **快路径 (Case 1)** | 3f+1 个 replica | 全票通过 | 安全完成 |
| **慢路径 (Case 2)** | 2f+1 签了 CC | Commit Certificate 可恢复 | 安全完成 |
| **Zyzzyva Gap** | 仅 f+1 个 replica | f+1 中可含 f 个拜占庭，view change 时集体否认 | **可能丢失** |

**Zyzzyva Gap**：命令被 f+1 个节点持有（恰好包括所有 f 个拜占庭 + 1 个 correct），在 view change 时，这 f+1 个支持者中的 f 个拜占庭可以在新视图的 VIEW-CHANGE 中"假装没有见过"这个命令，导致它从历史中消失。

### 10.2 历史沿革

| 时期 | 事件 | 影响 |
|------|------|------|
| 2007 | Zyzzyva 发表，获 SOSP 最佳论文 | BFT 性能"金标准" |
| 2010-2016 | 工业界广泛采用 | Tendermint、HotStuff 等均受其启发 |
| 2017 | Abraham et al. 证明 View Change 存在安全性 bug | **Zyzzyva Gap 被发现** |
| 2018-2022 | "安全性危机" | 大量 BFT 协议被重新审查 |
| 2023 | AZyzzyva 提出 | 针对性地修复 Gap |
| 2024-2026 | 形式化 SMT 验证 view change | 研究者共识：History Recovery 是 BFT 协议中**最危险的部分** |

### 10.3 总结教训

> "History Recovery is the most dangerous part of any BFT protocol."
> — BFT 研究者共识 (2025-2026)

- 推测执行带来了性能飞跃，但也创造了"灰色地带"——命令在"可能完成"和"可能未完成"之间
- View Change 必须正确处理这个灰色地带，否则安全性崩溃
- 这是一个**整个 BFT 领域花了 10 年才完全理解的 bug**
- 当代协议（AZyzzyva、HotStuff 变体）都加入了更强的恢复保证

---

## 十一、Zyzzyva 总结

**问题**：N=3f+1 节点，部分异步网络，允许 f 个拜占庭节点，对命令序列达成一致。

| 机制 | 要点 |
|------|------|
| **推测执行** | 乐观假设 primary 诚实，先执行再验证；正常情况 1 轮完成（vs PBFT 3 轮） |
| **三路径策略** | 快路径 (3f+1) → 零开销；慢路径 (2f+1) → 补 CC；故障路径 → 重发/view change |
| **View Change** | PoM 抓出拜占庭 primary；基于多数派交集保证已完成命令不丢失 |
| **History Recovery** | 从 2f+1 个 VIEW-CHANGE 中重建统一历史；⚠ 最脆弱的部分 |
| **Zyzzyva Gap** | f+1 持有者的命令可能在 view change 中丢失；花了 10 年才被完全理解 |

| 保证 | 状态 |
|------|------|
| 所有 correct node 执行相同命令序列（safety） | ✓ |
| 客户端请求最终完成（liveness，部分同步假设下） | ✓ |
| History Recovery 完全正确 | ⚠ 需形式化验证 |

**影响链**：PBFT → Zyzzyva → Tendermint (Cosmos) / HotStuff (Aptos, Sui, Diem)

---

## 十二、与 Paxos 的对比

| 维度 | Paxos | Zyzzyva |
|------|-------|---------|
| 故障模型 | 崩溃故障 (CFT) | 拜占庭故障 (BFT) |
| 节点数 | 2f+1 | 3f+1 |
| 认证 | 不需要签名 | 依赖数字签名防伪造 |
| 正常延迟 | 2 轮 (Prepare + Accept) | 1 轮（推测执行） |
| 最坏延迟 | 2 轮 | 4 轮+（需要 view change） |
| 共识粒度 | 单个值 | 有序命令序列（状态机复制） |
| 活锁风险 | 有（需 Leader 解决） | 有（需 View Change 解决） |
| 核心复杂度 | Phase 1/2 的交互 | History Recovery 的正确性 |

**本质区别**：Paxos 是"先共识再执行"，Zyzzyva 是"先执行再验证"。这个反转带来了极大的性能提升，也带来了 Zyzzyva Gap 这类隐蔽的正确性风险。

---

### 关键参考文献

- Kotla, Ramakrishna, et al. "Zyzzyva: speculative byzantine fault tolerance." *SOSP*, 2007. (Best Paper Award)
- Abraham, Ittai, et al. "Revisiting fast practical byzantine fault tolerance." *arXiv*, 2017. (发现 Zyzzyva Gap)
- Lamport, L., Shostak, R., & Pease, M. "The Byzantine generals problem." *TOPLAS*, 1982. (图灵奖工作)
- Buchman, Ethan. "Tendermint: Byzantine fault tolerance in the age of blockchains." 2016.
- Yin, Maofan, et al. "HotStuff: BFT consensus with linearity and responsiveness." *PODC*, 2019.
