## 1. 代理（Proxy）解决什么问题

代理的本质是：**在不改业务代码的前提下，为方法调用“加一层拦截”**，从而实现横切能力，例如：

- 日志、监控、链路追踪；
- 权限校验；
- 事务控制；
- 缓存、重试、限流；
- RPC 客户端桩（stub）与远程调用。

在 Java 生态里，最常见的两类动态代理是：**JDK 动态代理**与 **CGLIB**。

## 2. JDK 动态代理：基于接口的代理

### 2.1 适用前提与特点

- **前提**：目标对象必须有接口（代理的是接口方法）。
- **生成物**：运行时生成一个类，`implements` 目标接口，并 `extends Proxy`。
- **入口**：`Proxy.newProxyInstance(...)` + `InvocationHandler`。

### 2.2 核心调用链

调用代理对象的方法时，最终会进入：

- `InvocationHandler#invoke(Object proxy, Method method, Object[] args)`

你可以在 `invoke` 里做：

- 前置逻辑（鉴权、埋点）；
- 反射调用目标对象：`method.invoke(target, args)`；
- 后置逻辑（提交事务、记录耗时）；
- 异常处理（回滚、转换异常）。

示例：

```java
public interface UserService {
    String getName(long id);
}

public class UserServiceImpl implements UserService {
    public String getName(long id) { return "u" + id; }
}

UserService target = new UserServiceImpl();
UserService proxy = (UserService) java.lang.reflect.Proxy.newProxyInstance(
        target.getClass().getClassLoader(),
        new Class<?>[]{UserService.class},
        (p, method, args) -> {
            long start = System.nanoTime();
            try {
                return method.invoke(target, args);
            } finally {
                System.out.println("cost=" + (System.nanoTime() - start));
            }
        }
);
```

### 2.3 底层原理

- 运行时生成字节码（`ProxyGenerator`），并通过类加载器把字节码 `defineClass` 成真正的 Class。
- 代理类的方法体通常会把调用转发到 `InvocationHandler`，把“横切逻辑入口”统一到一个位置。
- 业务方法如何执行取决于你的 `InvocationHandler`：最常见的写法是 `Method.invoke` 调用真实对象方法，但也可以走 `MethodHandle`、RPC 转发等自定义逻辑。

1. `Proxy.newProxyInstance(loader, interfaces, h)`。
2. 通过缓存拿到或生成代理类 `Class<?> proxyClass`（接口数组 + 类加载器作为 key）。
3. 反射拿到构造器 `proxyClass.getConstructor(InvocationHandler.class)` 并 `newInstance(h)`。
4. 调用代理对象任意接口方法，最终进入 `h.invoke(proxy, method, args)`。

### 2.4 生成出来的代理类长什么样

JDK 代理类的关键特征：

- `extends Proxy`，并 `implements` 你传入的接口列表。
- `Proxy` 父类里持有 `InvocationHandler h`（字段名在 JDK 源码中常见为 `h`）。
- 每个接口方法在代理类中都会生成一个同名方法，方法体只做“参数封装 + 转发”。
- `equals`、`hashCode`、`toString` 通常也会在代理类中生成实现，并同样转发给 `InvocationHandler`，因此很多线上坑最终会落在 `InvocationHandler` 的实现细节上。

可以把代理类的方法体近似理解为：

```java
public final class $Proxy0 extends Proxy implements UserService {
  private static Method m0; // equals
  private static Method m1; // hashCode
  private static Method m2; // toString
  private static Method m3; // UserService#getName

  public $Proxy0(InvocationHandler h) { super(h); }

  public final String getName(long id) {
    return (String) this.h.invoke(this, m3, new Object[]{id});
  }
}
```

### 2.5 缓存与类加载器：为什么会有“类加载器泄漏”风险

JDK 动态代理会对生成的代理类做缓存（避免每次都生成新字节码）。缓存 key 通常包含：

