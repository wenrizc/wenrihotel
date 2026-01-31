## 1. 区别

- `HashMap`：**非线程安全**，底层是数组桶（bucket）+（链表 / 红黑树）；均摊 `O(1)`，冲突严重时树化为 `O(log n)`；允许 `null` key 和 `null` value。
- `ConcurrentHashMap`：**线程安全**，JDK 8+ 采用 **空桶 CAS + 冲突桶 bin 粒度 `synchronized` + 协助扩容**；迭代器为 **弱一致性**；不允许 `null` key/value。
- 选型一句话：单线程或外部已保证互斥用 `HashMap`；高并发读写用 `ConcurrentHashMap`；`Hashtable` 与 `Collections.synchronizedMap()` 通常不作为新代码首选。

## 2. HashMap（JDK 8+ 视角）

### 2.1 数据结构与关键参数

#### 2.1.1 核心结构

`HashMap` 的核心结构可以抽象为：

- `Node<K,V>[] table`：桶数组，长度始终保持为 **2 的幂**。
- `Node`：单向链表节点；冲突严重时桶会树化为 `TreeNode`（红黑树）。
- `loadFactor`：装载因子，默认 `0.75`。
- `threshold`：扩容阈值，通常为 `capacity * loadFactor`。

补充两个容易忽略的实现点：

- **延迟初始化**：`table` 通常在第一次 `put` 时才分配（避免空 Map 占用数组）。
- **最大容量上限**：容量增长有上限（`1 << 30` 量级），超过后不会再扩容，只能接受更高冲突。

#### 2.1.2 hash 扰动与索引定位（为什么要 2 的幂）

