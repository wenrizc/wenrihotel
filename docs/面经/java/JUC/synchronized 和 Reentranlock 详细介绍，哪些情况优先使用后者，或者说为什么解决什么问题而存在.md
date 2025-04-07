
**synchronized 详细介绍**

`synchronized` 是 Java 语言提供的**关键字**，用于实现**互斥同步**。它可以修饰方法或代码块，确保在同一时刻，只有一个线程可以执行被 `synchronized` 修饰的代码。

**特点:**

- **隐式锁:** `synchronized` 的锁的获取和释放是**隐式的**，由 JVM 自动管理。
- **基于 Monitor 对象:** `synchronized` 依赖于 Java 对象的 **Monitor 对象** (也称为管程或监视器锁)。每个 Java 对象都可以关联一个 Monitor 对象。
- **可重入性:** `synchronized` 是**可重入的**，即同一个线程可以多次获取同一个锁，避免了死锁的发生。
- **非公平锁:** `synchronized` 默认是**非公平锁**，线程获取锁的顺序不确定，可能导致某些线程一直无法获取到锁 (饥饿)。
- **阻塞等待:** 当线程尝试获取 `synchronized` 锁时，如果锁已被其他线程持有，则该线程会进入**阻塞状态**，等待锁的释放。
- **单一条件队列:** 每个 Monitor 对象只关联一个**隐式的条件队列**，用于线程的等待和唤醒。

**ReentrantLock 详细介绍**

`ReentrantLock` 是 Java `java.util.concurrent.locks` 包下提供的一个**类**，实现了 `Lock` 接口，提供了比 `synchronized` 更强大、更灵活的锁机制。

**特点:**

- **显式锁:** `ReentrantLock` 的锁的获取和释放是**显式的**，需要手动调用 `lock()` 方法获取锁，`unlock()` 方法释放锁。
- **基于 AQS (AbstractQueuedSynchronizer):** `ReentrantLock` 基于 **AQS 框架**实现，AQS 是一个用于构建锁和同步器的框架。
- **可重入性:** `ReentrantLock` 也是**可重入的**。
- **公平锁和非公平锁:** `ReentrantLock` 可以创建**公平锁** (FairSync) 或 **非公平锁** (NonfairSync)。
    - **公平锁:** 线程按照请求锁的顺序获取锁，先到先得，避免饥饿，但性能相对较低。
    - **非公平锁:** 线程获取锁的顺序不确定，可能导致饥饿，但性能相对较高，默认是**非公平锁**。
- **可中断等待:** 使用 `ReentrantLock` 可以响应中断，当线程在等待锁的过程中，可以被其他线程中断，并抛出 `InterruptedException` 异常。
- **超时等待:** `ReentrantLock` 提供了 `tryLock(long timeout, TimeUnit unit)` 方法，允许线程在等待锁时设置超时时间，如果在超时时间内未能获取到锁，则返回 `false`，线程可以执行其他操作。
- **非阻塞获取锁:** `ReentrantLock` 提供了 `tryLock()` 方法，尝试非阻塞地获取锁，如果立即获取到锁则返回 `true`，否则返回 `false`，线程不会被阻塞。
- **多个条件队列:** `ReentrantLock` 可以通过 `newCondition()` 方法创建**多个条件队列**，每个条件队列可以绑定不同的等待条件，实现更精细化的线程等待和唤醒控制。


**synchronized 和 ReentrantLock 的对比**

|特性|`synchronized`|`ReentrantLock`|
|---|---|---|
|**锁类型**|隐式锁|显式锁|
|**实现机制**|Monitor 对象|AQS (AbstractQueuedSynchronizer)|
|**可重入性**|可重入|可重入|
|**公平性**|非公平锁 (默认)|公平锁和非公平锁 (可选择)|
|**等待特性**|阻塞等待|可中断等待、超时等待、非阻塞获取锁|
|**条件队列**|单一隐式条件队列|可以创建多个条件队列|
|**锁信息获取**|较难获取锁状态|可以获取锁状态 (是否被持有、等待队列长度等)|
|**性能**|JDK 1.6 后性能优化，与 `ReentrantLock` 差距缩小|某些场景下可能略优于 `synchronized` (特别是公平锁场景)|
|**使用复杂度**|简单易用|使用更灵活，但需要手动管理锁的获取和释放，容易出错|
|**异常处理**|自动释放锁|需要在 `finally` 块中手动释放锁，否则可能导致死锁|


**哪些情况优先使用 ReentrantLock，或者说为什么解决什么问题而存在**

`ReentrantLock` 的存在主要是为了解决 `synchronized` 在某些场景下的局限性，提供更强大、更灵活的锁机制，以应对更复杂的并发需求。

**优先使用 `ReentrantLock` 的情况:**

1. **需要公平锁:** 当需要保证线程获取锁的公平性，避免某些线程饥饿时，必须使用 `ReentrantLock` 并创建公平锁。`synchronized` 无法实现公平锁。
    
2. **需要可中断等待:** 当线程在等待锁的过程中，需要能够响应中断 (例如取消操作) 时，必须使用 `ReentrantLock`。`synchronized` 的等待是不可中断的。
    
3. **需要超时等待:** 当线程在等待锁时，不希望无限期等待，而是希望在一定时间内尝试获取锁，超时后可以执行其他操作时，必须使用 `ReentrantLock` 的 `tryLock(timeout)` 方法。`synchronized` 没有超时等待机制。
    
4. **需要非阻塞获取锁:** 当线程需要尝试非阻塞地获取锁，立即知道是否获取成功，而不是一直等待时，可以使用 `ReentrantLock` 的 `tryLock()` 方法。`synchronized` 只能阻塞获取锁。
    
5. **需要多个条件队列:** 当需要实现更复杂的线程等待和唤醒机制，例如生产者-消费者模式中，需要区分 "缓冲区满" 和 "缓冲区空" 两种等待条件时，可以使用 `ReentrantLock` 创建多个条件队列 (Condition)。`synchronized` 只有一个隐式的条件队列，难以实现复杂的条件控制。
    
6. **需要获取锁的详细信息:** `ReentrantLock` 提供了 API 可以查询锁的状态，例如是否被持有、等待队列的长度等，方便监控和诊断并发问题。`synchronized` 很难获取这些信息。

**总结:**

- **简单场景:** 如果只是简单的互斥同步，对公平性、中断、超时、条件队列等没有特殊要求，`synchronized` 通常是更简洁易用的选择。JDK 1.6 之后 `synchronized` 的性能也得到了很大的提升，在很多场景下性能不输于 `ReentrantLock`。
- **复杂场景:** 当需要公平锁、可中断等待、超时等待、非阻塞获取锁、多个条件队列等高级功能时，或者需要更精细地控制锁的获取和释放时，`ReentrantLock` 是更合适的选择。

**选择建议:**

- **优先考虑 `synchronized`:** 在大多数情况下，`synchronized` 已经足够满足需求，并且更简单易用。
- **复杂场景使用 `ReentrantLock`:** 当遇到 `synchronized` 无法解决的问题，或者需要更高级的锁功能时，再考虑使用 `ReentrantLock`。