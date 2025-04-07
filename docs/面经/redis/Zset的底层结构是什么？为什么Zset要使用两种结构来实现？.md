
#### Zset 的底层结构
Redis 的 `Zset` 底层使用两种数据结构实现：
1. **跳表（Skip List）**：存储元素及其分数（score），支持高效的有序操作。
2. **哈希表（Hash Table）**：存储元素到分数的映射，快速定位元素。

#### 为什么使用两种结构
- **跳表**：提供有序性，支持范围查询（如 `ZRANGE`）和快速插入/删除。
- **哈希表**：提供 O(1) 的元素查找和更新能力。
两种结构结合，兼顾**高效排序**和**快速查找**，满足 `Zset` 的多种操作需求。

---

### 1. Zset 底层结构详解
#### (1) 跳表（Skip List）
- **定义**：
  - 一种随机化的多层链表结构，每层是元素的子集。
  - 底层包含所有元素，上层是稀疏索引。
- **实现**：
  - 每个节点存 `member`（元素）和 `score`（分数）。
  - 多级指针加速查找。
- **操作**：
  - 插入：O(log N)。
  - 删除：O(log N)。
  - 范围查询：O(log N + M)，M 为返回元素数。
- **源码表示**（Redis `zskiplist`）：
```c
typedef struct zskiplistNode {
    sds ele;           // 元素值
    double score;      // 分数
    struct zskiplistNode *backward; // 后向指针
    struct zskiplistLevel {
        struct zskiplistNode *forward; // 前向指针
        unsigned int span; // 跨度
    } level[]; // 多层索引
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;
```

#### (2) 哈希表（Hash Table）
- **定义**：
  - 一个键值对映射，键是元素，值是分数。
- **实现**：
  - Redis 用 `dict`（字典）实现。
- **操作**：
  - 查找：O(1)。
  - 更新：O(1)。
- **源码表示**（Redis `dict`）：
```c
typedef struct dictEntry {
    void *key;    // 元素
    union {
        double score; // 分数
    } v;
    struct dictEntry *next; // 链表解决冲突
} dictEntry;

typedef struct dict {
    dictEntry **table;
    unsigned long size;
} dict;
```

#### Zset 整体结构
- **zset 结构体**：
```c
typedef struct zset {
    dict *dict;        // 哈希表
    zskiplist *zsl;    // 跳表
} zset;
```
- 元素同时存入 `dict` 和 `zsl`，保证一致性。

---

### 2. 为什么使用两种结构
#### 单一结构的局限
- **仅用哈希表**：
  - 优点：O(1) 查找和更新。
  - 缺点：无序，无法支持范围查询（如 `ZRANGE`）。
- **仅用跳表**：
  - 优点：有序，范围查询快。
  - 缺点：查找特定元素慢（O(log N)），无 O(1) 操作。

#### 两种结构结合的原因
1. **功能需求**：
   - **快速查找**：`ZSCORE` 获取元素分数，需 O(1)。
   - **有序操作**：`ZRANGE`、`ZRANK` 需按分数排序。
2. **性能优化**：
   - 哈希表负责高效定位。
   - 跳表负责排序和范围操作。
3. **空间换时间**：
   - 元素重复存储（`dict` 和 `zsl`），换取操作效率。

#### 示例操作
- **ZADD key score member**：
  - `dict`：存 `<member, score>`。
  - `zsl`：按 `score` 插入跳表。
- **ZSCORE key member**：
  - `dict`：O(1) 返回分数。
- **ZRANGE key start stop**：
  - `zsl`：O(log N) 定位，遍历返回范围。

---

### 3. 为什么选择跳表而非其他
#### 与红黑树对比
- **跳表优点**：
  - 实现简单，代码量少。
  - 并发友好（插入不需全局调整）。
  - 范围查询快（链表遍历）。
- **红黑树优点**：
  - 平衡性好，稳定 O(log N)。
- **Redis 选择跳表**：
  - 简单性和范围操作优先。

#### 随机层数
- 跳表通过随机生成节点层数（概率如 1/2）保持平衡，平均 O(log N)。

---

### 4. 延伸与面试角度
- **时间复杂度**：
  - `ZADD`：O(log N)。
  - `ZSCORE`：O(1)。
  - `ZRANGE`：O(log N + M)。
- **内存开销**：
  - 双结构增加存储，但提升性能。
- **实际应用**：
  - 排行榜：按分数排序。
  - 延时队列：按时间戳排序。
- **面试点**：
  - 问“结构”时，提跳表和哈希表。
  - 问“原因”时，提功能和性能。

---

### 总结
`Zset` 底层用跳表（排序）和哈希表（查找）实现，两种结构结合满足快速定位和有序操作需求。跳表提供 O(log N) 的范围查询，哈希表提供 O(1) 的元素访问。面试时，可画跳表示意图或提时间复杂度，展示理解深度。