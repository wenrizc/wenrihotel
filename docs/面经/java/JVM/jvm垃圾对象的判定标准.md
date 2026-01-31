## 1. 判定标准：从 GC Roots 是否可达

在主流 JVM（以 HotSpot 为代表）里，“对象是否是垃圾”的核心判定标准是：**从一组固定的根对象（GC Roots）出发做图遍历，如果对象不可达，则该对象具备被回收的资格**。

这里要注意两个边界概念：

- **不可达 ≠ 立刻被回收**：回收发生在某次 GC 周期中，且可能受收集器策略、分代、停顿预算等影响。
- **可达 ≠ 一定有用**：对象被某条引用链“挂住”，就无法回收；这也是内存泄漏的本质形态（无用但可达）。

### 1.1 “不可达”与“可回收”的区别

不可达意味着“图遍历找不到它”。但对象最终能否被回收，还会受到一些机制影响，例如终结（finalization）与引用队列（`ReferenceQueue`）。

#### 1.1.1 `finalize()` 的影响与限制

`finalize()` 的执行具有以下特征：

- **不保证及时执行**：是否执行、何时执行都由 JVM 决定。
- **最多执行一次**：对象如果在 `finalize()` 中“自救”（让自己重新变为可达），后续再次变为不可达时通常不会再执行 `finalize()`。
- **不建议用于资源释放**：应使用 `try-with-resources`（`AutoCloseable`）或 `java.lang.ref.Cleaner` 等更可靠的机制。

## 2. 可达性分析（Reachability Analysis）

可达性分析可以理解为一次“从根开始的图搜索”：

1. 选取 GC Roots 作为起点集合。
2. 从根出发，沿着对象引用边遍历并标记可达对象。
3. 未被标记的对象即为不可达对象，具备回收资格。

在并发/增量收集器中，为了在应用线程同时运行时维持“可达性图”的一致性，会引入写屏障（Write Barrier）以及 Remembered Set 等结构，但核心思想仍然是“从根出发的可达性判定”。

## 3. 四种引用类型与清理语义

Java 提供了更细粒度的引用语义（`java.lang.ref`），用来描述“对象在内存紧张/GC 时应当如何被清理”：

| 引用类型 | 典型类 | 何时会被清理 | 常见用途 |
| --- | --- | --- | --- |
| 强引用 | 普通引用 | 不会因 GC 主动清理 | 业务对象的正常引用 |
| 软引用 | `SoftReference` | **内存压力较大时**可能清理（策略与收集器相关） | 内存敏感缓存（不推荐作为唯一缓存方案） |
| 弱引用 | `WeakReference` | 下一次 GC 时通常会清理 | `WeakHashMap`、对象关联缓存 |
| 虚引用 | `PhantomReference` | `get()` 永远返回 `null`，用于接收“回收通知” | 配合 `ReferenceQueue` 做堆外资源清理 |

### 3.1 `ReferenceQueue` 的落地模式

`ReferenceQueue` 常用于“在对象即将被回收时得到通知”，从而释放堆外资源或做辅助清理。

```java
import java.lang.ref.PhantomReference;
import java.lang.ref.ReferenceQueue;

class OffHeapResource {
    final long address; // 假设是堆外资源地址
    OffHeapResource(long address) { this.address = address; }
}

public class PhantomCleanupDemo {
    public static void main(String[] args) throws Exception {
        ReferenceQueue<OffHeapResource> queue = new ReferenceQueue<>();
        OffHeapResource res = new OffHeapResource(0x1234);
        PhantomReference<OffHeapResource> ref = new PhantomReference<>(res, queue);

        res = null; // 只保留虚引用
        System.gc();

        // 真实工程中不要 busy loop，可用阻塞/专用线程
        if (queue.remove(1000) == ref) {
            // 在这里释放堆外资源（示意）
            // free(address);
        }
    }
}
```

## 4. 工程实践：从“可达链”视角避免内存泄漏

面试与排障中，“对象为什么回收不了”通常可以用一句话概括：**它仍然被某条 GC Roots 引用链连接着**。常见来源：

- **静态集合/单例缓存**：`static Map`、全局缓存未做淘汰或上限。
- **线程相关引用**：线程池复用导致 `ThreadLocal` 未清理、任务上下文残留。
- **监听器/回调**：注册后不注销（事件总线、观察者模式）。
- **类加载器泄漏**：热部署/插件化场景中，类加载器被线程、静态变量或 JNI 引用挂住。

、