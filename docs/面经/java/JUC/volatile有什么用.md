
`volatile`关键字主要有两大核心作用：

1.  保证可见性（Visibility）
2.  禁止指令重排序（Ordering）

但需要特别强调的是，`volatile`**不保证原子性**。

### 保证可见性

这是`volatile`最核心、最直接的作用。

*   是什么意思？
    可见性指的是，当一个线程修改了被`volatile`修饰的共享变量的值时，这个修改能够立即被其他所有线程“看到”。

*   如何实现的？
    当一个线程对`volatile`变量进行写操作时，JMM（Java内存模型）会强制它做两件事：
    1.  立即将当前线程工作内存中该变量的最新值，刷新到主内存中。
    2.  这个写操作会导致其他线程工作内存中，缓存的该变量的旧值失效。
    当其他线程需要读取这个`volatile`变量时，它会发现自己工作内存中的值已经失效了，于是就必须去主内存中重新读取最新的值。通过这样一“写”一“读”的强制同步，就保证了变量的可见性。

*   应用场景：
    一个非常经典的场景就是使用一个`volatile`布尔标志来优雅地停止线程。
    ```java
    public class Worker extends Thread {
        private volatile boolean running = true;

        public void shutdown() {
            running = false;
        }

        @Override
        public void run() {
            while (running) {
                // ... do some work
            }
            System.out.println("Thread stopped.");
        }
    }

    // 在主线程中
    Worker worker = new Worker();
    worker.start();
    // ... 一段时间后
    worker.shutdown(); // 主线程的修改，能被worker线程立即看到
    ```
    如果没有`volatile`，`worker`线程很可能因为缓存了`running = true`而陷入死循环，无法停止。

### 禁止指令重排序

*   是什么意思？
    为了提高性能，编译器和处理器可能会对代码进行指令重排序。`volatile`关键字可以作为一个“内存屏障”（Memory Barrier），阻止它前后的指令被重排序，从而保证代码的执行顺序。

*   如何实现的？
    JMM会对`volatile`变量的读写操作，插入特定的内存屏障指令：
    *   在每个`volatile`写操作的**前面**，插入一个StoreStore屏障，保证之前的普通写操作都已完成。
    *   在每个`volatile`写操作的**后面**，插入一个StoreLoad屏障，防止它与后面的读操作重排序。
    *   在每个`volatile`读操作的**后面**，插入一个LoadLoad和LoadStore屏障，保证后续的读写操作不会被重排到它前面。

*   应用场景：
    最经典的应用就是解决双重检查锁定（Double-Checked Locking, DCL）单例模式的有序性问题。
    ```java
    public class Singleton {
        private static volatile Singleton instance; // 关键在于volatile

        private Singleton() {}

        public static Singleton getInstance() {
            if (instance == null) { // 第一次检查
                synchronized (Singleton.class) {
                    if (instance == null) { // 第二次检查
                        instance = new Singleton(); // 问题就出在这里
                    }
                }
            }
            return instance;
        }
    }
    ```
    `instance = new Singleton()`这个操作不是原子的，它可能被重排序为：
    1.  分配内存。
    2.  将`instance`引用指向内存。
    3.  初始化对象。
    如果没有`volatile`，线程A执行到第2步时，`instance`已经不为null，此时线程B进来判断，会直接返回一个尚未初始化的“半成品”对象。
    而`volatile`关键字通过禁止指令重排序，确保了操作必须按照1-2-3（或1-3-2的正确顺序）执行，从而避免了这个问题。

### volatile为什么不保证原子性？

我们来看`count++`这个操作，它至少包含“读-改-写”三个步骤。`volatile`只能保证在“读”的时候，能读到最新的值；在“写”的时候，能把新值立即刷回主内存。但它无法保证“读-改-写”这整个过程不被其他线程打断。

当线程A读取了`volatile`的`count`值（比如10）之后，在它计算`10+1`并准备写回之前，线程B可能已经抢先一步，也读取了`count=10`，并完成了加1写回的操作，此时主内存的`count`已经是`11`了。然后线程A再将它计算出的`11`写回，最终结果是`11`而不是期望的`12`。

所以，对于复合操作，如果需要保证原子性，我们仍然需要使用`synchronized`或者`java.util.concurrent.atomic`包下的原子类（如`AtomicInteger`）。
