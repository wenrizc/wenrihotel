
`ConcurrentHashMap` 是 Java 并发包 `java.util.concurrent` 中的一个高度优化的线程安全的 `Map` 实现，旨在提供比 `Hashtable` 和 `Collections.synchronizedMap` 更好的并发性能。

### ConcurrentHashMap 是如何实现线程安全的？

`ConcurrentHashMap` 的线程安全实现机制在 Java 7 及更早版本与 Java 8 及更高版本中有所不同，但核心思想都是通过**细粒度锁定**来提高并发性。

#### 1. Java 7 及更早版本 (基于 Segment 的分段锁)

在 Java 7 及更早版本中，`ConcurrentHashMap` 采用了一个称为 **Segment** 的分段锁机制。
*   **结构**: `ConcurrentHashMap` 内部维护一个 `Segment` 数组，每个 `Segment` 都是一个独立的 `ReentrantLock` 锁，并且每个 `Segment` 内部包含一个 `HashEntry` 数组（类似于 `HashMap` 的实现）。
*   **并发原理**:
    *   **写入操作 (put, remove 等)**: 当进行写入操作时，`ConcurrentHashMap` 会根据 key 的 hash 值定位到对应的 `Segment`，然后只对该 `Segment` 加锁。这意味着不同的线程可以同时对不同的 `Segment` 进行写入操作，互不影响，从而实现了较高的并发性。
    *   **读取操作 (get)**: 读取操作通常不需要加锁。它利用了 `volatile` 关键字的内存可见性保证。如果读取操作确实需要加锁（例如，在某个 `Segment` 正在扩容时），它也会只锁定相应的 `Segment`。
    *   **整个 Map 操作 (size, isEmpty)**: 对于需要遍历整个 `Map` 的操作，`ConcurrentHashMap` 会先尝试不加锁地计算，如果发现计算过程中有 `Segment` 被修改，则会重新计算，或者在必要时锁定所有 `Segment`，但这种情况非常罕见且代价高昂，通常通过优化来避免。

#### 2. Java 8 及更高版本 (基于 CAS 和 synchronized 的链表/红黑树)

Java 8 对 `ConcurrentHashMap` 的实现进行了重大改进，移除了 `Segment` 结构，转而使用 **CAS (Compare-And-Swap) 操作和 `synchronized` 关键字**来保证线程安全。其内部结构更类似于 `HashMap` (数组 + 链表/红黑树)。
*   **结构**: `ConcurrentHashMap` 内部是一个 `Node` 数组，每个 `Node` 代表一个桶（bucket）。当多个键发生哈希冲突时，数据会以链表或红黑树的形式存储在桶中。
*   **并发原理**:
    *   **初始化和扩容**: 涉及到整个数组的操作，会使用 CAS 操作和辅助的 `volatile` 字段来保证可见性和原子性，避免全局锁。
    *   **写入操作 (put, remove 等)**:
        *   首先，通过 CAS 尝试设置数组中的头节点 (如果该位置为空)。
        *   如果该位置不为空，则对该桶的头节点加 **`synchronized` 锁** (而不是整个 `Map` 或 `Segment` 锁)，然后再遍历链表或红黑树进行插入或修改。这意味着不同线程可以同时对不同的桶进行写入操作，极大地提高了并发度。
        *   对于链表转换成红黑树、扩容等复杂操作，会有更复杂的 CAS 和 `synchronized` 组合来保证原子性和可见性。
    *   **读取操作 (get)**: 读取操作完全不需要加锁。它依赖于 `volatile` 字段的内存可见性，并采用了无锁化的算法来读取链表或红黑树中的元素，保证了极高的读取并发性。

### 与 `Hashtable` 和 `Collections.synchronizedMap` 的区别

| 特性/比较点         | ConcurrentHashMap (Java 8+)                                  | Hashtable                                                    | Collections.synchronizedMap(new HashMap())                   |
| :------------------ | :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| **实现线程安全的机制** | **细粒度锁定**：基于 CAS 操作和对单个桶的 `synchronized` 锁。读取操作通常无锁。 | **粗粒度锁定**：对所有公共方法使用 `synchronized` 关键字，锁住整个 `Hashtable` 对象。 | **粗粒度锁定**：使用一个内部 `mutex` 对象锁住对底层 `Map` 的所有操作。 |
| **并发性能**        | **高**：支持高并发读写，多个线程可以同时访问不同的桶进行操作。写入操作只锁住少量资源。 | **低**：在任何时刻只允许一个线程访问 `Map`，所有操作都是串行的，导致性能瓶颈。 | **低**：与 `Hashtable` 类似，任何时刻只允许一个线程访问底层 `Map`。 |
| **Null 键/值**      | **不允许**：键和值都不能为 `null`，否则抛出 `NullPointerException`。 | **不允许**：键和值都不能为 `null`，否则抛出 `NullPointerException`。 | **允许**：如果底层是 `HashMap`，则允许 `null` 键和 `null` 值。 |
| **迭代器**          | **弱一致性 (Weakly Consistent)**：迭代器创建后，可能不会反映在迭代器创建后发生的修改，但不会抛出 `ConcurrentModificationException`。 | **快速失败 (Fail-Fast)**：如果 `Map` 在迭代过程中被修改，会立即抛出 `ConcurrentModificationException`。 | **快速失败 (Fail-Fast)**：如果 `Map` 在迭代过程中被修改，会立即抛出 `ConcurrentModificationException`。 |
| **扩容机制**        | 采用并发扩容，通过 CAS 和辅助变量进行协调，不会阻塞所有操作。 | 扩容时会锁定整个 `Hashtable`。                            | 扩容时会锁定整个底层 `Map`。                                |
| **版本**            | Java 5 引入，并在 Java 8 进行了重大优化。                    | Java 1.0 就有，是早期版本提供的线程安全集合。                  | Java 1.2 引入。                                            |

#### 总结：

*   **`Hashtable` 和 `Collections.synchronizedMap`**：它们都采用了 **粗粒度锁**（或称作表级锁、全局锁）的策略，即在执行任何操作（读或写）时，都会锁定整个 `Map`。这导致了在并发环境下性能非常差，因为所有操作都变成了串行的，极大地限制了吞吐量。`Collections.synchronizedMap` 只是 `Collections` 工具类提供的一个同步包装器，其同步机制与 `Hashtable` 类似，都是简单粗暴地在每个方法上加 `synchronized`。
*   **`ConcurrentHashMap`**：通过采用 **细粒度锁** 和 **无锁算法 (CAS)** 相结合的方式，允许多个线程同时对 `Map` 的不同部分进行操作。特别是在 Java 8 之后，其对单个桶的 `synchronized` 锁和读取操作的无锁化处理，使得它在并发读写场景下表现出卓越的性能。因此，在需要线程安全的 `Map` 时，`ConcurrentHashMap` 是首选。