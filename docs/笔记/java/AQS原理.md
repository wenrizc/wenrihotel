
#### ReentrantLock

##### 特性

`ReentrantLock` 是一种可重入锁，意味着同一个线程可以多次获取同一把锁。

|               | ReentrantLock              | Synchronized                     |
| :------------ | :------------------------- | :------------------------------- |
| **锁类型**       | 默认非公平，可选择公平锁               | 非公平锁                             |
| **实现机制**      | 基于AQS框架，CAS + 等待队列         | JVM层面实现，基于Monitor对象              |
| **灵活性**       | 支持中断、超时、尝试获取锁              | 不支持                              |
| **可重入性**      | 是                          | 是                                |
| **Condition** | 支持多个Condition，实现精确唤醒       | 仅支持一个等待队列，通过`wait`/`notifyAll`唤醒 |
| **释放锁**       | 必须在`finally`块中手动`unlock()` | JVM自动释放                          |

##### ReentrantLock 与 AQS 的关联

`ReentrantLock` 内部包含一个 `Sync` 抽象类，它继承自 AQS。`Sync` 有两个具体的子类：`FairSync` (公平锁) 和 `NonfairSync` (非公平锁)。

*   **核心关联点：** `ReentrantLock` 的加锁 (`lock`) 和解锁 (`unlock`) 操作，最终都委托给了其内部的 `Sync` 对象，而 `Sync` 的核心方法（如 `acquire`, `release`）则直接调用或依赖于 AQS 提供的框架方法。

**加锁流程的差异：**

*   **非公平锁 (`NonfairSync`) `lock()` 流程：**
    1.  尝试通过 CAS (Compare-And-Set) 直接将同步状态 `state` 从 0 修改为 1。
    2.  如果成功，表示获取锁成功，将当前线程设置为锁的持有者。
    3.  如果失败（锁已被占用或CAS竞争失败），则调用 AQS 的 `acquire(1)` 方法，进入排队等待流程。
*   **公平锁 (`FairSync`) `lock()` 流程：**
    1.  不尝试抢占，直接调用 AQS 的 `acquire(1)` 方法。

两种锁的实现虽然入口不同，但最终都会走到 AQS 的 `acquire` 方法，后续的排队、阻塞、唤醒逻辑都由 AQS 统一处理。

#### AQS (AbstractQueuedSynchronizer) 核心原理

AQS 是一个用于构建锁和同步器的框架，其核心思想是：

*   **核心状态：** 使用一个 `volatile int state` 变量作为同步状态。`state` 为 0 表示资源未被占用，大于 0 表示被占用。
*   **核心机制：** 如果资源空闲，请求线程获取锁并更新 `state`。如果资源被占用，则将线程封装成节点加入一个 **CLH 队列的变体** 中进行排队等待。

##### AQS 框架与数据结构

**AQS 框架分层：**

1.  **API层：** 暴露给自定义同步器重写的方法，如 `tryAcquire`, `tryRelease`。
2.  **获取/释放层：** AQS 提供的核心方法，如 `acquire`, `release`，它们会调用 API 层的方法。
3.  **队列操作层：** 封装了对等待队列的原子操作，如 `addWaiter`, `enq`。
4.  **等待/唤醒层：** 负责线程的阻塞 (`park`) 和唤醒 (`unpark`)。
5.  **基础数据层：** 包括 `state` 变量和等待队列的头 (`head`)、尾 (`tail`) 指针。

**核心数据结构 - `Node`：**

等待队列中的每个节点 (`Node`) 代表一个等待锁的线程。

*   **`thread`**: 节点关联的线程。
*   **`prev`, `next`**: 双向链表的前后指针。
*   **`waitStatus`**: 节点状态，非常关键。
    *   **`0` (默认)**: 节点的初始状态。
    *   **`SIGNAL` (-1)**: 表示后继节点已准备好，当前节点释放锁时需要唤醒 (`unpark`) 后继节点。
    *   **`CANCELLED` (1)**: 表示节点内的线程已取消等待（如超时或中断）。
    *   **`CONDITION` (-2)**: 用于 `Condition` 队列，表示节点正在等待条件。
    *   **`PROPAGATE` (-3)**: 用于共享锁，表示释放操作会向后传播。
*   **`nextWaiter`**: 用于 `Condition` 队列，指向下一个等待条件的节点。
*   **锁模式**: `SHARED` (共享) 和 `EXCLUSIVE` (独占)。

AQS 定义了一些 `protected` 方法，需要同步器（如 `ReentrantLock`）去实现它们，从而定义自己的同步逻辑。

| AQS 方法 (protected) | ReentrantLock (独占锁) 实现 | 描述 |
| :--- | :--- | :--- |
| `tryAcquire(int arg)` | **是** (`FairSync` / `NonfairSync`) | 尝试以独占方式获取锁。成功返回 `true`，失败返回 `false`。 |
| `tryRelease(int arg)` | **是** (`Sync`) | 尝试释放锁。成功返回 `true`，失败返回 `false`。 |
| `isHeldExclusively()` | **是** | 判断锁是否被当前线程独占。 |

