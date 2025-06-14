
#### 概述
- **定义**：
  - Redis 是一个内存数据库，当存储的数据量超过可用内存时，会触发 **内存淘汰机制**，根据配置的策略删除部分键值对以释放空间。
  - 内存淘汰通过配置 `maxmemory` 和 `maxmemory-policy` 实现，避免内存溢出（OOM）。
- **核心点**：
  - Redis 提供 **8 种内存淘汰策略**，分为 **随机淘汰**、**LRU（最近最少使用）**、**LFU（最不经常使用）** 和 **TTL（过期时间）** 相关策略。
  - 选择淘汰策略需根据业务场景（如缓存、持久化存储）、数据访问模式（热点数据、均匀访问）和性能要求。

---

### 1. Redis 内存淘汰的触发条件
- **内存限制**：
  - 配置 `maxmemory` 参数，限制 Redis 使用的最大内存（单位：字节）。
  - 示例：
    ```bash
    CONFIG SET maxmemory 2gb
    ```
  - 若未设置 `maxmemory`，Redis 将使用所有可用内存，可能导致 OOM。
- **触发时机**：
  - 当 Redis 内存使用量接近或超过 `maxmemory` 时，触发淘汰。
  - 每次写入操作（如 `SET`, `LPUSH`）检查内存，若超限，执行淘汰策略删除键，直到内存低于 `maxmemory`。
- **内存监控**：
  - 查看内存使用情况：
    ```bash
    INFO MEMORY
    ```
    输出示例：
    ```
    used_memory: 1500000000
    maxmemory: 2000000000
    ```

---

### 2. Redis 内存淘汰策略
Redis 提供以下 8 种内存淘汰策略（`maxmemory-policy`），通过 `CONFIG SET maxmemory-policy <policy>` 配置：

#### (1) noeviction（不淘汰）
- **描述**：
  - 不主动淘汰任何键，当内存不足时，新写入操作（如 `SET`）失败并返回错误。
- **适用场景**：
  - Redis 作为持久化存储（如数据库），数据不可丢失。
  - 数据量可控，不会超过 `maxmemory`。
- **缺点**：
  - 内存超限后无法写入，影响服务可用性。
- **示例**：
  ```bash
  CONFIG SET maxmemory-policy noeviction
  ```

#### (2) allkeys-lru（全局 LRU）
- **描述**：
  - 从所有键中选择 **最近最少使用（Least Recently Used）** 的键进行淘汰。
  - 使用近似 LRU 算法（随机采样，默认 5 个样本，`maxmemory-samples` 可调）。
- **适用场景**：
  - Redis 作为缓存，热点数据频繁访问，冷数据可淘汰。
  - 数据访问有明显热点（如 80/20 法则）。
- **优点**：
  - 保留高频访问数据，适合缓存场景。
- **缺点**：
  - 可能误删重要但访问频率低的键。
- **示例**：
  ```bash
  CONFIG SET maxmemory-policy allkeys-lru
  ```

#### (3) volatile-lru（带过期时间的 LRU）
- **描述**：
  - 从设置了过期时间（`EXPIRE`）的键中选择最近最少使用的键淘汰。
  - 未设置过期时间的键不会被淘汰。
- **适用场景**：
  - Redis 混合使用（部分键为缓存，带 TTL；部分键为持久化，无 TTL）。
  - 希望保护无过期时间的数据。
- **优点**：
  - 仅淘汰临时数据，保护持久化键。
- **缺点**：
  - 若带过期时间的键不足，可能无法释放足够内存。
- **示例**：
  ```bash
  CONFIG SET maxmemory-policy volatile-lru
  ```

#### (4) allkeys-random（全局随机）
- **描述**：
  - 从所有键中随机选择键进行淘汰。
- **适用场景**：
  - 数据访问均匀，无明显热点（如临时数据、测试环境）。
  - 对淘汰顺序不敏感。
- **优点**：
  - 实现简单，开销低。
- **缺点**：
  - 可能删除高频访问的键，降低缓存命中率。
- **示例**：
  ```bash
  CONFIG SET maxmemory-policy allkeys-random
  ```

#### (5) volatile-random（带过期时间的随机）
- **描述**：
  - 从设置了过期时间的键中随机选择键淘汰。
- **适用场景**：
  - 带 TTL 的键为临时数据，可随机删除。
  - 混合存储场景，需保护无过期时间键。
- **优点**：
  - 简单高效，保护持久化数据。
- **缺点**：
  - 随机性可能影响缓存效率。
- **示例**：
  ```bash
  CONFIG SET maxmemory-policy volatile-random
  ```

#### (6) volatile-ttl（按剩余 TTL 淘汰）
- **描述**：
  - 从设置了过期时间的键中选择 **剩余存活时间（TTL）最短** 的键淘汰。
- **适用场景**：
  - 键的过期时间有明确意义，优先删除即将过期的数据。
  - 缓存场景，TTL 表示数据时效性。
- **优点**：
  - 优先淘汰无用数据，延长有效数据存活。
