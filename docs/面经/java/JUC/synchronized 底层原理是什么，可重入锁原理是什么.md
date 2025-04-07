
#### synchronized 底层原理
- **`synchronized`** 是 Java 提供的内置锁机制，底层基于 **对象监视器（Monitor）** 和 **JVM 的锁优化** 实现。
- **核心**：
  - 通过 Monitor 对象管理线程的互斥和同步。
  - 底层用 **monitorenter** 和 **monitorexit** 指令实现。

#### 可重入锁原理
- **可重入锁（Reentrant Lock）**：
  - 同一个线程可多次获取同一锁而不死锁。
- **实现**：
  - Monitor 记录锁持有者和重入次数。

---

### 1. synchronized 底层原理详解
#### (1) 基本概念
- **作用**：
  - 确保同一时刻只有一个线程访问同步块或方法。
- **使用**：
  - 同步块：`synchronized (obj) {}`。
  - 同步方法：`synchronized void method()`。

#### (2) Monitor 机制
- **对象监视器**：
  - 每个对象有一个 Monitor（隐式绑定）。
  - Monitor 包含：
    - **Owner**：当前持有锁的线程。
    - **Entry List**：等待锁的线程队列。
    - **Wait Set**：调用 `wait()` 的线程。
- **状态**：
  - 未锁、锁定、等待。

#### (3) 字节码层面
- **指令**：
  - `monitorenter`：进入同步区，获取锁。
  - `monitorexit`：退出同步区，释放锁。
- **示例**：
```java
synchronized (obj) {
    // 代码
}
// 字节码：
monitorenter
// 同步代码
monitorexit
```

#### (4) 对象头与锁状态
- **对象头（Mark Word）**：
  - 存储锁信息（HotSpot 用 64 位）。
  - 结构：
    - 无锁：哈希码、GC 分代。
    - 锁状态：轻量锁指针、重量锁指针。
- **锁升级**（JDK 1.6+ 优化）：
  1. **无锁**：
     - 未加锁状态。
  2. **偏向锁**：
     - 单线程场景，记录线程 ID。
  3. **轻量锁**：
     - 无竞争或低竞争，用 CAS 更新指针。
  4. **重量锁**：
     - 多线程竞争，膨胀为 Monitor。

#### (5) 锁优化
- **偏向锁**：
  - 减少无竞争时的开销，CAS 标记线程。
- **轻量锁**：
  - CAS 在栈帧中记录锁，避免 Monitor。
- **锁粗化**：
  - 合并多次加锁为一次。
- **锁消除**：
  - JIT 编译器移除无意义的锁（如局部对象）。

#### 图示
```
无锁 -> 偏向锁 (单线程) -> 轻量锁 (低竞争) -> 重量锁 (高竞争)
```

---

### 2. 可重入锁原理详解
#### (1) 定义
- **可重入**：
  - 同一线程可多次获取同一锁，每次加锁计数加 1，释放时减 1，直至 0 才真正释放。

#### (2) synchronized 的可重入实现
- **Monitor 记录**：
  - 对象头的 Mark Word 或 Monitor 包含：
    - **线程 ID**：标识锁拥有者。
    - **计数器**：记录重入次数。
- **过程**：
  1. 线程首次获取锁：
     - Monitor 标记线程，计数器设为 1。
  2. 再次进入：
     - 检查线程 ID，相等则计数器 +1。
  3. 退出：
     - 每次 `monitorexit`，计数器 -1，减到 0 释放。
- **示例**：
```java
class Demo {
    synchronized void method1() {
        method2(); // 可重入
    }
    synchronized void method2() {
        System.out.println("Running");
    }
}

Demo demo = new Demo();
demo.method1(); // 同一线程两次加锁
```

#### (3) 与非可重入对比
- **非可重入**：
  - 线程二次获取锁会阻塞自己。
- **`synchronized`**：
  - 通过计数器避免自阻塞。

---

### 3. 源码与 JVM 实现
- **HotSpot**：
  - `monitorenter` 检查 Monitor：
    - 若空，获取并设计数器。
    - 若已持有，计数器 +1。
  - `monitorexit` 减计数器，0 时释放。
- **对象头变化**：
```
无锁:   | hashCode | age | 0 |
偏向锁: | threadID | epoch | 1 |
轻量锁: | 栈帧指针        | 00 |
重量锁: | Monitor 指针     | 10 |
```

---

### 4. 延伸与面试角度
- **与 ReentrantLock 对比**：
  - `synchronized`：JVM 内置，简单。
  - `ReentrantLock`：显式锁，支持超时、公平锁。
- **性能**：
  - JDK 1.6+ 优化后，`synchronized` 与 `ReentrantLock` 接近。
- **实际应用**：
  - 同步块：保护共享资源。
  - 可重入：递归调用。
- **面试点**：
  - 问“原理”时，提 Monitor 和锁升级。
  - 问“可重入”时，提计数器。

---

### 总结
`synchronized` 底层靠 Monitor 和对象头实现，锁升级优化性能；可重入通过线程 ID 和计数器支持同一线程多次加锁。面试时，可提字节码或画锁状态图，展示理解深度。