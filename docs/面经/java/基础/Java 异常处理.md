## 1. 异常体系与分类

### 1.1 异常的本质：可恢复失败与不可恢复失败

在 `Java` 中，异常是**非正常控制流**的显式建模：方法无法按正常路径产出结果时，通过 `throw` 抛出一个 `Throwable` 对象，调用栈沿着动态调用链回退，直到某个 `catch` 匹配并处理，或线程终止并交给默认处理器输出堆栈。

从工程语义上看，异常应区分两类：

- **可恢复失败**：调用方有合理手段处理或降级（例如参数不合法、外部依赖超时、业务校验失败）。
- **不可恢复失败**：进程状态已经不可信或无法继续（例如 `OutOfMemoryError`、`StackOverflowError`）。

### 1.2 继承层级：`Throwable`、`Exception`、`Error`

`Java` 的异常层级以 `java.lang.Throwable` 为根，分为两大分支：

- `java.lang.Exception`：面向应用可处理的异常。
- `java.lang.Error`：面向 `JVM` 或环境级错误，通常不建议捕获后继续运行。

常见子类与定位：

- `RuntimeException`：**非受检异常**（unchecked），通常代表编程错误或不可预期状态（空指针、数组越界、类型转换等）。
- 受检异常（checked）：`Exception` 中除 `RuntimeException` 及其子类以外的异常，例如 `IOException`。

### 1.3 受检异常与非受检异常：编译期约束

受检异常的核心特征是：**编译器要求显式处理**。

- 如果一个方法可能抛出受检异常，则必须：
  - 在方法签名中使用 `throws` 声明，或者
  - 在方法体内通过 `try`-`catch` 捕获并处理。
- 非受检异常（`RuntimeException`）与 `Error` 不强制声明和捕获。

这是一种“接口契约”机制：让调用方在编译期就看到失败路径，但也会带来“异常透传链”与样板代码的成本。

## 2. `throw` 与 `throws`：抛出与声明

### 2.1 `throw`：在运行时抛出一个对象

`throw` 的操作数必须是 `Throwable`（或其子类）实例。抛出时会触发栈回退，直到被捕获或线程终止。

```java
public void setAge(int age) {
    if (age < 0) {
        throw new IllegalArgumentException("age must be >= 0");
    }
}
```

### 2.2 `throws`：方法对外的失败契约

`throws` 只用于方法签名，表达“该方法在某些路径上可能抛出哪些异常”。注意：

- 子类覆写（override）方法时，`throws` 列表只能**更窄**：减少、删除，或替换为更具体的受检异常；不能新增更宽的受检异常。
- 对非受检异常，`throws` 只是文档化，不影响编译期约束。

## 3. `try` / `catch` / `finally`：控制流与语义细节

### 3.1 `catch` 匹配规则：从上到下、按类型兼容

`catch` 会按出现顺序依次尝试匹配抛出的异常对象类型：

- 先写子类，后写父类；否则父类会“吞掉”子类导致编译错误（不可达代码）。
- 匹配规则本质是 `catch (T e)` 中的 `T` 是否是异常对象实际类型的父类型（可赋值）。

```java
try {
    run();
} catch (IllegalArgumentException e) {
    // 先捕获更具体的异常
    handleBadArg(e);
} catch (RuntimeException e) {
    // 再捕获更通用的异常
    handleRuntime(e);
}
```

### 3.2 `finally` 的执行时机：几乎总会执行

`finally` 用于释放资源、清理状态。通常情况下，无论是否抛异常都会执行 `finally`，包括：

- `try` 正常结束；
- `try` 或 `catch` 中 `return`；
- `try` 或 `catch` 中再次 `throw`。

但存在“无法保证执行”的情形，例如：

- 进程被强制终止（如 `Runtime.getRuntime().halt(...)`）；
- `JVM` 崩溃、宿主机断电等不可控因素。

### 3.3 `finally` 覆盖异常：典型坑

如果在 `finally` 中抛出新异常，或在 `finally` 中 `return`，会导致原异常被覆盖，排障信息丢失。

```java
public int f() {
    try {
        throw new IllegalStateException("original");
    } finally {
        return 1; // 注意：会吞掉 original 异常
    }
}
```

工程建议：`finally` 中避免 `return`，避免抛新异常；必要时用“抑制异常”（见 `4.3`）保留上下文。

## 4. `try-with-resources`：资源管理与抑制异常

### 4.1 为什么需要：`finally` 关闭资源的复杂性

用 `finally` 手写关闭资源需要处理多层嵌套与“关闭失败”的异常路径，容易遗漏或吞异常。`try-with-resources` 将“关闭逻辑”内建到语言层面。

### 4.2 关键接口：`AutoCloseable` 与关闭顺序

`try-with-resources` 只能管理实现了 `AutoCloseable` 的资源。资源关闭顺序是**后创建先关闭**（栈式），这对依赖型资源（例如包装流）非常关键。

```java
try (var in = new java.io.FileInputStream(path);
     var out = new java.io.FileOutputStream(dest)) {
    in.transferTo(out);
}
```

### 4.3 抑制异常（suppressed）：保留“关闭失败”的信息

当 `try` 体内抛出异常，同时 `close()` 又抛异常时：

