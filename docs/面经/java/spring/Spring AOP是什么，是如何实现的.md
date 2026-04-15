## 1. Spring AOP 在解决什么问题

Spring AOP（Aspect-Oriented Programming，面向切面编程）用来解决“横切关注点”问题：同一类逻辑（日志、鉴权、事务、监控、限流等）分散在大量业务代码里，导致重复、难维护、易漏改。

它的核心价值是：**把横切逻辑从业务主流程中抽离出来，用统一的位置和规则进行织入**，同时保持业务方法的签名与调用方式不变。

### 1.1 常见横切关注点

- 事务：`@Transactional`。
- 日志与审计：接口耗时、入参脱敏、操作审计。
- 安全：鉴权、数据权限。
- 观测：Metrics、Tracing、告警埋点。
- 稳定性：重试、熔断、限流、幂等保护（通常更推荐在网关或组件层做）。

### 1.2 AOP 与 OOP 的关系

OOP 更擅长按“领域模型/职责”拆分代码；AOP 更擅长按“关注点/规则”统一治理。它们不是替代关系，而是互补：业务用 OOP 表达，横切逻辑用 AOP 织入。

## 2. AOP 术语与 Spring 的取舍

### 2.1 AOP 基本术语对照

| 术语 | 含义 | Spring AOP 中的落点 |
| --- | --- | --- |
| Join Point（连接点） | 可被增强的程序执行点 | **仅支持方法执行**（method execution） |
| Pointcut（切点） | 选出哪些连接点要增强的规则 | `Pointcut`、`@Pointcut`、AspectJ 表达式 |
| Advice（通知） | 在连接点处要执行的增强逻辑 | `MethodInterceptor` 及各类 Advice 适配器 |
| Aspect（切面） | 切点 + 通知的组合 | `@Aspect` Bean，最终会被拆成多个 `Advisor` |
| Weaving（织入） | 把增强应用到目标对象的过程 | **运行期代理织入**（Proxy） |
| Target（目标对象） | 被增强的原始对象 | `TargetSource` 持有的真实对象 |
| Proxy（代理对象） | 对外暴露的增强后对象 | JDK 动态代理或 CGLIB 代理 |

### 2.2 Spring AOP 与 AspectJ 的区别

Spring AOP 的关键取舍是“基于代理”，因此能力边界也很明确：

| 维度 | Spring AOP | AspectJ（编译期/加载期织入） |
| --- | --- | --- |
| 连接点类型 | 方法执行 | 方法、构造器、字段读写等更丰富 |
| 织入时机 | 运行期创建代理 | 编译期（CTW）或类加载期（LTW） |
| 对象范围 | Spring 容器管理的 Bean | 只要类被织入即可 |
| 典型优点 | 简单、侵入性低、与容器生命周期强集成 | 能力强、覆盖面广、可拦截更多场景 |
| 典型限制 | **自调用绕过代理**、非 Bean 无法增强 | 引入织入链路与构建复杂度 |

实际工程里，绝大多数业务场景 Spring AOP 足够；只有在需要更强连接点能力或必须绕开代理限制时，才考虑 AspectJ。

## 3. Spring AOP 的对象模型

Spring AOP 的实现不是直接用“切面”这个概念做运行时匹配，而是把它拆成更偏工程化的数据结构：`Advisor`、`Pointcut`、`Advice`。

### 3.1 `Advisor`、`Pointcut`、`Advice` 的关系

- `Advisor`：**“在哪儿增强 + 怎么增强”** 的组合体。
    - “在哪儿增强”：`Pointcut`（内部包含 `ClassFilter` + `MethodMatcher`）。
    - “怎么增强”：`Advice`（最终会被适配成 `MethodInterceptor`）。
- `@Aspect`：对开发者友好的一层语法糖。一个切面类里可以有多个通知方法，Spring 会把它们拆成多个 `Advisor` 并参与排序与匹配。

### 3.2 核心接口与类

| 类型 | 作用 | 典型实现/相关类 |
| --- | --- | --- |
| `Advice` | 增强逻辑的抽象 | `MethodBeforeAdvice`、`AfterReturningAdvice` 等 |
| `MethodInterceptor` | 统一的环绕拦截模型 | AOP Alliance `MethodInterceptor` |
| `Pointcut` | 类 + 方法匹配规则 | `AspectJExpressionPointcut` |
| `Advisor` | Advice + Pointcut | `PointcutAdvisor`、`DefaultPointcutAdvisor` |
| `AopProxy` | 代理创建与方法分发 | `JdkDynamicAopProxy`、`CglibAopProxy` |
| `MethodInvocation` | 拦截器链调用上下文 | `ReflectiveMethodInvocation` |
| `TargetSource` | 目标对象来源 | `SingletonTargetSource` 等 |

