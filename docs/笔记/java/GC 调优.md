
## 如何判断 GC 是否存在问题

### 评价核心指标

1.  延迟 (Latency): 指 GC 过程中 Stop-The-World (STW) 的最大停顿时间。目标是 STW 时间越短越好，通常需要低于服务的 TP9999 耗时。
2.  吞吐量 (Throughput): 指 `(1 - GC 时间 / 程序总运行时间) * 100%`。目标是吞吐量越高越好，通常要求不低于 99.99%。

### 判断问题的根因

当出现服务 RT 上涨、CPU 负载高、线程 Block 等现象时，需判断 GC 是问题的“因”还是“果”。
*   时序分析: 最先出现的异常指标，更有可能是根因。
*   概率分析: 结合历史经验，从最常出问题的模块开始排查。
*   实验分析: 在测试环境模拟单一变量（如高 CPU），观察是否能复现问题。
*   反证分析: 如果集群中某些节点无慢查询、CPU 正常但仍有问题，则可将嫌疑转向 GC。

### 读懂 GC Cause

通过 GC 日志中的 `GCCause` 可以了解触发 GC 的原因，常见的有：
*   `Allocation Failure`: 在年轻代中没有足够空间分配新对象时触发。
*   `System.gc()`: 手动调用 `System.gc()` 触发。
*   `CMS Initial Mark` / `CMS Final Remark`: CMS GC 过程中的 STW 阶段。
*   `Concurrent Mode Failure`: 在 CMS 并发执行期间，有新对象需要晋升到老年代，但老年代空间不足，导致并发失败，退化为一次 STW 时间很长的 Full GC。
*   `Promotion Failed`: 在进行 Minor GC 时，Survivor 区无法容纳存活对象，且老年代也无法容纳这些对象（可能是空间不足或碎片过多）。
*   `GCLocker Initiated GC`: JNI 调用期间，为保证数据一致性而阻止了 GC。当 JNI 调用结束后，触发一次 GC。

---

## 常见 CMS GC 问题场景分析与解决

### 场景一：动态扩容引起的空间震荡

*   现象: 服务启动初期 GC 频繁，即使堆内存最大值 (`-Xmx`) 很大，但仍会触发 GC。GC 日志显示堆空间在 GC 后被调整大小。
*   原因: `-Xms` (初始堆大小) 和 `-Xmx` (最大堆大小) 设置不一致。JVM 在空间不足时会扩容堆，这会触发 GC；空间空闲较多时会缩容堆，同样可能触发 GC。
*   解决策略: 将 `-Xms` 和 `-Xmx` 的值设置为相同，同样地，将 `-XX:NewSize` 和 `-XX:MaxNewSize`，以及 `-XX:MetaSpaceSize` 和 `-XX:MaxMetaSpaceSize` 设置为一致，以获得一个稳定的堆和元空间，避免运行时动态伸缩带来的开-销。

### 场景二：显式 GC 的去与留 (`System.gc()`)

*   现象: 在未达到其他 GC 触发条件时，发生了 GC，GC Cause 为 `System.gc()`。
*   原因:
    *   `System.gc()` 默认会触发一次 STW 时间非常长的 Full GC (采用 Mark-Sweep-Compact 算法)。
    *   禁用它 (`-XX:+DisableExplicitGC`) 看起来很美好，但会带来新问题：NIO 中使用的堆外内存 (DirectByteBuffer) 依赖 `System.gc()` 来通知 JVM 回收其关联的 PhantomReference，从而释放堆外内存。如果禁用，可能导致堆外内存泄漏。
*   解决策略: 不禁用 `System.gc()`。而是使用 `-XX:+ExplicitGCInvokesConcurrent` 参数，将 `System.gc()` 的行为从一次 STW 的 Full GC 转变为一次并发的 CMS GC，大大减少停顿时间，同时保证了堆外内存的正常回收。

### 场景三：MetaSpace 区 OOM

*   现象: MetaSpace 使用率持续增长，即使 GC 也无法有效回收，最终导致 OOM。
*   原因: 元空间的回收条件非常苛刻，只有当其关联的 ClassLoader 被卸载时，对应的类元数据才会被回收。问题通常由动态类加载技术（如 CGLIB、Groovy、Orika）引起，这些技术会不断创建新的类，但其 ClassLoader 却一直存活，导致元数据无法卸载。
*   解决策略:
    1.  使用 `jcmd <pid> GC.class_stats` 命令，多次执行并比对输出，定位是哪个包下的类在持续增长。
    2.  dump 内存快照，使用 MAT 等工具分析，查找 ClassLoader 的实例和它们加载的类数量。
    3.  从业务代码层面解决，例如缓存动态生成的类，避免重复创建。