`HashMap` 不直接使用 `hashCode()`，而是做一次高位扰动，降低低位碰撞概率：

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16); // 混合高 16 位
}
```

桶下标用位运算定位（要求 `n` 为 2 的幂）：

```java
int index = (n - 1) & hash; // 等价于 hash % n，但更快
```

容量为 2 的幂的收益不止是“快”：

- 扩容翻倍时迁移更简单：元素要么留在原桶，要么移动到 `oldIndex + oldCap`。
- 更容易让 `hash` 的低位均匀参与分桶，降低热点桶概率。

#### 2.1.3 Node 字段与线程语义（为什么并发读写不可靠）

`HashMap.Node` 的典型字段（概念化）是：

- `final int hash`、`final K key`：构造后不可变。
- `V value`、`Node<K,V> next`：**非 `volatile`**，且没有内建同步。

这意味着只要出现并发写，就可能出现可见性与结构一致性问题，属于未定义行为范畴。

### 2.2 put / get 的关键流程

#### 2.2.1 put（核心路径）

`put(k, v)` 的关键步骤（省略边界与细节）：

1. 计算 `hash`，定位桶 `index`。
2. 桶为空：直接放入新节点。
3. 桶非空：遍历链表或红黑树。
    - 命中同 key：更新 `value`。
    - 未命中：追加节点，并在满足条件时触发树化。
4. `size++` 后如果超过 `threshold`：触发 `resize()` 扩容。

```java
// 伪代码：强调关键点而非可编译源码
V put(K key, V value) {
    int h = hash(key);
    int i = (table.length - 1) & h;
    Node<K,V> first = table[i];
    if (first == null) {
        table[i] = new Node<>(h, key, value, null);
        size++;
        modCount++; // 结构性修改会影响迭代器 fail-fast
        return null;
    }
    // 桶非空：遍历链表 / 红黑树
    // 注意：这里没有任何并发保护，多个线程同时写会破坏结构
    ...
}
```

#### 2.2.2 get（为什么快）

`get(k)` 基本是：算 `hash` → 定位桶 → 遍历链表 / 树查找。平均情况下冲突少，因此 **均摊 `O(1)`**。

### 2.3 冲突、树化与复杂度

`HashMap` 通过“拉链法”解决冲突：同一桶下多个节点形成链表。

- 链表查找：最坏 `O(n)`。
- JDK 8+ 引入树化：桶内节点数达到 `TREEIFY_THRESHOLD = 8` 且表容量达到 `MIN_TREEIFY_CAPACITY = 64` 才会树化为红黑树。
- 退化回链表：当树节点数降到 `UNTREEIFY_THRESHOLD = 6` 左右会拆回链表，降低维护树的成本。

树化的工程意义：面对“坏 hash”或攻击型输入（hash flooding）时，最坏复杂度从 `O(n)` 下降到 `O(log n)`。

### 2.4 扩容（resize）机制与成本

#### 2.4.1 扩容触发与阈值

- `threshold = capacity * loadFactor`，默认约为 `capacity * 0.75`。
- 当 `size` 超过 `threshold` 时扩容，扩容后 capacity 翻倍，threshold 也随之翻倍。

#### 2.4.2 迁移拆分算法（JDK 8+ 的关键优化）

扩容时不是简单重新取模，而是基于 `hash & oldCap` 拆分链表为两段：

```java
// 伪代码：体现“低位链 / 高位链”的拆分思想
if ((e.hash & oldCap) == 0) {
    // 低位链：下标不变，仍在 oldIndex
} else {
    // 高位链：下标变为 oldIndex + oldCap
}
```

成本：`O(n)` 的元素搬迁会带来短期 CPU 峰值与 GC 压力，因此容量预估对线上稳定性很重要。

### 2.5 为什么 HashMap 线程不安全（源码级根因）

`HashMap` 的问题可以拆成三个维度理解：

- **原子性**：链表插入、树化、`size++`、`modCount++` 都是复合操作，多个线程交错会丢数据或覆盖写。
- **可见性**：`value`、`next` 不是 `volatile`，没有 happens-before，读线程可能看到旧值或中间态。
- **结构一致性**：并发 `resize()` 或并发插入会破坏 `next` 指针与桶头引用，导致遍历异常甚至死循环（JDK 7 更典型，JDK 8 缓解但仍不保证）。

结论：**只要出现并发写，`HashMap` 的正确性就不再成立。**

### 2.6 遍历与 fail-fast

`HashMap` 的迭代器通过 `modCount` 实现 fail-fast：

- 迭代器创建时保存 `expectedModCount`。
- 遍历过程中发现 `modCount != expectedModCount`，通常抛 `ConcurrentModificationException`。

注意：fail-fast 是“尽力而为”，多线程并发修改下可能抛，也可能不抛，但都不保证遍历结果正确。

### 2.7 适用场景与最佳实践

- key 设计：保证 `equals()` 与 `hashCode()` 一致，且 key 尽量 **不可变**（如 `String`）。
- 容量预估：已知元素规模时用 `new HashMap<>(expectedSize * 4 / 3 + 1)` 降低扩容次数。
- 避免热点桶：hash 分布差（低位变化小）会显著降低吞吐，必要时做 hash 设计或换数据结构。

## 3. ConcurrentHashMap（JDK 8+ 为主）

### 3.1 设计目标与约束（为什么不支持 null）

`ConcurrentHashMap` 的目标是在并发读写下同时做到：

- **结构正确**：不会因为并发更新破坏桶结构。
- **可见性正确**：写入对其他线程可见，读不会看到危险的中间态。
- **高吞吐**：尽量缩小锁粒度，避免整表锁。

它不允许 `null` key/value，核心原因是：`get(k)` 返回 `null` 时无法区分“key 不存在”与“value 为 null”，并发下会让语义与实现都变得不可靠。

### 3.2 JDK 7：Segment 分段锁（了解思路）

JDK 7 的 `ConcurrentHashMap` 由 `Segment[]` 组成，每个 `Segment` 继承 `ReentrantLock`，相当于“很多个小 `HashMap`”。

- 写：定位到 `Segment` 后对该段加锁。
- 读：多数情况下无锁读取，依赖 `volatile` 保证可见性。
- `concurrencyLevel` 决定段数，段数越多并发写能力越强，但管理开销更大。

### 3.3 JDK 8+：空桶 CAS + 冲突桶 synchronized

JDK 8+ 移除 `Segment`，回到“数组 +（链表 / 红黑树）”的形态，但通过更细粒度的同步与无锁原语实现并发。

#### 3.3.1 关键字段与发布语义

- `volatile Node<K,V>[] table`：桶数组引用是 `volatile`，保证表初始化与切换（扩容）对其他线程可见。
- `Node` 字段通常是：`final hash/key` + `volatile val/next`，读线程可在无锁情况下安全遍历。
- CAS 原语：JDK 8 使用 `Unsafe`，JDK 9+ 之后多用 `VarHandle`，本质都是原子读改写与内存语义保证。

`ConcurrentHashMap` 同样会做 hash 扰动（源码里常见为 `spread`），并保留部分负数 hash 作为“特殊节点标记”：

```java
static final int spread(int h) {
    return (h ^ (h >>> 16)) & 0x7fffffff; // 仅示意：保留正数位用于普通节点
}
// 负数 hash 常用于内部状态节点，例如：MOVED(-1) 表示转发节点（扩容迁移中）
```

#### 3.3.2 put 的并发控制点

- 桶为空：用 CAS 将新节点放到 `table[i]`，成功则无需加锁（快路径）。
- 桶非空：锁住该桶的头节点（bin 级别 `synchronized`），在锁内完成链表/树的插入或更新。
- 扩容迁移中：遇到 `ForwardingNode` 会帮助迁移或跳到新表。

```java
// 伪代码：强调并发控制点
V put(K key, V value) {
    int h = spread(key.hashCode());
    for (;;) {
        Node<K,V>[] tab = table;
        int i = (tab.length - 1) & h;
        Node<K,V> f = tabAt(tab, i);
        if (f == null) {
            if (casTabAt(tab, i, null, new Node<>(h, key, value, null))) {
                return null; // 空桶用 CAS 直接放入
            }
            continue; // CAS 失败重试
        }
        if (f.hash == MOVED) {
            tab = helpTransfer(tab, f); // 协助扩容
            continue;
        }
        synchronized (f) { // 只锁定一个桶（bin）的头节点
            if (tabAt(tab, i) != f) {
                continue; // 头节点已变化，重试，避免锁错对象
            }
            ...
        }
        return ...;
    }
}
```

#### 3.3.3 get 的并发语义（为什么可以无锁）

`get(k)` 典型是无锁路径：

- 利用 `table` 的 `volatile` 可见性拿到当前表引用。
- 利用 `Node.val`、`Node.next` 的 `volatile` 可见性安全遍历链表或树。

因此它能做到“高并发读”几乎不阻塞，但前提是遵循 `ConcurrentHashMap` 的约束（不允许 `null`，且结构更新受控）。

#### 3.3.4 计数与 size()（不是强一致快照）

JDK 8+ 的计数使用类似 `LongAdder` 的分片设计：

- 低竞争：更新 `baseCount`。
- 高竞争：分散到 `CounterCell[]`，降低 CAS 热点。

这带来两个结论：

- `size()` / `mappingCount()` 返回的是并发更新下“某一时刻”的结果，通常用于统计与监控，**不应假设为强一致快照**。
- 需要强一致计数时，应在业务层引入额外同步或采用其他数据结构与协议。

#### 3.3.5 扩容与 ForwardingNode（协助扩容）

扩容时，`ConcurrentHashMap` 允许多个线程一起搬迁桶数据（help transfer）：

- 桶迁移完成后，桶位会被设置为 `ForwardingNode`（其 `hash` 常量为 `MOVED = -1`）。
- 其他线程读写遇到该节点会转去新表，或加入迁移任务一起加速扩容。

这种“协作式扩容”把迁移成本分摊到并发线程上，降低单线程迁移造成的长时间停顿风险。

### 3.4 迭代器与一致性语义

`ConcurrentHashMap` 的迭代器是 **弱一致性（weakly consistent）**：

- 不抛 `ConcurrentModificationException`。
- 能遍历到迭代开始时存在的元素；并发更新可能被看到，也可能看不到，但不会破坏遍历安全。

### 3.5 常用 API 与正确用法

- `putIfAbsent(k, v)`：只在 key 不存在时写入，避免“先 `get` 再 `put`”的竞态。
- `computeIfAbsent(k, f)`：典型缓存写法；实现上会通过占位等手段减少同 key 的重复计算，但仍建议 `f` **无副作用、可重复调用**，并避免长耗时阻塞（否则会拖慢同 bin 内的并发）。
- `compute` / `merge`：原子复合更新入口，优先于手写“读-改-写”。

### 3.6 适用场景与最佳实践

- 适合：高并发读写的共享映射（配置缓存、连接表、会话表、去重表）。
- 不适合：需要强一致遍历快照、严格顺序、或 LRU/TTL 淘汰的缓存（优先用 Caffeine 等专业缓存）。
- 容量规划：减少扩容触发；扩容在高峰期仍会引入额外 CPU 与内存带宽消耗。
