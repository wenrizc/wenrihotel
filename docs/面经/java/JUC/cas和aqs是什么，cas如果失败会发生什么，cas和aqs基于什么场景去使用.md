
### 1. CAS 和 AQS 是什么
#### CAS（Compare And Swap）
- **定义**：一种乐观锁机制，通过比较并交换实现原子操作，由 CPU 硬件指令支持（如 `cmpxchg`），用于无锁并发。
- **原理**：比较内存值（V）与预期值（A），若相等则替换为新值（B）。
- **Java 实现**：`java.util.concurrent.atomic` 包（如 `AtomicInteger`）。
- **示例**：
```java
AtomicInteger num = new AtomicInteger(5);
num.compareAndSet(5, 10); // 成功改为 10
```

#### AQS（Abstract Queued Synchronizer）
- **定义**：Java 并发框架，基于队列和状态变量（`state`）实现锁和同步工具（如 `ReentrantLock`、`Semaphore`）。
- **原理**：用 CAS 修改 `state`，线程竞争失败时加入 CLH 队列等待。
- **Java 实现**：`java.util.concurrent.locks.AbstractQueuedSynchronizer`。
- **示例**：
```java
ReentrantLock lock = new ReentrantLock();
lock.lock(); // AQS 管理锁状态
lock.unlock();
```

---

### 2. CAS 如果失败会发生什么
#### 失败原因
- 内存值（V）与预期值（A）不匹配，说明其他线程已修改。

#### 失败结果
- **操作不执行**：新值（B）不会写入，CAS 返回 `false`。
- **后续处理**：
  - **自旋重试**：循环尝试，直到成功。
  - **放弃或切换策略**：根据业务决定。

#### 示例
```java
AtomicInteger num = new AtomicInteger(5);
// 线程 1
boolean success = num.compareAndSet(5, 10); // 成功，num = 10
// 线程 2
success = num.compareAndSet(5, 15); // 失败，预期 5，实际 10
if (!success) {
    System.out.println("CAS failed, retry or handle");
    // 自旋重试
    while (!num.compareAndSet(num.get(), 15)) {
        Thread.yield();
    }
}
```

#### 影响
- **性能**：高竞争下自旋耗 CPU。
- **ABA 问题**：值从 A->B->A，CAS 误判成功，需版本号解决（如 `AtomicStampedReference`）。

---

### 3. CAS 和 AQS 基于什么场景使用
#### CAS 使用场景
- **适用场景**：
  - **简单原子操作**：如计数器自增、状态标记。
  - **高并发读写**：无锁提升性能。
  - **轻量竞争**：线程冲突少，自旋成本低。
- **示例**：
  - `AtomicInteger`：统计在线人数。
  - `ConcurrentHashMap`：更新桶内数据。
- **不适用**：
  - 高竞争：自旋浪费 CPU。
  - 复杂逻辑：无法协调多步操作。

#### AQS 使用场景
- **适用场景**：
  - **复杂同步**：需要锁、等待、唤醒机制。
  - **多线程协作**：如线程倒计时、资源限制。
  - **高竞争**：阻塞式等待优于自旋。
- **示例**：
  - `ReentrantLock`：互斥访问共享资源。
  - `CountDownLatch`：多线程任务同步。
  - `Semaphore`：限流控制。
- **不适用**：
  - 简单操作：开销大于 CAS。

#### 对比选择
- **CAS**：轻量、无阻塞，适合单变量操作。
- **AQS**：功能丰富、阻塞式，适合复杂同步。

---

### 延伸与面试角度
- **CAS 失败处理**：
  - **自旋**：`while (!cas())`。
  - **降级**：切换到锁（如 `synchronized`）。
- **ABA 解决**：
  - `AtomicStampedReference` 加时间戳。
- **性能**：
  - CAS：低竞争快，高竞争慢。
  - AQS：稳定，适合阻塞。
- **底层支持**：
  - CAS：硬件指令。
  - AQS：CAS + 队列。
- **面试点**：
  - 问“CAS 失败”时，提自旋和 ABA。
  - 问“场景”时，提计数器 vs 锁。

---

### 总结
- **CAS**：无锁原子操作，失败则返回 false，常自旋重试，适合简单高并发。
- **AQS**：同步框架，管理锁和队列，适合复杂同步。
- **场景**：CAS 用于轻量原子操作，AQS 用于多线程协调。面试时，可写 `AtomicInteger` 自增或 `ReentrantLock` 示例，展示理解深度。