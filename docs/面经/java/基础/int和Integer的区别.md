
#### 区别
- **`int`**：
  - 基本数据类型，存储值，直接占用内存（4 字节）。
- **`Integer`**：
  - 包装类（`java.lang.Integer`），对象类型，包含 `int` 值并提供方法。

#### 核心点
- `int` 是值类型，无 null，效率高。
- `Integer` 是引用类型，可 null，功能丰富。

---

### 1. 区别详解
#### (1) 类型与存储
- **`int`**：
  - 基本类型，固定 32 位（4 字节），范围 -2^31 到 2^31-1。
  - 栈存储，直接值。
- **`Integer`**：
  - 对象类型，封装 `int`，包含一个 `private final int value`。
  - 堆存储，引用指向对象。

#### (2) 默认值
- **`int`**：
  - 默认值 0。
- **`Integer`**：
  - 默认值 `null`（未初始化时）。

#### (3) 使用方式
- **`int`**：
  - 直接运算，如 `int a = 1 + 2;`。
  - 无方法。
- **`Integer`**：
  - 提供方法，如 `Integer.parseInt("123")`。
  - 支持对象操作。

#### (4) 自动装箱/拆箱
- **装箱**：`int` 转 `Integer`（如 `Integer i = 10;`）。
- **拆箱**：`Integer` 转 `int`（如 `int j = i;`）。
- **实现**：编译器自动调用 `Integer.valueOf()` 和 `intValue()`。
- **示例**：
```java
int a = 10;
Integer b = a; // 装箱
int c = b;     // 拆箱
```

#### (5) 内存与性能
- **`int`**：
  - 无对象开销，占用 4 字节。
  - 计算效率高。
- **`Integer`**：
  - 含对象头（约 8-16 字节），总占用更多。
  - 创建和 GC 有开销。

#### (6) 缓存机制（Integer）
- **范围**：`-128` 到 `127`。
- **原理**：`Integer.valueOf()` 使用缓存池。
- **示例**：
```java
Integer x = 127;
Integer y = 127;
System.out.println(x == y); // true（缓存）

Integer m = 128;
Integer n = 128;
System.out.println(m == n); // false（新对象）
```

---

### 2. 示例对比
```java
int a = 10;          // 基本类型
Integer b = 10;      // 包装类
Integer c = null;    // 可为 null
// int d = null;     // 错误，int 不可 null

System.out.println(a + b); // 20，拆箱运算
System.out.println(b.equals(10)); // true，对象方法
// System.out.println(a.equals(10)); // 错误，无方法
```

---

### 3. 特点对比
| **特性**         | **`int`**         | **`Integer`**     |
|------------------|-------------------|-------------------|
| **类型**         | 基本类型          | 引用类型          |
| **存储**         | 栈               | 堆               |
| **默认值**       | 0                | null             |
| **方法**         | 无               | 有（如 parseInt） |
| **内存**         | 4 字节           | 更多（对象开销） |
| **性能**         | 高               | 稍低             |
| **可空**         | 不可             | 可               |

---

### 4. 延伸与面试角度
- **适用场景**：
  - **`int`**：计算密集型，简单变量。
  - **`Integer`**：集合（如 `List<Integer>`）、可空字段。
- **注意事项**：
  - `Integer` 拆箱时小心 NPE：
```java
Integer i = null;
int j = i; // NullPointerException
```
  - `==` vs `equals`：
    - `==` 比较地址（`Integer` 缓存影响）。
    - `equals` 比较值。
- **实际应用**：
  - `int`：循环计数。
  - `Integer`：数据库映射（可 null）。
- **面试点**：
  - 问“区别”时，提装箱和缓存。
  - 问“场景”时，提集合和性能。

---

### 总结
`int` 是高效的基本类型，`Integer` 是功能丰富的对象类型，支持 null 和方法。区别在于类型、存储和用途，自动装箱拆箱连接两者。面试时，可提缓存池或举 NPE 示例，展示理解深度。