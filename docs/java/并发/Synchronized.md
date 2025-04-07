
## Synchronized 的使用

使用 `synchronized` 关键字时，需要注意以下几点：

- **互斥性:** 一把锁同一时刻只能被一个线程持有，其他线程必须等待。
- **对象锁:** 每个实例拥有独立的锁 (`this`)，实例锁互不影响。
    - **例外:** 当锁对象是 `*.class` 或 `synchronized` 修饰 `static` 方法时，所有对象共享同一把**类锁**。
- **锁释放:** `synchronized` 修饰的方法或代码块，无论正常执行结束还是抛出异常，都会自动释放锁。

### 对象锁

对象锁包括两种形式：

1. **方法锁 (Method Lock):**
    
    - 使用 `synchronized` 修饰**普通方法**。
    - 默认锁对象为 `this` (当前实例对象)。
    
    **示例:**

    ```
    public class SynchronizedObjectLock implements Runnable {
        static SynchronizedObjectLock instance = new SynchronizedObjectLock();
    
        @Override
        public void run() {
            method(); // 调用同步方法
        }
    
        public synchronized void method() { // 方法锁，锁对象为 this
            System.out.println("我是线程" + Thread.currentThread().getName());
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread().getName() + "结束");
        }
    
        public static void main(String[] args) {
            Thread t1 = new Thread(instance);
            Thread t2 = new Thread(instance);
            t1.start();
            t2.start();
        }
    }
    ```
    
2. **同步代码块锁 (Synchronized Block Lock):**
    
    - 使用 `synchronized (lockObject)` 显式指定锁对象。
    - 锁对象可以是 `this`，也可以是自定义的对象。
    
    **示例 1 (锁为 `this`):**

    ```
    public class SynchronizedObjectLock implements Runnable {
        static SynchronizedObjectLock instance = new SynchronizedObjectLock();
    
        @Override
        public void run() {
            synchronized (this) { // 同步代码块，锁对象为 this
                System.out.println("我是线程" + Thread.currentThread().getName());
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName() + "结束");
            }
        }
    
        public static void main(String[] args) {
            Thread t1 = new Thread(instance);
            Thread t2 = new Thread(instance);
            t1.start();
            t2.start();
        }
    }
    ```
    
    **示例 2 (自定义锁对象):**

    ```
    public class SynchronizedObjectLock implements Runnable {
        static SynchronizedObjectLock instance = new SynchronizedObjectLock();
        Object block1 = new Object(); // 自定义锁对象 1
        Object block2 = new Object(); // 自定义锁对象 2
    
        @Override
        public void run() {
            synchronized (block1) { // 同步代码块，锁对象为 block1
                System.out.println("block1锁,我是线程" + Thread.currentThread().getName());
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("block1锁,"+Thread.currentThread().getName() + "结束");
            }
    
            synchronized (block2) { // 同步代码块，锁对象为 block2
                System.out.println("block2锁,我是线程" + Thread.currentThread().getName());
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("block2锁,"+Thread.currentThread().getName() + "结束");
            }
        }
    
        public static void main(String[] args) {
            Thread t1 = new Thread(instance);
            Thread t2 = new Thread(instance);
            t1.start();
            t2.start();
        }
    }
    ```
    

### 类锁

类锁用于控制对**类级别**的共享资源的并发访问，包括两种形式：

1. **`synchronized` 修饰静态方法:**
    
    - 锁对象为**当前类本身 (`Class` 对象)**。
    - 所有实例对象共享同一把类锁。
    
    **示例:**

    ```
    public class SynchronizedObjectLock implements Runnable {
        static SynchronizedObjectLock instance1 = new SynchronizedObjectLock();
        static SynchronizedObjectLock instance2 = new SynchronizedObjectLock();
    
        @Override
        public void run() {
            method(); // 调用静态同步方法
        }
    
        public static synchronized void method() { // 类锁，锁对象为 SynchronizedObjectLock.class
            System.out.println("我是线程" + Thread.currentThread().getName());
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread().getName() + "结束");
        }
    
        public static void main(String[] args) {
            Thread t1 = new Thread(instance1);
            Thread t2 = new Thread(instance2);
            t1.start();
            t2.start();
        }
    }
    ```
    