- **缺点**：
  - 若带 TTL 键不足，可能无法释放内存。
- **示例**：
  ```bash
  CONFIG SET maxmemory-policy volatile-ttl
  ```

#### (7) allkeys-lfu（全局 LFU）（Redis 4.0+）
- **描述**：
  - 从所有键中选择 **最不经常使用（Least Frequently Used）** 的键淘汰。
  - 使用近似 LFU 算法，基于访问频率计数（`LOGARITHMIC COUNTER`）。
- **适用场景**：
  - 缓存场景，数据访问频率差异大，需保留高频访问键。
  - 比 LRU 更适合长期热点数据。
- **优点**：
  - 精准保留高频数据，提升命中率。
- **缺点**：
  - LFU 计数需额外内存，维护成本略高。
- **示例**：
  ```bash
  CONFIG SET maxmemory-policy allkeys-lfu
  ```

#### (8) volatile-lfu（带过期时间的 LFU）（Redis 4.0+）
- **描述**：
  - 从设置了过期时间的键中选择最不经常使用的键淘汰。
- **适用场景**：
  - 混合存储，带 TTL 的键为缓存，需基于访问频率淘汰。
  - 保护无 TTL 的持久化数据。
- **优点**：
  - 结合 LFU 和 TTL，适合复杂缓存场景。
- **缺点**：
  - 同 `volatile-lru`，若带 TTL 键不足，可能内存不足。
- **示例**：
  ```bash
  CONFIG SET maxmemory-policy allkeys-lfu
  ```

---

### 3. 内存淘汰策略选择
选择淘汰策略需综合考虑 **业务场景**、**数据特性** 和 **性能要求**：

#### (1) 业务场景
- **Redis 作为缓存**：
  - **推荐**：`allkeys-lru` 或 `allkeys-lfu`。
  - 理由：全局淘汰，保留热点数据（LRU 适合近期热点，LFU 适合长期热点）。
  - 示例：Web 缓存、热点商品信息。
- **Redis 作为持久化存储**：
  - **推荐**：`noeviction`。
  - 理由：避免数据丢失，需确保内存充足。
  - 示例：配置数据、会话存储。
- **混合场景（缓存 + 持久化）**：
  - **推荐**：`volatile-lru` 或 `volatile-lfu`。
  - 理由：仅淘汰带 TTL 的缓存键，保护无 TTL 的持久化键。
  - 示例：缓存热点数据，持久化用户配置。
- **临时数据，TTL 重要**：
  - **推荐**：`volatile-ttl`。
  - 理由：优先淘汰即将过期的数据，延长有效数据存活。
  - 示例：验证码、临时令牌。

#### (2) 数据访问模式
- **热点数据明显（80/20 法则）**：
  - 推荐 `allkeys-lru` 或 `allkeys-lfu`。
  - LRU 适合短期热点，LFU 适合长期高频。
- **访问均匀，无明显热点**：
  - 推荐 `allkeys-random` 或 `volatile-random`。
  - 随机淘汰开销低，适合无序数据。
- **TTL 差异大**：
  - 推荐 `volatile-ttl`。
  - 优先删除快过期的数据。

#### (3) 性能与开销
- **低开销**：
  - `allkeys-random`、`volatile-random`：随机选择，计算量小。
- **高命中率**：
  - `allkeys-lru`、`allkeys-lfu`：基于访问模式，命中率高，但需维护 LRU/LFU 计数。
  - `volatile-lru`、`volatile-lfu`：同上，但限制在 TTL 键。
- **精确淘汰**：
  - `volatile-ttl`：按 TTL 排序，适合时效性数据。
- **无淘汰**：
  - `noeviction`：零开销，但内存不足时拒绝写入。

#### (4) 配置建议
- **maxmemory**：
  - 设为物理内存的 60%-80%，留空间给系统和缓冲。
  - 示例：16GB 机器，设 `maxmemory 12gb`。
- **maxmemory-samples**（LRU/LFU 采样数）：
  - 默认 5，增加到 10 可提高 LRU/LFU 精度，但增加开销。
  - 示例：
    ```bash
    CONFIG SET maxmemory-samples 10
    ```
- **TTL 使用**：
  - 尽量为缓存键设置 `EXPIRE`，配合 `volatile-*` 策略。
  - 示例：
    ```bash
    SET cache:key value EX 3600
    ```

---

### 4. 内存淘汰的工作原理
- **触发**：
  - 写入操作（如 `SET`, `LPUSH`）时，检查 `used_memory` 是否超过 `maxmemory`。
  - 若超限，调用淘汰算法。
- **LRU/LFU 实现**：
  - Redis 使用 **近似算法**，随机采样 N 个键（默认 5），选择最旧（LRU）或最不频繁（LFU）的键。
  - LRU：基于键的最后访问时间（`lru_clock`）。
  - LFU：基于访问频率计数（`lfu_log_factor` 控制衰减）。
- **随机淘汰**：
  - 直接随机选择键，效率高但命中率低。
- **TTL 淘汰**：
  - 比较带 TTL 键的剩余时间，选择最小者。
