
#### 什么是 ThreadLocal
`ThreadLocal` 是 Java 提供的一个线程局部变量工具类，用于在多线程环境中为每个线程提供独立的变量副本。每个线程访问 `ThreadLocal` 时，操作的是自己线程的私有数据，线程间互不干扰。

#### 原理
`ThreadLocal` 通过每个线程内部的 `ThreadLocalMap`（绑定在 `Thread` 对象中）存储数据，键是 `ThreadLocal` 实例，值是线程私有变量。核心是利用线程隔离实现数据独立性。

#### 问题
- **内存泄漏**：线程存活时间长（如线程池）且未清理，可能导致内存占用。
- **不适合共享**：仅限线程内使用，无法跨线程传递数据。

---

### 1. ThreadLocal 详解
#### 作用
- 为每个线程提供独立副本，避免并发竞争。
- 常用于线程上下文传递（如用户信息、事务 ID）。

#### 示例
```java
public class ThreadLocalDemo {
    private static final ThreadLocal<String> threadLocal = new ThreadLocal<>();

    public static void main(String[] args) {
        threadLocal.set("Main Thread"); // 主线程设置
        System.out.println(threadLocal.get()); // "Main Thread"

        new Thread(() -> {
            threadLocal.set("Thread-1"); // 子线程设置
            System.out.println(threadLocal.get()); // "Thread-1"
        }).start();
    }
}
```
- **输出**：
```
Main Thread
Thread-1
```

---

### 2. 原理
#### 数据结构
- **ThreadLocalMap**：
  - 每个 `Thread` 对象含一个 `ThreadLocalMap`（`Thread.threadLocals`）。
  - 键：`ThreadLocal` 对象（弱引用）。
  - 值：线程私有数据。
- **存储位置**：
  - 数据绑定在 `Thread` 而非 `ThreadLocal`。

#### 操作流程
1. **set(T value)**：
   - 获取当前线程的 `ThreadLocalMap`。
   - 若为空，创建新 `ThreadLocalMap`。
   - 以 `ThreadLocal` 为键，`value` 为值存入。
2. **get()**：
   - 获取当前线程的 `ThreadLocalMap`。
   - 用 `ThreadLocal` 键取值，若无返回 `null` 或初始值。
3. **remove()**：
   - 删除当前线程 `ThreadLocalMap` 中对应键值对。

#### 源码片段（简化）
```java
public class ThreadLocal<T> {
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = t.threadLocals;
        if (map == null) {
            map = new ThreadLocalMap(this, value);
            t.threadLocals = map;
        } else {
            map.set(this, value);
        }
    }

    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = t.threadLocals;
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) return (T) e.value;
        }
        return null;
    }
}
```

#### 图示
```
Thread 1           Thread 2
  |                  |
[ThreadLocalMap]   [ThreadLocalMap]
  | key: TL1        | key: TL1
  | value: "A"      | value: "B"
```

---

### 3. 潜在问题
#### (1) 内存泄漏
- **原因**：
  - `ThreadLocalMap` 的键是弱引用（`WeakReference`），但值是强引用。
  - 若 `ThreadLocal` 对象被 GC 回收，而线程（如线程池中的线程）长期存活，未调用 `remove()`，值仍占用内存。
- **场景**：
  - 线程池中使用，未清理。
- **解决**：
  - 使用后调用 `remove()`。
```java
threadLocal.set("value");
try {
    // 使用
} finally {
    threadLocal.remove(); // 清理
}
```

#### (2) 不适合跨线程共享
- **原因**：
  - 数据绑定线程，无法传递。
- **解决**：
  - 用 `InheritableThreadLocal` 实现父子线程继承。
```java
static InheritableThreadLocal<String> inheritable = new InheritableThreadLocal<>();
```

#### (3) 脏数据
- **原因**：
  - 线程复用（如线程池），未清理上次数据。
- **解决**：
  - 每次使用前 `remove()` 或初始化。

---

### 4. 延伸与面试角度
- **应用场景**：
  - **Spring**：`RequestContextHolder` 存请求上下文。
  - **日志**：线程独立的 Trace ID。
- **与 synchronized 对比**：
  - `ThreadLocal`：隔离数据，无竞争。
  - `synchronized`：共享数据，加锁。
- **性能**：
  - 无锁操作，效率高，但内存管理需注意。
- **面试点**：
  - 问“原理”时，提 `ThreadLocalMap`。
  - 问“问题”时，提内存泄漏和清理。

---

### 总结
`ThreadLocal` 通过线程内的 `ThreadLocalMap` 提供数据隔离，原理是将数据绑定到 `Thread` 对象。优点是线程安全无竞争，缺点是可能内存泄漏，需 `remove()`。面试时，可写示例或提线程池场景，展示理解深度。