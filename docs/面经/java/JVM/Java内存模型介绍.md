## 1. JMM 的核心抽象：主内存与工作内存

为了定义这套规则，JMM 提出了一个抽象的模型，将内存划分为两个逻辑部分：

1.  主内存 (Main Memory)：

    这是一个所有线程共享的区域，可以近似地对应于物理内存中的 Java 堆。
    Java 中所有的实例变量、静态变量都存储在主内存中。

2.  工作内存 (Working Memory)：

    这是每个线程私有的区域，可以近似地对应于 CPU 的高速缓存或寄存器。
    线程对变量的所有操作（读取、赋值等）都必须在自己的工作内存中进行，而不能直接读写主内存。
    不同线程之间也无法直接访问对方的工作内存，线程间的通信必须通过主内存来完成。

一个线程修改共享变量的完整流程如下：

1.  Read: 从主内存中读取变量的值。
2.  Load: 将读取到的值加载到工作内存的变量副本中。
3.  Use: 线程在工作内存中使用这个变量副本。
4.  Assign: 将计算后的新值赋给工作内存中的变量副本。
5.  Store: 将工作内存中变量副本的新值，存储到主内存中。
6.  Write: 将存储到主内存的值，更新到主内存的变量中。

JMM 正是通过定义这一系列操作的原子性、可见性和有序性规则，来保证并发的正确性。

## 2. JMM 提供的并发保证与工具

JMM 为我们开发者提供了几个关键的“武器”，来确保我们编写的并发程序能够正确地处理可见性和有序性问题。这些保证，我们称之为 Happens-Before 原则。

Happens-Before 原则是 JMM 的核心，它定义了在多线程环境中，两个操作之间存在的偏序关系。如果操作 A happens-before 操作 B，那么 A 操作的结果对 B 操作是可见的，并且 A 操作的执行顺序在 B 操作之前。

JMM 提供了以下几种方式来建立 Happens-Before 关系：

1.  程序次序规则 (Program Order Rule)：

    在一个线程内，按照代码的先后顺序，前面的操作 happens-before 后面的操作。这是单线程内有序性的基本保证。

2.  管程锁定规则 (Monitor Lock Rule)：

    一个 unlock 操作 happens-before 后续对同一个锁的 lock 操作。这就是 synchronized 关键字的魔力所在。当一个线程释放锁时，它会强制将自己工作内存中的修改刷新到主内存；当另一个线程获取锁时，它会清空自己的工作内存，强制从主内存加载最新的值。这同时保证了原子性和可见性。

3.  volatile 变量规则 (Volatile Variable Rule)：

    对一个 volatile 变量的写操作 happens-before 后续对这个变量的读操作。volatile 通过内存屏障，实现了两点：
    保证可见性：写操作会立即将新值刷新到主内存，并使其他线程的缓存失效。
    禁止指令重排序：保证了代码的有序性。

4.  线程启动规则 (Thread Start Rule)：

    Thread 对象的 start()方法 happens-before 此线程的任何一个动作。

5.  线程终止规则 (Thread Termination Rule)：

    线程中的所有操作都 happens-before 对此线程的终止检测（如 thread.join()或 thread.isAlive()的返回）。

6.  线程中断规则 (Thread Interruption Rule)：

    对线程 interrupt()方法的调用 happens-before 被中断线程的代码检测到中断事件的发生。

7.  传递性 (Transitivity)：

    如果 A happens-before B，且 B happens-before C，那么 A happens-before C。


