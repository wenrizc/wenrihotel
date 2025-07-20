
#### 分布式锁设计
分布式锁用于在分布式系统中协调多节点对共享资源的访问，通常基于 Redis、ZooKeeper 等实现。以 Redis 为例：
- **Key**：锁的唯一标识（如资源 ID）。
- **Value**：客户端标识（如 UUID）+超时时间，确保锁可释放。
- **实现**：用 `SETNX`（Set if Not Exists）加锁，`DEL` 解锁。

#### 避免死锁
1. **超时机制**：设置锁过期时间（如 10 秒），避免永久持有。
2. **唯一标识**：Value 带客户端 ID，仅持有者可释放。
3. **自动续期**：锁快过期时延长（如 watchdog）。
4. **原子操作**：加锁和解锁用 Lua 脚本保证一致性。

---

### 1. 分布式锁设计
#### 基本原理
- **目的**：确保同一时刻只有一个客户端持有锁。
- **工具**：Redis（简单高效）、ZooKeeper（强一致性）。

#### Key/Value 设计
- **Key**：
  - 表示锁定的资源，如 `lock:order:123`（订单 ID）。
  - 格式：`lock:<资源类型>:<资源ID>`。
- **Value**：
  - 包含客户端标识（如 UUID）和可选时间戳。
  - 示例：`client-uuid-123e4567-e89b-12d3-a456-426614174000`。
  - 作用：标识锁持有者，防止误删。

#### 加锁
- **命令**：
```bash
SET lock:order:123 client-uuid-123e4567 NX PX 10000
```
  - `NX`：仅当 Key 不存在时设置（SETNX）。
  - `PX 10000`：过期时间 10 秒。
- **返回**：
  - OK：加锁成功。
  - nil：锁被占用。

#### 解锁
- **命令**：需原子性，用 Lua 脚本。
```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```
- **执行**：
```bash
EVAL "if redis.call('GET', KEYS[1]) == ARGV[1] then return redis.call('DEL', KEYS[1]) else return 0 end" 1 lock:order:123 client-uuid-123e4567
```
  - 检查 Value 匹配才删除。

#### Java 示例（Jedis）
```java
Jedis jedis = new Jedis("localhost", 6379);
String lockKey = "lock:order:123";
String clientId = UUID.randomUUID().toString();

// 加锁
boolean locked = jedis.set(lockKey, clientId, "NX", "PX", 10000) != null;
if (locked) {
    try {
        // 业务逻辑
        System.out.println("Locked by " + clientId);
    } finally {
        // 解锁
        String script = "if redis.call('GET', KEYS[1]) == ARGV[1] then return redis.call('DEL', KEYS[1]) else return 0 end";
        jedis.eval(script, Collections.singletonList(lockKey), Collections.singletonList(clientId));
    }
}
```

---

### 2. 避免死锁
#### 死锁场景
- 客户端加锁后崩溃，未释放锁。
- 锁无超时，其他客户端永久等待。

#### 解决方案
##### (1) 超时机制
- **设计**：
  - 加锁时设过期时间（如 `PX 10000`）。
- **效果**：
  - 客户端异常，锁自动释放。
- **注意**：
  - 时间太短：业务未完锁失效。
  - 时间太长：释放延迟。

##### (2) 唯一标识
- **设计**：
  - Value 存客户端 UUID。
  - 解锁时校验持有者。
- **效果**：
  - 防止其他客户端误删。
- **示例**：
  - A 加锁，B 不可解锁。

##### (3) 自动续期
- **设计**：
  - 客户端启动守护线程（如 Watchdog）。
  - 每隔 1/3 过期时间（如 3 秒）检查：
```java
if (jedis.get(lockKey).equals(clientId)) {
    jedis.pexpire(lockKey, 10000); // 续期
}
```
- **效果**：
  - 业务超长时避免失效。
- **工具**：
  - Redisson 自带续期。

##### (4) 原子操作
- **设计**：
  - 解锁用 Lua 脚本，`GET` 和 `DEL` 一步完成。
- **效果**：
  - 避免检查和删除间被抢占。
- **问题**：
  - 无原子性可能误删。

#### 其他预防
- **监控**：
  - 锁占用时间异常告警。
- **重试**：
  - 加锁失败，指数退避重试。
```java
int retries = 5;
while (!locked && retries-- > 0) {
    Thread.sleep(100 * (5 - retries));
    locked = jedis.set(lockKey, clientId, "NX", "PX", 10000) != null;
}
```

---

### 3. 设计要点
#### 性能
- **加锁**：O(1)。
- **解锁**：O(1)。
- **Redis 单机**：10 万 QPS。

#### 可靠性
- **超时**：防死锁。
- **唯一性**：防误操作。
- **原子性**：防竞争。

#### 可扩展性
- **多资源**：Key 区分（如 `lock:user:1`）。
- **分布式**：Redis Cluster 支持。

---

### 4. 延伸与面试角度
- **与 ZooKeeper 对比**：
  - Redis：简单高效，最终一致。
  - ZooKeeper：强一致，顺序节点。
- **实际应用**：
  - 秒杀：锁库存。
  - 任务调度：锁任务。
- **局限**：
  - Redis 宕机：锁丢失。
  - 解决：RedLock（多节点锁）。
- **面试点**：
  - 问“设计”时，提 Key/Value 和 Lua。
  - 问“死锁”时，提超时和续期。

---

### 总结
分布式锁用 Redis 的 `SETNX` 实现，Key 标识资源，Value 带 UUID + 超时。避免死锁靠过期时间、唯一标识、续期和原子解锁。面试时，可写 Lua 脚本或提 Redisson，展示设计能力。