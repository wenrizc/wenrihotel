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

### 3.5 `MethodProxy.invokeSuper` 与 FastClass：为什么它通常比反射更快

`MethodProxy` 内部常见优化思路是 FastClass：

- 为目标类和代理类分别生成一个 “FastClass”，把 `Method` 映射到一个 `int` 索引。
- 调用时通过 `switch(index)` 或类似方式直接定位并执行目标方法，减少反射带来的检查与分派开销。

因此你会看到 CGLIB 的典型调用链是：

1. 覆盖方法进入 `MethodInterceptor#intercept`。
2. 业务决定调用原方法时执行 `MethodProxy.invokeSuper`。
3. `invokeSuper` 走 FastClass 的索引分派，调用父类实现。

## 4. JDK 动态代理 vs CGLIB

| 维度   | JDK 动态代理                               | CGLIB                                          |
| ---- | -------------------------------------- | ---------------------------------------------- |
| 代理对象 | 接口                                     | 类（子类）                                          |
| 约束   | 必须有接口                                  | 类/方法不能是 `final`                                |
| 调用成本 | 以 `InvocationHandler` 为入口，常见实现里会反射调用目标 | 以 `MethodInterceptor` 为入口，常见实现可走 `invokeSuper` |
| 生成成本 | 生成实现接口的代理类                             | 生成子类 + 可能生成 FastClass，启动成本更高                   |
| 典型场景 | 接口化良好的服务层                              | 无接口类、需要类级代理                                    |

工程上常见策略：

- 目标类实现了接口：优先使用 JDK 动态代理；
- 否则使用 CGLIB（或通过配置强制使用 CGLIB）。

## 5. SpringAOP 相关

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

## 6. 动态代理的典型应用场景

### 6.1 AOP 横切：日志、监控、鉴权、事务、限流

只要满足“在不侵入业务代码的前提下，对方法调用做统一增强”，动态代理都是首选思路之一。

常见落地方式：

- Spring AOP：对 `@Transactional`、`@Cacheable`、自定义 `@Around` 切面做方法级增强。
- 统一埋点：在 `InvocationHandler` / `MethodInterceptor` 里采集耗时、异常类型、QPS 统计。
- 权限校验：根据 `method`、参数、当前用户上下文决定是否放行。
- 限流与重试：对可重试错误做有限重试，对高危接口做限流与熔断。

### 6.2 RPC 客户端桩（Stub）：把接口方法映射成远程调用

典型模式是：业务代码只依赖接口，代理把“方法调用”翻译成网络请求（HTTP、TCP、自定义协议），并把响应反序列化成返回值。

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