**交互流程：**

*   **加锁 (`lock` -> `acquire`)**:
    1.  `ReentrantLock.lock()` 调用 `Sync.lock()`。
    2.  `Sync.lock()` (公平/非公平) 调用 AQS 的 `acquire()`。
    3.  AQS `acquire()` 调用 `ReentrantLock` 自己实现的 `tryAcquire()`。
    4.  如果 `tryAcquire()` 失败，AQS 框架负责将线程入队、挂起等后续操作。
*   **解锁 (`unlock` -> `release`)**:
    1.  `ReentrantLock.unlock()` 调用 `Sync.release()`。
    2.  `Sync.release()` 是 AQS 的方法，它会调用 `ReentrantLock` 自己实现的 `tryRelease()`。
    3.  如果 `tryRelease()` 成功（锁完全释放），AQS 框架负责唤醒等待队列中的后继节点。

#### QS 独占锁核心流程(通过 ReentrantLock)

##### 线程加锁与入队 (`acquire` -> `addWaiter` -> `acquireQueued`)

当 `tryAcquire()` 失败后，AQS 执行 `acquire(int arg)` 方法中的核心逻辑：`!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg)`

1.  **`addWaiter(Node.EXCLUSIVE)`**:
    *   将当前线程和独占模式 (`EXCLUSIVE`) 封装成一个新的 `Node`。
    *   **快速入队尝试**：通过 CAS 操作尝试将新节点直接设置为尾节点 (`tail`)。
        *   `node.prev = tail;`
        *   `compareAndSetTail(pred, node);`
        *   `pred.next = node;`
    *   **完整入队 (`enq`)**: 如果快速尝试失败（如队列为空或并发修改），则进入 `enq` 方法自旋。
        *   如果队列为空 (`tail == null`)，则初始化队列，创建一个虚拟的头节点 (`head`)，然后让 `tail` 指向它。
        *   通过 CAS 不断尝试将新节点设置为尾节点，直到成功为止。
    *   **注意**：队列的头节点是一个不关联任何线程的虚节点，仅作为标记。

2.  **`acquireQueued(final Node node, int arg)`**:
    *   节点入队后，此方法负责让节点在队列中自旋、阻塞和最终获取锁。
    *   **自旋 (`for (;;)`):**
        *   **检查是否可以获取锁**: 获取当前节点 `node` 的前驱节点 `p`。如果 `p == head`，说明当前节点是队列中的第一个有效节点，有资格尝试获取锁。
        *   **获取锁成功**: 如果 `p == head` 并且 `tryAcquire(arg)` 成功：
            1.  将当前节点 `node` 设置为新的头节点 (`setHead(node)`)，并清空其线程信息，使其变为新的虚节点。
            2.  将旧的头节点的 `next` 指针置为 `null`，帮助 GC。
            3.  方法返回，加锁流程结束。
        *   **获取锁失败**: 如果 `p != head` 或 `tryAcquire(arg)` 失败，则进入阻塞逻辑。

3.  **线程阻塞 (`shouldParkAfterFailedAcquire` -> `parkAndCheckInterrupt`)**:
    *   **`shouldParkAfterFailedAcquire(p, node)`**: 判断当前线程是否应该被挂起 (`park`)。
        *   检查前驱节点 `p` 的 `waitStatus`。
        *   如果 `p.waitStatus == Node.SIGNAL (-1)`，说明前驱节点已承诺会唤醒自己，所以当前线程可以安全地挂起。返回 `true`。
        *   如果 `p.waitStatus > 0` (即 `CANCELLED`)，说明前驱节点已取消，则向前遍历，将当前节点链接到第一个未被取消的节点上。返回 `false`，让外层循环继续。
        *   如果 `p.waitStatus` 是 `0` 或 `PROPAGATE`，则通过 CAS 将其状态设置为 `SIGNAL`，表示“我需要你将来唤醒我”。返回 `false`，让外层循环再试一次，下次循环时 `waitStatus` 就是 `SIGNAL` 了。
    *   **`parkAndCheckInterrupt()`**: 如果 `shouldParkAfterFailedAcquire` 返回 `true`，则调用 `LockSupport.park(this)` 挂起当前线程，线程将在此处停止执行，直到被 `unpark`。

##### 取消状态节点生成 (`cancelAcquire`)

如果在 `acquireQueued` 的 `try` 块中发生异常（如中断），`finally` 块中的 `cancelAcquire(node)` 会被调用。

*   **核心逻辑**: 将一个节点从等待队列中移除。
    1.  将节点的 `thread` 设为 `null`。
    2.  向前遍历，跳过所有已是 `CANCELLED` 状态的前驱节点。
    3.  将当前节点的 `waitStatus` 设为 `Node.CANCELLED`。
    4.  重新连接链表：将**有效的前驱节点**的 `next` 指针，指向当前节点的 `next` 指针，从而在逻辑上“跳过”当前节点。
    5.  如果当前节点是尾节点，则更新 `tail` 指针。
    6.  如果以上都不满足，则尝试唤醒后继节点 `unparkSuccessor(node)`。