## 4. Spring AOP 的基本形式

Spring AOP 对业务开发者最常见的使用方式，是：

- 定义一个切面类 `@Aspect`
- 用 `@Pointcut` 描述要拦截的方法
- 用不同类型的 Advice 标注增强逻辑

最常见的 5 种通知形式是：

- `@Before`
- `@After`
- `@AfterReturning`
- `@AfterThrowing`
- `@Around`

### 4.1 一个最小业务类

先有一个最小可增强的业务 Bean：

```java
import org.springframework.stereotype.Service;

@Service
public class UserService {

    public String getUser(Long id) {
        if (id == null) {
            throw new IllegalArgumentException("id must not be null");
        }
        return "user-" + id;
    }
}
```

### 4.2 切点：决定“拦谁”

通常先定义一个切点，避免表达式在多个通知里重复写：

```java
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.stereotype.Component;
import org.aspectj.lang.annotation.Aspect;

@Aspect
@Component
public class UserAspectPointcut {

    @Pointcut("execution(* com.example..UserService.*(..))")
    public void userServiceMethods() {}
}
```

这个表达式的含义可以粗略理解为：

- 拦截 `UserService` 里的所有方法执行

### 4.3 `@Before`：方法执行前做增强

`@Before` 适合做：

- 参数日志
- 鉴权前置检查
- 通用校验

```java
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class LogBeforeAspect {

    @Pointcut("execution(* com.example..UserService.*(..))")
    public void userServiceMethods() {}

    @Before("userServiceMethods()")
    public void before(JoinPoint joinPoint) {
        System.out.println("before method: " + joinPoint.getSignature());
    }
}
```

特点：

- 目标方法执行前触发
- 不能决定是否继续执行目标方法

### 4.4 `@AfterReturning`：方法正常返回后做增强

`@AfterReturning` 适合做：

- 返回值日志
- 成功埋点
- 结果审计

```java
import org.aspectj.lang.annotation.AfterReturning;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class LogAfterReturningAspect {

    @Pointcut("execution(* com.example..UserService.*(..))")
    public void userServiceMethods() {}

    @AfterReturning(pointcut = "userServiceMethods()", returning = "result")
    public void afterReturning(Object result) {
        System.out.println("method return: " + result);
    }
}
```

特点：

- 只有在目标方法**正常返回**时才会执行
- 如果方法抛异常，这个通知不会执行

### 4.5 `@AfterThrowing`：方法抛异常后做增强

`@AfterThrowing` 适合做：

- 异常日志
- 告警埋点
- 失败审计

```java
import org.aspectj.lang.annotation.AfterThrowing;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class LogAfterThrowingAspect {

    @Pointcut("execution(* com.example..UserService.*(..))")
    public void userServiceMethods() {}

    @AfterThrowing(pointcut = "userServiceMethods()", throwing = "ex")
    public void afterThrowing(Exception ex) {
        System.out.println("method throw exception: " + ex.getMessage());
    }
}
```

特点：

- 只有在目标方法**抛异常**时才会执行
- 很适合和监控、告警体系结合

### 4.6 `@After`：类似 finally

`@After` 适合做：

- 资源清理
- 统一收尾
- 无论成功失败都要执行的埋点

```java
import org.aspectj.lang.annotation.After;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class LogAfterAspect {

    @Pointcut("execution(* com.example..UserService.*(..))")
    public void userServiceMethods() {}

    @After("userServiceMethods()")
    public void after() {
        System.out.println("after method finish");
    }
}
```

特点：

- 无论目标方法正常返回还是抛异常，都会执行
- 语义上最像 `finally`

### 4.7 `@Around`：最强、最常用的形式

`@Around` 是最灵活的一种通知。

它适合做：

- 耗时统计
- 缓存包装
- 重试包装
- 权限校验后决定是否继续
- 自定义事务边界外层逻辑

```java
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class CostAroundAspect {

    @Pointcut("execution(* com.example..UserService.*(..))")
    public void userServiceMethods() {}

    @Around("userServiceMethods()")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.nanoTime();
        try {
            System.out.println("around before");
            return pjp.proceed(); // 不调用则目标方法不会执行
        } finally {
            long costMicros = (System.nanoTime() - start) / 1_000;
            System.out.println("around after, cost=" + costMicros + "us");
        }
    }
}
```

特点：

- 可以在目标方法前后都做逻辑
- 可以决定要不要调用 `proceed()`
- 可以修改返回值
- 可以捕获并转换异常

所以在工程里，`@Around` 通常是能力最强、使用最广的一种形式。

### 4.8 一张表看 5 种常见通知

