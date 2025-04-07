
#### 什么是 Java 的包装类
Java 的包装类是基本数据类型的对象封装形式，将 `int`、`double` 等基本类型包装为类（如 `Integer`、`Double`），位于 `java.lang` 包，提供方法和对象特性。

#### 包装类列表
| **基本类型** | **包装类**   |
|--------------|--------------|
| byte         | Byte         |
| short        | Short        |
| int          | Integer      |
| long         | Long         |
| float        | Float        |
| double       | Double       |
| char         | Character    |
| boolean      | Boolean      |

#### 为什么包装成类
基本数据类型是值类型，不具备对象特性（如方法、继承），而 Java 是面向对象语言，许多场景（如集合、泛型）需要对象。将基本类型包装成类是为了：
1. **支持面向对象特性**：提供方法和属性。
2. **适配对象场景**：如集合存储、泛型参数。
3. **类型转换和工具**：便于操作和转换。

---

### 详细解释
#### 1. 支持面向对象特性
- **基本类型**：仅存储值，无方法。
- **包装类**：是对象，继承 `Object`，有方法。
- **示例**：
```java
int i = 10; // 无方法
Integer integer = 10; // 可调用方法
System.out.println(integer.toString()); // "10"
```

#### 2. 适配对象场景
- **集合**：`List`、`Map` 等只能存对象。
- **示例**：
```java
List<int> list = new ArrayList<>(); // 错误
List<Integer> list = new ArrayList<>(); // 正确
list.add(10); // 自动装箱
```
- **泛型**：不支持基本类型。
```java
class Box<T> { T value; } // T 不能是 int，只能是 Integer
```

#### 3. 类型转换和工具
- **转换**：包装类提供方法，如 `parseInt`、`valueOf`。
```java
String s = "123";
int i = Integer.parseInt(s); // 字符串转 int
```
- **工具**：如 `Integer.MAX_VALUE` 获取最大值。
```java
System.out.println(Integer.MAX_VALUE); // 2147483647
```

---

### 自动装箱与拆箱
- **装箱（Boxing）**：基本类型转为包装类。
  - `Integer i = 10;`（隐式 `Integer.valueOf(10)`）。
- **拆箱（Unboxing）**：包装类转为基本类型。
  - `int j = i;`（隐式 `i.intValue()`）。
- **作用**：简化代码，但有性能开销。

#### 示例
```java
Integer a = 5; // 装箱
int b = a;     // 拆箱
```

---

### 优点与缺点
#### 优点
- **功能扩展**：提供方法（如 `toString`、`compareTo`）。
- **兼容性**：适配对象化 API。
- **缓存机制**：如 `Integer.valueOf` 缓存 -128 到 127。

#### 缺点
- **性能开销**：对象比基本类型占内存多，装拆箱耗时。
- **空指针风险**：
```java
Integer x = null;
int y = x; // NullPointerException
```

---

### 延伸与面试角度
- **缓存机制**：
  - `Integer.valueOf(100)` 复用缓存，`new Integer(100)` 不复用。
```java
Integer i1 = 100; // 缓存
Integer i2 = 100; // 相同对象
System.out.println(i1 == i2); // true
```
- **与基本类型对比**：
  - 基本类型：栈存储，高效。
  - 包装类：堆存储，功能强。
- **应用场景**：
  - 集合、反射、序列化。
- **面试点**：
  - 问“为什么用”时，提集合和方法。
  - 问“装箱问题”时，提性能和 NPE。

---

### 总结
Java 包装类将基本类型转为对象，提供方法和对象特性，解决基本类型在面向对象场景的局限。优点是功能强，缺点是开销大。面试时，可写装箱示例或提缓存，展示理解深度。