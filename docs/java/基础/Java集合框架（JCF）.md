
## 核心概念

**Java Collections Framework (JCF)** 是Java提供的标准容器库，自JDK 1.2起引入，具有以下优势：

- 降低编程复杂度
- 提升程序性能
- 增强API互操作性
- 简化学习曲线
- 促进代码重用

> 注：Java容器只能存储对象，基本类型需要先装箱（如int→Integer）才能使用，系统通常会自动完成装箱和拆箱操作。

## 容器分类

Java集合框架主要分为两大类：

1. **Collection** - 存储对象集合
2. **Map** - 存储键值对映射

### Collection体系

#### Set（集合，无重复元素）

- **TreeSet**：基于红黑树，支持排序，查找效率O(logN)
- **HashSet**：基于哈希表，查找效率O(1)，不保证元素顺序
- **LinkedHashSet**：兼具HashSet的高效查找和元素插入顺序保存的特性

#### List（列表，有序可重复）

- **ArrayList**：基于动态数组，支持随机访问，适合查询操作
- **Vector**：类似ArrayList但线程安全（性能较低）
- **LinkedList**：基于双向链表，适合频繁插入删除，可用作栈/队列/双向队列

#### Queue（队列）

- **LinkedList**：可实现双向队列
- **PriorityQueue**：基于堆结构，实现优先级队列

### Map体系（键值对映射）

- **TreeMap**：基于红黑树，键值有序存储
- **HashMap**：基于哈希表，高效查找
- **Hashtable**：线程安全版HashMap（已过时）
- **LinkedHashMap**：保持插入顺序或LRU（最近最少使用）顺序的HashMap
- **ConcurrentHashMap**：现代高效的线程安全Map实现，采用分段锁提高并发性能