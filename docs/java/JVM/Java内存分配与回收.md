
## 一、内存分配概述

Java对象的内存分配主要在**堆**上进行，通常在**新生代Eden区**分配，少数情况下可能直接进入**老年代**。分配规则因垃圾收集器组合和参数配置而异。对象可能被JIT编译器拆散为标量类型，间接分配在**栈**上。

---

## 二、内存分配规则

### 1. 对象优先在Eden区分配

- **规则**：大多数对象在新生代Eden区分配。
- **触发机制**：当Eden区空间不足时，虚拟机发起**Minor GC**。
- **特点**：Java对象多为“朝生夕灭”，因此Minor GC频率高，速度快。

**Minor GC vs Major GC/Full GC**：

- **Minor GC**：
    - 回收新生代（Eden + Survivor）。
    - 频率高，速度快。
- **Major GC / Full GC**：
    - 回收老年代，有时伴随Minor GC。
    - 速度通常比Minor GC慢10倍以上。
    - JVM规范中无正式定义，Major GC常指老年代回收，Full GC指整个堆回收。

---

### 2. 大对象直接进入老年代

- **定义**：大对象指需要大量连续内存的Java对象（如长字符串、大数组）。
- **原因**：
    - 大对象存入Eden区概率低，触发分配担保的概率高。
    - 分配担保涉及大量内存复制，效率低。
- **参数**：`-XX:PretenureSizeThreshold` 设置阈值，大于该值的对象直接分配到老年代。
- **目的**：避免Eden和Survivor区之间的频繁内存复制（新生代使用复制算法）。

---

### 3. 长期存活的对象进入老年代

- **机制**：
    - 每个对象有**年龄计数器**。
    - 每次Minor GC后，存活对象年龄+1。
    - 当年龄超过阈值（`-XX:MaxTenuringThreshold`），对象晋升到老年代。
- **参数**：`-XX:MaxTenuringThreshold` 设置新生代对象最大年龄。

---

### 4. 动态对象年龄判定

- **规则**：若Survivor中相同年龄对象大小总和超过Survivor空间一半，年龄≥该年龄的对象直接进入老年代。
- **特点**：无需等待达到`MaxTenuringThreshold`，动态优化晋升。

---

### 5. 空间分配担保

- **目的**：确保Minor GC安全进行，避免新生代对象因老年代空间不足无法晋升。
- **规则变迁**：
    - **JDK 6 Update 24之前**：
        1. 检查老年代最大连续空间是否大于新生代对象总空间：
            - 若大于，Minor GC安全。
            - 若小于，检查`HandlePromotionFailure`是否允许担保失败。
        2. 若允许担保失败，检查老年代最大连续空间是否大于历次晋升对象的平均大小：
            - 若大于，尝试Minor GC（有风险）。
            - 若小于或不允许担保失败，触发Full GC。
    - **JDK 6 Update 24之后**：
        - 只要老年代连续空间大于新生代对象总大小或历次晋升平均大小，进行Minor GC。
        - 否则，触发Full GC。
- **作用**：通过Full GC清理老年代废弃数据，扩大老年代空间，为新生代提供担保。

---

## 三、触发Full GC的场景

1. **`System.gc()`调用**：
    - 建议JVM进行Full GC（非强制）。
    - 可通过`-XX:+DisableExplicitGC`禁用。
2. **老年代空间不足**：
    - 触发Full GC，若仍不足，抛出`java.lang.OutOfMemoryError: Java heap space`。
3. **永久代/方法区空间不足**：
    - HotSpot JVM中方法区称为永久代，存储类信息、常量、静态变量等。
    - 空间不足触发Full GC，若仍不足，抛出`java.lang.OutOfMemoryError: PermGen space`。
4. **CMS GC异常**：
    - **Promotion Failed**：担保失败，Minor GC时对象无法晋升到老年代。
    - **Concurrent Mode Failure**：CMS GC过程中老年代空间不足。
5. **Minor GC晋升平均大小大于老年代剩余空间**：
    - 统计发现晋升对象平均大小超过老年代可用空间，触发Full GC。

---

## 四、总结与优化建议

1. **内存分配**：
    - 对象优先在Eden区分配，减少老年代直接分配。
    - 大对象和长期存活对象通过参数优化（如`PretenureSizeThreshold`和`MaxTenuringThreshold`）。
2. **垃圾回收**：
    - 尽量减少Full GC，优化Minor GC效率。
    - 避免频繁调用`System.gc()`，必要时禁用。
3. **参数调优**：
    - 根据应用场景调整`-XX:MaxTenuringThreshold`和`-XX:PretenureSizeThreshold`。
    - 监控老年代空间使用情况，防止空间不足触发Full GC。
4. **垃圾收集器选择**：
    - 选择合适的垃圾收集器组合（如CMS、G1），结合业务需求优化内存分配与回收。