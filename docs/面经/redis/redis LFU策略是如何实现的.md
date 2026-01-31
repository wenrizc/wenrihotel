## 1. LFU 在 Redis 内存淘汰中的位置

Redis 的 LFU（Least Frequently Used，最不经常使用）用于内存淘汰（eviction）阶段：当 `maxmemory` 达到上限且策略选择了 LFU 时，Redis 会优先删除“访问频率最低”的 key，从而尽量保留长期热点。

LFU 对应的策略是：

- `allkeys-lfu`：从所有 key 中按 LFU 淘汰。
- `volatile-lfu`：只从设置了过期时间（TTL，Time To Live）的 key 中按 LFU 淘汰。

需要强调：Redis 的 LFU 不是“精确计数的纯 LFU”，而是 **近似 LFU + 时间衰减** 的实现。它解决的核心痛点是：在存在批量扫描或一次性访问时，LRU 很容易被“最近访问”污染，而 LFU 更能识别长期热点。

## 2. Redis LFU 的核心思想：近似 + 衰减

### 2.1 为什么不是精确 LFU

精确 LFU 需要为每个 key 维护一个可能很大的访问计数，并且每次访问都要更新计数；淘汰时还要在全量 keyspace 上找到最小频次的 key。对于百万级 key 与高 QPS 的 Redis 来说，这会带来明显的内存与 CPU 负担。

Redis 的策略是把“每次访问更新成本”压到极低，把“淘汰时全量扫描”替换为随机采样，从而在工程上达到可接受的性价比。

### 2.2 为什么必须做时间衰减

如果只做“累积频次”，一个 key 可能在很久以前被大量访问过，之后长期不再访问，但它的频次仍然很高，导致一直难以被淘汰。这与缓存场景的目标不一致：我们更希望保留“近期仍然高频”的数据。

因此 Redis 的 LFU 引入衰减：**频次会随时间降低**，让“过气热点”最终回到可淘汰状态。

## 3. LFU 元数据：复用 `robj.lru` 的 24 位

Redis 的对象结构 `robj` 有一个 24 bit 的字段（历史名字是 `lru`）。在 LFU 策略下，这个字段不再表示“最近访问时钟”，而是编码为“时间 + 计数器”。

### 3.1 位布局：16 位时间 + 8 位计数

常见实现是把 24 bit 拆成两段：

- 高 16 bit：最近一次衰减时间（以分钟为粒度的滚动时钟）。
- 低 8 bit：LFU 计数器（`0 ~ 255`，饱和递增）。

可以用位布局表示：

| 字段 | 位宽 | 含义 |
| --- | --- | --- |
| `lfu_time` | 16 bit | 记录“上次衰减计算的时间”，用于计算衰减步数 |
| `lfu_count` | 8 bit | 记录“访问频率”的近似值（对数递增 + 衰减） |

这种设计的好处是：单 key 元数据开销非常小，且能同时表达“频次”和“时间维度”。

### 3.2 初始化与上限：`LFU_INIT_VAL` 与饱和

Redis 通常会给新 key 一个非零的初始频次（常见值是 `LFU_INIT_VAL = 5`），避免新写入的 key 在内存压力下立刻被淘汰（缓存抖动）。

计数器是 8 bit：

- 递增会在 `255` 饱和（不会溢出回 0）。
- 衰减会逐步把计数降到 `0`（不会降成负数）。

## 4. 计数如何增长：对数递增（`lfu-log-factor`）

Redis 不会在每次访问时都把 `lfu_count` 直接 `+1`，因为 8 bit 计数器很快就会打满，无法区分“更热”与“更冷”。它采用的是“对数递增”思想：**越热的 key，计数越难继续增长**。

### 4.1 概率递增函数的直觉解释

直觉上可以理解为：当 `lfu_count` 较小时，访问一次更容易让计数上涨；当 `lfu_count` 很大时，需要更多次访问才可能继续上涨。

常见实现是用一个概率函数控制是否递增（伪代码表达核心思路）：

```c
// 计数越大，p 越小；lfu_log_factor 越大，增长越慢
double base = max(0, (double)count - LFU_INIT_VAL);
double p = 1.0 / (base * lfu_log_factor + 1);
if (random01() < p && count < 255) {
    count++;
}
```

`lfu-log-factor` 是可配置参数，用来调整“增长曲线”：

- 值越大：计数增长越慢，更强调“长期稳定热点”。
- 值越小：计数增长更快，更敏感于短期访问高峰。

### 4.2 访问路径：`updateLFU` 何时被调用

LFU 元数据的维护发生在 key 被访问时，通常是在读/写路径上的查找函数中完成。也就是说：**只有真实访问会更新频次**，Redis 不会周期性地遍历全量 key 去更新计数。

与 LRU 类似，少数内部访问会使用 `LOOKUP_NOTOUCH` 这类标志，避免把“非业务访问”计入 LFU。

## 5. 计数如何衰减：惰性衰减（`lfu-decay-time`）