- **主体异常**会被优先抛出；
- `close()` 抛出的异常不会丢失，而是作为**抑制异常**挂在主体异常上，可通过 `Throwable#getSuppressed()` 获取。

```java
try (var r = new java.io.BufferedReader(new java.io.FileReader(path))) {
    throw new java.io.IOException("body");
} catch (java.io.IOException e) {
    for (Throwable s : e.getSuppressed()) {
        // 注意：这里能看到 close() 阶段的异常
        System.err.println("suppressed: " + s);
    }
    throw e;
}
```

## 5. 多重捕获、重新抛出与异常链

### 5.1 多重捕获（multi-catch）：减少重复代码

当多个异常处理逻辑一致时可使用 `|` 合并。注意 `catch (A | B e)` 中的 `e` 在语义上是“不可重新赋值”的。

```java
try {
    parse();
} catch (java.text.ParseException | java.io.IOException e) {
    // 同一处理策略：统一包装后抛出
    throw new IllegalStateException("parse failed", e);
}
```

### 5.2 重新抛出（rethrow）：保留原始类型信息

在 `catch` 中直接 `throw e;` 可能触发编译器的类型推断与“精确重抛”（precise rethrow）规则，使得方法签名的 `throws` 更精确，减少不必要的声明扩散。

实战建议：如果你只是补充日志或做少量清理，优先“原样抛出”而不是新建异常覆盖上下文。

### 5.3 异常链（chaining）：`cause` 的工程意义

异常链用于把底层失败原因上卷到更高抽象层，同时保留根因：

- 构造函数 `new X(message, cause)`；
- `Throwable#initCause(cause)`（通常不推荐后置设置，易造成非法状态）。

工程原则：**跨层包装要提升语义**。例如 DAO 层 `SQLException` 到 service 层可包装成 `DataAccessException`（或自定义），并把原异常作为 `cause`。

## 6. 堆栈信息：`StackTraceElement`、成本与可观测性

### 6.1 堆栈是什么：定位路径而非根因

堆栈跟踪（stack trace）是一系列 `StackTraceElement`，记录抛出点及调用链。它能定位“在哪里发生”，但“为什么发生”仍需要结合输入、状态与日志。

### 6.2 性能要点：异常不应该用于正常分支

创建异常通常需要填充堆栈信息（`fillInStackTrace`），这在高频路径上成本较高。常见优化策略：

- 不用异常表示“正常但无结果”的情况（例如用 `Optional`、返回码、空对象模式）。
- 对真正的异常路径，优先保证语义与可诊断性，而不是过度追求微优化。

## 7. 常见反模式与修正策略

### 7.1 捕获 `Exception` 或 `Throwable`：边界要清晰

除非在“线程边界”或“任务调度边界”（例如线程池执行器最外层）做兜底处理，否则不建议随意捕获过宽类型：

- 容易吞掉 `RuntimeException`，隐藏缺陷；
- 捕获 `Throwable` 可能把 `Error` 也吞掉，导致进程在不可信状态下继续运行。

推荐策略：在业务层捕获“你能处理的那一类异常”，其余让其向上抛到统一入口做日志与告警。

### 7.2 吞异常：`catch` 空块是事故制造机

```java
try {
    call();
} catch (java.io.IOException ignored) {
    // 反例：吞掉异常，后续行为变得不可解释
}
```

修正策略：

- 至少记录日志（含关键上下文）；
- 或转换为业务可理解的异常并返回明确错误码；
- 或明确注释为什么可以忽略，并用指标监控该分支频率。

### 7.3 只打印 `e.getMessage()`：信息不完整

`getMessage()` 常常不足以定位问题，建议至少记录：

- `message`；
- `cause` 链；
- 完整堆栈（日志框架通常支持 `log.error("...", e)`）。

## 8. 自定义异常：建模边界与 API 设计

### 8.1 何时自定义：表达领域语义

自定义异常的目标是让上层“基于语义处理”，而不是基于字符串或底层实现细节：

- `InvalidOrderStateException` 比 `IllegalStateException` 更可读、更可治理。

### 8.2 受检还是非受检：决策建议

- **偏业务语义且必须处理**：可考虑受检异常，但要控制传播链，避免方法签名污染。
- **偏编程契约或不可恢复**：更适合 `RuntimeException`。

工程中常用做法：业务异常使用 `RuntimeException`，配合统一异常处理（例如 `Spring` 的全局异常映射）转换为 HTTP 状态码与错误码。

### 8.3 异常对象的字段：让日志与排障更“结构化”

推荐在异常中携带：

- `errorCode`（稳定且可统计）；
- 关键业务标识（例如 `orderId`）；
- 必要时携带上下文数据，但避免放入大对象或敏感信息。

## 9. 与并发、反射、代理相关的异常形态

### 9.1 线程中的异常：`UncaughtExceptionHandler`

如果异常在某个线程中未被捕获，线程会终止，并交由线程的 `UncaughtExceptionHandler`（或默认处理器）处理。在线程池中，任务异常通常被 `Future` 封装，调用方需要通过 `get()` 触发并观察。

### 9.2 反射与代理：包装异常更常见

反射调用可能把目标方法抛出的异常包装在 `InvocationTargetException` 中；动态代理可能把受检异常包装为 `UndeclaredThrowableException`。排查时要沿 `cause` 链找到根因异常。
