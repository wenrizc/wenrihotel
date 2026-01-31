## 1. LRU 在 Redis 内存淘汰中的位置

Redis 的 LRU（Least Recently Used，最近最少使用）不是一个独立功能，它只在触发内存淘汰（eviction）时生效。也就是说：只有当 `maxmemory` 达到上限且当前 `maxmemory-policy` 选择了 LRU 相关策略时，Redis 才会按 LRU 规则选 key 删除。

LRU 对应的策略是：

- `allkeys-lru`：从所有 key 中按 LRU 淘汰。
- `volatile-lru`：只从设置了过期时间（TTL，Time To Live）的 key 中按 LRU 淘汰。

内存淘汰与过期删除是两条链路：过期删除解决“key 到点消失”，LRU 淘汰解决“内存不够要挪位置”。两者不要混淆。

## 2. 为什么 Redis 实现的是“近似 LRU”

Redis 对外提供的是 LRU 语义，但实现上是 **近似 LRU（approximate LRU）**，核心原因是成本权衡：Redis 要在“每次访问都更新元数据”的前提下，把额外 CPU 与内存开销控制在可接受范围。

### 2.1 精确 LRU 的成本

如果要实现精确 LRU，典型做法是维护一个全局双向链表：每次访问都把节点移动到表头，淘汰时从表尾删除。这会带来两个直接问题：

- 每次访问都要改链表指针，写放大明显；高 QPS 下会放大主线程开销。
- 需要为每个 key 维护链表节点，内存开销随 key 数线性增长（缓存场景里 key 往往是百万级）。

### 2.2 近似 LRU 的收益

Redis 用“时间戳 + 随机采样”的方式近似 LRU：

- 每个对象只记录一个小字段表示“最近访问时间”，写入开销是 O(1)。
- 淘汰时不全量扫描 keyspace，而是随机采样少量候选，再从中选“最久未访问”的。

这样在大多数热点分布明显的场景下，效果接近 LRU，但开销远低于精确实现。

## 3. LRU 元数据：`robj.lru` 24 位时钟

Redis 的对象结构 `robj` 里有一个用于淘汰策略的字段（历史上叫 `lru`）。在 LRU 策略下，它用来记录对象的“最近访问时间”，但注意它不是 Unix 时间戳，而是一个 **24 位滚动时钟**。

### 3.1 LRU 时钟如何生成：`server.lruclock` 与回绕

Redis 维护一个全局 LRU 时钟（常见实现是按秒级更新）：`server.lruclock` 会周期性刷新，取值范围是 `0 ~ (2^24 - 1)`。

这意味着：

- `robj.lru` 只占 24 bit，单 key 元数据很小。
- 时钟会回绕（wrap around）。以秒级计时为例，`2^24` 秒约 194 天；因此计算“空闲时长”必须用模运算处理回绕。

### 3.2 访问时如何更新：`lookupKey*` 的 touch 逻辑

当客户端访问一个 key（读或写路径上的查找）时，Redis 会在返回对象前更新它的 LRU 元数据。源码层面通常由 `lookupKeyReadWithFlags` / `lookupKeyWriteWithFlags` 这类函数完成。

极简伪代码如下：

```c
// 注意：这里记录的是 24 位滚动时钟，而不是毫秒时间戳
if (!(flags & LOOKUP_NOTOUCH)) {
    o->lru = LRU_CLOCK();
}
```

`LOOKUP_NOTOUCH` 这类标志用于少数“不要影响淘汰元数据”的内部访问（例如某些只做检查或统计的读取），避免无意义地污染 LRU。

### 3.3 如何计算空闲时间：`estimateObjectIdleTime`

淘汰时需要比较“多久没被访问”。Redis 通过当前时钟与对象时钟的差值估算空闲时间（idle time），并处理回绕：

```c
unsigned int now = LRU_CLOCK();
unsigned int idle = (now - o->lru) & LRU_CLOCK_MAX; // 处理 24 位回绕
```

对于 LRU 来说，**idle 越大，表示越久未被访问，越应该被淘汰**。

## 4. 淘汰算法主线：采样 + 淘汰池

LRU 的“实现难点”不在于记录时间戳，而在于“内存满了时如何快速找出最该淘汰的 key”。Redis 的主线是：**随机采样少量 key → 计算 idle → 选出 idle 最大的作为淘汰候选**。

### 4.1 触发时机：`maxmemory` 与 `freeMemoryIfNeeded`

