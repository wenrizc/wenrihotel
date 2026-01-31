## 1. 类加载器是什么，解决什么问题

类加载器（`ClassLoader`）的职责是：**把类的字节码（`.class`）加载进 JVM，并把它定义成可用的 `Class` 对象**。

它解决的核心问题包括：

- 类的来源可以多样化：本地文件、网络、加密包、内存生成等。
- 类的隔离与版本并存：同名类在不同加载器下可以“共存”（典型：应用服务器、插件系统）。
- 安全边界：通过委派模型避免核心类被篡改替换。

## 2. 常见类加载器层级（概念口径）

不同 JDK 版本在命名与实现上有所差异，但面试可用以下抽象层级表达：

- **Bootstrap ClassLoader**(启动类加载器)：加载 Java 核心类库（如 `java.lang.*`），通常由 JVM 内部实现（非 Java 代码）。
- **Platform/Extension ClassLoader**(扩展类加载器)：加载平台类库（JDK 自带的一些可选模块/扩展）。
- **Application ClassLoader**(应用程序类加载器)：加载应用 Classpath 下的类（业务代码与依赖）。
- **自定义 ClassLoader**：按需从特定来源加载（插件、隔离、热部署等）。

## 3. 双亲委派与 `loadClass` 关键流程

双亲委派的核心规则是：**先让父加载器尝试加载，父加载器加载不了再由子加载器加载**。

### 3.1 `loadClass` / `findClass` / `defineClass`

- `loadClass(name)`：模板方法，通常包含“检查缓存 → 委派父加载器 → `findClass` → `defineClass`”。
- `findClass(name)`：子类需要实现的“如何找到字节码”的逻辑。
- `defineClass(...)`：把字节码转换成 JVM 内部的类定义（最终产物是 `Class`）。

## 4. 类型隔离：为什么会有 `ClassCastException`

在 JVM 里，一个类的“身份”不是仅由类名决定，而是由以下二元组共同决定：

**类身份 =（全限定名，定义它的 ClassLoader）**

因此，即使两个类的字节码完全相同，只要由不同的加载器定义，它们在 JVM 看来就是两个不同的类型，互相强转就会失败，这也是插件化/容器中常见 `ClassCastException` 的根因。

## 5. 典型工程场景

### 5.1 破坏双亲委派用于隔离

应用服务器（如 Tomcat）往往会对某些类的加载策略做定制，目的是：

- WebApp 之间类隔离。
- 容器提供的类与应用自带类版本并存。

### 5.2 SPI 与线程上下文类加载器（TCCL）

Java SPI（`ServiceLoader`）常通过线程上下文类加载器加载实现类，以便在“接口在父加载器、实现类在子加载器”的场景下仍可工作。

## 6. 自定义类加载器最小示例

```java
import java.nio.file.Files;
import java.nio.file.Path;

public class FileSystemClassLoader extends ClassLoader {
    private final Path baseDir;

    public FileSystemClassLoader(Path baseDir, ClassLoader parent) {
        super(parent);
        this.baseDir = baseDir;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        try {
            String rel = name.replace('.', '/') + \".class\";
            byte[] bytes = Files.readAllBytes(baseDir.resolve(rel));
            return defineClass(name, bytes, 0, bytes.length);
        } catch (Exception e) {
            throw new ClassNotFoundException(name, e);
        }
    }
}
```

