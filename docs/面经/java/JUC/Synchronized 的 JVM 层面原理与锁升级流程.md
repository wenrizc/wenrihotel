
#### 概述
- **定义**：
  - `synchronized` 是 Java 提供的关键字，用于实现线程同步，保证多线程环境下的互斥访问，防止数据竞争。
  - 它可以修饰方法（静态或实例方法）或代码块，底层依赖 JVM 的 **监视器（Monitor）** 机制。
- **核心点**：
  - **JVM 原理**：通过 Monitor 对象（包含锁状态、线程 ID、等待队列等）实现互斥，涉及字节码指令 `monitorenter` 和 `monitorexit`。
  - **锁升级**：Java 6+ 引入偏向锁、轻量级锁、重量级锁，优化 `synchronized` 性能，根据竞争程度动态升级。

---

### 1. Synchronized 的 JVM 层面原理
#### (1) 基本概念
- **Monitor**：
  - JVM 为每个对象分配一个 Monitor（逻辑结构，非实体），包含：
    - **Owner**：持有锁的线程 ID。
    - **Entry List**：等待锁的线程队列。
    - **Wait Set**：调用 `wait()` 进入等待的线程。
    - **递归计数**：锁重入次数（支持可重入锁）。
  - Monitor 确保同一时刻只有一个线程持有锁。
- **字节码指令**：
  - `synchronized` 代码块在编译为字节码时，使用：
    - `monitorenter`：获取 Monitor 锁，进入同步块。
    - `monitorexit`：释放 Monitor 锁，退出同步块。
  - 示例：
    ```java
    public void method() {
        synchronized (this) {
            // 同步代码
        }
    }
    ```
    **字节码**：
    ```
    monitorenter
    // 同步代码
    monitorexit
    ```

#### (2) 工作机制
- **获取锁**：
  - 线程执行 `monitorenter`，尝试获取对象 Monitor：
    - 若 Monitor 无主（Owner 为空），线程成为 Owner，计数 +1。
    - 若线程已持有（重入），计数 +1。
    - 若 Monitor 被其他线程持有，当前线程进入 Entry List，阻塞等待。
- **释放锁**：
  - 线程执行 `monitorexit`：
    - 计数 -1，若减至 0，释放 Monitor，Owner 清空。
    - 唤醒 Entry List 中的等待线程竞争锁。
- **等待/通知**：
  - `wait()`：线程释放 Monitor，进入 Wait Set，等待唤醒。
  - `notify()`/`notifyAll()`：从 Wait Set 唤醒线程，重新竞争锁。
- **示例**：
  ```java
  synchronized (obj) {
      while (condition) {
          obj.wait(); // 释放锁，进入 Wait Set
      }
      // 同步操作
      obj.notify(); // 唤醒等待线程
  }
  ```

#### (3) Monitor 的 JVM 实现
- **对象头（Object Header）**：
  - Java 对象在堆中包含对象头（Mark Word）、实例数据和填充字节。
  - Mark Word 存储锁状态、线程 ID、GC 信息等。
  - 示例（64 位 JVM，Mark Word 结构）：
    ```
    |-------------------------------------------------------|
    |                 Mark Word (64 bits)                   |
    |-------------------------------------------------------|
    | 锁状态 | 锁标志 | 其他信息（如线程 ID、偏向 Epoch） |
    |-------------------------------------------------------|
    ```
    - 无锁：`01`（无锁状态）。
    - 偏向锁：`01`（含线程 ID）。
    - 轻量级锁：`00`（指向栈中 Lock Record）。
    - 重量级锁：`10`（指向 Monitor 对象）。
- **Monitor 对象**：
  - 重量级锁时，JVM 在堆中分配 Monitor 对象，Mark Word 指向它。
  - Monitor 维护 Owner、Entry List、Wait Set，实现线程调度。

---

### 2. Synchronized 锁升级流程
Java 6+ 优化 `synchronized`，引入 **偏向锁**、**轻量级锁**、**重量级锁**，根据竞争程度动态升级，降低锁开销。

