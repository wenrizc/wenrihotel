
ThreadLocal的核心思想，并不是它自己有一个Map来存储数据，而是它让每个线程自己来保管属于自己的那份数据副本。它的实现巧妙地利用了Java的类结构。

其底层实现可以概括为：每个Thread对象内部都有一个专门的成员变量 `threadLocals`，这个变量的类型是 `ThreadLocal.ThreadLocalMap`。这个Map才是真正存储数据的地方。

我们来拆解一下这个结构：

1. Thread类:
    
    每个线程实例（java.lang.Thread）都有一个 threadLocals 字段。这意味着，数据是与线程绑定的，而不是与ThreadLocal对象绑定。
    
2. ThreadLocalMap:
    
    这是一个定义在ThreadLocal类内部的、自定义的哈希表实现。它并不像HashMap那样对外开放，而是专门为ThreadLocal服务的。
    
3. 存储结构（Entry）:
    
    ThreadLocalMap中存储的键值对被封装在一个叫做Entry的内部类中。这个Entry的结构非常关键：
    
    - **Key**: `ThreadLocalMap`的键（Key）是`ThreadLocal`对象本身。但它不是一个普通的强引用，而是一个**弱引用（WeakReference）**。
        
    - **Value**: 值（Value）就是我们想要为这个线程存储的数据副本，它是一个普通的强引用。
        

#### 核心方法实现思路

- **set(T value) 方法**:
    
    1. 首先，获取当前正在执行的线程对象。
        
    2. 通过这个线程对象，拿到它内部的`threadLocals`这个Map。
        
    3. 如果这个Map不存在，就为当前线程创建一个新的`ThreadLocalMap`并赋值给它。
        
    4. 以当前的`ThreadLocal`对象实例作为Key，以传入的`value`作为Value，存入到当前线程的`ThreadLocalMap`中。
        
- **get() 方法**:
    
    1. 同样，先获取当前线程和它的`threadLocals` Map。
        
    2. 如果Map存在，就以当前的`ThreadLocal`对象实例为Key，去Map中查找对应的`Entry`。
        
    3. 如果找到了，就返回`Entry`中存储的Value。
        
    4. 如果Map不存在，或者Map中没有找到对应的`Entry`，就会执行一个`initialValue()`方法（该方法默认返回null，我们可以重写它来提供初始值），并将这个初始值存入Map后返回。
        

总结来说，`ThreadLocal`就像一个工具箱，它为每个线程提供了一个私有的存储空间（`ThreadLocalMap`），并提供了操作这个空间的钥匙（`ThreadLocal`实例本身）。

### 2. 使用时可能出现什么问题

`ThreadLocal`如果使用不当，最主要、最著名的问题就是**内存泄漏（Memory Leak）**。

#### 内存泄漏的原因

这个问题的根源，就在于`ThreadLocalMap`中`Entry`的弱引用设计。

1. 引用关系链:
    
    一个正常的引用链是这样的：线程对象（Thread）-> ThreadLocalMap -> Entry -> Key（弱引用指向ThreadLocal实例）和 Value（强引用指向数据对象）。
    
2. 弱引用被回收:
    
    当我们在代码中，不再持有对某个ThreadLocal实例的强引用时（比如 myThreadLocal = null;），在下一次垃圾回收（GC）发生时，由于ThreadLocalMap的Key只是一个弱引用，这个ThreadLocal实例就会被回收掉。
    
3. 泄漏发生:
    
    ThreadLocal实例被回收后，ThreadLocalMap中那个Entry的Key就变成了null。但是，它的Value部分依然存在一个强引用，指向我们存入的数据对象。只要这个线程本身不结束，它的ThreadLocalMap就会一直存在，这个Value也就无法被回收。这个Key为null但Value不为null的Entry，就成了一个永远无法被访问到，但又占用着内存的“僵尸”数据，从而导致了内存泄漏。
    

#### 如何避免内存泄漏

虽然`ThreadLocal`的`get()`, `set()`等方法在执行时，会顺便检查并清理一些Key为`null`的陈旧`Entry`（这是一种补偿机制），但这并不能保证内存泄漏一定不会发生，尤其是在线程池这种线程会被复用的场景下。

因此，最根本、最可靠的解决方案是：

**养成良好的编程习惯，在每次使用完`ThreadLocal`后，都在`finally`代码块中显式地调用其 `remove()` 方法。**

Java

```
ThreadLocal<MyData> myThreadLocal = new ThreadLocal<>();
try {
    myThreadLocal.set(new MyData());
    // ... 执行业务逻辑
} finally {
    my-thread-local.remove(); // 必须执行这一步
}
```

调用`remove()`方法会直接将当前线程的`ThreadLocalMap`中，与当前`ThreadLocal`实例对应的整个`Entry`（包括Key和Value）都移除掉，从而彻底断开引用链，避免内存泄漏的发生。

除了内存泄漏，另一个需要注意的点是，父子线程之间的数据传递。普通的`ThreadLocal`无法将父线程的值传递给子线程。如果需要这个功能，应该使用`InheritableThreadLocal`。