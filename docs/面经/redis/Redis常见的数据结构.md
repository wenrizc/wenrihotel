
#### Redis 常见数据结构
Redis 是一种键值数据库，支持丰富的数据结构，主要包括：
1. **字符串（String）**：简单键值对。
2. **哈希（Hash）**：键值对集合。
3. **列表（List）**：有序可重复元素。
4. **集合（Set）**：无序不重复元素。
5. **有序集合（Sorted Set）**：带分数的集合。

#### 核心点
- 数据结构多样，内存存储，高效操作。

---

### 1. 数据结构详解
#### (1) 字符串（String）
- **底层实现**：
  - **SDS（Simple Dynamic String）**：动态字符串，优于 C 字符串。
- **特点**：
  - 存储文本、数字、字节流。
  - 支持增减操作（如 `INCR`）。
- **命令**：
  - `SET key value`, `GET key`, `INCR key`。
- **时间复杂度**：
  - `O(1)`。
- **示例**：
```redis
SET name "Alice"
INCR count  # 自增
```
- **场景**：
  - 缓存、计数器。

#### (2) 哈希（Hash）
- **底层实现**：
  - **字典（dict）**：哈希表。
  - 小数据量用 **ziplist**（压缩列表）优化内存。
- **特点**：
  - 键值对集合，类似 Map。
  - 字段独立操作。
- **命令**：
  - `HSET key field value`, `HGET key field`, `HGETALL key`。
- **时间复杂度**：
  - `O(1)`（单字段）。
- **示例**：
```redis
HSET user:1 name "Alice" age "25"
HGET user:1 name  # 返回 "Alice"
```
- **场景**：
  - 对象存储（如用户信息）。

#### (3) 列表（List）
- **底层实现**：
  - **双向链表（quicklist）**：Redis 3.2+，结合 ziplist 和链表。
- **特点**：
  - 有序，可重复。
  - 支持两端操作。
- **命令**：
  - `LPUSH key value`, `RPOP key`, `LRANGE key 0 -1`。
- **时间复杂度**：
  - 头尾操作：`O(1)`。
  - 中间访问：`O(n)`。
- **示例**：
```redis
LPUSH list 1 2 3
RPOP list  # 返回 1
```
- **场景**：
  - 队列、栈、时间线。

#### (4) 集合（Set）
- **底层实现**：
  - **字典（dict）**：键存值，值为空。
  - 小集合用 **intset**（整数集合）优化。
- **特点**：
  - 无序，不重复。
  - 支持交并差操作。
- **命令**：
  - `SADD key value`, `SMEMBERS key`, `SINTER key1 key2`。
- **时间复杂度**：
  - 添加/查询：`O(1)`。
  - 集合运算：`O(n)`。
- **示例**：
```redis
SADD set 1 2 3
SADD set 2
SMEMBERS set  # 返回 1 2 3
```
- **场景**：
  - 去重、标签系统。

#### (5) 有序集合（Sorted Set）
- **底层实现**：
  - **跳表（skiplist）** + **字典（dict）**：
    - 跳表：排序。
    - 字典：快速查找。
- **特点**：
  - 每个元素带分数，按分数排序。
  - 支持排名查询。
- **命令**：
  - `ZADD key score value`, `ZRANGE key 0 -1`, `ZRANK key value`。
- **时间复杂度**：
  - 添加/查询：`O(log n)`。
  - 范围查询：`O(log n + k)`。
- **示例**：
```redis
ZADD rank 90 "Alice" 85 "Bob"
ZRANGE rank 0 1  # 返回 "Bob" "Alice"
```
- **场景**：
  - 排行榜、优先级队列。

---

### 2. 其他数据结构（扩展）
- **Bitmap**：
  - 字符串变种，按位操作。
  - `SETBIT`, `BITCOUNT`。
  - 场景：统计活跃用户。
- **HyperLogLog**：
  - 概率统计基数。
  - `PFADD`, `PFCOUNT`。
  - 场景：UV 计算。
- **Geo**：
  - 有序集合扩展，地理位置。
  - `GEOADD`, `GEODIST`。
  - 场景：附近的人。

---

### 3. 底层优化
- **SDS**：长度记录，O(1) 获取长度。
- **Ziplist**：连续内存，小数据节省空间。
- **Quicklist**：链表 + ziplist，平衡性能。
- **Intset**：整数优化，小集合高效。
- **Skiplist**：多层索引，快速排序。

---

### 4. 延伸与面试角度
- **与传统数据库**：
  - Redis 内存操作，复杂度低。
- **实际应用**：
  - String：Session 缓存。
  - Hash：购物车。
  - List：消息队列。
  - Set：好友列表。
  - ZSet：实时排名。
- **面试点**：
  - 问“结构”时，提五种及复杂度。
  - 问“实现”时，提 SDS 或跳表。

---

### 总结
Redis 提供 String、Hash、List、Set、ZSet 五大核心数据结构，底层用 SDS、字典、跳表等实现，高效支持多种场景。面试时，可提命令示例或场景应用，展示理解深度。