- `ClassLoader`；
- 接口列表（顺序与内容会影响生成结果）。

工程上的结论是：

- 只要你不断用新的 `ClassLoader` 或不断组合出新的接口列表，就可能不断生成新的代理类，带来元空间（Metaspace）压力。
- 典型风险场景：插件化、自研热加载、容器隔离、频繁创建短生命周期类加载器的系统。

### 2.6 默认方法（default method）与特殊方法的处理

接口 `default` 方法也会被 JDK 代理拦截，但“怎么执行 default 方法”取决于你的 `InvocationHandler`。如果你在 `invoke` 内直接 `method.invoke(target, args)`，而目标对象并没有实现该方法，可能会出现不符合预期的行为。

工程上更常见的做法是：把 default 方法当作一种特殊分支，用 `MethodHandles.Lookup` 做 `unreflectSpecial` 调用。

### 2.7 典型限制

- **必须基于接口**：没有接口就没法用 JDK 动态代理（除非你自己先抽接口或改为 CGLIB）。
- **可见性与模块边界**：在 Java 9+ 的模块系统下，某些非公开类型的反射访问会更严格，代理与反射调用要关注 `IllegalAccessException` 等问题。
- **`equals` / `hashCode` 语义**：如果你把 `equals` 也转发给目标对象，可能产生“目标对象与代理对象比较不对称”的问题，需要在 `InvocationHandler` 里明确策略。

## 3. CGLIB：基于继承的代理（生成子类）

### 3.1 适用前提与特点

- **前提**：目标类不能是 `final`，目标方法也不能是 `final`（否则无法覆盖）。
- **生成物**：运行时生成目标类的子类，通过方法重写实现拦截。
- **入口**：`Enhancer` + `MethodInterceptor`（Spring 内部会封装）。

### 3.2 核心调用链

调用代理对象的方法时，会进入：

- `MethodInterceptor#intercept(Object obj, Method method, Object[] args, MethodProxy proxy)`

典型写法会用 `proxy.invokeSuper(obj, args)` 调用父类原方法，避免直接反射调用。

### 3.3 底层原理要点

- CGLIB 基于 ASM 动态生成子类字节码（本质是：生成一个 `extends Target` 的新类，并覆盖可覆盖的方法）。
- 生成出来的子类会持有回调（callback），常见就是 `MethodInterceptor`，在被覆盖的方法中先绑定回调，再把调用转发给 `intercept`。
- `MethodProxy` 用于更高效地调用“父类原方法”（`invokeSuper`），避免每次都走 `Method.invoke` 的反射分派。

### 3.4 生成出来的子类结构

从字节码视角看，CGLIB 代理类通常会额外生成：

- 回调字段，例如 `CGLIB$CALLBACK_0`（具体命名随版本变化）。
- 回调绑定方法，例如 `CGLIB$BIND_CALLBACKS(this)`，用于在首次调用时把回调注入到实例上。
- 对每个可拦截方法的覆盖实现：先确保回调就绪，再进入 `MethodInterceptor#intercept`。

覆盖方法可以近似理解为：

```java
public String getName(long id) {
  MethodInterceptor cb = this.CGLIB$CALLBACK_0;
  if (cb != null) {
    return (String) cb.intercept(this, METHOD_getName, new Object[]{id}, METHODPROXY_getName);
  }
  return super.getName(id);
}
```

### 3.5 `MethodProxy.invokeSuper` 与 FastClass：为什么它通常比反射更快

`MethodProxy` 内部常见优化思路是 FastClass：

- 为目标类和代理类分别生成一个 “FastClass”，把 `Method` 映射到一个 `int` 索引。
- 调用时通过 `switch(index)` 或类似方式直接定位并执行目标方法，减少反射带来的检查与分派开销。

因此你会看到 CGLIB 的典型调用链是：