2. **`synchronized` 指定锁对象为 `Class` 对象:**
    
    - 使用 `synchronized (ClassName.class)` 显式指定锁对象为类对象。
    - 与 `synchronized` 修饰静态方法效果相同，都是类锁。
    
    **示例:**

	```
    public class SynchronizedObjectLock implements Runnable {
        static SynchronizedObjectLock instance1 = new SynchronizedObjectLock();
        static SynchronizedObjectLock instance2 = new SynchronizedObjectLock();
    
        @Override
        public void run() {
            synchronized(SynchronizedObjectLock.class){ // 类锁，锁对象为 SynchronizedObjectLock.class
                System.out.println("我是线程" + Thread.currentThread().getName());
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName() + "结束");
            }
        }
    
        public static void main(String[] args) {
            Thread t1 = new Thread(instance1);
            Thread t2 = new Thread(instance2);
            t1.start();
            t2.start();
        }
    }
    ```
    

### Synchronized 原理分析

#### 加锁和释放锁的原理

`synchronized` 的加锁和释放锁机制依赖于 JVM 底层的 **monitor (监视器锁)** 实现。

- **字节码指令:** `monitorenter` (加锁) 和 `monitorexit` (释放锁)。
- **Monitor 计数器:** 每个对象关联一个 monitor，monitor 内部维护计数器。
    - **`monitorenter`:**
        1. 检查 monitor 计数器是否为 0，为 0 表示锁未被获取，当前线程尝试获取锁，计数器 **+1**。
        2. 若计数器不为 0，检查当前线程是否已持有锁，若是，则**重入**锁，计数器 **+1**。
        3. 若锁已被其他线程持有，则当前线程进入**同步队列**等待锁释放。
    - **`monitorexit`:**
        1. 释放锁，monitor 计数器 **-1**。
        2. 若计数器减为 0，表示锁完全释放，唤醒同步队列中的等待线程竞争锁。
        3. 若计数器仍大于 0，表示是重入锁的释放，线程继续持有锁。


线程访问对象前必须先获取对象监视器。获取失败的线程进入同步队列，状态变为 `BLOCKED`。持有监视器的线程释放锁后，同步队列中的线程有机会重新竞争监视器。

#### 可重入原理：加锁次数计数器

**可重入性 (Reentrancy):** 同一线程在外层方法获取锁后，内层方法可以自动获取同一把锁，不会死锁。

**示例代码:**


```
public class SynchronizedDemo {
    public static void main(String[] args) {
        SynchronizedDemo demo =  new SynchronizedDemo();
        demo.method1();
    }

    private synchronized void method1() {
        System.out.println(Thread.currentThread().getId() + ": method1()");
        method2(); // 调用另一个 synchronized 方法
    }

    private synchronized void method2() {
        System.out.println(Thread.currentThread().getId()+ ": method2()");
        method3(); // 调用另一个 synchronized 方法
    }

    private synchronized void method3() {
        System.out.println(Thread.currentThread().getId()+ ": method3()");
    }
}
```

**执行流程 (Monitor 计数器变化):**

1. `monitorenter` (获取锁): 计数器 = 0 -> 1 (获取锁)
2. `method1()` 执行，计数器 +1 -> 1 (持有锁)
3. `method2()` 执行，计数器 +1 -> 2 (重入锁)
4. `method3()` 执行，计数器 +1 -> 3 (重入锁)
5. `monitorexit` (`method3()` 结束), 计数器 -1 -> 2
6. `monitorexit` (`method2()` 结束), 计数器 -1 -> 1
7. `monitorexit` (`method1()` 结束), 计数器 -1 -> 0 (释放锁)

**原理:** 每个对象拥有一个 monitor 计数器。线程获取锁时计数器 +1，释放锁时计数器 -1。同一线程重复获取锁时，只需累加计数器，无需再次竞争锁。

#### 保证可见性的原理：内存模型和 Happens-Before 规则

`synchronized` 的 Happens-Before 规则 (监视器锁规则): **对同一个监视器的解锁 (`monitorexit`)，Happens-Before 于后续对该监视器的加锁 (`monitorenter`)**。

**示例代码:**

