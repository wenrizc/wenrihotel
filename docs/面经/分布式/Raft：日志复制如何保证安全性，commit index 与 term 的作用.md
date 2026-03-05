## 1. Raft 日志复制流程（AppendEntries）

### 1.1 写入从 leader 开始

客户端把写请求发送给 leader（或先到任意节点再转发给 leader）。leader 的关键动作：

1. 把命令追加到本地日志 `log[]`，形成新条目：`(index, term, command)`。
2. 并行向 follower 发送 `AppendEntries`（携带 prev 信息与新条目）。
3. 收到多数派确认后，推进提交点（`commitIndex`）。
4. 让状态机按序应用（`apply`）已提交日志，并对客户端返回成功。

### 1.2 AppendEntries 的关键字段（最小集合）

leader → follower 的 `AppendEntries` 通常包含：

- `term`：leader 的任期。
- `leaderId`
- `prevLogIndex`、`prevLogTerm`：用于一致性检查。
- `entries[]`：本次要复制的新日志条目（心跳时可以为空）。
- `leaderCommit`：leader 当前的 `commitIndex`。

follower 的核心校验：

- 若 `term` 小于自身 `currentTerm`，拒绝并返回自己的 `currentTerm`（让 leader 退位）。
- 若本地在 `prevLogIndex` 处的 `term` 不等于 `prevLogTerm`，拒绝（表示日志发生分叉）。
- 若校验通过，则删除本地从冲突点开始的后缀，并追加 `entries[]`（对齐 leader）。

## 2. Raft 如何保证安全性（为什么不会提交两份冲突历史）

### 2.1 日志匹配性质（Log Matching Property）

核心性质：

- 如果两个日志在某个 `index` 位置的条目 `term` 相同，那么这两个日志在该 `index` 之前的所有条目都完全相同。

实现依赖：

- `AppendEntries(prevLogIndex, prevLogTerm)` 的一致性检查。

这使得 leader 能用“前驱匹配”把 follower 的日志强制对齐到自己的前缀。

### 2.2 Leader 完整性（Leader Completeness）

要保证“已提交的条目不会丢”，必须确保：

- **任何已提交的日志条目，一定会出现在未来所有 leader 的日志里。**

Raft 通过两点达成：

1. 投票时的“日志不落后”规则：只有日志足够新的 candidate 才可能当选。
2. 提交规则（见 3.2）：leader 只在特定条件下推进 `commitIndex`。

### 2.3 安全性与可用性的取舍（与 CAP 呼应）

Raft 在极端网络下倾向保证 Safety：

- 只要无法确认多数派复制，就不提交（可能牺牲可用性与延迟）。

## 3. commit index 与 term 的作用（面试必问）

### 3.1 `commitIndex` 是什么

`commitIndex` 表示：**已提交（Committed）日志的最大索引**。

含义：

- `index <= commitIndex` 的日志条目对外“不可撤销”，状态机可以应用，客户端可以认为成功。
- `index > commitIndex` 的条目只是“已复制但未提交”，可能在 leader 切换时被覆盖。

### 3.2 为什么提交需要 term：避免“旧日志被错误提交”

Raft 的关键提交规则（常见表述）：

- leader 只有在某个日志条目同时满足：
  - 已复制到多数派；
  - 并且该条目的 `term == currentTerm`；
  才能用它来推进 `commitIndex`。

直觉解释：

- 新 leader 可能携带一些“前任 leader 的旧 term 日志”，这些日志即使在多数派出现过，也可能存在复杂的分叉。
- 通过要求“用当前 term 的日志推进提交”，可以避免在某些边界情况下把不该提交的旧日志误判为已提交。

重要补充：

- 一旦某个当前 term 条目被提交，那么它之前的所有条目（即便来自旧 term）也都会被认为提交（因为日志前缀已被多数派认可）。

### 3.3 `term` 的三种核心用途

- 选举：区分新旧 leader，拒绝旧 term 的投票与追加。
- 日志一致性：`prevLogTerm` 用于发现分叉并回滚不一致后缀。
- 提交安全：提交规则用 `term` 避免错误提交与丢已提交日志。

## 4. follower 如何推进提交并应用

### 4.1 leaderCommit 如何影响 follower

follower 在处理 `AppendEntries` 成功后：

- 将自身 `commitIndex = min(leaderCommit, lastLogIndex)`。
- 然后把 `lastApplied` 到 `commitIndex` 之间的条目按序应用到状态机。

这保证所有节点最终在已提交前缀上达到一致。

## 5. 常见面试追问

### 5.1 “复制到多数派”就等于“写成功”吗

不一定，必须区分：

- replicated：日志条目写入了多数派日志（可能还没提交）。
- committed：leader 推进 `commitIndex` 并对外承诺不可回滚。

客户端通常应以“提交并应用”为成功语义，否则会出现“成功返回但随后回滚”的体验问题。

### 5.2 日志冲突怎么处理，leader 需要一条条回退吗

朴素实现是递减 `nextIndex` 重试，直到 `prevLogIndex/prevLogTerm` 匹配。

工程优化常见做法：

- follower 在拒绝时返回冲突 term 与该 term 的首个 index，leader 可以跳跃式回退，减少重试次数。
