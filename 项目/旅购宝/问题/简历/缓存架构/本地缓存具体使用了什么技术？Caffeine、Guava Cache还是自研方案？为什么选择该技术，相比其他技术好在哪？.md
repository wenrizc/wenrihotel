
项目中使用的是 **Caffeine** 作为本地缓存实现方案。

Caffeine 是一款高性能的 Java 缓存库，主要有以下优势：

1. **出色的性能表现**：Caffeine 在吞吐量和命中率方面表现极其优秀，远超其他同类产品。根据各种基准测试，Caffeine 的性能比 Guava Cache 和 EhCache 都要高出许多。
    
2. **先进的缓存淘汰算法**：Caffeine 使用基于 Window TinyLFU 的缓存淘汰算法，能够更智能地保留热点数据，提高缓存命中率。
    
3. **灵活的缓存策略**：支持基于容量、基于时间（访问后过期、写入后过期）等多种缓存驱逐策略。
    
4. **统计支持**：内置缓存使用统计功能，可以监控缓存的命中率、加载时间等关键指标。
    
5. **并发安全**：专为高并发环境设计，内部使用高效的并发数据结构。
## 为什么选择 Caffeine

Caffeine 相比其他本地缓存技术（如 Guava Cache、EhCache 等）有以下优点：

1. **卓越的性能**：Caffeine 在吞吐量和命中率方面表现极其出色，远超 Guava Cache 和 EhCache 等其他本地缓存方案。根据作者的基准测试，Caffeine 可以达到 Guava Cache 几倍的性能。

2. **先进的缓存淘汰算法**：Caffeine 使用基于 W-TinyLFU 的算法实现缓存淘汰策略，这比传统的 LRU 算法有更高的命中率和更智能的淘汰机制。

3. **丰富的功能特性**：支持缓存过期策略（基于写入时间、访问时间）、最大容量限制、缓存统计、异步加载等功能。

4. **更小的内存占用**：Caffeine 相比其他缓存框架内存占用更小，在高并发场景下性能更稳定。

5. **更现代的设计**：Caffeine 是 Guava Cache 的精神继承者，由 Guava Cache 的主要贡献者设计，吸取了 Guava Cache 的经验教训。

在项目中，Caffeine 被配置为最大容量 10,000 条目，过期时间为 60 秒，并开启了统计功能，这些配置很好地平衡了性能与内存占用。

## ## 与其他本地缓存方案的对比

### Caffeine vs Guava Cache

1. **性能**：
    
    - Caffeine 的读写性能大约是 Guava Cache 的 2-5 倍
    - 在高并发场景下，Caffeine 的优势更为明显
2. **算法**：
    
    - Guava Cache 使用的是传统的 LRU (Least Recently Used) 算法
    - Caffeine 使用的是 Window TinyLFU 算法，在各种工作负载下有更高的命中率
3. **API 设计**：
    
    - Caffeine 的 API 设计和 Guava Cache 非常相似，迁移成本低
    - Caffeine 是 Guava Cache 的精神继承者，由 Guava Cache 的主要贡献者设计开发

### Caffeine vs EhCache

1. **功能全面性**：
    
    - EhCache 功能更丰富，支持磁盘存储、分布式部署等
    - Caffeine 更专注于内存缓存性能优化
2. **资源占用**：
    
    - Caffeine 更轻量级，内存占用更小
    - EhCache 相对更重，特别是启用了磁盘存储功能时
3. **性能**：
    
    - 在纯内存缓存场景，Caffeine 的性能明显优于 EhCache
    - 对于需要持久化的场景，EhCache 有其优势


在这个项目中，Caffeine 作为本地缓存与 Redis 分布式缓存形成了多级缓存架构，为系统提供了高效的缓存解决方案，能够有效减轻数据库压力并提高响应速度。