```
public class MonitorDemo {
    private int a = 0;

    public synchronized void writer() {     // 1
        a++;                                // 2
    }                                       // 3

    public synchronized void reader() {    // 4
        int i = a;                         // 5
    }                                      // 6
}
```

**Happens-Before 关系图:** 

- **规则推导:**![[java-thread-x-key-schronized-3.webp]]
    
    - 程序顺序规则 (黑色箭头)
    - 监视器锁规则 (红色箭头): 线程 A 释放锁 Happens-Before 线程 B 加锁
    - 传递性规则 (蓝色箭头): 推导出 2 Happens-Before 5
- **可见性保证:** 根据 Happens-Before 定义，2 Happens-Before 5 意味着线程 A 在 `writer()` 方法中对共享变量 `a` 的修改 (a++)，对线程 B 在 `reader()` 方法中读取 `a` 的操作 (int i = a) 是**可见的**。线程 B 读取到的 `a` 值一定是最新的值 (至少是线程 A 修改后的值)。
    

### JVM 中锁的优化

JDK 1.6 对 `synchronized` 进行了大量优化，以减少锁操作开销，提高性能：

- **锁粗化 (Lock Coarsening):** 合并相邻的、对同一对象多次加锁/解锁操作，扩大锁的范围，减少锁的获取/释放次数。
- **锁消除 (Lock Elimination):** 通过逃逸分析，判断同步代码块中的数据是否只被单线程访问，若没有线程安全问题，则消除不必要的锁操作。
- **轻量级锁 (Lightweight Locking):** 在**无锁竞争**情况下，使用 CAS 操作和自旋代替重量级锁的阻塞，减少线程阻塞/唤醒开销。
- **偏向锁 (Biased Locking):** 在**单线程访问**同步代码块时，消除不必要的 CAS 操作，进一步提高性能。
- **适应性自旋 (Adaptive Spinning):** 动态调整自旋时间，根据锁竞争情况和自旋成功率，更智能地选择自旋策略。

### 锁的类型 (Synchronized 锁状态)

`synchronized` 锁有四种状态，随着竞争情况逐渐升级，但**不可降级**：

**锁膨胀方向:** 无锁 -> 偏向锁 -> 轻量级锁 -> 重量级锁

- **无锁 (No Lock):** 初始状态，无任何线程持有锁。
    
- **偏向锁 (Biased Lock):**
    
    - **适用场景:** **只有一个线程** 访问同步块的场景。
    - **优化:** 消除单线程重复获取锁的 CAS 操作。
    - **原理:** 首次获取锁时，将 **线程 ID** 记录在 **对象头 Mark Word** 中，后续该线程再次进入同步块时，只需检查 Mark Word 即可快速获得锁。
    - **撤销:** 当有**其他线程竞争**时，偏向锁撤销，升级为轻量级锁。
- **轻量级锁 (Lightweight Lock):**
    
    - **适用场景:** **多线程交替** 访问同步块 (竞争不激烈) 的场景。
    - **优化:** 使用 CAS + 自旋 尝试获取锁，避免线程阻塞。
    - **原理:** 线程在栈帧中创建 **锁记录 (Lock Record)**，通过 CAS 替换 **对象头 Mark Word** 为指向锁记录的指针。
    - **膨胀:** 当自旋超过一定次数或线程竞争激烈时，轻量级锁膨胀为重量级锁。
- **重量级锁 (Heavyweight Lock):**
    
    - **适用场景:** **多线程并发高**，锁竞争激烈的场景。
    - **原理:** 线程获取锁失败后，进入**等待队列**阻塞，等待操作系统唤醒。
    - **缺点:** 线程阻塞/唤醒开销大，性能较低。

### 自旋锁与自适应自旋锁

#### 自旋锁

- **引入背景:** 重量级锁线程阻塞/唤醒开销大，但锁的持有时间可能很短。
- **原理:** 线程在获取锁失败后，**不立即阻塞**，而是进行**忙循环 (自旋)**，等待锁释放。
- **优点:** 锁持有时间短时，自旋锁性能优于重量级锁，避免线程切换开销。
- **缺点:** 长时间自旋会**消耗 CPU 资源**。
- **自旋次数限制:** 默认 10 次，可配置 (`-XX:PreBlockSpin`)。

#### 自适应自旋锁

