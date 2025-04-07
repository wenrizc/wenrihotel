
`ArrayList` 和 `LinkedList` 都是 Java `List` 接口的实现，但底层结构和适用场景不同：
- **ArrayList**：
  - 底层是动态数组，随机访问快（O(1)），增删慢（O(n)）。
- **LinkedList**：
  - 底层是双向链表，增删快（O(1)），访问慢（O(n)）。

主要区别在于**数据结构**、**性能**和**使用场景**。

---

### 1. 区别详解
#### (1) 底层数据结构
- **ArrayList**：
  - 动态数组（`Object[]`）。
  - 容量不足时扩容（默认 1.5 倍）。
- **LinkedList**：
  - 双向链表（`Node` 节点）。
  - 每个节点含前驱（prev）、后继（next）和数据。

#### (2) 时间复杂度
| **操作**      | **ArrayList** | **LinkedList** | **说明**                  |
|---------------|---------------|----------------|---------------------------|
| 随机访问（如 get(index)） | O(1)          | O(n)           | 数组直接索引，链表需遍历  |
| 头部插入/删除 | O(n)          | O(1)           | 数组移动元素，链表改指针  |
| 尾部插入/删除 | O(1)          | O(1)           | 数组尾追加快，链表改尾指针|
| 中间插入/删除 | O(n)          | O(n) 或 O(1)*  | 数组移位，链表定位后 O(1)|

*注：`LinkedList` 中间操作需 O(n) 找到位置，找到后 O(1) 修改指针。

#### (3) 空间复杂度
- **ArrayList**：
  - 连续内存，预留空间（capacity > size）。
  - 浪费少量未用空间。
- **LinkedList**：
  - 非连续内存，每个节点存指针。
  - 额外开销（prev/next 引用）。

#### (4) 线程安全
- **两者**：
  - 均非线程安全。
  - 需 `Collections.synchronizedList()` 或 `CopyOnWriteArrayList`。

#### (5) 功能扩展
- **ArrayList**：
  - 仅 `List` 功能。
- **LinkedList**：
  - 实现 `Deque` 和 `Queue`，支持栈/队列操作（如 `push`、`pop`）。

---

### 2. 源码视角
#### ArrayList
- **核心字段**：
```java
private Object[] elementData; // 数组
private int size;            // 元素个数
```
- **扩容**：
```java
int newCapacity = oldCapacity + (oldCapacity >> 1); // 1.5 倍
elementData = Arrays.copyOf(elementData, newCapacity);
```

#### LinkedList
- **核心字段**：
```java
private Node<E> first; // 头节点
private Node<E> last;  // 尾节点

static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;
}
```
- **插入**：
```java
void linkLast(E e) {
    Node<E> l = last;
    Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null) first = newNode;
    else l.next = newNode;
}
```

---

### 3. 适用场景
#### ArrayList
- **适合**：
  - 随机访问频繁（如 `get(index)`）。
  - 数据追加为主（如日志列表）。
- **示例**：
  - 存储查询结果，快速遍历。
- **不适合**：
  - 频繁头部插入/删除。

#### LinkedList
- **适合**：
  - 频繁增删（如队列、栈）。
  - 双端操作（如 `addFirst`、`removeLast`）。
- **示例**：
  - 实现任务队列，动态调整。
- **不适合**：
  - 随机访问（如查找第 n 个元素）。

---

### 4. 性能对比示例
```java
List<Integer> arrayList = new ArrayList<>();
List<Integer> linkedList = new LinkedList<>();

// 添加 10 万元素
for (int i = 0; i < 100000; i++) {
    arrayList.add(0, i); // 头部插入
    linkedList.add(0, i);
}
// ArrayList 慢（O(n) 移位），LinkedList 快（O(1) 改指针）

// 随机访问
System.out.println(arrayList.get(50000)); // O(1)
System.out.println(linkedList.get(50000)); // O(n)
```

---

### 5. 延伸与面试角度
- **内存使用**：
  - `ArrayList`：紧凑，但扩容浪费。
  - `LinkedList`：指针开销大。
- **与 Vector 对比**：
  - `ArrayList`：非同步，性能高。
  - `Vector`：同步，线程安全但慢。
- **实际应用**：
  - `ArrayList`：分页数据。
  - `LinkedList`：任务调度。
- **面试点**：
  - 问“区别”时，提结构和复杂度。
  - 问“场景”时，提查询 vs 增删。

---

### 总结
`ArrayList` 基于数组，适合查询；`LinkedList` 基于链表，适合增删。性能差异源于底层结构，选型看操作类型。面试时，可提时间复杂度或画链表图，展示理解深度。