#### (1) 锁状态
- **无锁**：
  - 对象无线程竞争，Mark Word 标记为 `01`（无锁）。
- **偏向锁（Biased Locking）**：
  - 适合单线程反复获取锁，记录偏向线程 ID，无需 CAS 操作。
- **轻量级锁（Lightweight Locking）**：
  - 适合少量线程竞争，使用 CAS（Compare-And-Swap）操作，避免系统调用。
- **重量级锁（Heavyweight Locking）**：
  - 适合多线程激烈竞争，使用操作系统互斥量（Mutex），涉及线程阻塞/唤醒。

#### (2) 锁升级流程
1. **无锁状态**：
   - 新创建对象，Mark Word 为无锁（`01`），无线程持有。
2. **偏向锁**：
   - **触发**：单线程首次获取锁（默认启用，`-XX:+UseBiasedLocking`）。
   - **过程**：
     - Mark Word 记录线程 ID 和偏向标志（`01`）。
     - 后续该线程重入锁无需 CAS，直接检查 ID。
   - **开销**：极低，仅更新 Mark Word。
   - **撤销**：
     - 其他线程竞争时，暂停偏向线程，撤销偏向锁（批量偏向或禁用）。
     - 升级为轻量级锁。
3. **轻量级锁**：
   - **触发**：多线程竞争，但竞争不激烈。
   - **过程**：
     - 线程在栈中创建 Lock Record，复制对象 Mark Word。
     - 使用 CAS 将 Mark Word 替换为指向 Lock Record 的指针（锁标志 `00`）。
     - 获取成功：线程持有轻量级锁。
     - 获取失败：自旋（默认 10 次，`-XX:PreBlockSpin`）等待锁释放。
   - **开销**：中等，CAS 操作比系统调用轻量。
   - **升级**：
     - 自旋失败或竞争加剧，膨胀为重量级锁。
4. **重量级锁**：
   - **触发**：多线程激烈竞争，自旋无效。
   - **过程**：
     - JVM 分配 Monitor 对象，Mark Word 指向 Monitor（锁标志 `10`）。
     - 未获取锁的线程进入 Entry List，阻塞（调用操作系统 `park()`）。
     - 锁释放时，唤醒 Entry List 线程（`unpark()`）竞争。
   - **开销**：高，涉及上下文切换和系统调用。
   - **降级**：
     - 竞争减少时不会回退（单向升级），需等待锁释放。

#### (3) 锁升级图示
```
无锁 -> 偏向锁 -> 轻量级锁 -> 重量级锁
  |        |           |             |
  | 单线程 | 多线程少竞争 | 多线程激烈竞争 |
  | 低开销 | 中等开销     | 高开销       |
```

#### (4) 示例
```java
public class Example {
    private final Object lock = new Object();

    public void syncMethod() {
        synchronized (lock) {
            // 同步操作
        }
    }
}
```
- **单线程**：
  - 首次调用 `syncMethod`，`lock` 进入偏向锁，记录线程 ID。
  - 后续调用无需锁操作，性能最高。
- **两线程竞争**：
  - 线程 2 竞争，撤销偏向锁，升级为轻量级锁。
  - 线程 1 和 2 通过 CAS 竞争，失败者自旋。
- **多线程竞争**：
  - 自旋失败，升级为重量级锁。
  - 未获取锁的线程阻塞，进入 Monitor 的 Entry List。

---

### 3. 锁升级的优化机制
- **偏向锁延迟**：
  - JVM 启动时延迟启用偏向锁（默认 4 秒，`-XX:BiasedLockingStartupDelay`），避免初始化阶段频繁撤销。
- **批量偏向/撤销**：
  - 高并发下，类级别批量调整偏向锁状态，减少逐对象撤销开销。
- **自适应自旋**：
  - 轻量级锁自旋次数动态调整（`-XX:+UseAdaptiveSpin`），根据 CPU 和竞争情况优化。
- **锁消除**：
  - JIT 编译器通过逃逸分析，消除无竞争的 `synchronized`（如局部对象）。
  - 示例：
    ```java
    public String concat(String a, String b) {
        StringBuilder sb = new StringBuilder(); // 局部对象
        synchronized (sb) { // 锁消除
            sb.append(a).append(b);
        }
        return sb.toString();
    }
    ```
