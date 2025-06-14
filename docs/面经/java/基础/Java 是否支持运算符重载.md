
- **答案**：
  - **Java 不支持运算符重载**（Operator Overloading）。
  - 与 C++、Python 等语言不同，Java 不允许开发者自定义运算符（如 `+`、`-`、`*`）的行为。
- **原因**：
  1. **简化语言设计**：
     - Java 的设计目标是简单性和可读性，运算符重载可能导致代码复杂，难以理解。
     - 示例：C++ 中 `+` 可重载为字符串拼接或向量加法，增加调试难度。
  2. **避免歧义**：
     - 重载运算符可能引发语义不清，Java 选择通过方法名明确功能。
  3. **一致性**：
     - Java 强制通过方法（如 `equals`、`compareTo`）实现类似功能，保持统一性。
- **例外**：
  - Java 内部对某些运算符有**内置支持**，但这不是用户可自定义的重载：
    - **字符串拼接**：`+` 用于 `String` 拼接，编译器将其转换为 `StringBuilder` 操作。
```java
String s = "Hello" + "World"; // 编译为 StringBuilder.append()
```
    - **基本类型运算**：`+`、`-` 等对 `int`、`double` 等有固定实现。
- **替代方案**：
  - 使用方法代替运算符：
    - 比较：`equals()`、`compareTo()`。
    - 算术：自定义方法（如 `Vector.add(Vector)`）。
  - 示例：
```java
public class Vector {
    private double x, y;

    public Vector(double x, double y) {
        this.x = x;
        this.y = y;
    }

    public Vector add(Vector other) {
        return new Vector(this.x + other.x, this.y + other.y);
    }
}
```
- **与内部类的关系**：
  - 内部类或匿名内部类无法直接实现运算符重载，但可通过方法封装类似逻辑（如 `add` 方法）。
  - 匿名内部类常用于实现 `Comparator` 或 `Comparable`，间接支持自定义比较逻辑。

#### 为什么不支持运算符重载
- **James Gosling（Java 之父）观点**：
  - 运算符重载在 C++ 中常被滥用，导致代码难以维护。
  - Java 选择明确的方法名（如 `plus`、`minus`）替代，增强可读性。
- **Kotlin 对比**：
  - Kotlin（基于 JVM）支持运算符重载（如 `operator fun plus`），但 Java 保持简洁设计。
