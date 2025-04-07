
**Caffeine** 是一个高性能的本地缓存库（Java），基于 Java 8 开发，替代 Guava Cache。它通过高效的数据结构和淘汰策略（如 W-TinyLFU），提供快速访问和内存管理。

**优点**：高性能、灵活淘汰策略、线程安全、丰富的配置选项。  
**缺点**：内存占用、本地局限、复杂配置可能增加学习成本。

---

### 1. Caffeine 简介
- **定位**：本地缓存，运行于 JVM 内存。
- **核心功能**：
  - 键值存储。
  - 自动加载（同步/异步）。
  - 淘汰机制（大小、时间、引用）。
- **依赖**：Maven 引入 `com.github.ben-manes.caffeine:caffeine`。

---

### 2. 优点
#### (1) 高性能
- **W-TinyLFU 算法**：
  - 结合 LFU（最少使用）和 LRU（最近最少使用），命中率高。
  - 使用 Window 和 Main 缓存分区，优化热点数据。
- **并发性**：
  - 基于 `ConcurrentHashMap`，读写性能优于 Guava。
- **基准测试**：
  - 比 Guava Cache 快 2-3 倍（读写吞吐量）。

#### (2) 灵活淘汰策略
- **大小淘汰**：限制缓存条目数。
  - `maximumSize(1000)`。
- **时间淘汰**：
  - `expireAfterWrite(10, TimeUnit.MINUTES)`：写入后过期。
  - `expireAfterAccess(5, TimeUnit.MINUTES)`：访问后过期。
- **引用淘汰**：
  - `weakKeys()`、`weakValues()`：弱引用，GC 可回收。

#### (3) 线程安全
- 无需外部同步，支持高并发。
- 示例：
```java
Cache<String, String> cache = Caffeine.newBuilder()
    .maximumSize(100)
    .build();
cache.put("key", "value"); // 线程安全
```

#### (4) 丰富配置
- **自动加载**：
  - `LoadingCache` 同步加载。
  - `AsyncLoadingCache` 异步加载。
- **统计**：
  - `recordStats()` 提供命中率、耗时等指标。
- **监听器**：
  - `removalListener` 监听淘汰事件。

#### 示例
```java
LoadingCache<String, String> cache = Caffeine.newBuilder()
    .maximumSize(100)
    .expireAfterWrite(10, TimeUnit.MINUTES)
    .build(key -> loadValue(key)); // 自动加载
String value = cache.get("key");
```

---

### 3. 缺点
#### (1) 内存占用
- **本地缓存**：数据存 JVM 堆，受限于内存大小。
- **问题**：
  - 大数据量可能引发 OOM（OutOfMemoryError）。
  - 需手动设置 `maximumSize` 控制。

#### (2) 本地局限
- **非分布式**：
  - 仅限单 JVM，无法跨进程或机器共享。
  - 对比：Redis 支持分布式缓存。
- **一致性**：
  - 多实例间数据不同步，需额外机制。

#### (3) 配置复杂性
- **学习成本**：
  - 参数多（如 `expireAfterAccess` vs `expireAfterWrite`），易混淆。
- **调试难度**：
  - 淘汰策略调优需经验，命中率低可能影响性能。

---

### 延伸与面试角度
- **与 Guava Cache 对比**：
  - **Caffeine**：W-TinyLFU 更高效，支持异步。
  - **Guava**：LRU 淘汰，性能稍逊。
- **优化建议**：
  - 小数据集：用 Caffeine。
  - 分布式需求：结合 Redis。
- **实际应用**：
  - Spring Cache 集成：`@Cacheable` 用 Caffeine。
  - 配置缓存：频繁读取的热点数据。
- **面试点**：
  - 问“优点”时，提 W-TinyLFU 和性能。
  - 问“缺点解决”时，提分布式替代。

---

### 总结
Caffeine 优点是高性能、灵活淘汰和线程安全，缺点是内存占用、本地局限和配置复杂。适合单机高频访问场景，面试时可写示例或提淘汰算法，展示理解深度。