当 Redis 发现内存使用超过 `maxmemory`，并且当前命令可能继续分配内存时，会进入淘汰流程：

- 若策略是 `noeviction`：直接对写入类命令返回错误。
- 若策略允许淘汰：执行 `freeMemoryIfNeeded` 之类流程，循环淘汰 key，直到内存回到阈值以下或无法继续淘汰。

### 4.2 采样范围：`allkeys-lru` vs `volatile-lru`

采样集合由策略决定：

- `allkeys-lru`：从 `db->dict`（全量 keyspace）随机取样。
- `volatile-lru`：从 `db->expires`（带 TTL 的 key 集合）随机取样。

如果选了 `volatile-lru`，但当前没有任何带 TTL 的 key，那么淘汰阶段“无 key 可删”，行为会退化为类似 `noeviction`：写入会报错。

### 4.3 淘汰池：`EVPOOL_SIZE` 与插入排序规则

直接“采样 N 个 key，然后选 idle 最大的”仍然有随机性。Redis 进一步引入一个固定大小的淘汰池（eviction pool，常见大小为 `EVPOOL_SIZE = 16`）：

- 每轮从 keyspace 采样 `maxmemory-samples` 个 key。
- 把候选按 idle 值插入淘汰池（池内保持有序）。
- 真正淘汰时，优先从淘汰池取“最该淘汰”的条目（LRU 下就是 idle 最大）。

这相当于“多轮采样 + TopK 保留”，用很小的额外成本显著提升近似 LRU 的准确率。

### 4.4 `maxmemory-samples` 对准确率的影响

`maxmemory-samples` 决定每轮采样数量（默认通常是 `5`）：

- 采样数越大，越接近真实 LRU，但每次淘汰的 CPU 开销越高。
- 采样数越小，开销更低，但随机性更大，淘汰质量可能波动。

工程上通常从默认值开始，结合 `evicted_keys`、延迟与命中率变化进行调参。

## 5. 与过期删除（TTL）的关系：两条不同链路

LRU 淘汰只在“内存压力”下触发；过期删除在“TTL 到期”时触发（惰性删除 + 定期删除）。两者的交互点在于：

- `volatile-lru` 把淘汰范围限制在“带 TTL 的 key”，因此业务上常用“缓存 key 都设置 TTL”来保护非缓存数据不被淘汰。
- `allkeys-lru` 会淘汰未过期甚至无 TTL 的 key，因此更像“纯缓存模式”。

## 6. 工程实践与调优建议

### 6.1 避免“扫描污染”

LRU 只看“最近访问”，因此对“批量扫库式访问”敏感：一次性访问大量冷数据，会把它们的 `robj.lru` 刷新成“最近使用”，导致真正的热点 key 反而更容易被淘汰。

常见应对：

- 如果业务存在明显的批量扫描场景，优先考虑 `allkeys-lfu`（LFU 对扫描污染更不敏感）。
- 必要时提高 `maxmemory-samples`，让淘汰更接近真实 LRU（但要评估 CPU 开销）。

### 6.2 大 key 释放与延迟抖动

淘汰本质是删除 key。删除大对象（大 list、big hash、超大 string）可能产生明显的释放成本，影响主线程延迟。

工程上可以结合 Redis 的 lazyfree 相关配置（例如 `lazyfree-lazy-eviction`）降低同步释放带来的抖动，并在设计阶段避免大 key。

### 6.3 观测指标与验证方法

- `INFO stats`：关注 `evicted_keys`、延迟指标与命中率变化。
- `OBJECT IDLETIME key`：可用于理解某个 key 的空闲时间（其精度与维护策略相关，且是估算值）。
- 配合压测：在不同 `maxmemory-samples` 下观察“淘汰时延”和“命中率”曲线，找到业务可接受的折中点。

### 6.4 常用配置与检查命令

```shell
CONFIG GET maxmemory
CONFIG GET maxmemory-policy
CONFIG GET maxmemory-samples
```

## 7. 总结

- Redis 的 LRU 是“24 位访问时钟 + 随机采样 + 淘汰池”的近似实现，不做精确全局链表维护。
- `allkeys-lru` 从全量 keyspace 淘汰，`volatile-lru` 只淘汰带 TTL 的 key。
- `maxmemory-samples` 决定近似质量与 CPU 开销；批量扫描场景下 LRU 可能被污染，必要时改用 LFU 或调整参数。
