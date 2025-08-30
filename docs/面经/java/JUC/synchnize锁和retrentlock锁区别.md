
### 本质

1. **来源不同**：
    
    - `synchronized` 是 Java 语言的一个关键字，是内置的语言特性。它的实现是基于 JVM 层面的，具体来说是依赖于对象头的 Mark Word 和操作系统的 Monitor（监视器锁）来实现的。
        
    - `ReentrantLock` 是一个纯 Java 实现的 API 类，它位于 `java.util.concurrent.locks` 包下，实现了 `Lock` 接口。它的实现依赖于 JUC (Java Util Concurrent) 包中的核心工具，特别是 AQS (AbstractQueuedSynchronizer)。
        
2. **使用方式不同**：
    
    - `synchronized` 的使用非常简洁，是隐式的。它通过代码块或方法修饰符来使用，JVM 会自动负责锁的获取和释放。即使代码块中发生异常，JVM 也能保证锁被正确释放，不易出错。
        
    - `ReentrantLock` 的使用则需要显式地手动操作。必须在代码中调用 `lock()` 方法来获取锁，并且必须在 `finally` 语句块中调用 `unlock()` 方法来释放锁。这样做是为了确保即使在业务逻辑中出现异常，锁也一定能被释放，否则可能导致死锁。这种方式相对繁琐，也增加了忘记释放锁的风险。
        

### 功能和特性的差异

这是两者最核心的区别，`ReentrantLock` 提供了 `synchronized` 所不具备的许多高级功能。

1. **可中断的等待**：
    
    - `synchronized` 的等待是不可中断的。如果一个线程在等待获取一个被其他线程持有的 `synchronized` 锁，那么它会一直阻塞下去，除非它获取到锁，否则无法被中断。
        
    - `ReentrantLock` 提供了 `lockInterruptibly()` 方法，允许等待锁的线程响应中断。当一个线程正在等待锁时，如果其他线程调用了该等待线程的 `interrupt()` 方法，那么该线程会放弃等待并抛出 `InterruptedException` 异常，这为处理死锁等问题提供了可能。
        
2. **可限时的等待**：
    
    - `synchronized` 没有提供尝试获取锁的机制，只能一直死等。
        
    - `ReentrantLock` 提供了 `tryLock()` 方法。`tryLock()` 会尝试立即获取锁，如果成功则返回 `true`，失败则立即返回 `false`，不会阻塞。它还有一个重载版本 `tryLock(long timeout, TimeUnit unit)`，允许线程在指定的时间内尝试获取锁，超时后如果仍未获取到，则返回 `false`。这个特性在需要避免线程无限期等待的场景中非常有用。
        
3. **公平性选择**：
    
    - `synchronized` 是一种非公平锁。当锁被释放时，任何等待的线程都有机会获取锁，JVM 为了优化吞吐量，可能会让刚来的线程“插队”，而不是严格按照先来后到的顺序。
        
    - `ReentrantLock` 提供了公平性的选择。通过其构造函数 `public ReentrantLock(boolean fair)`，可以创建公平锁或非公平锁（默认为非公平）。公平锁会按照线程请求锁的时间顺序来分配锁，但通常会带来额外的性能开销，降低吞吐量。非公平锁则性能更好。
        
4. **绑定多个条件（Condition）**：
    
    - `synchronized` 配合的线程通信方式是 `wait()`、`notify()` 和 `notifyAll()`。它们都作用于锁对象的同一个监视器，这意味着所有在该锁上等待的线程都在同一个等待队列中。`notify()` 只能随机唤醒一个等待线程，而 `notifyAll()` 会唤醒所有等待线程，但后者效率不高，且可能导致“惊群效应”。
        
    - `ReentrantLock` 通过 `newCondition()` 方法可以获取一个 `Condition` 对象。一个 `ReentrantLock` 可以关联多个 `Condition` 对象。这允许我们实现更精细的线程通信，比如在一个生产者-消费者模型中，可以为“生产者队列未满”和“消费者队列不空”创建两个不同的 `Condition`，从而可以做到定向通知，只唤醒特定类型的线程，大大提高了调度的灵活性和效率。
        

### 性能差异

在早期的 JDK 版本（如 JDK 1.5），`ReentrantLock` 的性能通常优于 `synchronized`。但是，从 JDK 1.6 开始，JVM 团队对 `synchronized` 进行了大量的优化，引入了偏向锁、轻量级锁、自旋锁以及锁消除等技术，使其性能得到了极大的提升。

目前，在锁竞争不激烈的情况下，两者的性能差距已经非常小。`synchronized` 由于其内置的优化，在某些场景下甚至可能表现得更好。因此，在现代 Java 开发中，性能不应再是选择它们的首要决定因素，而应更多地关注功能需求。
