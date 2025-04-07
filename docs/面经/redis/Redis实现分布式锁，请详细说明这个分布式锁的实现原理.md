
使用 Redis 实现分布式锁是通过 Redis 的 **单线程特性和原子操作**，以键值对的形式设置锁，确保在分布式系统中多个进程或线程只有一个能获取锁。核心原理是利用 `SETNX`（Set if Not Exists）命令设置锁，结合过期时间防止死锁，解锁时用 Lua 脚本保证原子性。

---

### 关键事实
1. **目标**：
   - 在分布式环境中（如多服务器）实现互斥访问。
2. **Redis 特性**：
   - 单线程执行命令，保证原子性。
   - 高性能，支持分布式部署。
3. **基本步骤**：
   - 加锁：设置唯一键，值带标识。
   - 解锁：检查标识后删除键。
   - 超时：设置过期时间，避免死锁。

---

### 实现原理详解
#### 1. 加锁
- **命令**：`SET key value NX PX milliseconds`
  - `NX`：键不存在时设置（Set if Not Exists）。
  - `PX`：设置过期时间（毫秒）。
  - `value`：唯一标识（如客户端 ID 或随机 UUID）。
- **原理**：
  - 多个客户端尝试设置同一键，只有第一个成功，后续失败。
  - 过期时间防止锁未释放导致死锁。
- **代码**（Java + Jedis）：
```java
public boolean lock(Jedis jedis, String key, String value, int expireMs) {
    String result = jedis.set(key, value, "NX", "PX", expireMs);
    return "OK".equals(result);
}
```
- **示例**：
  - 客户端 A：`SET lock:resource clientA NX PX 30000` -> 成功。
  - 客户端 B：`SET lock:resource clientB NX PX 30000` -> 失败。

#### 2. 解锁
- **问题**：直接用 `DEL key` 可能误删其他客户端的锁。
- **解决**：用 Lua 脚本检查 value 匹配后再删除。
- **Lua 脚本**：
```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```
- **原理**：
  - 确保只有锁的拥有者能释放。
  - `GET` 和 `DEL` 在单次执行中完成，原子性。
- **代码**：
```java
public void unlock(Jedis jedis, String key, String value) {
    String script = "if redis.call('GET', KEYS[1]) == ARGV[1] then " +
                    "return redis.call('DEL', KEYS[1]) else return 0 end";
    jedis.eval(script, Collections.singletonList(key), Collections.singletonList(value));
}
```

#### 3. 完整示例
```java
public class DistributedLock {
    private Jedis jedis = new Jedis("localhost", 6379);
    private static final String LOCK_KEY = "lock:resource";
    private static final int EXPIRE_MS = 30000; // 30秒

    public boolean tryLock(String clientId) {
        return lock(jedis, LOCK_KEY, clientId, EXPIRE_MS);
    }

    public void releaseLock(String clientId) {
        unlock(jedis, LOCK_KEY, clientId);
    }

    public static void main(String[] args) {
        DistributedLock lock = new DistributedLock();
        String clientId = UUID.randomUUID().toString();
        if (lock.tryLock(clientId)) {
            System.out.println("Lock acquired");
            // 执行业务
            lock.releaseLock(clientId);
            System.out.println("Lock released");
        } else {
            System.out.println("Failed to acquire lock");
        }
    }
}
```

---

### 实现细节与原理
#### 1. 为什么用 SETNX？
- Redis 单线程执行，`SET NX` 保证只有一个客户端设置成功，天然互斥。

#### 2. 为什么加过期时间？
- 防止客户端崩溃未释放锁，过期后自动解锁。
- 示例：客户端 A 获取锁后宕机，30 秒后锁失效。

#### 3. 为什么用 Lua 解锁？
- 避免误删：直接 `DEL` 可能删除其他客户端的锁。
- 原子性：`GET` 和 `DEL` 分开执行可能被中断，Lua 脚本一次完成。

#### 4. 锁续期问题
- **问题**：业务执行超 30 秒，锁过期被他人获取。
- **解决**：客户端定时检查并延长锁（`PEXPIRE`），如 Redisson 的看门狗机制。

---

### 延伸与面试角度
- **优缺点**：
  - **优点**：简单、高性能、跨节点。
  - **缺点**：依赖 Redis 单点，过期时间难估。
- **与 Zookeeper 对比**：
  - **Redis**：高性能，基于内存，弱一致性。
  - **Zookeeper**：强一致性，基于顺序节点，性能低。
- **改进**：
  - **Redisson**：封装分布式锁，支持锁续期、公平锁。
  - **多节点**：用 Redlock 算法，多 Redis 实例投票。
- **实际应用**：
  - 秒杀系统：锁库存。
  - 分布式任务：锁任务 ID。
- **面试点**：
  - 问“死锁怎么防”时，提过期时间。
  - 问“误删怎么防”时，提 Lua 脚本。

---

### 总结
Redis 分布式锁用 `SETNX` 加锁，设置过期时间防死锁，Lua 脚本原子解锁。原理基于 Redis 单线程和原子性，保证互斥。面试时，可写代码或提 Redlock 改进，展示深入理解。