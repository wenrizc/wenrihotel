
### 答案

#### 是什么

- **StringBuffer**：Java 中的 **线程安全的可变字符串类**，位于 java.lang 包，用于动态构建和修改字符串，适合多线程环境。
- **StringBuilder**：Java 中的 **非线程安全的可变字符串类**，同样位于 java.lang 包（JDK 5 引入），用于单线程环境下的字符串操作，性能更高。

#### 区别

两者功能相同（都支持 append、insert 等操作），主要区别在于 **线程安全性和性能**：

- **StringBuffer**：线程安全，方法加 synchronized，性能稍低。
- **StringBuilder**：非线程安全，无同步开销，性能更高。

### 关键事实详解

#### 1. StringBuffer

- **定义**：可变字符串，内部用字符数组（char[]）存储。
- **线程安全**：方法使用 synchronized 同步。
- **引入**：JDK 1.0。

#### 2. StringBuilder

- **定义**：与 StringBuffer 功能相同，但无同步。
- **线程安全**：无保护，多线程可能数据混乱。
- **引入**：JDK 5.0。

### 具体区别

#### 1. 线程安全性

- **StringBuffer**：
    
    - 方法加锁
    - 多线程安全，但锁开销大。
- **StringBuilder**：
    
    - 无同步
    - 多线程可能覆盖或错乱。

#### 2. 性能

- **StringBuffer**：同步导致性能较低。
- **StringBuilder**：无锁，执行更快。
- **测试**：
    - 循环拼接 10 万次，StringBuilder 比 StringBuffer 快约 20%-50%。

#### 3. 使用场景

- **StringBuffer**：多线程环境，如日志拼接。
- **StringBuilder**：单线程环境，如字符串构建。

#### 4. 底层实现

- **相同点**：
    - 都基于动态字符数组（char[]）。
    - 初始容量 16，扩容规则：新容量 = 旧容量 * 2 + 2。
- **不同点**：仅同步机制差异。

### 延伸与面试角度

- **与 String 对比**：
    - String：不可变，拼接慢（创建新对象）。
    - StringBuffer/StringBuilder：可变，高效。
- **为什么引入 StringBuilder？**：
    - StringBuffer 线程安全开销大，单线程场景浪费性能。
    - JDK 5 优化，新增 StringBuilder。
- **性能数据**：
    - 单线程拼接 10 万次：StringBuilder ~100ms，StringBuffer ~150ms。
- **实际应用**：
    - StringBuffer：多线程日志。
    - StringBuilder：JSON 构建。
- **面试点**：
    - 问“线程安全原理”时，提 synchronized。
    - 问“选择依据”时，提线程环境。

### 总结

StringBuffer 和 StringBuilder 都是可变字符串类，区别在于线程安全（StringBuffer 有，StringBuilder 无）和性能（StringBuilder 更快）。多线程用 StringBuffer，单线程用 StringBuilder。面试时，可写示例或对比性能，展示理解深度。