*   **Prev 指针的处理**: `cancelAcquire` 主要处理 `next` 指针。`prev` 指针的修正发生在 `shouldParkAfterFailedAcquire` 中，因为那时队列相对稳定。

##### 解锁与唤醒 (`release` -> `unparkSuccessor`)

1.  **`release(int arg)`**:
    *   调用自定义同步器实现的 `tryRelease(arg)`。对于 `ReentrantLock`，此方法会减少 `state` 的值。
    *   如果 `tryRelease` 返回 `true` (表示锁已完全释放，`state` 变为 0)，则执行唤醒逻辑。
    *   获取头节点 `h`，如果 `h != null` 且 `h.waitStatus != 0`，则调用 `unparkSuccessor(h)`。
        *   `h.waitStatus != 0`: 通常意味着 `waitStatus` 是 `SIGNAL`，表明有后继节点在等待被唤醒。

2.  **`unparkSuccessor(Node node)`**:
    *   `node` 参数此时是头节点。此方法负责唤醒它的后继节点。
    *   获取头节点的后继节点 `s = node.next`。
    *   **核心逻辑**：如果 `s` 为 `null` 或 `s` 已被取消 (`waitStatus > 0`)，则**从尾部向前遍历**队列，找到离头节点最近的、未被取消的节点。
    *   **为什么从后往前遍历？**
        1.  `addWaiter` 入队时，`node.prev = tail` 和 `compareAndSetTail` 是原子性的，但 `pred.next = node` 这一步在之后。在极端并发下，从前向后遍历可能无法触及新加入的节点。
        2.  `cancelAcquire` 时，会先断开 `next` 指针，`prev` 指针的连接相对可靠。
    *   找到合适的后继节点 `s` 后，调用 `LockSupport.unpark(s.thread)` 唤醒其线程。

##### 中断处理

*   AQS 对中断的处理是**协作式**的。在等待过程中，如果线程被中断，`parkAndCheckInterrupt` 会返回 `true`，`acquireQueued` 方法会记录中断状态 (`interrupted = true`)，但**并不会立即停止获取锁**。
*   线程会继续尝试获取锁。
*   当成功获取锁后，`acquireQueued` 返回记录的中断状态。
*   如果返回 `true`，`acquire` 方法会调用 `selfInterrupt()`，即 `Thread.currentThread().interrupt()`，重新设置当前线程的中断标志。这是一种“延迟”中断，将中断的响应权交还给调用者。

#### AQS 的应用

##### ReentrantLock 的可重入性实现

可重入性是通过 `tryAcquire` 方法中的逻辑实现的：

1.  获取当前 `state` 值 `c`。
2.  如果 `c == 0`，表示锁未被持有，尝试通过 CAS 获取锁。
3.  如果 `c > 0`，则检查当前锁的持有者是否是当前线程 (`current == getExclusiveOwnerThread()`)。
4.  如果是，将 `state` 的值加 1 (`nextc = c + acquires`)，代表重入次数增加。
5.  解锁时 (`tryRelease`)，相应地将 `state` 减 1，直到 `state` 减为 0，锁才被真正释放。

##### JUC 中的其他应用

| 同步工具 | AQS 应用方式 |
| :--- | :--- |
| **Semaphore** | `state` 用于表示当前可用的信号量许可数量。`acquire` 减少 `state`，`release` 增加 `state`。使用共享模式。 |
| **CountDownLatch** | `state` 用于表示倒计数的计数值。`countDown()` 方法通过 `release` 减少 `state`。当 `state` 为 0 时，所有在 `await()` 的线程被唤醒。 |
| **ReentrantReadWriteLock** | `state` 被拆分为两部分：高 16 位表示读锁的持有数量，低 16 位表示写锁的重入次数。 |
| **ThreadPoolExecutor** | 其内部工作线程 `Worker` 继承自 AQS，使用 `state` 来实现对线程的独占控制，防止任务执行时被不当中断。 |

##### 自定义同步工具示例 (`LeeLock`)

通过继承 AQS，可以轻松实现一个简单的、不可重入的独占锁：

1.  创建一个私有静态内部类 `Sync` 继承 `AbstractQueuedSynchronizer`。
2.  重写 `tryAcquire`：使用 `compareAndSetState(0, 1)`，成功则获取锁。
3.  重写 `tryRelease`：直接 `setState(0)` 释放锁。
4.  重写 `isHeldExclusively`：判断 `getState() == 1`。
5.  外部 `lock()` 和 `unlock()` 方法分别调用 `sync.acquire(1)` 和 `sync.release(1)`。

这个例子完美展示了 AQS 作为同步器框架的强大之处：开发者只需关注状态的获取与释放逻辑，而线程的排队、阻塞、唤醒等复杂工作全部由 AQS 框架完成。