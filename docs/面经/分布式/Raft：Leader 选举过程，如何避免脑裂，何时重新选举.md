## 1. Raft Leader 选举过程（从 follower 到 leader）

### 1.1 角色与超时：选举从“心跳消失”开始

Raft 的选举由 `electionTimeout` 驱动：

- follower 在一段随机超时内收不到 leader 的心跳（`AppendEntries` 空日志也算心跳），就会认为 leader 可能不可用。
- follower 转为 candidate，发起选举。

随机化超时是关键：降低多个节点同时发起选举造成“分票（split vote）”的概率。

### 1.2 term 的推进与投票（RequestVote）

candidate 发起选举时：

1. `currentTerm++`，进入新 `term`。
2. 投票给自己（`votedFor = self`）。
3. 并行向其他节点发送 `RequestVote(term, candidateId, lastLogIndex, lastLogTerm)`。

对方（follower）处理投票请求时的核心规则：

- 若 `term` 比自己小，拒绝（说明对方信息旧）。
- 若 `term` 比自己大，更新 `currentTerm` 并退回 follower（承认新任期）。
- 每个 `term` 最多投一票（`votedFor` 防止重复投票）。
- **日志新旧校验（log up-to-date）**：只有当候选人的日志“不落后”于自己时才投票。

### 1.3 获得多数派即当选

candidate 在同一 `term` 内获得**多数派**投票（`> N/2`）就成为 leader，然后：

- 立即发送心跳（`AppendEntries`）宣告领导权并抑制其他节点超时。

若出现分票：

- 多个 candidate 都拿不到多数派，直到某个 candidate 超时进入更高 `term`，重新发起选举，最终收敛。

## 2. 如何避免脑裂（为什么不会出现两个 leader 同时对外写）

### 2.1 核心原因：多数派集合必相交

在同一个 `term` 内：

- leader 必须拿到多数派投票。
- 任何两个多数派集合必然至少有一个交集节点。
- 交集节点在同一 `term` 只能投一票，因此不可能同时选出两个 leader。

这就是 Raft 防脑裂的数学基础。

### 2.2 “避免脑裂”不等于“避免旧 leader 误以为自己仍是 leader”

真实系统里常见现象：

- 旧 leader 处在少数派网络分区里，可能短时间还以为自己是 leader。

Raft 通过 `term` 纠偏：

- 旧 leader 一旦收到更高 `term` 的 RPC（`RequestVote` 或 `AppendEntries`），就必须退位为 follower。

工程实现要点：

- RPC 必须携带 `term`，并且每次处理 RPC 都要更新 `term` 与角色。

## 3. 什么情况下会重新选举

### 3.1 leader 崩溃或不可达

最典型：

- follower 在 `electionTimeout` 内收不到心跳，就会触发选举。

### 3.2 网络抖动导致心跳丢失

即便 leader 没崩溃，只要心跳延迟超过超时，也会触发选举。

工程后果：

- 频繁选举会显著降低吞吐，写入会被打断。

常见工程手段：

- 合理配置 `heartbeatInterval` 与 `electionTimeout`（通常后者是前者的 5～10 倍以上）。
- 增加抖动容忍，避免短抖动触发重选。

### 3.3 leader 发现自己落后（term 过期）

leader 收到更高 `term` 的消息（例如某个节点发起选举），必须立刻退位。

### 3.4 日志复制长期失败（间接触发）

如果 leader 无法把日志复制到多数派（例如多数节点不可达），提交无法推进，客户端可能超时。此时：

- 多数派侧可能选出新 leader（如果 leader 不在多数派）。
- 少数派侧的旧 leader 最终会因为看到更高 `term` 而退位。

## 4. “日志不落后”投票规则为什么重要

### 4.1 防止选出“缺日志”的 leader

如果允许一个落后节点当选 leader，它可能覆盖或丢失已经提交的日志，破坏 Safety。

因此 Raft 要求投票时比较候选人的 `(lastLogTerm, lastLogIndex)`：

- `lastLogTerm` 更大者更新；
- 若 `lastLogTerm` 相同，`lastLogIndex` 更大者更新。

这条规则与后续的“leader 完整性（Leader Completeness）”性质一起，保证已提交日志不会被回滚。

## 5. 常见面试追问（简答）

### 5.1 选举期间还能写吗

通常不能保证写成功：

- 没 leader 或 leader 不稳定时，写入要么失败，要么等待新 leader 产生后重试。

### 5.2 为什么要随机超时

避免所有节点同时变 candidate 导致反复分票，提升收敛速度与稳定性。
