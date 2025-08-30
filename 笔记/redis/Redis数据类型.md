
Redis 是一个开源的、基于内存的、使用 C 语言编写的 Key-Value 数据库。它之所以如此流行，不仅仅是因为其高性能，更在于它提供了多种丰富且高效的数据结构，使得开发者可以非常灵活地解决各种问题。

Redis 官方暴露给用户的主要有以下几种核心数据类型：

1. **String (字符串)**
    
2. **List (列表)**
    
3. **Hash (哈希/字典)**
    
4. **Set (集合)**
    
5. **Sorted Set (有序集合 / ZSet)**
    

此外，还有一些基于这些核心类型实现的特殊数据类型：

- **Bitmap (位图)**
    
- **HyperLogLog (基数统计)**
    
- **Geospatial (地理空间)**
    
- **Stream (流)**
    

下面我们逐一详细剖析。

---

### 1. String (字符串)

#### 简介

String 是 Redis 最基本的数据类型，一个 Key 对应一个 Value。它的 Value 不仅可以是普通的字符串，还可以是数字（整数或浮点数），甚至是二进制数据（如图片、序列化后的对象）。一个 String 类型的 Value 最大可以存储 512MB。

#### 底层实现原理

String 的底层实现并非简单的 C 语言字符串（以 \0 结尾），而是自己构建的一种名为 **简单动态字符串 (Simple Dynamic String, SDS)** 的结构。

**SDS 相比 C 字符串的优势：**

1. **O(1) 复杂度获取字符串长度**：直接读取 len 属性，而 C 字符串需要遍历整个字符串。
    
2. **杜绝缓冲区溢出**：当对 SDS 进行修改时，API 会先检查 free 空间是否足够，如果不够，会自动扩容，安全可靠。
    
3. **减少修改字符串时的内存重分配次数**：SDS 采用了**空间预分配**和**惰性空间释放**策略。
    
    - **空间预分配**：当对 SDS 进行扩展时，程序不仅会分配所需空间，还会额外分配一些未用空间（free）。如果修改后 SDS 长度小于 1MB，则分配 len * 2 的空间；如果超过 1MB，则额外分配 1MB 的空间。
        
    - **惰性空间释放**：当对 SDS 进行缩短时，程序不会立即回收多出来的字节，而是更新 free 属性，以备将来使用。
        
4. **二进制安全**：SDS 的 buf 数组可以存储任意二进制数据（包括 \0），因为它使用 len 属性来判断字符串结束，而不是 \0 字符。
    

**编码方式：**  
为了优化存储，String 类型有三种编码方式：

- **int**: 如果一个字符串值可以被解析为 64 位有符号整数，那么 Redis 会将其编码为 long 类型存储，以节省内存并提高 INCR 等操作的效率。
    
- **embstr**: 如果字符串长度小于等于 44 字节（Redis 3.2 版本后），会使用 embstr 编码。它将 RedisObject 对象头和 SDS 连续存放在一起，只需要一次内存分配，效率更高。
    
- **raw**: 如果字符串长度大于 44 字节，则使用 raw 编码。RedisObject 对象头和 SDS 分开分配内存，用指针连接。
    

#### 常用命令

- SET key value: 设置值
    
- GET key: 获取值
    
- MSET key1 value1 key2 value2: 批量设置
    
- MGET key1 key2: 批量获取
    
- INCR key: 将 key 中储存的数字值增一（原子操作）
    
- DECR key: 将 key 中储存的数字值减一（原子操作）
    
- SETEX key seconds value: 设置带过期时间的值
    
- SETNX key value: 当 key 不存在时才设置
    

#### 使用场景

1. **缓存**：最常见的用途。缓存用户信息、商品信息、API 响应、HTML 片段等。
    
2. **计数器**：利用 INCR / DECR 的原子性，实现文章阅读量、点赞数、库存数量等。
    
3. **分布式锁**：利用 SETNX (SET if Not eXists) 命令的特性，当 key 不存在时才能设置成功，可以实现简单的分布式锁。
    
4. **共享 Session**：在分布式系统中，将用户的 Session 信息存储在 Redis 中，实现多台应用服务器间的 Session 共享。
    

---

### 2. List (列表)

#### 简介

List 是一个**有序的、可重复的**字符串集合。它按照元素插入的顺序进行排序，可以在列表的头部（左侧）或尾部（右侧）添加或删除元素。

