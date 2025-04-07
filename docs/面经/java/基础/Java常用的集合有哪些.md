
Java SE 的集合框架（Java Collections Framework, JCF）位于 java.util 包，提供了一组接口和类，用于存储和操作一组对象。主要包括 **List**（有序、可重复）、**Set**（无序、不可重复）、**Map**（键值对）和 **Queue**（队列），底层实现基于数组、链表、哈希表或树等数据结构。

1. **核心接口**：
    - **Collection**：单列集合的根接口。
        - **List**：有序、可重复。
        - **Set**：无序、不可重复。
        - **Queue**：队列/双端队列。
    - **Map**：键值对集合。
2. **常用实现类**：
    - **List**：ArrayList、LinkedList。
    - **Set**：HashSet、TreeSet。
    - **Map**：HashMap、TreeMap。
    - **Queue**：PriorityQueue、LinkedList（实现 Deque）。
3. **特点**：
    - 动态大小（对比数组）。
    - 泛型支持（JDK 5+）。
    - 线程安全需额外处理（如 Collections.synchronizedList）。

### 核心接口与实现详解

#### 1. List（列表）

- **特点**：有序（按插入顺序），允许重复元素。
- **实现**：
    - **ArrayList**：
        - 底层：动态数组。
        - 查询快 O(1)，增删慢 O(n)。
    - **LinkedList**：
        - 底层：双向链表。
        - 增删快 O(1)，查询慢 O(n)。

#### 2. Set（集合）

- **特点**：无序（或特定顺序），元素唯一。
- **实现**：
    - **HashSet**：
        - 底层：哈希表（HashMap）。
        - 增删查 O(1)，无序。
    - **TreeSet**：
        - 底层：红黑树。
        - 增删查 O(log n)，有序。

#### 3. Map（映射）

- **特点**：键值对，键唯一。
- **实现**：
    - **HashMap**：
        - 底层：哈希表（数组 + 链表/红黑树，JDK 8+）。
        - 增删查 O(1)，无序。
    - **TreeMap**：
        - 底层：红黑树。
        - 增删查 O(log n)，键有序。

#### 4. Queue（队列）

- **特点**：先进先出（FIFO）或优先级排序。
- **实现**：
    - **PriorityQueue**：
        - 底层：堆（二叉堆）。
        - 出队按优先级 O(log n)。
    - **LinkedList**（实现 Deque）：
        - 支持双端队列，增删 O(1)。

### 数据结构与性能

|集合类型|实现类|底层结构|增删复杂度|查询复杂度|特点|
|---|---|---|---|---|---|
|**List**|ArrayList|动态数组|O(n)|O(1)|查询快|
||LinkedList|双向链表|O(1)|O(n)|增删快|
|**Set**|HashSet|哈希表|O(1)|O(1)|无序唯一|
||TreeSet|红黑树|O(log n)|O(log n)|有序唯一|
|**Map**|HashMap|哈希表|O(1)|O(1)|无序键值|
||TreeMap|红黑树|O(log n)|O(log n)|有序键值|
|**Queue**|PriorityQueue|堆|O(log n)|O(1)|优先级出队|

### 延伸与面试角度

- **线程安全**：
    - 默认非线程安全。
    - 解决：Collections.synchronizedList 或 ConcurrentHashMap。
- **JDK 8+ 改进**：
    - HashMap：链表过长转红黑树（桶内 > 8）。
    - Stream API：集合操作更简洁。
- **实际应用**：
    - ArrayList：频繁查询。
    - HashMap：键值存储。
    - PriorityQueue：任务调度。
- **面试点**：
    - 问“选择依据”时，提复杂度与场景。
    - 问“线程安全”时，提并发集合。

### 总结

Java SE 集合框架包括 List（ArrayList、LinkedList）、Set（HashSet、TreeSet）、Map（HashMap、TreeMap）和 Queue（PriorityQueue），基于数组、链表、哈希表、红黑树实现，满足不同需求。面试时，可结合代码示例或复杂度分析，展示掌握程度。