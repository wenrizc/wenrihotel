## 1. 分布式锁解决什么问题

### 1.1 目标：跨进程互斥与正确释放

分布式锁用于在多个进程/节点之间对某个共享资源实现互斥访问，核心目标包括：

- 互斥性：同一时刻最多一个持有者进入临界区。
- 不死锁：持有者崩溃或失联后，锁最终能被释放。
- 可恢复：异常场景下系统能收敛，不永久卡死。

面试加分点：分布式锁不仅要“能加锁”，更要回答“**如何防止锁被错误释放、如何防止锁过期导致并发进入**”。

### 1.2 两个常见误区

- 误区 1：拿到锁就安全。实际上还需要处理锁过期、暂停（GC Stop-The-World）、网络抖动导致的“锁漂移”。
- 误区 2：只要 `SETNX` 就够了。正确实现需要原子性、唯一标识、续租、释放校验与 fencing。

## 2. 分布式锁的关键设计点（不按中间件划分）

### 2.1 幂等与唯一持有者标识

每次加锁生成唯一 token（例如 UUID），并把 token 写入锁值中：

- 释放锁时必须验证 token，避免“我把别人的锁删了”。

### 2.2 TTL 与续租（Lease）

为了避免死锁，锁通常需要 TTL：

- 持有者正常工作时续租。
- 持有者崩溃时 TTL 到期自动释放。

风险：如果持有者发生长时间暂停（例如 GC），可能导致 TTL 过期但持有者继续执行临界区，出现并发进入。

### 2.3 Fencing Token（栅栏令牌）

这是工程上防“锁过期后仍在执行”的关键手段：

- 每次成功获取锁，系统返回一个单调递增的 `fence`（例如递增序号、revision）。
- 临界资源（如数据库）在写入时携带 `fence`，并拒绝小于已见最大值的写入。

直觉：即便旧持有者“误以为仍持锁”，它携带的 `fence` 更小，也会被资源层拒绝。

## 3. 用 etcd 实现分布式锁：事务 + Lease + Revision

### 3.1 核心思路

etcd 提供：

- 线性一致的 KV（基于 Raft）。
- 原子事务（Compare-And-Swap）。
- Lease（租约）与 keepalive。
- revision（全局递增版本号，天然可做 fencing token）。

### 3.2 最小实现（事务抢锁）

下面用 `etcd` 的事务表达“检查并写入”的原子性（示例为 Go 客户端伪代码，省略错误处理）：

```go
leaseResp, _ := cli.Grant(ctx, 10) // TTL=10s
leaseID := leaseResp.ID

token := uuid.NewString()
key := "/locks/job-123"

txnResp, _ := cli.Txn(ctx).
	If(clientv3.Compare(clientv3.Version(key), "=", 0)). // key 不存在
	Then(clientv3.OpPut(key, token, clientv3.WithLease(leaseID))).
	Else(clientv3.OpGet(key)).
	Commit()

if !txnResp.Succeeded {
	// 未抢到锁：读取当前持有者信息或直接退避重试
}

fence := txnResp.Header.Revision // 可作为 fencing token（单调递增）
_ = fence
```

工程落地通常是：

- 先创建 lease（带 TTL）。
- 再用 txn：`if version(key)==0 then put(key, value, lease) else fail`。

### 3.3 etcd 的坑与注意点

- 续租失败：网络抖动或客户端暂停会导致 lease 过期，必须配合 fencing。
- watch 不是强一致读：watch 是事件流，消费端仍要基于 revision 做校验，避免漏事件或重放导致的错误判断。
- 热点 key：大量竞争同一把锁会造成 etcd 压力上升，需要限流与退避。

适用场景：

- 对正确性要求高的锁（配置变更、主节点竞选、全局互斥任务）。

## 4. 用 ZooKeeper 实现分布式锁：临时顺序节点 + Watch

### 4.1 核心思路（经典配方）

ZooKeeper 提供：

- 临时节点（Ephemeral）：会话断开自动删除，避免死锁。
- 顺序节点（Sequential）：天然排队。
- Watch：事件通知，减少轮询。

常见做法：

1. 创建临时顺序节点：`/lock/lock-00000001`。
2. 判断自己是否最小序号：是则持锁。
3. 否则 watch 前一个节点（predecessor），等待它删除后再判断。

### 4.2 ZooKeeper 的坑与注意点

- 羊群效应：如果所有人都 watch 同一个节点，节点删除会唤醒大量客户端，造成抖动。正确做法是 watch 前驱节点。
- 会话抖动：网络闪断可能导致 session 过期，临时节点被删除，持有者必须立刻停止临界区或依赖 fencing。
- 读写语义：ZK 的一致性与通知语义要结合 session 与 zxid 理解，不能把 watch 当成可靠消息队列。

适用场景：

- 强一致协调、选主、分布式锁等（很多系统的元数据协调层）。

## 5. 用 Redis 实现分布式锁：SET NX PX + Lua 释放

### 5.1 单实例 Redis 的最小正确姿势

加锁：

- `SET key token NX PX ttl_ms`

解锁必须校验 token，建议 Lua 原子脚本：

```lua
-- 只删除“自己持有”的锁，避免删错
if redis.call("GET", KEYS[1]) == ARGV[1] then
  return redis.call("DEL", KEYS[1])
else
  return 0
end
```

### 5.2 Redis 的坑与注意点

- 主从复制异步：主挂了但锁写入未复制，故障切换后锁可能“丢失”，导致多个客户端都以为自己拿到锁。
- 续租与暂停：客户端 GC 或网络抖动导致锁过期，旧持有者继续执行，必须配合 fencing 或把临界区设计为可重入幂等。
- Redlock 争议：多实例多数派加锁方案在某些时钟漂移与网络模型下存在安全性争议，面试中建议谨慎表述“适用前提与风险”，不要一口咬定“绝对安全”。

适用场景：

- 对性能敏感、可容忍偶发并发进入且有业务兜底的场景（例如缓存重建、非关键任务互斥）。

## 6. etcd / ZooKeeper / Redis 对比表（面试高频）

| 方案 | 正确性基础 | 自动释放 | 典型优势 | 典型风险 |
| --- | --- | --- | --- | --- |
| etcd | Raft 共识的线性一致 KV + txn | Lease | 语义清晰、强一致、易做 fencing（revision） | 热点锁压力、续租失败需处理 |
| ZooKeeper | Zab 协调 + 会话语义 + 临时节点 | 会话过期删除 | 顺序节点实现公平锁、生态成熟 | watch 羊群效应、会话抖动 |
| Redis | 单机原子命令 + TTL | TTL | 性能好、实现简单 | 主从切换一致性、锁过期并发风险 |

## 7. 面试回答模板（30 秒）

1. 先讲目标：互斥、自动释放、避免误删与避免锁过期并发。
2. etcd/ZK 更偏强一致协调（CP），更适合关键互斥；Redis 更偏性能与可用性，需要业务兜底。
3. 真正的工程防线是 token 校验 + TTL/续租 + **fencing token**，否则锁过期与暂停会破坏互斥。
