
#### Redis 使用场景概述
- **定义**：
  - Redis（Remote Dictionary Server）是一个开源的、内存-based的高性能键值存储数据库，支持多种数据结构（如字符串、哈希、列表、集合、有序集合等），以其低延迟和高吞吐量著称。
  - Redis 提供持久化、事务、发布/订阅、Lua 脚本等功能，广泛应用于分布式系统。
- **核心点**：
  - Redis 适用于需要**高性能**、**低延迟**、**简单键值存储**或**复杂数据结构**的场景，常见于缓存、会话管理、排行榜、计数器、消息队列等。

---

### 1. Redis 的主要使用场景
以下是 Redis 的典型使用场景，结合其数据结构和特性：

#### (1) 缓存（Cache）
- **描述**：
  - Redis 作为缓存层，存储热点数据，减少数据库（如 MySQL）压力，提升响应速度。
- **适用场景**：
  - 频繁查询但更新较少的数据，如商品信息、文章内容、配置数据。
  - Web 应用缓存：加速页面加载（如电商商品详情）。
- **数据结构**：
  - **String**：存储 JSON 或序列化对象。
  - **Hash**：存储结构化对象（如用户信息）。
- **特性**：
  - 高性能：内存操作，单线程模型，读写延迟低（<1ms）。
  - 过期机制：通过 `EXPIRE` 设置 TTL（Time-To-Live）。
- **示例**：
  ```bash
  # 缓存商品信息，TTL 1小时
  SET product:123 "{\"id\":123, \"name\":\"Phone\", \"price\":999}" EX 3600
  GET product:123
  ```
- **实现**：
  - Spring Boot 集成 `Spring Cache` + Redis：
    ```java
    @Cacheable(value = "products", key = "#id")
    public Product getProduct(int id) {
        return productRepository.findById(id);
    }
    ```

#### (2) 会话管理（Session Store）
- **描述**：
  - 存储用户会话数据（如登录状态、购物车），支持分布式系统中的会话共享。
- **适用场景**：
  - Web 应用的分布式会话管理（如 Spring Session）。
  - 单点登录（SSO）系统。
- **数据结构**：
  - **Hash**：存储会话属性（如 `user_id`, `token`）。
  - **String**：存储简单会话 ID 和值。
- **特性**：
  - 高并发：支持高频读写。
  - 过期机制：自动清理过期会话。
- **示例**：
  ```bash
  # 存储用户会话，TTL 30分钟
  HMSET session:abc123 user_id 1001 token xyz123
  EXPIRE session:abc123 1800
  HGETALL session:abc123
  ```
- **实现**：
  - Spring Session 配置 Redis 存储：
    ```yaml
    spring:
      session:
        store-type: redis
        timeout: 1800s
    ```

#### (3) 排行榜（Leaderboard）
- **描述**：
  - 使用有序集合（Sorted Set）实现实时排行榜，按分数排序。
- **适用场景**：
  - 游戏排行榜（如玩家分数）。
  - 社交平台（如热门帖子、用户积分）。
- **数据结构**：
  - **Sorted Set**：存储元素和分数，支持排名查询。
- **特性**：
  - 高效排序：`ZRANK`、`ZRANGE` 操作 O(log N)。
  - 实时更新：支持增量更新分数。
- **示例**：
  ```bash
  # 添加玩家分数
  ZADD leaderboard 1000 player1 2000 player2
  # 获取排名前3
  ZREVRANGE leaderboard 0 2 WITHSCORES
  # 获取 player1 排名
  ZRANK leaderboard player1
  ```
- **实现**：
  - Java Jedis 操作：
    ```java
    Jedis jedis = new Jedis("localhost");
    jedis.zadd("leaderboard", 1000, "player1");
    jedis.zrevrangeWithScores("leaderboard", 0, 2);
    ```