- **改进:** **动态调整自旋时间**，根据**前一次自旋**在**同一把锁**上的**成功率**和**锁持有者**状态决定自旋时长。
- **优势:** 更智能的自旋策略，减少无效自旋，提高性能。

### 锁消除和锁粗化

#### 锁消除 (Lock Elimination)

- **原理:** JVM 运行时编译器 (JIT) 通过 **逃逸分析**，判断同步代码块中的数据是否 **线程私有**，不会被其他线程访问。
- **优化:** 若数据线程私有，则 **消除** 不必要的锁操作，提高性能。
- **适用场景:** 例如 `StringBuffer`、`Vector` 等线程安全类在方法内部使用，但对象不会逃逸出线程安全范围的情况。

#### 锁粗化 (Lock Coarsening)

- **原理:** 将**连续的**、对**同一对象**的**多次加锁/解锁**操作，合并为一个**范围更大的锁**，减少锁的获取/释放次数。
- **适用场景:** 例如循环体内反复加锁/解锁，或连续的 `append()` 操作等。

### 轻量级锁和偏向锁

#### 轻量级锁

- **加锁过程:**
    
    1. 线程在栈帧中创建 **锁记录 (Lock Record)**。
    2. CAS 尝试将 **对象头 Mark Word** 复制到锁记录，并将 Mark Word 指向锁记录。
    3. CAS 成功，获得轻量级锁，Mark Word 锁标志位设为 `00` (轻量级锁定)。
    4. CAS 失败，检查 Mark Word 是否指向当前线程栈帧，若是，表示已获得锁 (重入)；否则，锁被其他线程抢占，轻量级锁膨胀为重量级锁。
- **解锁过程:**
    
    1. CAS 尝试将锁记录中的 **Displaced Mark Word** 替换回 **对象头 Mark Word**。
    2. CAS 成功，解锁成功，无竞争。
    3. CAS 失败，表示存在竞争，锁膨胀为重量级锁，需要唤醒等待线程。

#### 偏向锁

- **引入背景:** 轻量级锁在无竞争情况下仍需 CAS 操作，存在开销。
- **优化目标:** 进一步消除 **无竞争** 场景下的锁开销。
- **原理:** 线程首次获取锁时，将 **线程 ID** 偏向到对象头 Mark Word。后续该线程再次进入同步块，只需检查 Mark Word 是否偏向当前线程，若是，则直接获得锁，无需 CAS 操作。
- **撤销:** 当有**其他线程竞争**偏向锁时，偏向锁撤销，升级为轻量级锁或无锁状态。撤销过程需要在**全局安全点**进行。

### 锁的优缺点对比

|锁类型|优点|缺点|适用场景|
|---|---|---|---|
|偏向锁|加锁/解锁无需 CAS 操作，性能开销极小|线程竞争时，锁撤销开销大|只有一个线程访问同步块|
|轻量级锁|竞争线程不阻塞，响应速度快|竞争激烈时，自旋消耗 CPU 性能|追求响应时间，同步块执行快|
|重量级锁|线程竞争不自旋，不消耗 CPU|线程阻塞，响应时间慢，频繁获取/释放锁开销大|追求吞吐量，同步块执行时间长，多线程竞争激烈|


### Synchronized 与 Lock (ReentrantLock) 的比较

#### Synchronized 的缺陷

1. **效率较低:**
    
    - 锁释放情况少：代码执行完毕或异常结束才释放。
    - 获取锁时无法设置超时，无法中断等待线程。
2. **不够灵活:**
    
    - 加锁/释放时机单一。
    - 每个锁只有一个单一的条件 (是否获取锁)。
3. **无法获取锁状态:** 无法判断是否成功获得锁。
    

#### Lock (ReentrantLock) 的优势

1. **可中断等待:** `lockInterruptibly()` 方法允许线程在等待锁的过程中响应中断，避免死锁。
2. **可设置超时时间:** `tryLock(long, TimeUnit)` 方法允许线程在等待锁超时后放弃等待。
3. **可判断锁状态:** `tryLock()` 方法尝试非阻塞地获取锁，并返回 boolean 值指示是否成功获取锁。
4. **Condition 接口:** `ReentrantLock` 可以绑定多个 `Condition` 对象，实现更灵活的线程协作和条件等待/唤醒机制。

