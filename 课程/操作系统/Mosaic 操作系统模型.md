
Mosaic 是一个操作系统模型和状态空间检查器，它通过 Python 实现了操作系统的核心行为，特别关注 "Three Easy Pieces" 中的三个核心概念：并发(Concurrency)、虚拟化(Virtualization)和持久性(Persistence)。

## 核心架构

Mosaic 主要包含以下核心组件：

1. **系统调用接口** - 暴露给应用程序的接口
2. **操作系统模拟器** - `OperatingSystem` 类实现
3. **状态空间探索器** - `Mosaic` 类处理状态空间遍历

## 关键行为实现

### 1. 并发 (Concurrency)

Mosaic 实现了线程和进程管理的基本机制：

- **`sys_fork()`**: 创建一个进程，通过克隆当前线程并复制其堆区实现
  ```python
  @syscall
  def sys_fork(self):
      """Create a clone of the current thread with a copied heap."""
      # ...创建进程的实现，返回进程ID
  ```

- **`sys_spawn()`**: 创建共享堆的线程
  ```python
  @syscall
  def sys_spawn(self, func: Callable, *args):
      """Spawn a heap-sharing new thread executing func(args)."""
      # ...创建线程的实现
  ```

- **`sys_sched()`**: 实现非确定性调度
  ```python
  @syscall
  def sys_sched(self):
      """Return a non-deterministic context switch to a runnable thread."""
      # ...返回所有可能的线程切换选择
  ```

**实现思路**：使用 Python 的生成器(Generator)作为线程上下文，巧妙地利用了生成器可以暂停执行的特性。每个线程由包含上下文和堆的 `Thread` 数据类表示。

### 2. 虚拟化 (Virtualization)

实现了内存和设备的虚拟化：

- **内存虚拟化**：`Heap` 类为每个线程提供独立或共享的内存空间
- **设备虚拟化**：
  - 字符设备：`sys_write()` 将字符串写入标准输出
  - 块设备：`sys_bread()`、`sys_bwrite()`、`sys_sync()` 提供键值对形式的存储访问

### 3. 持久性 (Persistence)

模拟了存储系统的持久性，特别是崩溃一致性：

- **`sys_bread(key)`**: 从块设备读取数据
- **`sys_bwrite(key, value)`**: 向块设备缓冲区写入数据
- **`sys_sync()`**: 将缓冲区数据持久化
- **`sys_crash()`**: 模拟系统崩溃，非确定性地决定哪些写入会被持久化

```python
@syscall
def sys_crash(self):
    """Simulate a system crash that non-deterministically persists
    outstanding writes in the buffer.
    """
    # 通过组合不同的子集实现非确定性的持久化
    # 允许任意子集的写入操作被持久化或丢失
```

**实现思路**：存储分为缓冲区(`buf`)和持久存储(`persist`)，模拟真实系统中的缓存与磁盘关系。

### 4. 状态空间探索

Mosaic 的一个重要特点是能够探索操作系统执行的非确定性：

- **`run()`**: 随机选择一条执行路径
- **`check()`**: 穷尽所有可能的状态空间(BFS 搜索)

## 核心实现思路

1. **系统调用作为生成器的暂停点**：
   - 将应用程序中的系统调用重写为 `yield` 表达式
   - 系统调用时，控制权从应用程序转移到操作系统

   ```python
   # 应用程序视角
   pid = sys_fork()
   
   # 转换后
   res = yield 'sys_fork', []  # 暂停点
   ```

2. **系统调用的延迟执行模式**：
   - 系统调用不立即执行，而是返回可能的选择和对应的处理逻辑
   - 例如，`sys_choose(['A', 'B'])` 返回：
     ```python
     {
         'A': (lambda: 'A'),
         'B': (lambda: 'B')
     }
     ```

3. **操作系统作为状态机**：
   - 每个系统调用可能导致状态转换
   - 状态包括线程、当前运行线程、标准输出和存储状态
   - 通过 `state_dump()` 方法序列化状态

4. **AST 转换**：
   - 使用 `ast` 模块解析和转换应用程序代码
   - `Transformer` 类将系统调用转换为 `yield` 表达式

Mosaic 是一个极简但功能强大的操作系统模型，它巧妙地利用 Python 语言特性来模拟操作系统核心行为，并提供了状态空间探索工具，帮助理解并分析操作系统中的复杂交互和潜在问题。