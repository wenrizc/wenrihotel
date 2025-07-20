
在Java中，一切反射操作的起点都是获取目标类的Class对象。通过反射动态加载一个类，最核心的方法就是 `Class.forName(String className)`。

当我们调用这个方法时，例如 `Class.forName("com.example.MyService")`，我们就向JVM发出了一个加载指定类的请求。这个方法不仅仅是加载类，它默认还会对类进行链接和初始化。

需要区分的是，获取Class对象的其他方式，如 `MyService.class` 或 `new MyService().getClass()`，它们不完全等同于反射加载。`.class`是在编译期就确定的，而 `obj.getClass()` 是在对象已经存在（意味着类早已被加载）时调用。只有`Class.forName()` 是在运行时根据一个字符串来动态加载和链接一个类，这是反射机制的典型体现。

JVM的类加载过程

调用 `Class.forName()` 后，JVM内部会启动一个标准的类加载流程。这个流程主要分为三个大的阶段：加载（Loading）、链接（Linking）和初始化（Initialization）。

阶段一：加载 (Loading)

这是类加载的第一个阶段。JVM需要完成三件事：

1. 通过类的全限定名（例如 "com.example.MyService"），获取定义这个类的二进制字节流。这个动作是由类加载器（ClassLoader）完成的。
    
2. 将这个字节流所代表的静态存储结构，转换为方法区（在Java 8后是元空间Metaspace）中的运行时数据结构。
    
3. 在内存中生成一个代表这个类的`java.lang.Class`对象，作为方法区这个类的各种数据的访问入口。
    

这个阶段的核心是类加载器（ClassLoader）。`Class.forName()` 实际上是请求当前的类加载器，按照双亲委派模型去找到并加载对应的.class文件。

阶段二：链接 (Linking)

链接阶段是将加载进来的二进制数据组装成JVM可运行状态的过程，它又分为三小步：

1. 验证（Verification）：确保被加载的类文件字节流符合JVM规范，不会危害虚拟机自身的安全。比如检查文件格式、元数据、字节码和符号引用等。
    
2. 准备（Preparation）：为类的静态变量（static variables）分配内存，并设置其初始的“零值”。例如 `int` 类型为0，`Object` 类型为 `null`。注意，此时并不会执行我们代码中写的具体赋值语句。
    
3. 解析（Resolution）：将常量池内的符号引用替换为直接引用（即内存地址）。比如，把对某个方法名的引用，转变为指向方法区中该方法入口的直接指针。
    

阶段三：初始化 (Initialization)

这是类加载过程的最后一步。到了这个阶段，JVM才真正开始执行我们编写的Java代码。

- 主要工作是执行类的构造器方法 `<clinit>()`。这个方法不是我们写的，是Java编译器自动收集类中所有的静态变量的赋值动作和静态代码块（`static{}`块），合并产生的一个方法。
    
- JVM会保证 `<clinit>()` 方法在多线程环境中被正确地加锁、同步。也就是说，一个类的初始化过程是线程安全的，并且只会执行一次。
    

3.反射加载的细节控制

`Class.forName()` 方法有一个重载版本：`Class.forName(String name, boolean initialize, ClassLoader loader)`。

- 这里的 `initialize` 参数非常关键。
    
    - 当我们调用 `Class.forName("com.example.MyService")` 时，它等同于 `Class.forName("com.example.MyService", true, currentClassLoader)`，即默认会执行到“初始化”阶段。
        
    - 如果我们调用 `Class.forName("com.example.MyService", false, currentClassLoader)`，那么类加载过程只会完成“加载”和“链接”阶段，并不会触发类的“初始化”，也就是不会执行静态代码块和静态变量的赋值。
        

这种可以控制是否初始化的能力，在某些框架（如JDBC驱动加载）中非常有用。例如，JDBC规范就曾利用 `Class.forName(driverClassName)` 来自动完成驱动的注册，因为它会触发驱动类的静态代码块。

整个过程可以概括为：

1. 代码触发：我们调用 `Class.forName("类的全限定名")`。
    
2. 委派加载：JVM的当前类加载器启动，并按照双亲委派模型向上委托，最终找到并加载.class文件字节流，在方法区创建数据结构，并生成Class对象。
    
3. 链接：JVM对加载的类进行验证、准备（为静态变量分配内存并设零值）和解析。
    
4. 初始化：JVM执行类的`<clinit>()`方法，运行静态代码块和静态变量赋值语句。
    

最终，`Class.forName()` 返回一个已经完成所有加载、链接和初始化步骤的、功能完备的Class对象，我们就可以用它来进行后续的反射操作，如创建实例、调用方法等。