#### 底层实现原理

在 Redis 3.2 版本之前，List 的底层实现是 ziplist (压缩列表) 和 linkedlist (双向链表) 的结合。

- 当列表元素较少且每个元素都较小时，使用 ziplist 存储，节省内存。
    
- 当列表元素变多或元素变大时，转换为 linkedlist。
    

**Redis 3.2 之后，统一使用 quicklist 作为 List 的底层实现。**  
quicklist 是 ziplist 和 linkedlist 的混合体。它是一个双向链表，但每个链表节点（quicklistNode）都包含一个 ziplist。

**quicklist 的优势：**

- **空间效率**：多个小的 ziplist 串联起来，相比 linkedlist 减少了大量的指针开销，也避免了单个 ziplist 过大时连锁更新的性能问题。
    
- **时间效率**：在两端进行 PUSH 和 POP 操作时，复杂度仍然是 O(1)。在中间插入/删除的效率也优于大的 ziplist。
    

#### 常用命令

- LPUSH key value1 [value2 ...]: 将一个或多个值插入到列表头部
    
- RPUSH key value1 [value2 ...]: 将一个或多个值插入到列表尾部
    
- LPOP key: 移除并获取列表的第一个元素
    
- RPOP key: 移除并获取列表的最后一个元素
    
- LRANGE key start stop: 获取列表指定范围内的元素
    
- LLEN key: 获取列表长度
    
- BLPOP/BRPOP key [key ...] timeout: LPOP/RPOP 的阻塞版本，当列表为空时会阻塞等待，常用于消息队列。
    

#### 使用场景

1. **消息队列/任务队列**：利用 LPUSH + RPOP (或 RPUSH + LPOP) 的模式，可以实现一个简单的先进先出（FIFO）的消息队列。BRPOP 的阻塞特性使得消费者可以高效地等待新任务。
    
2. **时间线 (Timeline) / 最新动态**：例如微博、朋友圈的 Feed流。当用户发布新内容时，LPUSH 到其关注者的列表中。LRANGE 可以分页获取最新的 N 条动态。
    
3. **最新 N 个数据**：例如网站首页展示的“最新登录的 5 位用户”，每次用户登录时 LPUSH，并用 LTRIM 保持列表长度为 5。
    

---

### 3. Hash (哈希)

#### 简介

Hash 是一个 String 类型的 field 和 value 的映射表，特别适合用于存储对象。你可以把它看作是 Map<String, String>。

#### 底层实现原理

Hash 的底层实现也有两种编码方式：

- **ziplist (压缩列表)**：当哈希中键值对数量较少，且每个键和值的大小也较小时，使用 ziplist。它将 field 和 value 紧凑地连续存储，非常节省内存。
    
- **hashtable (哈希表/字典)**：当键值对数量超过配置阈值（hash-max-ziplist-entries，默认 512）或任一键/值大小超过阈值（hash-max-ziplist-value，默认 64 字节）时，会自动转换为 hashtable。hashtable 的结构和 Java 中的 HashMap 类似，通过数组+链表（或红黑树）的方式解决哈希冲突。
    

**hashtable 的 rehash 机制：**  
为了避免在扩容或缩容时因数据迁移导致长时间阻塞，Redis 的 hashtable 采用了**渐进式 rehash**。它会保留新旧两个哈希表，在后续的每次增删改查操作时，除了执行本身的操作，还会顺便将旧哈希表中的一部分数据迁移到新表中，将迁移成本分摊到多次操作中。

#### 常用命令

- HSET key field value: 设置哈希中指定字段的值
    
- HGET key field: 获取哈希中指定字段的值
    
- HMSET key field1 value1 [field2 value2 ...]: 批量设置
    
- HMGET key field1 [field2 ...]: 批量获取
    
- HGETALL key: 获取所有字段和值
    
- HKEYS key: 获取所有字段
    
- HVALS key: 获取所有值
    
- HINCRBY key field increment: 为哈希中指定字段的整数值增加指定增量
    

#### 使用场景

1. **存储对象数据**：用户信息（id, name, email, age）、商品信息等。相比于将每个属性都用一个 String key 存储（如 user:1:name, user:1:email），使用 Hash 可以：
    
    - **节省内存**：当使用 ziplist 编码时，内存占用远小于多个 String key。
        
    - **便于管理**：将同一对象的属性聚合在一个 key 中，逻辑更清晰。
        