| 通知类型 | 触发时机 | 能否拿到返回值 | 能否拿到异常 | 能否阻止目标方法执行 |
| --- | --- | --- | --- | --- |
| `@Before` | 方法执行前 | 否 | 否 | 否 |
| `@AfterReturning` | 方法正常返回后 | 是 | 否 | 否 |
| `@AfterThrowing` | 方法抛异常后 | 否 | 是 | 否 |
| `@After` | 方法结束后（类似 finally） | 否 | 间接 | 否 |
| `@Around` | 方法前后都可控制 | 是 | 是 | 是 |

### 4.9 基本形式在实际项目中的常见落点

- `@Before`
  - 参数打印、鉴权、租户信息校验

- `@AfterReturning`
  - 结果审计、成功埋点

- `@AfterThrowing`
  - 异常日志、监控告警

- `@After`
  - 清理 ThreadLocal、统一收尾

- `@Around`
  - 耗时统计、缓存、限流、重试、分布式锁包装

## 5. Spring AOP 是如何“自动”创建代理的

Spring AOP 的核心机制可以概括为：**用一个 Auto Proxy Creator 充当 `BeanPostProcessor`，在 Bean 初始化后判断是否需要增强，需要则用 `ProxyFactory` 包一层代理**。

### 5.1 启用 AOP：`@EnableAspectJAutoProxy` 注册 Auto Proxy Creator

使用 `@EnableAspectJAutoProxy`（或 Spring Boot 的 AOP 自动配置）后，Spring 会向容器注册 `AnnotationAwareAspectJAutoProxyCreator` 这一类基础设施 Bean。

它是一个 `BeanPostProcessor`（更准确是 `SmartInstantiationAwareBeanPostProcessor`），会在 Bean 创建的关键阶段介入，决定是否对该 Bean 做“包装”。

常用参数：

- `proxyTargetClass`：强制使用 CGLIB（即便存在接口）。
- `exposeProxy`：把当前代理暴露到 `AopContext`，用于少数需要解决自调用的场景。

### 5.2 把 `@Aspect` 变成一组可执行的 `Advisor`

`AnnotationAwareAspectJAutoProxyCreator` 会扫描容器中标注了 `@Aspect` 的 Bean，然后通过工厂把每个通知方法转换为 `Advisor`，其中切点通常会被编译成 `AspectJExpressionPointcut`。

理解到这一步很关键：**Spring 在运行期真正“执行”的不是 `@Aspect`，而是排好序的 `Advisor` 列表**。

### 5.3 包装 Bean：`wrapIfNecessary` 的决策过程（核心算法）

对每个初始化完成的 Bean，Auto Proxy Creator 会做类似下面的判断：

1. 跳过基础设施类：`Advice`、`Advisor`、`AopInfrastructureBean` 等（避免对 AOP 自己再套代理）。
2. 找到所有候选 `Advisor`（包含 `@Aspect` 转换出的，也包含事务、缓存等框架内置的）。
3. 对当前 Bean 的 Class 进行匹配：`Pointcut` 的 `ClassFilter`、`MethodMatcher` 是否命中。
4. 命中则创建代理：把 `Advisor` 列表塞进 `ProxyFactory`（底层是 `AdvisedSupport`），再由 `AopProxy` 生成最终代理对象。

匹配与代理创建都会做缓存，以降低运行期开销。

### 5.4 选择 JDK 代理还是 CGLIB 代理

`DefaultAopProxyFactory` 负责决策代理类型，常见规则可以记成：

- 有接口且未强制 `proxyTargetClass`：优先 JDK 动态代理。
- 没有接口或强制 `proxyTargetClass`：使用 CGLIB 子类代理。

工程建议：优先面向接口注入（JDK 代理友好）；确实需要代理类本身（例如无接口、需要强转实现类）再考虑 CGLIB。

## 6. 方法调用时发生了什么

Spring AOP 在运行期的关键不是“代理能不能创建”，而是“一个方法进来时，如何组装并执行拦截器链”。

### 6.1 从 `Advisor` 到 `MethodInterceptor`：拦截器链如何生成

当调用代理对象的某个方法时，Spring 会为“这个方法”挑出所有命中的 `Advisor`，并把其中的 `Advice` 适配成统一的 `MethodInterceptor` 列表。

在源码层面常见的关键点：

- `DefaultAdvisorChainFactory`：把 `Advisor` 转成拦截器链。
- `AdvisorAdapterRegistry`：把不同类型的 Advice 适配成 Interceptor（例如 Before、AfterReturning、Throws 等）。
- `MethodMatcher#isRuntime()`：如果切点需要基于参数做运行期匹配，会生成动态匹配包装，增加一次运行期判断。

### 6.2 `proceed()` 模型：责任链 + 递归推进

拦截器链的执行模型类似“责任链”：每个 `MethodInterceptor` 都决定“做点事，然后调用 `proceed()` 让下一个拦截器继续”，最后才会真正反射调用目标方法。