#### (4) 计数器（Counter）
- **描述**：
  - 使用原子操作（如 `INCR`）实现高并发计数，适合统计点击量、点赞数等。
- **适用场景**：
  - 文章阅读量、商品浏览量。
  - 社交媒体点赞、评论计数。
- **数据结构**：
  - **String**：存储计数器值。
- **特性**：
  - 原子性：`INCR`、`DECR` 避免并发冲突。
  - 高性能：单命令操作，延迟低。
- **示例**：
  ```bash
  # 文章浏览量 +1
  INCR article:123:views
  GET article:123:views
  ```
- **实现**：
  - Spring Data Redis：
    ```java
    @Autowired
    private StringRedisTemplate redisTemplate;
    public void incrementViewCount(String articleId) {
        redisTemplate.opsForValue().increment("article:" + articleId + ":views");
    }
    ```

#### (5) 消息队列（Message Queue）
- **描述**：
  - 使用列表（List）或发布/订阅（Pub/Sub）实现轻量级消息队列，处理异步任务。
- **适用场景**：
  - 任务队列：如订单处理、邮件发送。
  - 实时通知：如聊天消息、事件广播。
- **数据结构**：
  - **List**：实现 FIFO 队列（`LPUSH`/`RPOP`）。
  - **Pub/Sub**：实现事件广播。
  - **Stream**（Redis 5.0+）：支持持久化、消费者组。
- **特性**：
  - 高吞吐：支持高频消息推送/消费。
  - 简单实现：无需复杂 MQ（如 RabbitMQ）。
- **示例**：
  - 队列：
    ```bash
    # 生产者推送任务
    LPUSH task_queue "{\"task\":\"send_email\"}"
    # 消费者拉取任务
    RPOP task_queue
    ```
  - Pub/Sub：
    ```bash
    # 订阅频道
    SUBSCRIBE chat_channel
    # 发布消息
    PUBLISH chat_channel "Hello, World!"
    ```
- **实现**：
  - Java Redis 队列：
    ```java
    jedis.lpush("task_queue", "{\"task\":\"send_email\"}");
    String task = jedis.rpop("task_queue");
    ```

#### (6) 分布式锁（Distributed Lock）
- **描述**：
  - 使用 Redis 的原子操作（如 `SETNX`）实现分布式锁，解决分布式系统中的资源竞争。
- **适用场景**：
  - 分布式任务调度（如定时任务）。
  - 库存扣减（如秒杀系统）。
- **数据结构**：
  - **String**：存储锁标识和过期时间。
- **特性**：
  - 原子性：`SETNX` 确保互斥。
  - 过期机制：防止死锁。
- **示例**：
  ```bash
  # 获取锁，TTL 10秒
  SET lock:resource123 thread1 NX EX 10
  # 释放锁
  DEL lock:resource123
  ```
- **实现**：
  - Redisson 分布式锁：
    ```java
    @Autowired
    private RedissonClient redisson;
    public void lockResource(String resourceId) {
        RLock lock = redisson.getLock("lock:" + resourceId);
        lock.lock(10, TimeUnit.SECONDS);
        try {
            // 业务逻辑
        } finally {
            lock.unlock();
        }
    }
    ```

#### (7) 地理位置（Geo-Spatial）
- **描述**：
  - 使用 Geo 数据结构存储经纬度，支持距离计算和附近搜索。
- **适用场景**：
  - 附近的人（如社交 App）。
  - 地点搜索（如外卖、打车）。
- **数据结构**：
  - **Geo**（基于 Sorted Set）：存储经纬度。
- **特性**：
  - 高效查询：`GEORADIUS` 计算距离。
  - 实时更新：支持动态添加/删除坐标。
- **示例**：
  ```bash
  # 添加地点
  GEOADD locations 116.40 39.90 beijing
  # 查找 10km 内的地点
  GEORADIUS locations 116.40 39.90 10 km
  ```
