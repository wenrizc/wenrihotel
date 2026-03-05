## 1. 为什么需要 JMM：并发不是“按代码顺序执行”

Java 内存模型（JMM，Java Memory Model）是一套**语言层面的并发语义规范**：它规定了线程如何通过共享内存交互、哪些重排序被允许、以及在什么条件下一个线程的写入对另一个线程可见。

### 1.1 三大根因：缓存、乱序、编译器优化

在现代 CPU 上，线程对共享变量的访问通常会经过多层缓存、寄存器、写缓冲（store buffer）。同时，编译器与处理器会为了性能做重排序。

因此，即使你在源码里写的是“先写 a，再写 b”，另一个线程也**未必**能以同样顺序观察到这些写入。

### 1.2 并发三大问题：可见性、原子性、有序性

- **可见性**：线程 A 写入的值，线程 B 何时能看到？
- **原子性**：一个操作是否“不可分割”？例如 `count++` 实际是“读-改-写”三步。
- **有序性**：编译器或 CPU 是否能把两条指令交换顺序？哪些交换会破坏正确性？

## 2. JMM 的核心抽象：主内存与工作内存

JMM 用“主内存 / 工作内存”来描述线程与共享变量的交互，这是**规范层面的抽象**，不等价于 JVM 运行时的某个具体内存区域。

### 2.1 主内存（Main Memory）与工作内存（Working Memory）

- **主内存**：共享变量的“规范视角下的存放处”，所有线程共享。
- **工作内存**：每个线程私有的变量副本，线程对共享变量的读写，必须先在工作内存中进行，再与主内存同步。

注意：这里的“工作内存”更接近“CPU 缓存 / 寄存器 / 写缓冲”等硬件现象的抽象，而不是“JVM 栈”。

### 2.2 八种交互操作

JMM 以一组抽象操作描述主内存与工作内存的交互（常见说法是 8 种）：

- `read` / `load`：从主内存读取，并加载到工作内存副本。
- `use`：线程在执行引擎中使用工作内存副本。
- `assign`：线程把新值赋给工作内存副本。
- `store` / `write`：把工作内存新值刷新到主内存。
- `lock` / `unlock`：与监视器锁（`synchronized`）相关的互斥与同步语义。

这些操作不是 JVM 字节码指令，也不是你能直接调用的 API，而是为了**定义可见性与有序性规则**所用的形式化工具。

## 3. Happens-Before：JMM 的可见性与有序性契约

Happens-Before（简称 HB）是 JMM 的核心：它定义了并发程序中操作之间的偏序关系，用于推导“可见性 + 必要的有序性”。

### 3.1 定义与常见误区

如果操作 A happens-before 操作 B，那么：

- A 的结果对 B **可见**；
- A 在“规范语义上”先于 B（不是指真实时间的先后）。

误区：HB 不是“谁先执行谁后执行”的物理时序，而是“在并发语义上必须能这样观察”的约束。

### 3.2 HB 的组成：程序次序 + 同步次序 + 传递性

在 JLS 的表述中，你可以把 HB 理解为：

- 同一线程内的**程序次序**（Program Order）。
- 由同步原语建立的 **synchronizes-with**（如锁、`volatile`、线程启动/终止等）。
- **传递性**：A HB B 且 B HB C，则 A HB C。

### 3.3 常用 HB 规则清单

| 规则 | 形式化表述（简化） | 常见落地方式 |
| --- | --- | --- |
| 程序次序 | 单线程内，前面 HB 后面 | 普通代码顺序（不跨线程） |
| 监视器锁 | 对同一把锁，`unlock` HB 后续 `lock` | `synchronized` 退出 / 进入 |
| `volatile` | 对同一变量，`volatile` 写 HB 后续读 | 状态发布、停止标志 |
| 线程启动 | `t.start()` HB `t` 内的任意动作 | 主线程启动子线程 |
| 线程终止 | `t` 内动作 HB `t.join()` 返回 | `join()` 获取结果 |
| 线程中断 | `interrupt()` HB 中断被检测到 | `isInterrupted()` / 抛 `InterruptedException` |
| `final` 语义 | 构造函数对 `final` 字段写入，对外可见（需正确发布） | 不可变对象安全初始化 |
| 传递性 | A HB B，B HB C，则 A HB C | 多段发布链路 |

## 4. `volatile`：可见性 + 有序性

`volatile` 解决两类问题：

- **可见性**：对 `volatile` 变量的写入会被其他线程及时观察到。
- **有序性**：它会对读写两侧施加重排序约束（通过内存屏障与编译器约束实现）。

### 4.1 语义要点

可以用“释放（release）/ 获取（acquire）”来记忆：

- `volatile` 写：把该线程在写之前的普通写入“释放”出去。
- `volatile` 读：获取到该变量的最新值，并禁止后续操作被重排到它之前。