2. **购物车**：以用户 ID 为 key，商品 ID 为 field，商品数量为 value，可以方便地对购物车中的商品进行增删改查。HINCRBY 可以方便地实现商品数量的增减。
    
3. **缓存结构化数据**：将关系型数据库中的一行数据或一个 JSON 对象缓存到 Hash 中。
    

---

### 4. Set (集合)

#### 简介

Set 是一个**无序的、不重复的**字符串集合。它的核心价值在于其成员的唯一性以及能够高效地进行集合间的运算。

#### 底层实现原理

Set 的底层实现也有两种编码方式：

- **intset (整数集合)**：当集合中的所有元素都是整数，并且元素数量不超过配置阈值（set-max-intset-entries，默认 512）时，使用 intset。intset 是一个有序的、紧凑的整数数组，查找时使用二分查找，非常高效且节省内存。
    
- **hashtable (哈希表)**：当集合元素不全是整数，或元素数量超过阈值时，转换为 hashtable。hashtable 的 key 存储集合成员，value 则统一为 NULL。
    

#### 常用命令

- SADD key member1 [member2 ...]: 向集合添加一个或多个成员
    
- SREM key member1 [member2 ...]: 移除一个或多个成员
    
- SMEMBERS key: 返回集合中的所有成员
    
- SISMEMBER key member: 判断 member 是否是集合的成员
    
- SCARD key: 获取集合的成员数
    
- SINTER key1 [key2 ...]: 返回给定所有集合的交集
    
- SUNION key1 [key2 ...]: 返回给定所有集合的并集
    
- SDIFF key1 [key2 ...]: 返回给定所有集合的差集
    

#### 使用场景

1. **标签系统 (Tagging)**：一篇文章可以有多个标签，一个标签可以对应多篇文章。可以使用 Set 存储每篇文章的标签，或每个标签下的文章 ID。SINTER 可以方便地找出同时拥有多个标签的文章。
    
2. **共同好友/关注**：使用 SINTER 可以轻松实现“查看你和 Ta 的共同好友”功能。
    
3. **抽奖系统**：将参与抽奖的用户 ID 存入 Set，利用其成员唯一性保证每个用户只能参与一次。SPOP 或 SRANDMEMBER 可以随机抽取中奖用户。
    
4. **去重统计**：例如统计网站的独立访客（UV），将每个访客的 ID SADD 到一个 Set 中，最后通过 SCARD 得到总数。
    

---

### 5. Sorted Set (有序集合 / ZSet)

#### 简介

ZSet 和 Set 一样，也是不重复的字符串集合。但不同的是，ZSet 的每个成员都会关联一个 double 类型的**分数 (score)**。Redis 正是根据这个 score 对集合中的成员进行排序。成员是唯一的，但 score 可以重复。

#### 底层实现原理

ZSet 是 Redis 中最复杂的数据结构，它的底层实现也是两种编码方式：

- **ziplist (压缩列表)**：当 ZSet 的元素数量较少，且每个元素和 score 都较小时，使用 ziplist。在 ziplist 中，元素和 score 成对出现，并按 score 有序排列。
    
- **skiplist (跳表) + hashtable (哈希表)**：当不满足 ziplist 的条件时，会转换为这种组合结构。
    
    - **skiplist**: 一种高效的、支持范围查找的有序数据结构，其查找、插入、删除的平均时间复杂度都是 O(logN)。它负责按 score 排序和范围查询。
        
    - **hashtable**: 负责存储从成员 (member) 到分数 (score) 的映射，使得通过成员名查找其 score 的时间复杂度为 O(1)。
        
    - **精妙之处**：skiplist 和 hashtable 会共享成员和分数的内存，不会造成数据冗余。这两种结构的结合，使得 ZSet 既能高效地按 score 排序和范围查找，也能高效地通过 member 查找 score。
        

#### 常用命令

- ZADD key score1 member1 [score2 member2 ...]: 添加或更新一个或多个成员及其 score
    
- ZREM key member1 [member2 ...]: 移除一个或多个成员
    
- ZRANGE key start stop [WITHSCORES]: 按 score 从小到大返回指定排名范围的成员
    
- ZREVRANGE key start stop [WITHSCORES]: 按 score 从大到小返回指定排名范围的- ZSCORE key member: 获取指定成员的 score
    
- ZRANK key member: 获取指定成员的排名（从小到大）
    