- **实现**：
  - Java Redis Geo：
    ```java
    jedis.geoadd("locations", 116.40, 39.90, "beijing");
    jedis.georadius("locations", 116.40, 39.90, 10, GeoUnit.KM);
    ```

#### (8) 限流（Rate Limiting）
- **描述**：
  - 使用计数器或滑动窗口实现接口限流，防止服务过载。
- **适用场景**：
  - API 限流（如每秒 100 次请求）。
  - 防刷机制（如登录尝试限制）。
- **数据结构**：
  - **String**：计数器限流。
  - **List** 或 **Sorted Set**：滑动窗口限流。
- **特性**：
  - 原子操作：确保计数准确。
  - Lua 脚本：实现复杂限流逻辑。
- **示例**：
  ```bash
  # 计数器限流：1秒内限制 100 次
  INCR rate:api:123
  EXPIRE rate:api:123 1
  ```
- **实现**：
  - Lua 脚本限流：
    ```lua
    local key = KEYS[1]
    local limit = tonumber(ARGV[1])
    local current = redis.call("INCR", key)
    if current == 1 then
        redis.call("EXPIRE", key, 1)
    end
    return current <= limit
    ```

#### (9) 数据分析与统计
- **描述**：
  - 使用 HyperLogLog、Bitmap 进行高效统计，如独立用户数、活跃用户。
- **适用场景**：
  - UV（独立访客）统计。
  - 用户行为分析（如签到记录）。
- **数据结构**：
  - **HyperLogLog**：近似计数，误差 ~0.81%。
  - **Bitmap**：位图，精确记录布尔值。
- **特性**：
  - 低内存：HyperLogLog 固定 12KB，Bitmap 高效存储。
  - 高性能：快速计数和位运算。
- **示例**：
  - HyperLogLog：
    ```bash
    # 记录访问用户
    PFADD visitors:2023-01-01 user1 user2
    # 统计 UV
    PFCOUNT visitors:2023-01-01
    ```
  - Bitmap：
    ```bash
    # 记录用户签到
    SETBIT signin:2023-01-01:uid1001 1 1
    # 统计签到人数
    BITCOUNT signin:2023-01-01
    ```

#### (10) 实时数据处理
- **描述**：
  - 使用 Stream 数据结构处理实时数据流，支持消费者组。
- **适用场景**：
  - 日志收集与分析。
  - 事件流处理（如订单状态更新）。
- **数据结构**：
  - **Stream**：支持持久化、消费者组、ACK 机制。
- **特性**：
  - 类 Kafka：支持多消费者、分组。
  - 高效：O(1) 添加，O(log N) 读取。
- **示例**：
  ```bash
  # 添加事件
  XADD mystream * event_type order_placed order_id 123
  # 消费者组读取
  XREADGROUP GROUP mygroup consumer1 COUNT 10 STREAMS mystream >
  ```
- **实现**：
  - Java Redis Stream：
    ```java
    jedis.xadd("mystream", StreamEntryID.NEW_ENTRY, Map.of("event_type", "order_placed"));
    jedis.xreadGroup("mygroup", "consumer1", StreamOptions.builder().count(10).build(), "mystream");
    ```

---

### 2. Redis 特性与场景匹配
| **特性**                | **支持的场景**                              |
|-------------------------|---------------------------------------------|
| 内存存储，延迟 <1ms     | 缓存、会话、计数器、限流                   |
| 多种数据结构            | 排行榜、队列、Geo、统计                    |
| 原子操作               | 计数器、分布式锁、限流                     |
| 过期机制（TTL）         | 缓存、会话、限流、临时数据                 |
| 持久化（RDB/AOF）       | 消息队列、数据分析（需持久化）             |
| 发布/订阅、Stream       | 消息队列、实时通知、事件流                 |
| Lua 脚本                | 复杂原子操作（如限流、分布式锁）           |
| 高可用（主从、Sentinel）| 生产环境（缓存、会话、队列）               |
| 集群（Cluster）         | 大规模分布式场景（高并发、大数据量）       |

