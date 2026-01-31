## 1. “GC 类型”常见两种问法

面试里说的 “GC 类型”通常有两层含义：

- 回收事件类型：Young GC / Full GC / Mixed GC 等（更偏现象与日志）。
- 垃圾收集器类型：Serial、Parallel、G1、ZGC 等（更偏实现与选型）。


## 2. 经典收集器（吞吐优先）

### 2.1 Serial GC（串行）

- 单线程回收，Stop-The-World（STW）。
- 适合小堆、单核或对停顿不敏感场景（例如简单工具、单机任务）。

启用：

```bash
-XX:+UseSerialGC
```

### 2.2 Parallel GC（并行，吞吐优先）

- 多线程并行回收，目标是提高吞吐。
- 常见搭配是新生代 Parallel Scavenge + 老年代 Parallel Old（具体实现取决于 JVM）。

启用：

```bash
-XX:+UseParallelGC
```

## 3. 低停顿收集器（并发/区域化）

### 3.1 G1（Garbage-First）

- 把堆划分为多个 Region，按收益优先回收“垃圾最多”的 Region。
- 支持并发标记与增量整理，目标是提供可预测的停顿时间。

启用：

```bash
-XX:+UseG1GC
```

### 3.2 ZGC（低延迟）

- 以低停顿为主要目标，通过并发标记/转移与屏障技术减少 STW。
- 更适合大堆、对尾延迟敏感的场景（具体效果依赖版本与参数）。

启用：

```bash
-XX:+UseZGC
```

### 3.3 Shenandoah（低延迟，部分发行版提供）

Shenandoah 与 ZGC 目标类似，强调并发整理以降低停顿。是否可用与具体 JDK 发行版、版本与参数有关。

## 4. 历史收集器：CMS（已逐步退出主流）

CMS（Concurrent Mark Sweep）以低停顿为目标，但基于标记-清除容易产生碎片，并且在较新的 HotSpot 中已被标记为过时并逐步移除。面试回答中更建议把 CMS 作为“历史方案”提及，并说明为何被 G1 等替代。

## 5. 特殊用途：Epsilon（不回收）

Epsilon 是 no-op GC，只分配不回收，主要用于：

- 性能基线测试（排除 GC 干扰）
- 研究与调试场景