- **锁粗化**：
  - 合并多次 `synchronized` 操作，减少锁开销。
  - 示例：
    ```java
    // 原始
    synchronized (lock) { a++; }
    synchronized (lock) { b++; }
    // 粗化后
    synchronized (lock) { a++; b++; }
    ```

---

### 4. 性能与适用场景
- **偏向锁**：
  - **场景**：单线程或低竞争（如单例对象）。
  - **性能**：接近无锁，适合高频重入。
- **轻量级锁**：
  - **场景**：少量线程竞争（如线程池任务）。
  - **性能**：CAS 操作轻量，适合短时锁。
- **重量级锁**：
  - **场景**：多线程高竞争（如共享资源）。
  - **性能**：高开销，适合长时任务或复杂同步。
- **对比 ReentrantLock**：
  - `synchronized`：JVM 内置，简单，自动升级，适合通用场景。
  - `ReentrantLock`：Java API，灵活（可中断、超时），无锁升级，适合复杂场景。

---

### 5. 注意事项
- **锁升级单向**：
  - 偏向 → 轻量级 → 重量级，不可降级。
  - 竞争减少后，需释放锁后重新获取低级别锁。
- **偏向锁禁用**：
  - 高并发场景禁用偏向锁（`-XX:-UseBiasedLocking`），避免撤销开销。
- **自旋开销**：
  - 自旋消耗 CPU，单核或高竞争下慎用（可调 `-XX:PreBlockSpin`）。
- **Monitor 开销**：
  - 重量级锁涉及系统调用，上下文切换耗时（几十微秒）。
- **内存影响**：
  - Monitor 对象占用堆内存，高并发下需关注。
- **调试**：
  - 使用 `-XX:+PrintGCDetails` 或 JVisualVM 分析锁竞争。
- **JDK 版本**：
  - Java 6 引入锁升级，Java 8+ 优化偏向锁和自旋。
  - Java 11+ 推荐结合 `VarHandle` 或 `Lock` 替代部分场景。

---

### 6. 面试角度
- **问“synchronized 的 JVM 原理”**：
  - 提 Monitor 机制、`monitorenter`/`monitorexit`、对象头（Mark Word）、线程阻塞/唤醒。
- **问“锁升级流程”**：
  - 提无锁 → 偏向锁（单线程）→ 轻量级锁（CAS）→ 重量级锁（Monitor），说明触发条件和开销。
- **问“偏向锁作用”**：
  - 提减少单线程锁开销，记录线程 ID，无需 CAS，举例单例。
- **问“轻量级 vs 重量级”**：
  - 提轻量级用 CAS 自旋，适合少竞争；重量级用 Monitor 阻塞，适合高竞争。
- **问“优化机制”**：
  - 提偏向延迟、批量撤销、自适应自旋、锁消除、锁粗化。
- **问“synchronized vs ReentrantLock”**：
  - 提 synchronized 简单、JVM 优化；ReentrantLock 灵活、无升级。

---

### 7. 总结
- **JVM 原理**：
  - `synchronized` 基于 Monitor 实现，对象头存储锁状态，`monitorenter`/`monitorexit` 控制互斥。
  - Monitor 维护 Owner、Entry List、Wait Set，支持阻塞/唤醒/重入。
- **锁升级流程**：
  - 无锁 → 偏向锁（单线程，记录 ID）→ 轻量级锁（少竞争，CAS 自旋）→ 重量级锁（高竞争，Monitor 阻塞）。
  - 升级单向，优化性能（偏向低开销 → 重量级高开销）。
- **优化机制**：
  - 偏向延迟、批量撤销、自旋调整、锁消除/粗化。
- **场景**：
  - 偏向锁：单线程；轻量级锁：少竞争；重量级锁：高竞争。
- **面试建议**：
  - 提 Monitor 结构、字节码指令、锁升级步骤、开销对比、优化机制，举代码（同步块）与图示（Mark Word），清晰展示理解。