LFU 的时间衰减是为了让“曾经很热但现在冷了”的 key 逐步变得可淘汰。Redis 采用的是惰性衰减：**不做后台全量扫描，只在访问或采样评估时计算衰减**。

### 5.1 衰减计算与时间回绕

衰减通常以分钟为单位：

- 当前时间取一个 16 bit 的分钟时钟 `now`（会回绕）。
- 根据 `elapsed = now - lfu_time` 计算经过了多少分钟（用模运算处理回绕）。
- 每经过 `lfu-decay-time` 分钟，计数衰减 1（最多衰减到 0）。

伪代码如下：

```c
uint16_t now = LFU_TIME_MINUTES();                 // 16 位分钟时钟
uint16_t last = (uint16_t)(o->lru >> 8);           // lfu_time
uint8_t  cnt  = (uint8_t)(o->lru & 0xff);          // lfu_count

uint16_t elapsed = (uint16_t)(now - last);         // 处理 16 位回绕
unsigned int periods = elapsed / lfu_decay_time;   // 每个周期衰减 1
cnt = (periods >= cnt) ? 0 : (cnt - periods);

o->lru = ((uint32_t)now << 8) | cnt;               // 写回时间与计数
```

### 5.2 `lfu-decay-time = 0` 的语义

如果把 `lfu-decay-time` 设为 `0`，表示关闭衰减：计数只增不减。此时 LFU 会更接近“累计频次”，但更容易出现“过气热点常驻”的问题，一般不建议在缓存场景中关闭衰减。

## 6. 淘汰时如何选择 key：采样 + 频率优先

LFU 淘汰阶段不会扫描所有 key，而是走与 LRU 相同的框架：**随机采样 + 淘汰池（eviction pool）**。区别在于候选评分规则。

### 6.1 评分规则：先 `freq`，再 `idle`

LFU 的核心排序逻辑可以理解为一个二元组：

1. 主排序键：衰减后的 `lfu_count`（越小越应该被淘汰）。
2. 次排序键：访问时间（当频次相同时，越久没访问越应该被淘汰）。

因此 LFU 不是单纯“只看频次”，而是频次优先、时间兜底，避免出现“两个同频 key 随机淘汰”的抖动。

### 6.2 `maxmemory-samples` 与淘汰池对精度的影响

影响淘汰质量的关键参数仍然是：

- `maxmemory-samples`：每轮采样数量。越大越接近真实 LFU，但 CPU 开销越高。
- 淘汰池（固定大小 TopK）：多轮保留更差候选，提高近似质量。

另外，`volatile-lfu` 与 `allkeys-lfu` 的差别与 LRU 一致：前者只从“带 TTL 的 key 集合”采样，后者从全量 keyspace 采样。

## 7. LFU 与 LRU 的对比与选型

| 维度 | LRU | LFU |
| --- | --- | --- |
| 关注点 | 最近访问（recency） | 访问频率（frequency）+ 时间衰减 |
| 抗扫描污染 | 较弱 | 较强 |
| 适合场景 | 热点变化快、短期热点明显 | 长期热点明显、希望抵抗一次性访问干扰 |
| 关键参数 | `maxmemory-samples` | `maxmemory-samples`、`lfu-log-factor`、`lfu-decay-time` |

选型经验：如果业务经常出现批量扫库、全量预热、离线任务穿透等访问模式，LFU 往往比 LRU 更稳；如果热点切换很快，LRU 可能更敏感、更“跟手”。

## 8. 调优与排查

### 8.1 参数调节：`lfu-log-factor` / `lfu-decay-time`

- `lfu-log-factor`：控制增长速度。过大可能让新热点“爬升太慢”，过小可能让一次性访问把计数冲高。
- `lfu-decay-time`：控制遗忘速度。过大可能导致“过气热点”长期不被淘汰，过小可能导致热点需要持续访问才能维持计数。

建议按业务访问节奏调：热点持续时间越短，衰减时间越小；热点更稳定，衰减时间可适当增大。

### 8.2 诊断命令：`OBJECT FREQ` / `OBJECT IDLETIME`

Redis 提供了 `OBJECT FREQ key` 用于查看 LFU 计数器（只有在 LFU 相关策略启用时更有参考价值），也可以结合 `OBJECT IDLETIME key` 观察“同频 key 的时间差”：

```shell
OBJECT FREQ some:key
OBJECT IDLETIME some:key
```

当你发现“LFU 看起来不符合预期”，优先排查：

- 是否真的启用了 `allkeys-lfu` / `volatile-lfu`。
- 是否存在大量一次性访问造成计数爬升（调整 `lfu-log-factor`）。
- 是否衰减太慢导致旧热点常驻（调整 `lfu-decay-time`）。

### 8.3 常用配置与检查命令

```shell
CONFIG GET maxmemory
CONFIG GET maxmemory-policy
CONFIG GET maxmemory-samples
CONFIG GET lfu-log-factor
CONFIG GET lfu-decay-time
```