---

### 3. 注意事项
- **内存管理**：
  - Redis 内存占用需监控，避免 OOM（Out of Memory）。
  - 设置 `maxmemory` 和淘汰策略（如 `volatile-lru`、`allkeys-lru`）。
    ```bash
    CONFIG SET maxmemory 2gb
    CONFIG SET maxmemory-policy allkeys-lru
    ```
- **持久化选择**：
  - RDB（快照）：适合冷备份，数据丢失可接受。
  - AOF（日志）：适合高一致性，性能略低。
  - 混合模式（Redis 4.0+）：兼顾性能和一致性。
- **高可用**：
  - 使用主从复制 + Sentinel 实现故障转移。
  - 大规模场景用 Redis Cluster 分片。
- **性能优化**：
  - 批量操作（如 `MSET`、`PIPELINE`）降低网络开销。
  - 避免大键（如大 List/Set），拆分为小键。
  - 示例（Pipeline）：
    ```java
    Pipeline pipeline = jedis.pipelined();
    pipeline.set("key1", "value1");
    pipeline.set("key2", "value2");
    pipeline.sync();
    ```
- **事务限制**：
  - Redis 事务（`MULTI`/`EXEC`）仅保证原子性，不支持回滚。
  - 复杂逻辑用 Lua 脚本。
- **数据一致性**：
  - 缓存与数据库一致性需处理（如双写、延迟双删）。
  - 示例（缓存失效）：
    ```java
    public void updateProduct(Product product) {
        productRepository.save(product);
        redisTemplate.delete("product:" + product.getId()); // 失效缓存
    }
    ```
- **版本兼容**：
  - Stream（5.0+）、Geo（3.2+）、HyperLogLog（2.8+）需确认 Redis 版本。

---

### 4. 面试角度
- **问“Redis 的使用场景”**：
  - 提缓存（热点数据）、会话（分布式）、排行榜（Sorted Set）、计数器（INCR）、队列（List/Stream）、分布式锁（SETNX）、Geo（附近搜索）、限流（计数器）、统计（HyperLogLog）。
- **问“Redis 适合缓存的原因”**：
  - 提内存存储、低延迟、TTL、淘汰策略，举例（`SET EX`）。
- **问“分布式锁实现”**：
  - 提 `SETNX` + `EXPIRE`，或 Redisson（带看门狗），说明防死锁。
- **问“Redis 队列 vs MQ”**：
  - 提 Redis 队列（List/Stream）简单、延迟低，但无复杂功能（如 RabbitMQ 的 DLX、Kafka 的分区）。
- **问“Redis 高可用”**：
  - 提主从 + Sentinel（故障转移）、Cluster（分片），说明优缺点。
- **问“Redis 性能优化”**：
  - 提批量操作（Pipeline）、小键设计、Lua 脚本、内存管理。

---

### 5. 总结
- **Redis 使用场景**：
  - **缓存**：热点数据，加速查询。
  - **会话管理**：分布式会话，TTL 清理。
  - **排行榜**：Sorted Set 实时排序。
  - **计数器**：INCR 原子计数。
  - **消息队列**：List/Stream/Pub/Sub 异步任务。
  - **分布式锁**：SETNX 互斥操作。
  - **Geo**：地理位置查询。
  - **限流**：计数器、滑动窗口。
  - **统计**：HyperLogLog、Bitmap。
- **特性支持**：
  - 内存存储、低延迟、丰富数据结构、原子操作、持久化、高可用。
- **注意事项**：
  - 内存管理、持久化选择、事务限制、一致性、高可用部署。
- **面试建议**：
  - 提场景（缓存、锁、队列）、数据结构（String、Sorted Set）、代码（`SETNX`、`ZADD`）、优化（Pipeline、内存）、高可用（Sentinel、Cluster），清晰展示理解。
