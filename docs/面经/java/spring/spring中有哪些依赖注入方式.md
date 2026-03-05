## 1. 依赖注入（DI）在 Spring 里是什么意思

DI（Dependency Injection，依赖注入）指：对象不自己 `new` 依赖，而是把“创建依赖并组装依赖”的职责交给容器。Spring 的 IoC（Inversion of Control，控制反转）是理念，DI 是实现 IoC 的主要手段。

在 Spring 中，一个 Bean 的依赖解析与赋值发生在“实例化之后、初始化之前”的属性填充阶段（`populateBean`）。对开发者而言，注入点可以是构造器、字段、Setter、普通方法参数等。

## 2. Spring 支持的依赖注入方式总览

从“注入点”维度看，常用方式如下：

| 注入方式      | 注入点           | 是否推荐 | 典型场景            |
| --------- | ------------- | ---- | --------------- |
| 构造器注入     | 构造器参数         | 推荐   | 强制依赖、不可变对象、便于测试 |
| Setter 注入 | `setXxx(...)` | 视情况  | 可选依赖、需要后置替换依赖   |
| 字段注入      | 成员字段          | 不推荐  | 仅在极少数简单项目使用     |

## 3. 构造器注入（Constructor Injection）

构造器注入是 Spring 官方长期推荐的方式之一，原因是它能让依赖关系显式化，并且更容易做不可变设计（`final` 字段）。

### 3.1 基本写法（单构造器可省略 `@Autowired`）

```java
import org.springframework.stereotype.Service;

@Service
public class OrderService {
    private final UserRepository userRepository;

    public OrderService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

Spring 对“只有一个构造器”的类会默认使用该构造器进行装配；如果存在多个构造器，通常需要用 `@Autowired` 显式指定，或者通过参数可解析性决定优先级。

### 3.2 优缺点

- 优点：依赖强约束（没注入就无法构造）、便于单元测试、便于发现循环依赖、适合不可变对象。
- 缺点：可选依赖表达不自然（通常配合 `Optional<T>` 或 `ObjectProvider<T>` 解决）。

## 4. Setter 注入（Setter Injection）

Setter 注入通过 `setXxx(...)` 方法完成依赖赋值，适合“可选依赖”或“需要在特定条件下替换实现”的场景。

### 4.1 基本写法

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class ReportService {
    private MetricsClient metricsClient;

    @Autowired(required = false)
    public void setMetricsClient(MetricsClient metricsClient) {
        this.metricsClient = metricsClient;
    }
}
```

### 4.2 什么时候用 Setter 更合适

- 依赖是可选的（没有也能工作）。
- 依赖需要被“延后设置”（例如按条件装配）。
- 需要通过 setter 暴露“可替换点”（例如测试替换）。

## 5. 字段注入（Field Injection）

字段注入通常写起来最省事，但不推荐作为主流方案。它通过反射直接给字段赋值，会带来可测试性与可维护性问题。

### 5.1 基本写法

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class PayService {
    @Autowired
    private PayGateway payGateway;
}
```

### 5.2 不推荐的原因

- 依赖不显式：从构造器签名看不出该类依赖什么，代码阅读与重构成本上升。
- 难以测试：不借助 Spring 容器时需要反射塞字段，或引入额外测试框架。
- 不利于不可变：字段无法声明为 `final`（反射也可以改，但语义更差）。
- 更容易隐藏循环依赖：直到运行期创建 Bean 才暴露问题。
