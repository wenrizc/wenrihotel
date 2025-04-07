
JDK 1.7 和 1.8 中 `HashMap` 的主要区别在于**底层结构**、**性能优化**和**实现细节**：
1. **数据结构**：
   - 1.7：数组 + 链表。
   - 1.8：数组 + 链表 + 红黑树（链表超长时转换）。
2. **插入方式**：
   - 1.7：头插法（可能导致循环链表）。
   - 1.8：尾插法（更安全）。
3. **扩容机制**：
   - 1.7：先扩容后插入。
   - 1.8：先插入后扩容。
4. **哈希计算**：
   - 1.7：复杂扰动函数。
   - 1.8：简化扰动，提高效率。
5. **并发安全性**：
   - 1.7：扩容可能死循环（多线程）。
   - 1.8：优化避免死循环，但仍非线程安全。

---

### 1. 区别详解
#### (1) 数据结构
- **JDK 1.7**：
  - 底层：`Entry[]` 数组 + 单向链表。
  - 冲突：哈希碰撞用链表存储。
- **JDK 1.8**：
  - 底层：`Node[]` 数组 + 单向链表 + 红黑树。
  - 冲突：
    - 链表长度 ≤ 8：用链表。
    - 链表长度 > 8 且数组容量 ≥ 64：转红黑树。
    - 红黑树节点数 < 6：退回链表。
- **影响**：
  - 1.7：链表长时查询 O(n)。
  - 1.8：红黑树优化为 O(log n)。

#### (2) 插入方式
- **JDK 1.7**：
  - 头插法：新元素插链表头部。
  - 源码：
```java
void addEntry(int hash, K key, V value, int bucketIndex) {
    Entry<K,V> e = table[bucketIndex];
    table[bucketIndex] = new Entry<>(hash, key, value, e);
}
```
- **JDK 1.8**：
  - 尾插法：新元素插链表尾部。
  - 源码：
```java
Node<K,V> newNode(int hash, K key, V value, Node<K,V> next) {
    return new Node<>(hash, key, value, next);
}
// 插入时遍历到尾部
```
- **影响**：
  - 1.7：多线程扩容可能形成环。
  - 1.8：顺序稳定，避免环。

#### (3) 扩容机制
- **JDK 1.7**：
  - 先扩容（`resize`）再插入。
  - 触发：`size >= threshold`（默认负载因子 0.75）。
  - 源码：
```java
void resize(int newCapacity) {
    Entry[] newTable = new Entry[newCapacity];
    transfer(newTable);
}
```
- **JDK 1.8**：
  - 先插入再扩容。
  - 优化：元素按高低位拆分（无需全重新计算哈希）。
  - 源码：
```java
if (++size > threshold) resize();
```
- **影响**：
  - 1.7：多线程扩容易错。
  - 1.8：更高效，线程安全提升。

#### (4) 哈希计算
- **JDK 1.7**：
  - 复杂扰动：多次位运算（9 次）。
```java
static int hash(int h) {
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}
```
- **JDK 1.8**：
  - 简化扰动：一次高低位异或。
```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
- **影响**：
  - 1.8：减少计算量，冲突更均匀。

#### (5) 并发安全性
- **JDK 1.7**：
  - 多线程扩容，头插法可能形成循环链表，死循环。
  - 示例：
    - 线程 A 和 B 同时 `resize`，链表顺序颠倒。
- **JDK 1.8**：
  - 尾插法 + 扩容优化，避免死循环。
  - 但仍非线程安全，需 `ConcurrentHashMap`。
- **影响**：
  - 1.8：单线程更稳定，多线程仍需注意。

---

### 2. 源码对比
#### 1.7 Entry
```java
static class Entry<K,V> {
    final K key;
    V value;
    Entry<K,V> next;
    int hash;
}
```

#### 1.8 Node/TreeNode
```java
static class Node<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;
}

static class TreeNode<K,V> extends Node<K,V> {
    TreeNode<K,V> parent;
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    boolean red;
}
```

---

### 3. 性能与适用性
#### JDK 1.7
- **优点**：
  - 简单，内存占用略低。
- **缺点**：
  - 长链表性能差。
  - 多线程不安全。

#### JDK 1.8
- **优点**：
  - 红黑树优化查询。
  - 扩容和插入更高效。
- **缺点**：
  - 复杂性增高，内存略多。

#### 示例
```java
HashMap<String, Integer> map = new HashMap<>();
for (int i = 0; i < 10000; i++) {
    map.put("key" + i, i); // 1.8 长链表转红黑树
}
```

---

### 4. 延伸与面试角度
- **红黑树条件**：
  - 链表长度 > 8 且容量 ≥ 64。
- **实际应用**：
  - 1.7：小规模简单场景。
  - 1.8：大数据、高并发。
- **与 ConcurrentHashMap**：
  - 1.8 借鉴红黑树，线程安全更优。
- **面试点**：
  - 问“区别”时，提结构和扩容。
  - 问“死循环”时，提头插法。

---

### 总结
JDK 1.7 的 `HashMap` 用数组 + 链表，头插法，易死循环；1.8 引入红黑树，尾插法，优化性能和安全。面试时，可提红黑树转换条件或画扩容图，展示理解深度。