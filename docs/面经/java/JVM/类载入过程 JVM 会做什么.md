
#### 类载入过程
JVM 的类加载过程是指将类的字节码（`.class` 文件）加载到内存并准备使用的过程，主要分为三个阶段：
1. **加载（Loading）**：读取字节码到方法区，生成 `Class` 对象。
2. **链接（Linking）**：
   - 验证（Verification）：检查字节码合法性。
   - 准备（Preparation）：为静态变量分配内存并赋默认值。
   - 解析（Resolution）：将符号引用转为直接引用。
3. **初始化（Initialization）**：执行静态初始化代码（`<clinit>`）。

#### 核心点
- JVM 通过类加载器完成，从加载到初始化，确保类可用。

---

### 1. 详细过程分解
#### (1) 加载（Loading）
- **做什么**：
  - 通过类加载器读取 `.class` 文件的二进制数据。
  - 在方法区创建 `Class` 对象，表示类的元信息（如字段、方法）。
  - 在堆中存储 `Class` 实例，作为反射入口。
- **输入**：
  - 类的全限定名（如 `java.lang.String`）。
- **输出**：
  - 方法区中的类数据结构。
- **类加载器**：
  - **引导类加载器**（Bootstrap）：加载 JDK 核心类（如 `rt.jar`）。
  - **扩展类加载器**（Extension）：加载 `ext` 目录类。
  - **应用类加载器**（Application）：加载用户类路径。
- **双亲委派**：
  - 子加载器委托父加载器，避免重复加载。
- **示例**：
  - `Class.forName("java.util.List")` 触发。

#### (2) 链接（Linking）
##### a. 验证（Verification）
- **做什么**：
  - 检查字节码的正确性和安全性。
- **内容**：
  - 文件格式：魔数（`CAFEBABE`）、版本号。
  - 语义：方法调用、变量访问合法。
  - 安全性：无栈溢出、不访问越界。
- **目的**：
  - 防止恶意或错误代码运行。
- **异常**：
  - `VerifyError`：验证失败。

##### b. 准备（Preparation）
- **做什么**：
  - 为类的静态变量分配内存，赋默认值。
- **规则**：
  - `int`：0。
  - `Object`：`null`。
  - 不包括实例变量（对象创建时处理）。
- **示例**：
```java
class Demo {
    static int x = 10;
    static String s;
}
// 准备阶段：x = 0, s = null
```

##### c. 解析（Resolution）
- **做什么**：
  - 将常量池中的符号引用转为直接引用。
- **内容**：
  - 类/接口引用：转为内存地址。
  - 字段引用：转为偏移量。
  - 方法引用：转为方法表索引。
- **时机**：
  - 可延迟到使用时（懒解析）。
- **示例**：
  - `invokevirtual #2` 解析为具体方法地址。

#### (3) 初始化（Initialization）
- **做什么**：
  - 执行类的静态初始化代码，由 JVM 生成 `<clinit>` 方法。
- **内容**：
  - 静态变量赋值（如 `x = 10`）。
  - 静态代码块（`static {}`）。
- **触发条件**：
  - `new` 创建对象。
  - 调用静态方法/字段。
  - `Class.forName()`。
- **规则**：
  - 父类先初始化（递归）。
  - 单线程执行，线程安全。
- **示例**：
```java
class Demo {
    static int x = 10;
    static { System.out.println("Static block"); }
}
// <clinit>: x = 10, 打印 "Static block"
```

---

### 2. JVM 的具体动作
- **内存分配**：
  - 方法区：存储类元信息（字段、方法表）。
  - 堆：存储 `Class` 对象。
- **数据结构**：
  - **方法表**：存储方法引用。
  - **常量池**：存储字面量和符号引用。
- **线程安全**：
  - 类加载器加锁，避免重复加载。
- **异常处理**：
  - `ClassNotFoundException`：类未找到。
  - `NoClassDefFoundError`：加载失败。

#### 图示
```
.class 文件 --> 加载 (ClassLoader) --> 方法区 (Class 数据)
                --> 链接 (验证/准备/解析) --> 初始化 (<clinit>)
```

---

### 3. 示例分析
```java
class Parent {
    static int p = 1;
    static { System.out.println("Parent init"); }
}

class Child extends Parent {
    static int c = 2;
    static { System.out.println("Child init"); }
}

public class Test {
    public static void main(String[] args) {
        new Child();
    }
}
```
- **过程**：
  1. **加载**：`Parent.class`, `Child.class` 到方法区。
  2. **链接**：
     - 验证：检查字节码。
     - 准备：`Parent.p = 0`, `Child.c = 0`。
     - 解析：符号引用。
  3. **初始化**：
     - `Parent.<clinit>`：`p = 1`, 打印 "Parent init"。
     - `Child.<clinit>`：`c = 2`, 打印 "Child init"。
- **输出**：
```
Parent init
Child init
```

---

### 4. 延伸与面试角度
- **双亲委派**：
  - 防止核心类被覆盖。
  - 可破坏（如 OSGi）。
- **类卸载**：
  - 类加载器和 `Class` 对象无引用时 GC。
- **实际应用**：
  - Spring：动态加载 Bean。
  - Tomcat：多应用隔离。
- **面试点**：
  - 问“过程”时，提三阶段五步骤。
  - 问“初始化”时，提 `<clinit>`。

---

### 总结
JVM 类加载过程包括加载、链接（验证、准备、解析）和初始化，将 `.class` 文件转化为可用的 `Class` 对象。涉及内存分配、字节码校验和静态初始化，确保类安全可用。面试时，可提双亲委派或举初始化例子，展示理解深度。