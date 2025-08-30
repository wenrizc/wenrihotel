
自 Redis 3.2 版本起，列表（List）类型的底层实现从 `ziplist`（压缩列表）和 `linkedlist`（双向链表）的组合转向了一个更为高效的数据结构——`quicklist`（快速列表）。 `quicklist` 巧妙地结合了 `ziplist` 和 `linkedlist` 的优点，在内存使用效率和操作性能之间取得了出色的平衡。

---

### 什么是 Quicklist？

从宏观上看，`quicklist` 是一个双向链表。但与传统链表不同的是，它的每个节点（`quicklistNode`）都包含一个 `ziplist`，而不是单个数据元素。 换言之，**`quicklist` 是一个由 `ziplist` 构成的双向链表**。

这种设计解决了以往实现的痛点：
*   **对于 `ziplist`**：虽然其内存紧凑，能有效减少内存碎片，但当列表过长时，任何修改都可能引发连锁更新（cascading updates），导致性能下降。
*   **对于 `linkedlist`**：它的插入删除操作效率很高（O(1)），但每个节点都需要额外的指针来维护链表关系，内存开销较大，且容易产生内存碎片。

`quicklist` 将一个大的列表切分成多个小的 `ziplist` 进行存储，既减少了单个 `ziplist` 过大带来的性能问题，也通过复用节点内存储空间，大大降低了 `linkedlist` 的内存开销。

---

### Quicklist 的核心结构

`quicklist` 的主要由两部分构成：`quicklist` 结构体和 `quicklistNode` 结构体。

*   **`quicklist` 结构体**：这是快速列表的“控制器”，定义了整个列表的元信息，主要包含：
    *   `head`: 指向链表的头节点。
    *   `tail`: 指向链表的尾节点。
    *   `count`: 列表中的元素总数。
    *   `len`: `quicklistNode` 的节点总数。
    *   `fill`: 用于控制每个 `quicklistNode` 中 `ziplist` 的大小，对应配置参数 `list-max-ziplist-size`。
    *   `compress`: 用于控制节点压缩深度，对应配置参数 `list-compress-depth`。

*   **`quicklistNode` 结构体**：这是构成 `quicklist` 的基本单元，主要包含：
    *   `prev`: 指向前一个 `quicklistNode` 的指针。
    *   `next`: 指向后一个 `quicklistNode` 的指针。
    *   `zl`: 指向当前节点所包含的 `ziplist`。
    *   `encoding`: 描述 `ziplist` 的存储形式，是原生字节数组还是经过 LZF 压缩的。
    *   `container`: 一个预留字段，目前固定表示使用 `ziplist` 作为数据容器。

---

### 关键特性：数据压缩

为了进一步节省内存，`quicklist` 支持对其中间的 `quicklistNode` 进行 LZF 算法无损压缩。 这一行为由配置参数 `list-compress-depth` 控制，其含义如下：

*   **`0` (默认值):** 表示不进行任何压缩。
*   **`1`:** 表示跳过首尾各 1 个节点不压缩，对其余所有中间节点进行压缩。
*   **`2`:** 表示跳过首尾各 2 个节点不压缩，对其余节点进行压缩。
*   以此类推。

由于列表操作大多在两端进行（如 `LPUSH`, `RPOP`），保持两端节点不被压缩可以确保这些高频操作的性能。 而对于不常访问的中间数据进行压缩，则可以显著降低内存占用。

---

### 配置参数与性能权衡

`quicklist` 的性能和效率很大程度上取决于以下两个配置参数：

1.  **`list-max-ziplist-size`**: 这个参数决定了每个 `quicklistNode` 内部 `ziplist` 的最大容量。
    *   **正值**: 表示 `ziplist` 内最多包含的元素数量。
    *   **负值**: 表示 `ziplist` 的最大字节大小。例如，`-2` 表示每个 `ziplist` 的大小不能超过 8KB（Redis 默认值）。

2.  **如何权衡？**
    *   **较大的 `ziplist`**: 意味着更少的 `quicklistNode`，从而减少了指针带来的内存开销，内存利用率更高。但每次修改 `ziplist` 时，需要操作的数据量也更大。
    *   **较小的 `ziplist`**: 意味着更多的 `quicklistNode`，更接近于普通双向链表的形态，内存碎片会增多，指针开销也更大。但单次操作 `ziplist` 的成本更低。

---

### 总结：Quicklist 的优势

*   **空间与时间的平衡**: 它是对 `ziplist` 和 `linkedlist` 优缺点的完美中和，既避免了 `ziplist` 在大数据量下的更新效率低下问题，也解决了 `linkedlist` 的内存开销和碎片化问题。
*   **高内存利用率**: 通过将多个元素存储在一个 `ziplist` 中，并对不常用的中间节点进行压缩，实现了比纯 `linkedlist` 更高的内存效率。
*   **稳定的性能**: 对于列表两端的操作，其时间复杂度为 O(1)，而对于索引查找等操作，虽然仍需遍历，但其结构设计使其在实践中表现稳定。