可以用极简伪代码抓住本质：

```java
Object proceed() throws Throwable {
    if (index == interceptors.size() - 1) {
        return invokeJoinpoint(); // 反射调用目标方法
    }
    MethodInterceptor interceptor = interceptors.get(++index);
    return interceptor.invoke(this); // 决定是否继续 proceed
}
```

### 6.3 `@Before`、`@Around`、`@After*` 的执行顺序怎么理解

对 `@Aspect` 来说，不同通知类型最终都会变成拦截器，但语义不同：

- `@Around`：最像“手写 try/catch/finally”，它是否调用 `proceed()` 决定了目标方法是否会执行。
- `@Before`：在 `proceed()` 之前执行。
- `@AfterReturning`：`proceed()` 正常返回后执行。
- `@AfterThrowing`：`proceed()` 抛异常时执行。
- `@After`：类似 finally，总会执行。

多个切面时的顺序由 `@Order` / `Ordered` 决定：**数值越小优先级越高，越靠外层包裹**。外层切面的 `@Around` 会先进入，最后退出。

## 7. JDK 动态代理 vs CGLIB：差异与限制

### 7.1 JDK 动态代理的特点

- 只能代理接口：代理类实现目标对象的接口，方法分发由 `InvocationHandler` 完成。
- 不能强转为实现类：`(Impl) proxy` 会失败（除非你本来就是 CGLIB）。
- 适合“面向接口编程”的服务层。

### 7.2 CGLIB 子类代理的特点

- 通过生成目标类的子类并重写方法实现拦截：拦截逻辑由 CGLIB 的 `MethodInterceptor` 完成。
- 对 `final` 类或 `final` 方法无能为力：无法继承或重写就无法织入。
- 对 `static` 方法无效：静态方法不参与动态分派，本质上没有可拦截的“虚方法”。

### 7.3 Spring AOP 的天然边界

无论 JDK 还是 CGLIB，只要是“基于代理”，就逃不开两条硬约束：

- **必须从代理对象调用才能拦截**：类内部 `this.xxx()` 的自调用不会走代理。
- **必须是 Spring 管理的 Bean**：容器之外 `new` 出来的对象没有代理，也就没有增强。

## 8. 常见坑与最佳实践

### 8.1 自调用导致 AOP 不生效（最常见）

```java
import org.springframework.stereotype.Service;

@Service
public class OrderService {
    public void create() {
        this.pay(); // 通过 this 调用，绕过代理，AOP 不生效
    }

    public void pay() {}
}
```

常见解决思路：

- 推荐：拆分方法到另一个 Bean，通过注入的代理对象调用。
- 备选：自注入代理对象（`@Autowired private OrderService self;`）再用 `self.pay()` 调用。
- 谨慎使用：开启 `exposeProxy = true`，用 `AopContext.currentProxy()` 获取代理后再调用（侵入性强）。

### 8.2 代理类型与注入方式不匹配

- JDK 代理场景：优先按接口类型注入与引用。
- 需要按实现类注入或强转：考虑开启 `proxyTargetClass = true` 使用 CGLIB。

### 8.3 多切面顺序与事务边界

`@Transactional` 本身就是基于 AOP 的一组 `Advisor`（核心拦截器是 `TransactionInterceptor`）。当日志、鉴权、缓存等切面与事务叠加时，你需要明确“事务应该包住哪些逻辑”，并用 `@Order` 调整顺序，避免出现：

- 你以为在事务里执行的逻辑，其实在事务外（例如外层切面先捕获并吞掉异常）。
- 你以为一定会回滚，但异常在外层切面被转换或吞掉，导致事务提交。

## 9. 一个耗时统计切面示例

### 9.1 切面定义

```java
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

@Aspect
@Component
@Order(0) // 数值越小越靠外层包裹
public class CostAspect {

    @Around("execution(* com.example..service..*(..))") // 拦截 service 包下所有方法
    public Object cost(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.nanoTime();
        try {
            return pjp.proceed(); // 关键：不调用则目标方法不会执行
        } finally {
            long costMicros = (System.nanoTime() - start) / 1_000;
            // 注意：这里不要做耗时 IO，避免放大延迟
            System.out.println(pjp.getSignature() + \" cost=\" + costMicros + \"us\");
        }
    }
}
```

### 9.2 开启 AOP（非 Boot 项目常见）

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.EnableAspectJAutoProxy;

@Configuration
@EnableAspectJAutoProxy(proxyTargetClass = true, exposeProxy = false)
class AopConfig {}
```

### 9.3 验证是否拦截生效

要验证 AOP 是否生效，务必满足两个条件：

- 调用发生在“另一个 Bean → 代理对象”的路径上，而不是 `this.xxx()` 自调用。
- 目标对象由 Spring 容器管理，而不是 `new` 出来的实例。