#### 使用选择

- **优先使用 `synchronized`:** JVM 原生支持，简单易用，JVM 优化后性能与 `ReentrantLock` 相近。
- **在需要 `ReentrantLock` 高级功能时选择 `Lock`:** 如可中断等待、公平锁、多条件等。
- **优先考虑 `java.util.concurrent` 包下的工具类:** 更高级、更灵活的并发工具。

#### 深入理解 Synchronized

- **注意事项:**
    
    - 锁对象不能为空。
    - 作用域不宜过大。
    - 避免死锁。
- **公平性:** `synchronized` 实际上是 **非公平锁**，新来的线程可能优先获得锁，提高性能，但也可能导致饥饿现象。


1. Synchronized可以作用在哪里？分别通过对象锁和类锁进行举例。  
    • 对象锁：用于实例方法或同步实例代码块，每个对象都有一把独立的锁。例如，在一个对象的实例方法上加synchronized，则同一个对象的多个线程不能同时执行该方法，但不同对象之间互不影响。  
    • 类锁：用于static方法或同步static代码块，是针对类的Class对象的锁。例如，在一个static方法上加synchronized，则所有线程在访问该方法时，会竞争同一个锁（即这个类的Class对象）。
    
    示例代码：
    
    Java
    
    ```
    public class Example {
        // 对象锁示例：synchronized实例方法
        public synchronized void instanceMethod() {
            // 临界区代码
        }
    
        // 类锁示例：synchronized静态方法
        public static synchronized void staticMethod() {
            // 临界区代码
        }
    
        // 对象锁示例：synchronized代码块（锁定this）
        public void blockMethod() {
            synchronized (this) {
                // 临界区代码
            }
        }
    
        // 类锁示例：synchronized代码块（锁定类对象）
        public void staticBlockMethod() {
            synchronized (Example.class) {
                // 临界区代码
            }
        }
    }
    ```
    

---

1. Synchronized本质上是通过什么保证线程安全的？分三个方面回答：  
    • 加锁和释放锁的原理：  
    当线程进入synchronized块（或方法）时，会尝试获得对应的监视器锁（Monitor），底层通过字节码指令“monitorenter”进行加锁，执行完毕后通过“monitorexit”释放锁。JVM确保在退出同步块时，无论是否正常退出，都能正确释放锁。
    
    • 可重入原理：  
    Synchronized是可重入锁，即同一个线程可以多次获取同一个锁而不会死锁。JVM内部为每个锁维护了一个计数器，如果同一线程再次请求已持有的锁，只需增加计数器，不必再次阻塞，直到所有匹配的退出（monitorexit）操作完成后才真正释放锁。
    
    • 保证可见性原理：  
    Synchronized保证了内存的可见性。当线程进入同步块时，会清空工作内存中的共享变量副本，从主内存重新加载最新值；当退出同步块时，会把工作内存中对共享变量的修改刷新到主内存。这样可以保证修改对其他线程的及时可见。
    

---

1. Synchronized有什么缺陷？Java Lock是怎么弥补这些缺陷的。  
    • Synchronized的缺陷：
    
    - 不能尝试非阻塞方式获取锁（即没有tryLock等机制）。
    - 不能响应中断，即在等待获取锁期间线程无法手动中断。
    - 缺少公平性控制，竞争激烈时无法保证先等待的线程一定先获得锁。
    - 灵活性较差，不能精确控制加锁和解锁的时机，这是由JVM自动管理的。
    
    • Java Lock（如ReentrantLock）的改进：
    - 提供tryLock方法和带超时参数的锁获取方式，允许非阻塞式尝试获取锁。
    - 提供lockInterruptibly方法，可以响应中断。
    - 提供公平锁的实现，允许按照FIFO顺序分配锁。
    - 用户可以灵活地控制加锁和解锁的时机，实现更精细的同步控制。

---

