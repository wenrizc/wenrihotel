
#### 概述
- **定义**：
  - Redisson 是一个基于 Redis 的 Java 框架，提供分布式锁（`RLock`）等功能，用于在分布式系统中实现线程安全的资源互斥访问。
  - 分布式锁通过 Redis 的原子操作和键值存储，确保多节点环境下锁的正确性和高可用性。
- **核心点**：
  - Redisson 分布式锁基于 Redis 的 `SETNX`（Set if Not Exists）命令、Lua 脚本、过期机制和看门狗（Watchdog）续期，结合发布/订阅机制实现高效锁管理。
  - 它支持可重入锁、公平锁、读写锁等高级特性，解决死锁、锁失效等问题。

---

### 1. Redisson 分布式锁的核心原理
Redisson 的分布式锁（`RLock`）实现依赖 Redis 的以下特性：

#### (1) 基础机制
- **Redis 键**：
  - 锁以 Redis 键存储，键名通常为 `redisson_lock:{resource}`，值为锁的元数据（如线程 ID、锁计数）。
- **SETNX 命令**：
  - 使用 `SETNX` 确保锁的互斥性，仅当键不存在时设置成功。
  - 结合 `EXPIRE` 设置 TTL（Time-To-Live），防止锁未释放导致死锁。
- **可重入锁**：
  - Redisson 支持锁重入，通过记录锁的持有线程和重入计数（类似 `synchronized`）。
  - 使用 Hash 结构存储：
    - 键：`redisson_lock:{resource}`。
    - 字段：线程 ID 或 UUID（标识锁持有者）。
    - 值：重入计数。
- **Lua 脚本**：
  - 所有锁操作（加锁、释放、续期）通过 Lua 脚本执行，保证原子性。
  - Lua 脚本在 Redis 服务端运行，避免网络往返和并发冲突。

#### (2) 看门狗（Watchdog）续期
- **问题**：
  - 锁的 TTL 可能导致锁在任务未完成时过期，引发并发问题。
- **解决**：
  - Redisson 引入看门狗机制，自动为锁续期。
  - 原理：
    - 获取锁后，Redisson 启动后台线程（默认每 TTL/3 秒检查一次）。
    - 若锁仍被持有（任务未完成），通过 `PEXPIRE` 延长 TTL（默认 30 秒，`-Dredisson.lockWatchdogTimeout` 可调）。
    - 任务完成后，锁释放，看门狗停止。
- **示例**：
  - 锁 TTL 设为 30 秒，看门狗每 10 秒检查，若线程仍在执行，续期至 30 秒。

#### (3) 发布/订阅（Pub/Sub）机制
- **作用**：
  - 实现锁的阻塞等待和通知，减少轮询开销。
- **原理**：
  - 锁释放时，Redisson 通过 Redis 的 Pub/Sub 发布消息到特定频道（如 `redisson_lock__channel:{resource}`）。
  - 等待锁的客户端订阅该频道，收到消息后尝试获取锁。
  - 避免客户端频繁轮询 `GET` 锁状态，提高效率。
- **流程**：
  - 线程 A 获取锁，线程 B 阻塞并订阅锁释放频道。
  - 线程 A 释放锁，发布消息到频道。
  - 线程 B 收到消息，重新尝试 `SETNX` 获取锁。

#### (4) 锁释放与清理
- **释放**：
  - 释放锁时，Redisson 使用 Lua 脚本检查调用者是否为锁持有者：
    - 若重入计数 > 1，减 1。
    - 若计数 = 0，删除锁键并发布释放消息。
  - 示例 Lua 脚本（简化）：
    ```lua
    if redis.call("hexists", KEYS[1], ARGV[1]) == 1 then
        local count = redis.call("hincrby", KEYS[1], ARGV[1], -1);
        if count > 0 then
            redis.call("pexpire", KEYS[1], ARGV[2]);
            return 0;
        else
            redis.call("del", KEYS[1]);
            redis.call("publish", KEYS[2], ARGV[3]);
            return 1;
        end
    end
    return 0;
    ```
    - `KEYS[1]`：锁键。
    - `ARGV[1]`：线程 ID。
    - `ARGV[2]`：TTL。
    - `ARGV[3]`：释放消息。
