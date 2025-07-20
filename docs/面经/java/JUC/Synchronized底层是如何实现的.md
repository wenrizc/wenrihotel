
### 1. Synchronized在字节码层面的实现

`synchronized` 在Java中有两种使用形式，它们的字节码实现是不同的。

形式一：同步方法

当我们将 `synchronized` 用于修饰一个方法时，如下所示：

``` Java
public synchronized void testMethod() {
    // 方法体
}
```

编译成字节码后，我们查看方法的访问标志（access flags），会发现多了一个 `ACC_SYNCHRONIZED` 标志。

当JVM在执行方法时，如果检测到方法上有这个 `ACC_SYNCHRONIZED` 标志，它会首先尝试获取该方法所属对象的监视器锁（Monitor）。

- 如果是实例方法，它会获取 `this` 对象的锁。
    
- 如果是静态方法，它会获取该方法所属类的 Class 对象的锁。
    
    只有成功获取锁之后，线程才能执行方法体。方法执行完毕（无论是正常返回还是抛出异常），JVM会自动释放这个锁。这个过程是由JVM隐式完成的。
    

形式二：同步代码块

当我们将 `synchronized` 用于一个代码块时：

``` Java
public void testMethod() {
    synchronized (lockObject) {
        // 同步代码块
    }
}
```

编译成字节码后，它不再是通过方法标志位来实现，而是由两条特定的字节码指令来完成：`monitorenter` 和 `monitorexit`。

编译器会在同步代码块的开始位置插入一个 `monitorenter` 指令，在同步代码块的结束位置插入一个 `monitorexit` 指令。

- `monitorenter`：当线程执行到这条指令时，会尝试获取 `lockObject` 所引用的对象的监视器锁。
    
- `monitorexit`：当线程执行完同步代码块，执行到这条指令时，会释放持有的监视器锁。
    

一个重要的细节是，为了保证即使在同步代码块中发生异常，锁也一定能被释放，编译器实际上会生成两个 `monitorexit` 指令。一个在正常结束路径的末尾，另一个在异常处理路径的末尾（类似于`finally`块的逻辑）。

### 2.锁的本质：Java对象头与Monitor

无论是 `ACC_SYNCHRONIZED` 标志还是 `monitorenter/exit` 指令，它们操作的“锁”到底是什么呢？这个锁的实体，与Java的对象头（Object Header）和Monitor机制密切相关。

- **Java对象头**：在HotSpot虚拟机中，每个Java对象都有一个对象头。对象头里包含一部分叫做“Mark Word”的区域。这个Mark Word非常关键，它在不同的锁状态下，存储的内容是不同的，比如对象的哈希码、GC分代年龄，以及最重要的，锁状态信息。
    
- **Monitor（监视器锁）**：每个Java对象都可以唯一地关联一个Monitor。当一个线程想要获取这个对象的锁时，实际上就是想要获取这个对象关联的Monitor的所有权。Monitor可以理解为一个同步工具，它内部包含了等待队列、持有线程等信息。在早期的JDK版本中，Monitor是直接依赖于操作系统底层的互斥量（Mutex）实现的，这种锁被称为“重量级锁”，因为它涉及到用户态和内核态的切换，开销很大。
    

### 3. 锁优化：锁的升级过程

为了解决重量级锁的性能问题，从JDK 1.6开始，JVM对`synchronized`引入了一系列优化，核心思想就是“锁升级”。即锁不会一开始就是重量级的，而是会根据竞争情况，从低到高进行升级，以适应不同的并发场景。
