
`wait()` 和 `sleep()` 是Java多线程中两个非常重要且容易混淆的方法。它们都能让当前线程暂停执行，但其设计目的、工作机制和对锁的处理方式完全不同。

下面是它们之间的核心区别，我将通过一个表格和详细解释来进行说明。

### 核心区别一览

| 特性 | `Object.wait()` | `Thread.sleep()` |
| :--- | :--- | :--- |
| **所属的类** | `java.lang.Object` | `java.lang.Thread` |
| **对锁的处理** | **会释放持有的监视器锁** | **不会释放任何锁** |
| **主要用途** | 线程间的协作与通信 | 暂停当前线程的执行，通常用于控制执行节奏 |
| **唤醒方式** | 需由其他线程调用`notify()`或`notifyAll()`来唤醒 | 1. 睡眠时间到期后自动唤醒<br>2. 被其他线程调用`interrupt()`中断 |
| **使用前提** | **必须在 `synchronized` 代码块或方法中调用** | 可以在任何地方调用 |
| **抛出异常** | 抛出 `InterruptedException` | 抛出 `InterruptedException` |

---

### 详细解释

#### 1. 所属的类不同

*   **`wait()`**: 这是 `java.lang.Object` 类的方法。这意味着Java中的任何对象都可以调用 `wait()` 方法（以及 `notify()` 和 `notifyAll()`）。
*   **`sleep()`**: 这是 `java.lang.Thread` 类的静态方法。调用时通常写作 `Thread.sleep()`，它作用于**当前正在执行的线程**，而不是指定的某个线程。

#### 2. 对锁（监视器锁）的处理方式是根本区别

这是两者之间最本质、最重要的区别。

*   **`wait()` 会释放锁**:
    *   当一个线程调用某个对象的 `wait()` 方法时，它会**立即释放**它所持有的该对象的监视器锁（即`synchronized`锁）。
    *   释放锁后，其他等待该锁的线程就有机会获得锁并进入 `synchronized` 代码块。
    *   这使得 `wait()` 成为实现线程间高效协作和通信的关键。一个线程可以等待某个条件成立，而在它等待期间，其他线程可以进入并修改这个条件。

*   **`sleep()` 不会释放锁**:
    *   当一个线程调用 `Thread.sleep()` 时，它仅仅是让出了CPU的执行时间片，进入休眠状态。
    *   如果这个线程在休眠前持有了任何监视器锁，它在整个休眠期间会**一直持有这些锁，不会释放**。
    *   这意味着，如果 `sleep()` 是在一个 `synchronized` 代码块中被调用的，那么其他线程在这段休眠时间内将无法进入该代码块，因为它们无法获取锁。这可能会导致性能问题甚至死锁。

#### 3. 主要用途和设计目的不同

*   **`wait()` 用于线程间的通信/协作 (Inter-thread communication)**:
    *   它的设计场景是：一个线程需要等待另一个线程完成某个操作或满足某个条件后才能继续执行。例如，在经典的“生产者-消费者”模型中，当队列为空时，消费者线程会调用 `wait()` 等待，直到生产者线程放入商品并调用 `notify()` 将其唤醒。

*   **`sleep()` 用于暂停当前线程 (Pausing execution)**:
    *   它的设计场景很简单：就是想让当前线程暂停执行一段时间。例如，你可能想在一个循环中减慢执行速度，或者模拟一个耗时操作，或者等待某个外部资源准备就绪。它不涉及线程间的直接通信。

#### 4. 唤醒方式不同

*   **`wait()`**: 线程调用 `wait()` 后会进入对象的等待集（Wait Set），它无法自己醒来。必须依赖于**其他线程**对**同一个对象**调用 `notify()` 或 `notifyAll()` 方法来将其唤醒。被唤醒后，该线程并不会立即执行，而是会进入入口集（Entry Set）重新竞争锁，只有当它再次获得锁之后，才能从 `wait()` 的调用处继续执行。
*   **`sleep()`**: 线程主要通过两种方式恢复执行：
    1.  **睡眠时间结束**: 当指定的时间（毫秒）过去后，线程会自动变为就绪（Runnable）状态，等待CPU调度。
    2.  **被中断**: 其他线程可以调用该休眠线程的 `interrupt()` 方法，这会让 `sleep()` 抛出 `InterruptedException`，从而提前结束休眠。

#### 5. 使用前提条件不同

*   **`wait()`**: 必须在 `synchronized` 代码块或 `synchronized` 方法中调用。这是因为调用 `wait()` 的前提是线程必须持有该对象的监视器锁。如果在没有锁的情况下调用，JVM无法知道你要等待的是哪个锁，也无法进行后续的锁释放和重入操作，因此会抛出 `IllegalMonitorStateException`。
*   **`sleep()`**: 可以在代码的任何地方调用，没有特殊限制。

### 代码示例对比

**使用 `sleep()`（不释放锁）**

```java
public class SleepExample {
    private static final Object LOCK = new Object();

    public static void main(String[] args) {
        new Thread(() -> {
            synchronized (LOCK) {
                System.out.println("Thread A got the lock.");
                try {
                    System.out.println("Thread A is sleeping for 2 seconds.");
                    Thread.sleep(2000); // 持有锁并休眠
                    System.out.println("Thread A woke up.");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();

        // 稍微等待，确保线程A先启动并获得锁
        try { Thread.sleep(100); } catch (InterruptedException e) {}

        new Thread(() -> {
            synchronized (LOCK) {
                // 这段代码必须等待线程A休眠结束并释放锁后才能执行
                System.out.println("Thread B got the lock.");
            }
        }).start();
    }
}
// 输出:
// Thread A got the lock.
// Thread A is sleeping for 2 seconds.
// (等待约2秒)
// Thread A woke up.
// Thread B got the lock.
```
**分析**: 线程B在线程A休眠的2秒内完全被阻塞，因为它无法获得`LOCK`对象锁。

**使用 `wait()`（释放锁）**

```java
public class WaitExample {
    private static final Object LOCK = new Object();

    public static void main(String[] args) {
        new Thread(() -> {
            synchronized (LOCK) {
                System.out.println("Thread A got the lock.");
                try {
                    System.out.println("Thread A is waiting.");
                    LOCK.wait(); // 释放锁并进入等待
                    System.out.println("Thread A has been notified and got the lock again.");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();

        // 稍微等待，确保线程A先启动并进入wait状态
        try { Thread.sleep(100); } catch (InterruptedException e) {}

        new Thread(() -> {
            synchronized (LOCK) {
                System.out.println("Thread B got the lock because Thread A released it.");
                LOCK.notify(); // 唤醒正在等待LOCK对象的线程
                System.out.println("Thread B notified Thread A.");
                // 线程B将在这里执行完同步块后才真正释放锁
            }
        }).start();
    }
}
// 输出:
// Thread A got the lock.
// Thread A is waiting.
// Thread B got the lock because Thread A released it.
// Thread B notified Thread A.
// Thread A has been notified and got the lock again.
```
**分析**: 线程A调用`wait()`后立即释放了锁，所以线程B能够马上获得锁并执行。线程B调用`notify()`后，线程A被唤醒，并在线程B释放锁之后重新获得锁并继续执行。