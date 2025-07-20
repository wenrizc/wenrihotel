
Redis是一个高性能的键值存储系统，它支持多种丰富的数据类型，这些数据类型是Redis强大功能和灵活性的基础。以下是Redis中几种最常见且核心的数据类型：

1.  String（字符串）：
    *   描述：String是Redis中最基本的数据类型，可以存储任何形式的字符串，包括文本、序列化的JSON对象、二进制数据（如图片或视频文件，但通常不建议直接存放大文件）。一个String类型的值最大可以存储512MB。
    *   常见操作：
        *   `SET key value`：设置键的值。
        *   `GET key`：获取键的值。
        *   `INCR key` / `DECR key`：对存储整数的字符串进行原子性的自增/自减1操作。
        *   `INCRBY key increment` / `DECRBY key decrement`：按指定步长自增/自减。
        *   `APPEND key value`：向字符串末尾追加内容。
        *   `STRLEN key`：获取字符串长度。
        *   `MSET key value [key value ...]` / `MGET key [key ...]`：批量设置/获取。
        *   `SETEX key seconds value` / `PSETEX key milliseconds value`：设置带过期时间的键值。
        *   `SETNX key value`：仅当键不存在时设置（常用于实现分布式锁）。
    *   应用场景：缓存（页面缓存、对象缓存）、计数器、分布式锁、存储Session信息、存储简单的配置信息等。

2.  List（列表）：
    *   描述：List是一个双向链表结构（Redis 3.2之后底层实现为Quicklist，是ziplist和linkedlist的混合体），可以存储一个有序的字符串元素集合。可以在列表的两端进行元素的推入（push）和弹出（pop）操作。
    *   常见操作：
        *   `LPUSH key element [element ...]` / `RPUSH key element [element ...]`：从列表左/右端插入一个或多个元素。
        *   `LPOP key` / `RPOP key`：从列表左/右端弹出一个元素。
        *   `LLEN key`：获取列表长度。
        *   `LRANGE key start stop`：获取列表指定范围内的元素。
        *   `LINDEX key index`：通过索引获取列表中的元素。
        *   `LSET key index element`：通过索引设置列表元素的值。
        *   `LREM key count element`：从列表中移除指定数量的匹配元素。
        *   `BLPOP key [key ...] timeout` / `BRPOP key [key ...] timeout`：阻塞式地从列表左/右端弹出一元素（可用于实现消息队列）。
    *   应用场景：消息队列（生产者消费者模式）、任务队列、栈、微博/朋友圈关注列表或时间线、最新N条记录等。

3.  Hash（哈希/字典）：
    *   描述：Hash是一个键值对集合，它是一个字符串字段（field）和字符串值（value）之间的映射表。非常适合存储对象。可以看作是一个小的、内嵌在Redis键中的哈希表。
    *   常见操作：
        *   `HSET key field value [field value ...]`：设置哈希中一个或多个字段的值。
        *   `HGET key field`：获取哈希中指定字段的值。
        *   `HMSET key field value [field value ...]`：同时设置多个字段的值 (在Redis 4.0.0后，HSET可以替代HMSET)。
        *   `HMGET key field [field ...]`：获取多个字段的值。
        *   `HGETALL key`：获取哈希中所有字段和值。
        *   `HDEL key field [field ...]`：删除一个或多个字段。
        *   `HLEN key`：获取哈希中字段的数量。
        *   `HEXISTS key field`：判断字段是否存在。
        *   `HINCRBY key field increment`：对哈希中指定字段的整数值进行原子性自增。
    *   应用场景：存储对象信息（如用户信息、商品信息），其字段对应对象的属性。相比将整个对象序列化为JSON字符串存入String类型，Hash可以单独对对象的某个字段进行读写，更灵活高效。

4.  Set（集合）：
    *   描述：Set是一个无序的、不重复的字符串元素集合。
    *   常见操作：
        *   `SADD key member [member ...]`：向集合添加一个或多个成员。
        *   `SREM key member [member ...]`：从集合移除一个或多个成员。
        *   `SMEMBERS key`：获取集合中的所有成员。
        *   `SISMEMBER key member`：判断成员是否在集合中。
        *   `SCARD key`：获取集合的成员数量。
        *   `SPOP key [count]`：随机移除并返回一个或多个成员。
        *   集合运算：
            *   `SINTER key [key ...]`：求多个集合的交集。
            *   `SUNION key [key ...]`：求多个集合的并集。
            *   `SDIFF key [key ...]`：求多个集合的差集。
            *   `SINTERSTORE destination key [key ...]` 等，将运算结果存储到新集合。
    *   应用场景：标签系统（为一个对象打多个标签）、共同好友/关注、抽奖（随机取成员）、去重统计（如独立IP访问量，但HyperLogLog更适合大数据量去重估算）等。

5.  Sorted Set（有序集合，也称ZSet）：
    *   描述：Sorted Set与Set类似，也是一个不重复的字符串元素集合，但每个成员都会关联一个double类型的分数（score）。Redis通过这个分数来为集合中的成员进行从小到大排序。成员是唯一的，但分数可以重复。
    *   常见操作：
        *   `ZADD key [NX|XX] [CH] [INCR] score member [score member ...]`：向有序集合添加一个或多个成员，或者更新已存在成员的分数。
        *   `ZREM key member [member ...]`：移除一个或多个成员。
        *   `ZRANGE key start stop [WITHSCORES]`：按分数升序返回指定排名范围内的成员。
        *   `ZREVRANGE key start stop [WITHSCORES]`：按分数降序返回指定排名范围内的成员。
        *   `ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count]`：按分数区间返回成员。
        *   `ZREVRANGEBYSCORE key max min [WITHSCORES] [LIMIT offset count]`：按分数区间逆序返回成员。
        *   `ZCARD key`：获取成员数量。
        *   `ZSCORE key member`：获取成员的分数。
        *   `ZRANK key member` / `ZREVRANK key member`：获取成员的升序/降序排名。
        *   `ZINCRBY key increment member`：对成员的分数进行原子性自增。
        *   `ZINTERSTORE destination numkeys key [key ...] [WEIGHTS weight [weight ...]] [AGGREGATE SUM|MIN|MAX]` 等集合运算。
    *   应用场景：排行榜（如游戏得分榜、热门商品榜）、带权重的任务队列、范围查找（如根据时间戳或分数范围获取数据）、社交网络中的Timeline（如微博Feed流，用时间戳作score）等。

除了这些核心数据类型，Redis还提供了一些更专门的数据结构和功能：

*   Bitmaps（位图）：基于String类型实现，可以对字符串的任意位进行操作（置0、置1、统计）。适合用于状态标记、用户签到、布隆过滤器等。
*   HyperLogLog：一种基数估计算法，用于以极小的内存代价估算一个集合中不同元素的数量（不精确但误差可控）。适合大数据量的去重计数，如网站UV统计。
*   Geospatial（地理空间索引）：用于存储地理位置信息（经纬度），并支持基于半径的范围查询、计算两点距离等。
*   Streams（流）：Redis 5.0引入，是一个类似日志的、可追加的、支持消费组的消息队列系统。

Redis之所以强大，很大程度上归功于其对这些数据类型的原生支持和高效的内存操作。开发者可以根据业务需求选择最合适的数据类型来存储和处理数据。

