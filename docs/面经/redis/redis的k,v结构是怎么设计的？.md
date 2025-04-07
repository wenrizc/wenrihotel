
#### Redis 的 K,V 结构设计
Redis 是一个高性能的键值存储数据库，其键值（Key-Value）结构基于**哈希表（Hash Table）实现，底层通过 `dict`（字典）数据结构管理所有键值对。键（Key）是字符串，值（Value）支持多种数据类型（如字符串、列表、哈希等），通过高效的内存管理和动态类型设计实现快速存取。

#### 核心设计
- **键**：`SDS`（简单动态字符串）存储，高效且节省内存。
- **值**：`redisObject` 封装，支持多种类型，动态编码优化存储。
- **存储结构**：全局哈希表（`dict`），键映射到值。

---

### 1. Redis K,V 结构详解
#### (1) 键（Key）的设计
- **类型**：字符串，使用 `SDS`（Simple Dynamic String）实现。
- **SDS 结构**：
```c
struct sdshdr {
    int len;       // 已用长度
    int free;      // 剩余空间
    char buf[];    // 数据缓冲区
};
```
- **优点**：
  - O(1) 获取长度（`len` 字段）。
  - 动态扩展，预分配空间（`free`）。
  - 二进制安全（支持任意字节）。
- **示例**：
  - Key：`"user:1001"`，存为 SDS。

#### (2) 值（Value）的设计
- **类型**：支持多种数据类型（String、List、Set、Hash、Zset 等）。
- **封装**：用 `redisObject` 结构表示。
```c
typedef struct redisObject {
    unsigned type:4;    // 数据类型（如 REDIS_STRING）
    unsigned encoding:4;// 编码方式（如 REDIS_ENCODING_RAW）
    unsigned lru:24;    // LRU 时间戳
    int refcount;       // 引用计数
    void *ptr;          // 指向实际数据
} robj;
```
- **动态编码**：
  - String：
    - `REDIS_ENCODING_INT`：小整数直接存（节省内存）。
    - `REDIS_ENCODING_RAW`：大字符串用 SDS。
  - List：`REDIS_ENCODING_LINKEDLIST` 或 `REDIS_ENCODING_ZIPLIST`（压缩列表）。
  - Zset：`REDIS_ENCODING_SKIPLIST` + `dict`。
- **优点**：
  - 灵活性：支持多类型。
  - 优化：小数据用压缩编码。

#### (3) 键值映射（dict）
- **结构**：全局哈希表 `dict`。
```c
typedef struct dictEntry {
    void *key;          // 键（SDS）
    union {
        void *val;      // 值（redisObject）
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next; // 链表解决冲突
} dictEntry;

typedef struct dictht {
    dictEntry **table;  // 哈希表数组
    unsigned long size; // 数组大小
    unsigned long used; // 已用槽数
} dictht;

typedef struct dict {
    dictType *type;     // 类型特定函数
    dictht ht[2];       // 两个哈希表（渐进式 rehash）
    long rehashidx;     // rehash 进度
} dict;
```
- **存储**：
  - Redis 实例含一个 `redisDb`，每个 `redisDb` 含一个 `dict`。
- **冲突解决**：链地址法（`next` 指针）。

---

### 2. 设计细节
#### (1) 哈希表实现
- **哈希函数**：`siphash`（Redis 4.0+），抗碰撞。
- **扩容/缩容**：
  - 当 `used/size > 1`（负载因子），触发扩容。
  - 用双表（`ht[0]` 和 `ht[1]`），渐进式 rehash。
- **渐进式 rehash**：
  - 分批迁移，避免一次性阻塞。
  - `rehashidx` 标记进度。

#### (2) 内存管理
- **SDS**：预分配减少内存分配次数。
- **redisObject**：引用计数（`refcount`）共享对象。
- **编码优化**：小数据用压缩格式（如 `ZIPLIST`）。

#### (3) 数据访问
- **SET key value**：
  - 计算 key 的哈希，定位 `dictEntry`。
  - 创建或更新 `redisObject`，存入 `val`。
- **GET key**：
  - 哈希定位，O(1) 返回 `val`。

---

### 3. 为什么这样设计
#### (1) 高性能
- **哈希表**：O(1) 平均复杂度。
- **SDS**：快速长度查询和拼接。
- **渐进 rehash**：避免阻塞。

#### (2) 内存效率
- **动态编码**：小数据压缩存储。
- **引用计数**：复用对象。

#### (3) 灵活性
- **redisObject**：支持多类型。
- **dict**：通用键值映射。

---

### 4. 延伸与面试角度
- **时间复杂度**：
  - `SET`/`GET`：O(1)。
  - 扩容：均摊 O(1)。
- **与 HashMap 对比**：
  - Redis：全局单表，支持多类型。
  - HashMap：对象内，单一类型。
- **实际应用**：
  - 缓存：`SET user:1 "data"`。
  - 会话：`HSET session:1 key val`。
- **面试点**：
  - 问“结构”时，提 `dict` 和 `redisObject`。
  - 问“设计”时，提性能和内存。

---

### 示例
```c
// SET "key" "value"
dictAdd(db->dict, sdsnew("key"), createStringObject("value"));
// GET "key"
dictEntry *entry = dictFind(db->dict, sdsnew("key"));
return entry ? entry->v.val : NULL;
```

---

### 总结
Redis 的 K,V 结构用 `dict` 哈希表管理，键用 `SDS`，值用 `redisObject` 封装，支持多类型和高性能。设计兼顾速度、内存和灵活性。面试时，可画 `dict` 结构或提 rehash，展示理解深度。