1. 覆盖方法进入 `MethodInterceptor#intercept`。
2. 业务决定调用原方法时执行 `MethodProxy.invokeSuper`。
3. `invokeSuper` 走 FastClass 的索引分派，调用父类实现。

### 3.6 典型限制与坑

- **`final` 限制**：`final` 类不能代理，`final` 方法不能增强。
- **`private` 方法**：无法被覆盖，因此也无法被拦截（调用路径决定）。
- **构造器与副作用**：生成子类并实例化时会涉及构造器调用。Spring 为了避免构造器副作用，部分场景会用 Objenesis 等方式实例化代理，但前提仍是可生成子类。
- **`equals` / `hashCode`**：覆盖方法的行为与父类实现、拦截器实现有关，仍需要明确语义，否则集合类场景容易出问题。

## 4. JDK 动态代理 vs CGLIB：区别与选型

| 维度 | JDK 动态代理 | CGLIB |
| --- | --- | --- |
| 代理对象 | 接口 | 类（子类） |
| 约束 | 必须有接口 | 类/方法不能是 `final` |
| 调用成本 | 以 `InvocationHandler` 为入口，常见实现里会反射调用目标 | 以 `MethodInterceptor` 为入口，常见实现可走 `invokeSuper` |
| 生成成本 | 生成实现接口的代理类 | 生成子类 + 可能生成 FastClass，启动成本更高 |
| 典型场景 | 接口化良好的服务层 | 无接口类、需要类级代理 |

工程上（例如 Spring）常见策略：

- 目标类实现了接口：优先使用 JDK 动态代理；
- 否则使用 CGLIB（或通过配置强制使用 CGLIB）。

### 4.1 怎么验证“到底生成了什么字节码”

- JDK 动态代理：通过系统属性保存生成的代理类（不同 JDK 版本可能存在属性名差异）。

```shell
# 常见属性名（不同 JDK 版本/实现可能不同）
JAVA_TOOL_OPTIONS="-Djdk.proxy.ProxyGenerator.saveGeneratedFiles=true"
```

- CGLIB：通过调试开关设置 class 输出目录（不同 cglib 版本属性名可能不同）。

```shell
JAVA_TOOL_OPTIONS="-Dcglib.debugLocation=D:\\tmp\\cglib"
```

导出后可以用 `javap -c` 观察方法体是否“纯转发”，以及是否存在 FastClass 相关生成物。

## 5. Spring AOP 相关

### 5.1 自调用导致 @Transactional 失效

同一个类内部方法 A 调用方法 B（B 上有 `@Transactional`），如果是 `this.b()` 这种自调用，通常不会经过代理，因此事务不会生效。

解决方向：

- 把 B 抽到另一个 Bean；
- 或通过注入自身代理（不推荐滥用）；
- 或使用 `AopContext.currentProxy()`（需要开启暴露代理，谨慎）。

### 5.2 final / private 方法无法被代理增强

- CGLIB 无法覆盖 `final` 方法；
- 代理通常也不会拦截 `private` 方法（调用路径与可见性决定）。

### 5.3 Spring 为什么“有接口默认用 JDK 代理”

这是一个工程权衡：

- JDK 代理生成的类结构更简单（实现接口 + 统一转发），对继承层次影响更小。
- CGLIB 需要生成子类，会引入 `final` 限制，并对构造器、`equals`/`hashCode` 等语义更敏感。

对应到 Spring 配置上：

- `spring.aop.proxy-target-class=true`：倾向强制使用 CGLIB（即使有接口）。
- `@EnableAspectJAutoProxy(exposeProxy = true)`：允许通过 `AopContext.currentProxy()` 取当前代理（解决自调用，但会增加理解与维护成本）。

## 6. 动态代理的典型应用场景（怎么用，何时用）

### 6.1 AOP 横切：日志、监控、鉴权、事务、限流

只要满足“在不侵入业务代码的前提下，对方法调用做统一增强”，动态代理都是首选思路之一。

常见落地方式：

