
#### 原理
- **`ConcurrentHashMap`** 是 Java 提供的一种线程安全的高并发哈希表，基于 **分段锁**（JDK 1.7）或 **CAS + 同步**（JDK 1.8）实现。
- **核心思想**：
  - 将数据分段存储，减少锁粒度，提升并发性能。
- **JDK 1.8 实现**：
  - 底层：数组 + 链表 + 红黑树。
  - 并发控制：CAS（无锁操作）+ `synchronized`（锁桶头）。

#### 关键点
- 高并发：多线程安全读写。
- 无锁读：读取无需加锁。
- 优化锁：写操作锁粒度小。

---

### 1. ConcurrentHashMap 原理详解
#### (1) 数据结构
- **JDK 1.7**：
  - **Segment 数组 + HashEntry 数组**：
    - `Segment` 是分段锁单元，每个包含一个 `HashEntry[]`。
    - 每个 `HashEntry` 是链表。
- **JDK 1.8**：
  - **Node 数组 + 链表/红黑树**：
    - 直接用 `Node[]` 存储，类似 `HashMap`。
    - 链表长度 > 8 且数组容量 ≥ 64 时转红黑树。
  - **变化**：
    - 移除 `Segment`，锁粒度更细。

#### (2) 并发控制
- **JDK 1.7**：
  - **分段锁（Segment Lock）**：
    - 每个 `Segment` 继承 `ReentrantLock`，锁住一段数据。
    - 并发度由 `Segment` 数量决定（默认 16）。
  - **读写**：
    - 写操作锁 `Segment`，读操作无锁（`volatile` 保证可见性）。
- **JDK 1.8**：
  - **CAS + synchronized**：
    - **CAS**：无锁操作（如初始化、扩容）。
    - **synchronized**：锁桶头节点（`Node`），只锁单链表或树。
  - **读写**：
    - 读：无锁，靠 `volatile`。
    - 写：锁首节点，粒度更小。

#### (3) 核心操作
- **put 操作**：
  1. 计算哈希，定位桶。
  2. 若桶为空，用 CAS 插入。
  3. 若桶非空，`synchronized` 锁首节点，插入链表/红黑树。
  4. 链表超长，转红黑树。
- **get 操作**：
  - 无锁读取，遍历桶内链表/红黑树。
- **扩容**：
  - 多线程协同，分段迁移（`transfer`）。
  - 用 `sizeCtl` 控制扩容状态。

#### (4) 关键字段
- **`Node[] table`**：
  - 主数组，存储键值对。
- **`volatile int sizeCtl`**：
  - 控制初始化和扩容（负数表示进行中）。
- **`Node` 的 `volatile`**：
  - 保证可见性。

#### 源码片段（put）
```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    int hash = spread(key.hashCode());
    int binCount = 0;
    Node<K,V>[] tab = table;
    for (;;) {
        if (tab == null) initTable(); // 初始化
        int n = tab.length, i = (n - 1) & hash;
        Node<K,V> f = tabAt(tab, i);
        if (f == null) { // 桶为空，CAS 插入
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value)))
                break;
        } else { // 桶非空，锁首节点
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    // 链表或红黑树插入
                }
            }
        }
    }
    addCount(1L, binCount); // 更新 size
    return null;
}
```

---

### 2. JDK 1.7 vs 1.8
| **特性**         | **JDK 1.7**             | **JDK 1.8**             |
|------------------|-------------------------|-------------------------|
| **结构**         | Segment + HashEntry    | Node + 链表/红黑树      |
| **锁机制**       | Segment Lock           | CAS + synchronized     |
| **并发度**       | Segment 数量（默认 16）| 数组长度（动态调整）    |
| **读性能**       | 无锁读                | 无锁读（更细粒度）      |
| **写性能**       | 锁整段                | 锁单桶                 |
| **扩容**         | 单线程                | 多线程协同             |

---

### 3. 性能优势
- **高并发**：
  - 1.8 锁粒度更小，写冲突减少。
- **无锁读**：
  - `volatile` 保证可见性，读无需同步。
- **红黑树**：
  - 长链表转树，查询 O(log n)。
- **多线程扩容**：
  - 1.8 分担迁移任务。

#### 数据
- 单线程：与 `HashMap` 接近。
- 多线程：比 `Hashtable` 高数倍（后者全表锁）。

---

### 4. 延伸与面试角度
- **与 HashMap 对比**：
  - `HashMap`：非线程安全。
  - `ConcurrentHashMap`：线程安全，性能优于 `synchronized HashMap`。
- **实际应用**：
  - 缓存：多线程共享数据。
  - 配置：高并发读写。
- **局限**：
  - 不保证强一致性（读写间可能不即时同步）。
- **面试点**：
  - 问“原理”时，提 CAS 和红黑树。
  - 问“区别”时，提 1.7 分段锁。

---

### 总结
`ConcurrentHashMap` 靠分段存储和细粒度锁实现高并发，1.8 用 CAS + `synchronized` 替代 1.7 的 `Segment` 锁，结合红黑树优化性能。面试时，可提源码片段或画桶结构，展示理解深度。