- **清理**：
  - 若客户端异常退出（未释放锁），TTL 确保锁最终过期。
  - 看门狗停止续期，锁自动释放。

#### (5) 可重入锁实现
- **存储**：
  - 锁键为 Hash，结构如：
    ```bash
    redisson_lock:{resource}
        thread_id:UUID -> count
    ```
    - 示例：`thread_id:uuid123 -> 2`（重入 2 次）。
- **加锁**：
  - 检查 Hash 是否存在：
    - 不存在：`SETNX` 创建锁，计数设为 1。
    - 存在且线程匹配：计数 +1。
- **释放**：
  - 计数 -1，若为 0，删除键。

---

### 2. Redisson 分布式锁的工作流程
#### (1) 加锁（lock）
- **代码**：
  ```java
  RLock lock = redisson.getLock("myLock");
  lock.lock(); // 默认 TTL 30 秒
  ```
- **流程**：
  1. 客户端尝试通过 Lua 脚本执行 `SETNX`：
     - 脚本检查锁键是否存在：
       - 不存在：设置锁（`HSET` 记录线程 ID 和计数 1），设置 TTL。
       - 存在：检查线程 ID，若匹配，计数 +1（重入）；否则失败。
  2. 若获取失败：
     - 订阅锁释放频道（Pub/Sub）。
     - 定期重试（默认 100ms，`-Dredisson.lockRetryInterval` 可调）。
  3. 获取成功：
     - 启动看门狗，定时续期（默认每 10 秒）。
- **Lua 脚本（简化）**：
  ```lua
  if redis.call("exists", KEYS[1]) == 0 then
      redis.call("hset", KEYS[1], ARGV[1], 1);
      redis.call("pexpire", KEYS[1], ARGV[2]);
      return nil;
  end
  if redis.call("hexists", KEYS[1], ARGV[1]) == 1 then
      redis.call("hincrby", KEYS[1], ARGV[1], 1);
      redis.call("pexpire", KEYS[1], ARGV[2]);
      return nil;
  end
  return redis.call("pttl", KEYS[1]);
  ```

#### (2) 阻塞等待
- **代码**：
  ```java
  lock.lock(10, TimeUnit.SECONDS); // 最多等待 10 秒
  ```
- **流程**：
  - 若锁被占用，客户端订阅释放频道，等待消息。
  - 等待期间，定期检查锁状态或超时退出。
  - 收到释放消息后，重新尝试加锁。

#### (3) 释放锁（unlock）
- **代码**：
  ```java
  lock.unlock();
  ```
- **流程**：
  1. 执行 Lua 脚本：
     - 验证调用者是否为锁持有者（线程 ID 匹配）。
     - 计数 -1，若为 0，删除锁键。
     - 发布锁释放消息到频道。
  2. 看门狗停止续期。
- **Lua 脚本**：见上文释放脚本。

#### (4) 看门狗续期
- **流程**：
  - 加锁成功后，Redisson 启动定时任务（默认 30 秒 TTL，每 10 秒检查）。
  - 检查锁键是否存在且线程 ID 匹配：
    - 若存在，执行 `PEXPIRE` 续期。
    - 若不存在（已释放或过期），停止续期。
  - 任务完成或锁释放，定时任务终止。

---

### 3. Redisson 分布式锁的特性
- **可重入**：
  - 支持同一线程多次获取锁，计数累加，释放时计数减 1。
- **防死锁**：
  - TTL 确保锁最终释放，看门狗动态续期避免任务未完成时锁失效。
- **高性能**：
  - Lua 脚本保证原子性，Pub/Sub 减少轮询，单 Redis 实例 QPS 可达万级。
- **高可用**：
  - 支持 Redis 主从、Sentinel、Cluster 模式，故障转移不影响锁。
- **灵活性**：
  - 支持阻塞锁（`lock`）、非阻塞锁（`tryLock`）、超时等待、公平锁（`RFairLock`）、读写锁（`RReadWriteLock`）。

---