1. Synchronized和Lock的对比，以及如何选择？  
    • Synchronized：
    
    - 语法简单，使用方便，异常情况下自动释放锁。
    - 由JVM优化，在无竞争的场景下性能非常好。
    - 缺乏灵活性，不能定时等待，不能响应中断，且没有公平性等控制。
    
    • Lock：
    
    - 提供更多高级特性，如尝试锁、可中断锁、定时获取锁以及公平性保证。
    - 开发者需要手动释放锁，容易出现死锁或忘记释放锁的问题。
    
    • 选择建议：
    - 在简单场景或竞争较低场景下，synchronized语法简洁且安全；
    - 在高竞争、需要精细控制或需要公平性、中断响应等高级功能时，可以选择Lock。

---

1. Synchronized在使用时有何注意事项？  
    • 避免将过大范围的代码块放入synchronized区，尽量减少临界区代码的粒度。  
    • 注意死锁问题，尤其是在嵌套使用多个锁时，要避免顺序不一致。  
    • 避免在同步代码块中执行可能阻塞很长时间或调用外部不可控代码（如IO操作）。  
    • 对于共享资源的操作，确保所有访问都通过相同的锁进行保护，防止出现竞态问题。

---

1. Synchronized修饰的方法在抛出异常时，会释放锁吗？  
    是的。无论方法正常返回还是抛出异常，都会在退出synchronized代码块时自动释放锁（由JVM保证执行monitorexit）。

---

1. 多个线程等待同一个Synchronized锁的时候，JVM如何选择下一个获取锁的线程？  
    JVM并不保证公平性，具体的调度策略依赖于JVM实现和操作系统调度，一般采用的是无序的竞争机制。也就是说，在没有其他明确调度策略时，下一个获得锁的线程是随机的（或由底层系统决定），不一定是最先等待的线程。

---

1. Synchronized使得同时只有一个线程可以执行，性能比较差，有什么提升的方法？  
    • 减小同步代码块的粒度，尽量缩小锁住的范围；  
    • 可以使用JVM的优化机制，如偏向锁、轻量级锁等，在低竞争情况下减少锁操作的开销；  
    • 使用局部变量和无竞争的数据结构，减少共享变量的使用；  
    • 在高并发场景下，可以考虑使用Lock或其他并发控制工具（如原子变量、CAS算法等）来提升性能。

---

1. 我想更加灵活的控制锁的释放和获取（现在释放锁和获取锁的时机都被规定死了），怎么办？  
    你可以选择使用java.util.concurrent.locks.Lock接口及其实现（例如ReentrantLock），它允许你在代码中明确调用lock()和unlock()，从而更加灵活地控制锁的获取和释放时机，以及支持中断响应、定时锁和公平策略等功能。

---

1. 什么是锁的升级和降级？什么是JVM里的偏斜锁、轻量级锁、重量级锁？  
    • 锁的升级：  
    当线程请求一个锁时，JVM根据当前的竞争情况不断调整锁的状态。一般情况下，当线程获取锁并且没有竞争时，JVM会使用偏向锁，如果出现竞争，则会升级为轻量级锁；如果竞争激烈，则可能进一步膨胀为重量级锁。  
    • 锁的降级：  
    锁的降级通常是指在竞争结束后，JVM能将锁从重量级状态降低为轻量级或偏向状态，以提高后续无竞争情况下的性能。  
    • JVM锁的分类：
    - 偏向锁（Biased Lock）：在无竞争的情况下，锁会偏向于首次获得它的线程，这样后续该线程再次获得锁时无需额外同步操作。
    - 轻量级锁（Lightweight Lock）：当偏向锁检测到多个线程竞争时，会升级为轻量级锁，通过自旋（spin）来等待锁释放，避免进入阻塞状态。
    - 重量级锁（重量级锁/Heavyweight Lock）：在激烈竞争或超出自旋时间时，锁会膨胀为重量级锁，采用操作系统的互斥量（mutex）机制，线程会被挂起和唤醒。

---

1. 不同的JDK中对Synchronized有何优化？  
    • 在JDK 1.5以后，JVM对synchronized做了大量优化：
    - 引入了偏向锁和轻量级锁技术，降低了无争用时的性能开销。
    - 采用逃逸分析（Escape Analysis）和同步消除（Lock Elision），当发现同步块不会产生实际竞争时，可能将同步代码块消除，进一步提高性能。
    - 在后续版本中对锁膨胀的策略、锁升级的机制以及自旋次数等都进行了优化，旨在平衡性能和并发安全性。