### 场景四：过早晋升 (Premature Promotion)

*   现象:
    *   Young GC 频繁，但 Old GC 后老年代使用率大幅下降（例如从 80% 降到 10%），说明晋升到老年代的对象大多是“短命”的。
    *   对象的晋升年龄很小（例如 GC 日志中 `new threshold 1`）。
*   原因:
    *   年轻代空间过小: Eden 区很快被填满，导致大量本应在年轻代就被回收的对象，不得不提前晋升到老年代。
    *   分配速率过大: 瞬间的大流量或批处理任务导致内存分配速率激增。
    *   动态年龄判断: 当 Survivor 区中同一年龄的对象总大小超过 Survivor 空间的一半（由 `-XX:TargetSurvivorRatio` 控制）时，JVM 会动态降低晋升年龄门槛，导致大量对象提前晋升。
*   解决策略:
    *   在总堆内存不变的前提下，适当增大年轻代空间。一个经验法则是，老年代大小应为活跃对象大小的 2-3 倍，其余空间可分配给年轻代。使用 `-Xmn` 参数或调整 `-XX:NewRatio`。
    *   如果分配速率过大，需从业务层面进行优化，例如处理流量尖峰或优化数据加载逻辑。

### 场景五：CMS Old GC 频繁

*   现象: CMS GC 频繁发生，虽然单次 STW 时间不长，但总体上拉低了服务吞吐量。
*   原因:
    *   老年代的使用率达到了触发 CMS GC 的阈值 (`-XX:CMSInitiatingOccupancyFraction`，默认为 92%)。
    *   这通常是由于存在“中等生命周期”的内存泄漏，例如缓存、数据库连接等对象，它们存活时间比一次 Young GC 长，但又不是永久存活，不断累积在老年代。
*   解决策略: 这是典型的内存泄漏排查场景。
    1.  在 CMS GC 前后分别 dump 内存快照。
    2.  使用 MAT 等工具进行对比分析（Diff），或直接分析单个快照的支配树（Dominator Tree）和泄漏嫌疑（Leak Suspects）。
    3.  重点关注 Unreachable Objects，这些对象即将被回收，可以反推出哪些是“短命”的。

### 场景六：单次 CMS Old GC 耗时长

*   现象: CMS GC 的 STW 阶段（特别是 Final Remark）耗时过长，可能达到数秒。
*   原因: Final Remark 阶段耗时长的主要原因有两个：
    1.  处理大量的引用 (Reference): 特别是 `FinalReference`。当一个类实现了 `finalize()` 方法，其对象在被回收前需要被放入 `Finalizer` 的队列中，由一个低优先级的线程去执行 `finalize()` 方法。如果这个队列堆积了大量对象，处理过程会很慢。
    2.  类卸载和符号表/字符串表清理: `scrub symbol table` 耗时过长。如果启用了 `-XX:+CMSClassUnloadingEnabled`（JDK 8 默认开启），CMS 会在 Remark 阶段尝试卸载不再使用的类，这个过程可能很耗时。
*   解决策略:
    *   开启 `-XX:+PrintReferenceGC` 查看各类引用的处理耗时。
    *   对于 `FinalReference` 问题，dump 内存并分析 `java.lang.ref.Finalizer` 对象的支配树，找到是哪些对象在滥用 `finalize()` 方法并优化代码。可以开启 `-XX:+ParallelRefProcEnabled` 并行处理引用以缓解问题。
    *   对于类卸载问题，如果确认元空间使用稳定，没有动态类加载，可以考虑使用 `-XX:-CMSClassUnloadingEnabled` 关闭类卸载来避免这部分开销。

### 场景七：内存碎片与收集器退化

*   现象: GC 日志中出现 `Concurrent Mode Failure` 或 `promotion failed`，随后发生了一次 STW 时间极长的 Full GC。
*   原因:
    *   CMS 使用“标记-清除”算法，会产生内存碎片。当 Young GC 后需要晋升一个大对象到老年代时，可能因为找不到一块足够大的连续内存而导致 `promotion failed`，进而退化为 Full GC。
    *   `Concurrent Mode Failure` 是因为 CMS 在并发回收过程中，业务线程仍在运行并产生新的对象（浮动垃圾）。如果预留的老年代空间不足以容纳这些新对象，就会导致并发失败，退化为 Full GC。
