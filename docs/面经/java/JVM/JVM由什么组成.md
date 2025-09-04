
Java虚拟机（JVM）是Java平台的核心，它使得Java程序能够实现“一次编写，到处运行”的跨平台特性。 JVM本质上是一个虚拟的计算机，拥有自己的指令集和内存管理机制。 当一个Java程序运行时，它实际上是在一个JVM实例中执行的。

JVM主要由以下几个核心部分组成：

*   **类加载器（Class Loader）**
*   **运行时数据区（Runtime Data Area）**
*   **执行引擎（Execution Engine）**
*   **本地方法接口（Native Interface）**

### 类加载器（Class Loader）

类加载器的主要职责是根据类的全限定名查找并加载类的字节码文件（.class文件），然后在内存中创建一个对应的`java.lang.Class`对象。

类加载过程主要包括三个阶段：
*   **加载（Loading）**: 查找并导入.class文件。
*   **链接（Linking）**:
    *   **验证（Verification）**: 确保被加载的类符合JVM规范，没有安全问题。
    *   **准备（Preparation）**: 为类的静态变量分配内存，并设置默认初始值。
    *   **解析（Resolution）**: 将符号引用替换为直接引用。
*   **初始化（Initialization）**: 执行类的构造器`<clinit>()`方法，对静态变量进行初始化。

JVM中的类加载器通常采用**双亲委派模型**，主要包含以下几种：
*   **启动类加载器（Bootstrap Class Loader）**: 负责加载Java核心库（如`JAVA_HOME/lib`目录下的类），由C++实现。
*   **扩展类加载器（Extension Class Loader）**: 负责加载扩展目录（如`JAVA_HOME/jre/lib/ext`）中的类库。
*   **应用程序类加载器（Application Class Loader）**: 也称为系统类加载器，负责加载用户类路径（Classpath）上的类。
*   **用户自定义类加载器（User-Defined Class Loader）**: 允许开发者根据需求自定义类的加载方式。

### 运行时数据区（Runtime Data Area）

运行时数据区是JVM在执行Java程序时用来存储数据的内存区域，它分为多个部分，有些是线程共享的，有些是线程私有的。

#### 线程共享的区域：
*   **方法区（Method Area）**: 用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。 在Java 8及之后版本中，方法区的实现方式变为了元空间（Metaspace），它使用的是本地内存而不是JVM堆内存。
*   **堆（Heap）**: 是JVM所管理的内存中最大的一块，几乎所有的对象实例和数组都在这里分配内存。 堆是垃圾收集器（Garbage Collector）管理的主要区域。

#### 线程私有的区域：
*   **Java虚拟机栈（Java Virtual Machine Stack）**: 每个线程在创建时都会创建一个虚拟机栈。 每当一个方法被调用时，JVM会为其创建一个**栈帧（Stack Frame）**并压入栈中，方法执行完毕后，栈帧出栈。 栈帧中存储了局部变量表、操作数栈、动态链接和方法出口等信息。
*   **本地方法栈（Native Method Stack）**: 与虚拟机栈类似，但它服务于本地（Native）方法，即由C、C++等非Java语言编写的方法。
*   **程序计数器（Program Counter Register）**: 是一块较小的内存空间，可以看作是当前线程所执行的字节码的行号指示器。 它是唯一一个在Java虚拟机规范中没有规定任何`OutOfMemoryError`情况的区域。

### 执行引擎（Execution Engine）

执行引擎负责执行包含在已加载类中的指令。 它读取字节码，并逐条解释或编译执行。

执行引擎主要包含以下部分：
*   **解释器（Interpreter）**: 逐条读取字节码指令，并解释执行。
*   **即时编译器（Just-In-Time, JIT Compiler）**: 为了提高性能，JIT编译器会在运行时将频繁执行的“热点代码”编译成与本地平台相关的机器码，并进行缓存。 HotSpot虚拟机默认采用混合模式，结合了解释执行和即时编译的优点。
*   **垃圾收集器（Garbage Collector, GC）**: 垃圾收集器是JVM内存管理的核心功能，负责自动回收堆内存中不再被引用的对象，以释放内存空间。

### 本地方法接口（Native Interface）

本地方法接口（Java Native Interface, JNI）是一个编程框架，它允许Java代码与其他语言（如C和C++）编写的代码进行交互。 当Java程序需要调用本地方法库时，就会通过JNI来实现。 这为Java程序提供了与底层操作系统和硬件交互的能力。