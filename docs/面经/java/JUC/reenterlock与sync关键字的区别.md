
#### ReentrantLock 与 synchronized 概述
- **synchronized**：
  - Java 关键字，内置锁机制，基于 JVM 监视器（Monitor）实现。
- **ReentrantLock**：
  - Java 的显式锁，`java.util.concurrent.locks` 包中的类，基于 AQS（AbstractQueuedSynchronizer）实现。

#### 主要区别
1. **实现方式**：
   - synchronized：JVM 级别，自动管理。
   - ReentrantLock：Java API，需手动加解锁。
2. **功能灵活性**：
   - synchronized：基本加锁、等待/通知。
   - ReentrantLock：支持公平锁、超时、条件变量。
3. **锁释放**：
   - synchronized：自动释放（代码块或异常）。
   - ReentrantLock：需显式调用 `unlock()`。
4. **性能**：
   - synchronized：Java 6 后优化（偏向锁、轻量锁），性能接近。
   - ReentrantLock：早期性能优，现差距小。
5. **使用场景**：
   - synchronized：简单同步场景。
   - ReentrantLock：复杂并发需求。

#### 核心点
- ReentrantLock 更灵活，synchronized 更简洁。

---

### 1. 区别详解
#### (1) 实现方式
- **synchronized**：
  - 内置于 JVM，通过对象头中的 Monitor 实现。
  - 锁状态由 JVM 管理，无需显式操作。
  - 例：
```java
synchronized (obj) {
    // 同步块
}
```
- **ReentrantLock**：
  - 基于 AQS，使用 CAS 和队列管理锁。
  - 需手动调用 `lock()` 和 `unlock()`。
  - 例：
```java
ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    // 同步块
} finally {
    lock.unlock();
}
```

#### (2) 功能灵活性
- **synchronized**：
  - 仅支持基本加锁和 `wait()/notify()`。
  - 无公平性控制、无超时机制。
- **ReentrantLock**：
  - **公平锁**：可选公平策略（`new ReentrantLock(true)`），FIFO 顺序。
  - **超时锁**：`tryLock(long, TimeUnit)` 避免死等。
  - **条件变量**：支持多个 `Condition`（如 `newCondition()`），比 `wait/notify` 更灵活。
  - **中断响应**：`lockInterruptibly()` 可被中断。
  - 例：
```java
ReentrantLock lock = new ReentrantLock();
Condition condition = lock.newCondition();
lock.lock();
try {
    condition.await(); // 等待
    condition.signal(); // 唤醒
} finally {
    lock.unlock();
}
```

#### (3) 锁释放
- **synchronized**：
  - 自动释放锁（离开同步块或异常）。
  - 例：
```java
synchronized (obj) {
    throw new RuntimeException(); // 锁自动释放
}
```
- **ReentrantLock**：
  - 需显式 `unlock()`，通常放 `finally` 块。
  - 忘记 `unlock()` 可能导致死锁。
  - 例：
```java
lock.lock();
try {
    // 可能抛异常
} finally {
    lock.unlock(); // 确保释放
}
```

#### (4) 性能
- **synchronized**：
  - 早期性能差（重量锁）。
  - Java 6 优化：偏向锁 → 轻量锁 → 重量锁，性能提升。
- **ReentrantLock**：
  - 早期优于 synchronized（细粒度控制）。
  - 现性能接近，复杂场景仍具优势。
- **选择**：
  - 简单场景用 synchronized，复杂场景选 ReentrantLock。

#### (5) 使用场景
- **synchronized**：
  - 简单同步：方法或代码块。
  - 例：计数器加锁。
```java
public synchronized void increment() {
    count++;
}
```
- **ReentrantLock**：
  - 复杂并发：超时、中断、条件等待。
  - 例：生产者-消费者多条件。
```java
ReentrantLock lock = new ReentrantLock();
Condition notFull = lock.newCondition();
Condition notEmpty = lock.newCondition();
```

---

### 2. 共同点
- **可重入**：
  - 两者都是可重入锁，同一线程可多次获取锁。
  - 例：
```java
synchronized (obj) {
    synchronized (obj) { // 可重入
    }
}
```
```java
lock.lock();
lock.lock(); // 可重入
lock.unlock();
lock.unlock();
```
- **线程安全**：
  - 都保证同步块内数据一致性。

---

### 3. 优缺点对比
| **特性**         | **synchronized**           | **ReentrantLock**          |
|------------------|----------------------------|----------------------------|
| **实现**         | JVM Monitor               | AQS (Java API)            |
| **加锁/解锁**    | 自动                      | 手动                      |
| **灵活性**       | 基本功能                  | 公平锁、超时、条件        |
| **性能**         | 优化后接近                | 复杂场景略优              |
| **异常处理**     | 自动释放                  | 需 finally unlock         |
| **适用场景**     | 简单同步                  | 高级并发控制              |

---

### 4. 延伸与面试角度
- **与并发工具**：
  - ReentrantLock 用于 `ConcurrentHashMap`、`BlockingQueue`。
  - synchronized 常用于简单类。
- **实际应用**：
  - synchronized：日志同步。
  - ReentrantLock：线程池条件等待。
- **选择依据**：
  - 简单逻辑用 synchronized。
  - 需高级功能用 ReentrantLock。
- **面试点**：
  - 问“区别”时，提灵活性和释放。
  - 问“场景”时，提示例代码。

---

### 总结
synchronized 是 JVM 内置锁，简单但功能有限；ReentrantLock 是显式锁，灵活支持公平、超时和条件变量。synchronized 适合简单场景，ReentrantLock 适合复杂并发。面试时，可提功能对比或代码示例，展示理解深度。