- ZRANGEBYSCORE key min max [WITHSCORES]: 按 score 范围查询成员
    

#### 使用场景

1. **排行榜/积分榜**：最经典的应用。例如游戏玩家积分榜、热门商品榜。ZADD 更新分数，ZREVRANGE 获取 Top N 列表。
    
2. **带权重的消息队列**：如果消息需要有优先级，可以将优先级作为 score，高优先级的消息先被处理。
    
3. **延迟任务队列**：将任务的执行时间戳作为 score，然后用一个定时任务轮询 ZRANGEBYSCORE 获取当前需要执行的任务。
    
4. **范围查找**：例如，查找工资在 5000 到 10000 之间的员工。
    

---

### 其他特殊数据类型

#### 6. Bitmap (位图)

- **本质**：它不是一个独立的数据类型，而是对 String 类型的一种扩展。可以看作是一个以 bit 为单位的数组，每个 bit 只能是 0 或 1。
    
- **原理**：底层就是 String (SDS)，通过 SETBIT 和 GETBIT 等命令在字节级别上操作位。
    
- **场景**：
    
    - **用户签到统计**：用一个 Bitmap 记录一个用户一年的签到情况，365 天只需要 365/8 ≈ 46 字节，非常节省空间。
        
    - **活跃用户统计**：BITCOUNT 可以快速统计指定时间段内活跃用户数量。
        
    - **用户在线状态**：用一位表示一个用户是否在线。
        

#### 7. HyperLogLog

- **本质**：一种概率性数据结构，用于进行**基数统计**（即统计集合中不重复元素的数量）。
    
- **原理**：它不是精确计数，而是给出一个带有很小标准误差（约 0.81%）的近似值。其惊人之处在于，无论统计多少个元素，它占用的内存都非常小且固定（约 12KB）。
    
- **场景**：
    
    - **网站 UV 统计**：统计一个页面的每日独立访客数，相比 Set，HyperLogLog 在数据量巨大时能节省大量内存。
        
    - **大规模数据去重计数**。
        

#### 8. Geospatial (地理空间)

- **本质**：基于 ZSet 实现，用于存储地理位置信息（经度、纬度）。
    
- **原理**：它使用 **Geohash** 算法将二维的经纬度坐标编码成一个一维的字符串（或整数），然后将这个编码值作为 ZSet 的 score，地点名称作为 member。利用 ZSet 的有序性和范围查询能力，可以实现地理位置的邻近查询。
    
- **场景**：
    
    - **附近的人/地点**：如“查找我附近 1 公里内的餐厅”。
        
    - **计算两点间距离**。
        

#### 9. Stream (流)

- **本质**：Redis 5.0 新增的数据结构，是一个**支持多播的、可持久化的、追加式的日志 (append-only log)**。
    
- **原理**：结构上类似一个简版的 Kafka。它包含消息 ID、消息内容，并支持**消费组 (Consumer Group)**，允许多个消费者协同消费同一个流中的消息，并能记录每个消费者的消费进度。
    
- **场景**：
    
    - **消息队列**：相比 List 实现的消息队列，Stream 功能更强大，支持消费组、消息持久化和 ACK 机制，更像一个专业的 MQ。
        
    - **事件溯源 (Event Sourcing)**。
        
    - **实时数据流处理**。
        

### 总结

|   |   |   |   |
|---|---|---|---|
|数据类型|底层实现|核心特点|主要使用场景|
|**String**|SDS (int, embstr, raw)|键值对，二进制安全|缓存、计数器、分布式锁、共享Session|
|**List**|quicklist (ziplist + linkedlist)|有序、可重复|消息队列、时间线、最新列表|
|**Hash**|ziplist / hashtable|field-value 映射，对象存储|存储对象（用户信息）、购物车|
|**Set**|intset / hashtable|无序、唯一|标签、共同好友、抽奖、去重统计|
|**ZSet**|ziplist / (skiplist + hashtable)|有序、唯一、带分数|排行榜、延迟任务、范围查找|
|**Bitmap**|String (SDS)|位操作，节省空间|签到、活跃用户统计、在线状态|
|**HyperLogLog**|HLL 算法|概率性基数统计，极省内存|海量数据去重计数（如UV）|
|**Geospatial**|ZSet (Geohash)|地理位置存储与查询|附近的人/地点|
|**Stream**|类 Radix Tree|追加式日志，支持消费组|消息队列、事件溯源、日志系统|