- Spring AOP：对 `@Transactional`、`@Cacheable`、自定义 `@Around` 切面做方法级增强。
- 统一埋点：在 `InvocationHandler` / `MethodInterceptor` 里采集耗时、异常类型、QPS 统计。
- 权限校验：根据 `method`、参数、当前用户上下文决定是否放行。
- 限流与重试：对可重试错误做有限重试，对高危接口做限流与熔断。

工程要点：

- 代理更适合做“横切逻辑”，不要在代理里塞复杂业务流程，否则可测试性与可观测性会变差。
- 代理增强对 `final` / `private` 的限制（尤其是 CGLIB）要在设计阶段就规避。

### 6.2 RPC 客户端桩（Stub）：把接口方法映射成远程调用

典型模式是：业务代码只依赖接口，代理把“方法调用”翻译成网络请求（HTTP、TCP、自定义协议），并把响应反序列化成返回值。

最小示例（只展示结构，省略序列化与网络细节）：

```java
import java.lang.reflect.*;
import java.util.*;

public final class RpcStub {
  public interface UserApi {
    String getName(long id);
  }

  public static <T> T create(Class<T> api, String endpoint) {
    Object proxy = Proxy.newProxyInstance(
        api.getClassLoader(),
        new Class<?>[] { api },
        (p, method, args) -> {
          // 横切：统一打点、超时、重试、鉴权签名等都可以放在这里
          Map<String, Object> req = new HashMap<>();
          req.put("method", method.getName());
          req.put("args", args == null ? List.of() : List.of(args));
          req.put("endpoint", endpoint);
          // TODO: send(req) -> resp
          return "mock"; // 示例返回
        }
    );
    return api.cast(proxy);
  }
}
```

对应的真实工程会补齐：

- 超时（connect/read/deadline）与重试边界（次数、退避、抖动）。
- 连接池与限流（per-host 并发、客户端保护）。
- traceId 透传、灰度路由、熔断与降级。

### 6.3 ORM / DAO：用接口表达数据访问，把实现交给框架生成

典型例子是 MyBatis 的 Mapper 接口：你只写 `interface` 与 SQL 映射，框架用 JDK 动态代理生成实现，把方法调用转换为 SQL 执行。

这种场景的收益是：

- 统一入口：所有 DAO 调用都能被拦截做审计、慢查询告警、读写分离路由。
- 实现可替换：同一个接口可以接不同实现（本地、远程、Mock）。

### 6.4 缓存与幂等包装：为接口加“结果复用”与“重复抑制”

当你无法或不想在业务方法内部显式编写缓存/幂等逻辑时，可以用代理在调用边界做包装：

- 缓存：key 通常由 `method` + 参数生成，命中直接返回。
- 幂等：以业务幂等键为 key，重复调用直接返回已完成结果或拒绝执行。

要点：

- 缓存与幂等都必须先定义一致的 key 语义，否则命中率与正确性都会出问题。
- 代理侧不要吞异常，避免把故障隐藏成“缓存 miss”或“业务失败”。

### 6.5 测试与 Mock：用代理快速替换依赖

当依赖以接口形式暴露时，测试可以直接用动态代理构造一个最小实现：

- 返回固定数据、或按入参返回不同结果。
- 记录调用次数与参数，用于断言。

这种方式适合轻量测试；复杂场景仍建议使用成熟的 Mock 框架，并明确线程安全与行为约束。

### 6.6 统一拦截链（Interceptor Chain）：可插拔的增强体系

把 `InvocationHandler` / `MethodInterceptor` 设计成“链式处理”，可以得到类似 Netty Pipeline 的可插拔能力：

- 认证拦截器
- 限流拦截器
- 熔断拦截器
- 监控拦截器
- 兜底降级拦截器

工程要点：

- 拦截器要可配置与可观测：开关、阈值、命中次数、耗时分布。
- 拦截器要有顺序：例如先鉴权再限流，先限流再执行业务。