*   解决策略:
    *   针对内存碎片: 使用 `-XX:UseCMSCompactAtFullCollection` (默认开启) 让 Full GC 时进行碎片整理。使用 `-XX:CMSFullGCsBeforeCompaction=N` 控制每 N 次不带压缩的 Full GC 后，执行一次带压缩的。
    *   针对并发失败: 适当调低 CMS 的触发阈值 `-XX:CMSInitiatingOccupancyFraction`（例如 75% 或 80%），让 GC 更早地启动，以预留更多空间。需要配合 `-XX:+UseCMSInitiatingOccupancyOnly` 使用。
    *   使用 `-XX:+CMSScavengeBeforeRemark` 在 Final Remark 之前强制进行一次 Young GC，以减少 Remark 阶段需要扫描的新增对象。

### 场景八：堆外内存 OOM

*   现象: Java 进程的实际物理内存占用（RES）远超 `-Xmx` 设置，GC 时间飙升，服务卡顿。
*   原因: 未能正确释放由 `Unsafe.allocateMemory` 或 `DirectByteBuffer` 分配的堆外内存，或者 JNI 调用中 C 代码分配的内存未释放。
*   解决策略:
    1.  使用 `-XX:NativeMemoryTracking=detail` 启动 JVM，然后用 `jcmd <pid> VM.native_memory detail` 查看本地内存分布。
    2.  如果 Committed 值和 RES 接近，说明是 `DirectByteBuffer` 等 Java 代码申请的内存泄漏。检查是否禁用了 `System.gc`，或相关资源是否被正确关闭。
    3.  如果相差巨大，说明是 JNI 调用的 Native Code 泄漏。可使用 `gperftools` 等外部工具来追踪 `malloc` 调用栈。

### 场景九：JNI 引发的 GC 问题

*   现象: GC 日志中 GC Cause 为 `GCLocker Initiated GC`。
*   原因: JNI 提供了 `GetPrimitiveArrayCritical` 这样的方法，可以让 Native Code 直接访问 JVM 堆内数组的指针，性能很高。在此期间，JVM 会加一个锁（JNI Critical Region），禁止 GC 发生以保证数据安全。当所有持有该锁的线程都释放后，会触发一次 GC。
*   解决策略:
    *   JNI 调用应非常谨慎，其性能提升可能被 GC 问题所抵消。
    *   添加 `-XX:+PrintJNIGCStalls` 参数，可以打印出发生 JNI 调用时阻塞 GC 的线程信息，帮助定位问题代码。
    *   升级 JDK 版本可以规避一些已知的 GCLocker 相关的 Bug。

## 总结与调优建议

### 处理流程 (SOP)

1.  制定标准: 结合服务 SLA，为延迟和吞吐量设定明确的、可量化的 GC 监控指标。
2.  保留现场: 出现问题时，优先通过摘流来恢复服务，而不是立即重启，以便保留堆转储（Heap Dump）、线程转储（Thread Dump）、GC 日志等关键信息。
3.  因果分析: 使用时序、概率等方法，判断 GC 是“因”还是“果”。
4.  根因分析: 确认为 GC 问题后，借助工具和上述场景进行匹配，定位到具体原因。
5.  实施优化: 根据根因选择合适的策略进行调优。
6.  复盘总结: 问题解决后进行复盘，将经验固化为团队知识。

### 核心调优建议

*   不要过早优化: 大部分性能问题源于业务代码，而非 JVM 参数。GC 调优应在应用层面优化无效后，作为最终手段。
*   控制变量法: 每次调优只修改一个参数，以便清晰地观察其效果。
*   善用工具和搜索: 绝大多数问题都有前人遇到过。熟练使用 MAT, JProfiler, Arthas 等工具，并善用搜索引擎。
*   重要的 GC 参数建议:
    *   `-XX:+HeapDumpOnOutOfMemoryError`: 在 OOM 时自动 dump 堆内存。
    *   `-XX:HeapDumpPath=<path>`: 指定 dump 文件的路径。
    *   `-XX:+PrintGCDetails -XX:+PrintGCDateStamps`: 打印详细的 GC 日志并带上时间戳。
    *   `-Xloggc:<file>`: 将 GC 日志输出到文件。