### 4. 代码示例
```java
import org.redisson.Redisson;
import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;
import org.redisson.config.Config;

public class RedissonLockExample {
    public static void main(String[] args) {
        // 配置 Redis
        Config config = new Config();
        config.useSingleServer().setAddress("redis://127.0.0.1:6379");
        RedissonClient redisson = Redisson.create(config);

        // 获取锁
        RLock lock = redisson.getLock("myLock");
        try {
            // 尝试获取锁，等待 10 秒，持有 30 秒
            if (lock.tryLock(10, 30, TimeUnit.SECONDS)) {
                try {
                    System.out.println("Lock acquired by " + Thread.currentThread().getName());
                    // 业务逻辑
                    Thread.sleep(5000);
                } finally {
                    lock.unlock();
                    System.out.println("Lock released");
                }
            } else {
                System.out.println("Failed to acquire lock");
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            redisson.shutdown();
        }
    }
}
```

**Redis 存储**：
- 锁键：`redisson_lock:myLock`。
- Hash 内容：`{uuid:thread-id -> count}`。
- 频道：`redisson_lock__channel:myLock`。

---

### 5. 注意事项
- **Redis 依赖**：
  - Redis 故障可能导致锁不可用，需部署主从或 Cluster 模式。
  - 网络分区可能引发脑裂，需配置 Sentinel 或 Paxos 保证一致性。
- **性能开销**：
  - Lua 脚本和 Pub/Sub 高效，但高并发下 Redis 单点可能成为瓶颈。
  - 建议监控 QPS 和延迟，必要时分片（Cluster）。
- **看门狗限制**：
  - 续期依赖客户端存活，若客户端崩溃，锁仍可能因 TTL 过期。
  - 建议设置合理 TTL（如 30-60 秒），避免任务超长。
- **锁竞争**：
  - 高竞争下，阻塞等待可能增加延迟，考虑 `tryLock` 或公平锁。
- **参数调优**：
  - 调整 `lockWatchdogTimeout`（续期间隔）、`lockRetryInterval`（重试间隔）。
  - 示例：
    ```bash
    -Dredisson.lockWatchdogTimeout=60000 # 60 秒
    ```
- **一致性**：
  - Redis 主从复制可能导致锁数据延迟，建议同步写入（`WAIT` 命令）。

---

### 6. 面试角度
- **问“Redisson 分布式锁原理”**：
  - 提 `SETNX` + Lua 脚本（原子性）、看门狗（续期）、Pub/Sub（通知）、Hash（重入）。
- **问“看门狗如何工作”**：
  - 提后台线程每 TTL/3 秒检查，调用 `PEXPIRE` 续期，锁释放后停止。
- **问“Redisson vs 手动 SETNX”**：
  - 提 Redisson 自动续期、可重入、Pub/Sub 高效；手动 `SETNX` 需自己处理过期和等待。
- **问“高可用性保障”**：
  - 提主从 + Sentinel、Cluster 分片，说明网络分区应对（Paxos、同步写入）。
- **问“锁释放流程”**：
  - 提 Lua 脚本验证线程 ID、减计数、删除键、发布消息。
- **问“与 synchronized 区别”**：
  - 提 synchronized 单 JVM、Monitor 机制；Redisson 分布式、Redis 协调。

---

### 7. 总结
- **Redisson 分布式锁原理**：
  - **核心机制**：`SETNX` + Lua 脚本（原子加锁/释放）、Hash（可重入）、Pub/Sub（阻塞通知）、看门狗（续期）。
  - **工作流程**：尝试加锁（SETNX）→ 阻塞等待（Pub/Sub）→ 续期（Watchdog）→ 释放（Lua + Pub/Sub）。
  - **存储结构**：Hash 键存储线程 ID 和计数，频道发布释放消息。
- **特性**：
  - 可重入、防死锁、高性能、高可用、灵活（阻塞/非阻塞/公平）。
- **与之前讨论关联**：
  - 相比 `synchronized`（单 JVM，Monitor），Redisson 跨节点，依赖 Redis。
  - 类似并发原语（CAS），通过 Lua 保证原子性，Pub/Sub 类似条件变量。
- **面试建议**：
  - 提核心原理（SETNX、Lua、Watchdog、Pub/Sub）、代码（`RLock` 使用）、优缺点（性能 vs Redis 依赖）、调优（TTL、Cluster），清晰展示理解。