这也是为什么“先写普通变量，再写 `volatile` 标志”可以用来做发布。

### 4.2 典型用法 1：停止线程（可见性）

```java
class Worker implements Runnable {
    private volatile boolean running = true;

    @Override
    public void run() {
        while (running) {
            // do work
        }
    }

    public void shutdown() {
        running = false; // volatile 写：其他线程能及时看到
    }
}
```

### 4.3 典型用法 2：发布结果（有序性 + 可见性）

```java
class Holder {
    private int x;
    private int y;
    private volatile boolean ready;

    public void publish() {
        x = 1;
        y = 2;
        ready = true; // volatile 写：禁止把 x/y 的写重排到 ready 之后
    }

    public int consume() {
        if (ready) { // volatile 读：禁止后续读被重排到 ready 之前
            return x + y; // 一旦看到 ready==true，必须能看到 x==1 且 y==2
        }
        return 0;
    }
}
```

### 4.4 `volatile` 不保证原子性：不要用它做计数器

`count++` 不是原子操作，即使 `count` 是 `volatile` 也不行：

```java
class Counter {
    private volatile int count = 0;

    public void inc() {
        count++; // 读-改-写，仍可能丢失更新
    }
}
```

需要原子性时，使用 `AtomicInteger.incrementAndGet()` 或在更高并发下用 `LongAdder`。

## 5. `synchronized` 与 `Lock`：互斥 + 同步

互斥类同步原语同时解决：

- 临界区的**原子性**（同一时刻只有一个线程进入）。
- 进入 / 退出临界区时的**可见性**（建立 HB）。

### 5.1 `synchronized` 的内存语义

对同一把锁：

- 退出 `synchronized`（`monitorexit`）等价于一次释放：把临界区内的写入对外可见。
- 进入 `synchronized`（`monitorenter`）等价于一次获取：读取到其他线程在释放前写入的结果。

这正对应 HB 规则中的“`unlock` HB 后续 `lock`”。

## 6. `final`：初始化安全性与不可变对象

`final` 不只是“不可重新赋值”，它还有重要的并发语义：在满足条件时，其他线程能**安全地看到**构造期间写入的 `final` 字段值。

### 6.1 `final` 字段的发布语义

当对象在构造完成后被正确发布（safe publication），其他线程读取到该对象引用后：

- 必须能看到构造函数对 `final` 字段写入的值；
- 不会把对 `final` 字段的读取重排到构造完成之前。

### 6.2 构造函数中让 `this` 逸出

如果在构造期间把 `this` 发布到其他线程可见的位置（例如注册回调、放入全局集合），会破坏初始化安全性：

```java
class ThisEscape {
    private final int a;

    ThisEscape(EventSource source) {
        source.registerListener(e -> {
            System.out.println(a); // 可能看到默认值 0（发布过早）
        });
        a = 42;
    }
}
```

修复思路：避免构造期间发布 `this`，或使用工厂方法在构造完成后再注册。

## 7. 安全发布（Safe Publication）：把“写入”变成“可见”

很多并发 Bug 的本质是：对象或数据被发布到其他线程时，没有建立足够的 HB 关系。

### 7.1 什么叫安全发布

一个对象被安全发布，意味着：其他线程获取到该对象引用时，**必须**能看到发布线程在发布前对该对象状态做的初始化写入。

### 7.2 常见安全发布方式

- 通过 `static` 初始化（类初始化的同步保证）。
- 把引用写入 `volatile` 变量（`volatile` 写 HB 后续读）。
- 通过锁保护发布与获取（`unlock` HB 后续 `lock`）。
- 放入线程安全容器（容器内部通过同步保证 HB）。
- 使用不可变对象（`final` 字段 + 无 `this` 逸出 + 正确发布）。

### 7.3 双重检查锁（DCL）为什么必须配合 `volatile`

没有 `volatile` 时，DCL 可能读到“半初始化对象”，根因是对象创建可能被重排序：

1. 分配内存
2. 调用构造初始化
3. 把引用赋给共享变量

2 和 3 可能被重排，导致另一个线程看到非 null 引用但对象未初始化完成。

```java
class Singleton {
    private static volatile Singleton instance;

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

## 8. CAS 与原子类：原子性来自哪里

JUC 的原子类（如 `AtomicInteger`）基于 CAS（Compare-And-Swap）实现：

- **原子性**：CAS 在硬件层面以“读-比较-写”的原子指令序列实现。
- **可见性 / 有序性**：原子变量的读写会带有 JMM 规定的内存语义（可类比 `volatile` 的获取/释放效果）。

注意：CAS 只解决单个变量（或特定结构）的原子更新，跨多个变量的一致性仍需要更高层的同步手段。