- **流程**：
  1. 检查内存是否超限。
  2. 根据策略选择候选键。
  3. 删除键，释放内存。
  4. 重复直到内存低于 `maxmemory`。

---

### 5. 代码与配置示例
```java
import redis.clients.jedis.Jedis;

public class RedisEvictionExample {
    public static void main(String[] args) {
        Jedis jedis = new Jedis("localhost", 6379);

        // 设置最大内存 100MB
        jedis.configSet("maxmemory", "100mb");

        // 设置淘汰策略为 allkeys-lru
        jedis.configSet("maxmemory-policy", "allkeys-lru");

        // 设置 LRU 采样数
        jedis.configSet("maxmemory-samples", "10");

        // 插入数据
        for (int i = 0; i < 100000; i++) {
            jedis.set("key:" + i, "value:" + i);
            if (i % 1000 == 0) {
                System.out.println("Inserted " + i + " keys");
            }
        }

        // 检查内存使用
        String info = jedis.info("memory");
        System.out.println(info);

        jedis.close();
    }
}
```

**配置文件（redis.conf）**：
```conf
maxmemory 2gb
maxmemory-policy allkeys-lru
maxmemory-samples 10
```

**查看当前策略**：
```bash
CONFIG GET maxmemory-policy
CONFIG GET maxmemory
```

---

### 6. 注意事项
- **内存规划**：
  - 设置 `maxmemory` 小于物理内存，留空间给 Redis 管理开销（如复制缓冲区）。
  - 监控内存使用：
    ```bash
    INFO MEMORY
    ```
- **TTL 使用**：
  - 为缓存键设置合理 TTL（如 1 小时），减少手动淘汰压力。
  - 示例：
    ```bash
    SETEX cache:key 3600 value
    ```
- **策略选择**：
  - 避免 `noeviction` 导致写入失败，除非数据不可丢。
  - 优先 `allkeys-lru`/`allkeys-lfu` 确保缓存命中率。
- **性能影响**：
  - LRU/LFU 淘汰需采样和比较，增加 CPU 开销。
  - 增大 `maxmemory-samples` 提高精度，但降低性能。
- **数据丢失风险**：
  - `allkeys-*` 策略可能删除持久化数据，需评估业务影响。
  - `volatile-*` 策略需确保足够带 TTL 键。
- **高可用**：
  - 主从复制中，主节点淘汰可能导致从节点数据不一致。
  - 集群模式下，各节点独立淘汰，需统一配置 `maxmemory`。
- **碎片管理**：
  - 淘汰可能导致内存碎片，定期重启或使用 `MEMORY PURGE` 清理。
    ```bash
    MEMORY PURGE
    ```

---

### 7. 面试角度
- **问“Redis 内存淘汰策略有哪些”**：
  - 提 8 种：`noeviction`, `allkeys-lru`, `volatile-lru`, `allkeys-random`, `volatile-random`, `volatile-ttl`, `allkeys-lfu`, `volatile-lfu`。
- **问“如何选择淘汰策略”**：
  - 提缓存用 `allkeys-lru`/`allkeys-lfu`，持久化用 `noeviction`，混合用 `volatile-lru`/`volatile-lfu`，TTL 重要用 `volatile-ttl`。
- **问“LRU vs LFU 区别”**：
  - 提 LRU 基于最近访问时间，适合短期热点；LFU 基于访问频率，适合长期高频。
- **问“volatile-lru 的局限”**：
  - 提仅淘汰带 TTL 键，若 TTL 键不足，可能内存不足。
- **问“如何监控内存淘汰”**：
  - 提 `INFO MEMORY` 查看 `used_memory`、`evicted_keys`，设置 `maxmemory` 和 `maxmemory-policy`。
- **问“如何避免内存溢出”**：
  - 提设置 `maxmemory`，用有界数据结构（如 List 裁剪），定期清理过期键，优化淘汰策略。

---

### 8. 总结
- **Redis 内存淘汰策略**：
  - **noeviction**：不淘汰，拒绝写入。
  - **allkeys-lru/lfu**：全局 LRU/LFU，适合缓存。
  - **volatile-lru/lfu**：带 TTL 的 LRU/LFU，适合混合场景。
  - **allkeys-random/volatile-random**：随机淘汰，简单高效。
  - **volatile-ttl**：按 TTL 淘汰，适合时效性数据。
- **选择依据**：
  - **缓存**：`allkeys-lru`（短期热点）或 `allkeys-lfu`（长期高频）。
  - **持久化**：`noeviction`。
  - **混合**：`volatile-lru`/`volatile-lfu`。
  - **TTL 优先**：`volatile-ttl`。
- **实现原理**：
  - 内存超 `maxmemory` 触发，LRU/LFU 使用近似采样，随机和 TTL 直接选择。
- **面试建议**：
  - 提 8 种策略、适用场景（缓存 vs 持久化）、选择依据（热点 vs TTL）、配置（`maxmemory`、`maxmemory-samples`），举例（`CONFIG SET`），清晰展示理解。
