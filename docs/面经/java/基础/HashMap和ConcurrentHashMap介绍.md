# HashMap和ConcurrentHashMap介绍

## 1. 结论

ConcurrentHashMap 是 `java.util.concurrent` 包下的一个类，是 HashMap 的线程安全版本。它提供了比 `Hashtable` 和 `Collections.synchronizedMap()` 更高的并发性能。它允许多个线程同时读写集合，而不会导致数据不一致或抛出异常。

## 2. ConcurrentHashMap

ConcurrentHashMap 是 `java.util.concurrent` 包下的一个类，是 HashMap 的线程安全版本。它提供了比 `Hashtable` 和 `Collections.synchronizedMap()` 更高的并发性能。它允许多个线程同时读写集合，而不会导致数据不一致或抛出异常。

### 2.1 线程安全的实现原理

ConcurrentHashMap 的线程安全实现也经历了重要的版本迭代。

- **JDK 1.7：分段锁 (Segment Locking)**

    1. **基本结构**：ConcurrentHashMap 内部由一个 `Segment` 数组和一个 `HashEntry` 数组组成。`Segment` 本身继承自 `ReentrantLock`，可以看作是一个小的、线程安全的 HashMap。

    2. **分段思想**：整个 ConcurrentHashMap 被分成多个 `Segment`（默认 16 个）。当一个线程需要对某个数据进行操作时，它不是锁定整个 Map，而是先定位到数据所在的 `Segment`，然后只锁定那一个 `Segment`。

    3. **如何工作**：

        - **put 操作**：先通过哈希值定位到具体的 `Segment`，然后对该 `Segment` 加锁。加锁成功后，再执行与 HashMap 类似的 `put` 操作。由于锁只作用于单个 `Segment`，其他线程可以同时访问并修改其他 `Segment` 中的数据，从而实现了并发。`Segment` 的数量被称为“并发度”。

        - **get 操作**：`get` 操作大部分时候是不需要加锁的。`HashEntry` 中的 `value` 和 `next` 指针都使用 `volatile` 关键字修饰，这保证了内存可见性。当一个线程修改了某个 `value` 后，其他线程能够立即看到这个修改，从而可以安全地读取。只有在读取到的值为 null 时，才会尝试加锁来保证获取到最新的值。

        - **size 操作**：计算 `size` 时，会先尝试不加锁地累加两次所有 `Segment` 的 `count` 值。如果两次结果一致，就直接返回。如果不一致，则会依次锁住所有的 `Segment` 来进行精确计算。

    4. **优点**：通过将锁的粒度从整个 Map 缩小到一个 `Segment`，大大提高了并发访问的效率。

    5. **缺点**：分段锁的实现相对复杂，且在某些场景下（例如 `size()` 操作）的开销较大。

- **JDK 1.8 及之后：CAS + synchronized**

    1. **基本结构**：摒弃了 `Segment` 的设计，回归到与 JDK 1.8 中 HashMap 类似的“数组 + 链表 + 红黑树”的结构。

    2. **锁粒度更细**：锁的粒度进一步降低，从锁定一个 `Segment` 缩小到只锁定数组中某个具体的索引位置（`bucket`）。

    3. **实现方式**：

        - **CAS (Compare-And-Swap)**：在很多关键操作中，都使用了 CAS 这种无锁算法。例如，在向一个为 null 的 `bucket` 中添加第一个节点时，会使用 CAS 操作来尝试写入。如果 CAS 成功，说明没有竞争，操作完成。如果失败，说明有其他线程已经占用了该位置，则会进入下一步的同步逻辑。

        - **synchronized**：当 CAS 操作失败，或者需要修改的 `bucket` 中已经存在节点（链表或红黑树的头节点）时，就会使用 `synchronized` 关键字锁住这个头节点。

        - **put 操作流程**：

            1. 计算哈希值，定位到数组的索引位置。

            2. 如果该位置为 null，使用 CAS 尝试插入新节点。成功则返回。

            3. 如果该位置不为 null，则使用 `synchronized` 锁住该位置的头节点。

            4. 在同步块内，遍历链表或红黑树，判断是更新 `value` 还是在末尾添加新节点。

            5. 这个过程中，其他线程如果也想操作同一个 `bucket`，就会因为获取不到 `synchronized` 锁而阻塞。但如果它们操作的是不同的 `bucket`，则完全不受影响，可以并发执行。

    4. **优点**：

        - **锁粒度更小**：只在发生哈希冲突时才加锁，并且锁定的只是单个 `bucket` 的头节点，并发性能更高。

        - **实现更简洁**：相比分段锁，实现逻辑更清晰。

        - **更好的性能**：在大多数情况下，尤其是在高并发场景下，性能优于 JDK 1.7 的版本。

## 3. 常见追问/易错点

- 追问：HashMap和ConcurrentHashMap介绍的核心流程或关键点是什么？
  - 答：核心结论是：ConcurrentHashMap 是 `java.util.concurrent` 包下的一个类，是 HashMap 的线程安全版本。它提供了比 `Hashtable` 和 `Collections.synchronizedMap()` 更高的并发性能。展开时可按“ConcurrentHashMap”组织，先概述再逐点展开，保证结构完整。其中ConcurrentHashMap侧重ConcurrentHashMap 是 `java.util.concurrent` 包下的一个类，是 HashMap 的线程安全版本。它提供了比 `Hashtable` 和 `Collections.synchronizedMap()` 更高的并发性能。回答时要体现步骤、关键点与适用场景，必要时补充示例或对比。
- 易错点：HashMap和ConcurrentHashMap介绍中最容易混淆或踩坑的点是什么？
  - 答：常见易错点是只给结论不讲依据、边界条件与前提不清。比如ConcurrentHashMap中提到：ConcurrentHashMap 是 `java.util.concurrent` 包下的一个类，是 HashMap 的线程安全版本。它提供了比 `Hashtable` 和 `Collections.synchronizedMap()` 更高的并发性能。这些细节很容易被忽视。回答时应明确边界、关键步骤与适用场景，并用实例或对比验证。
