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

最小示例：

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

### 2.3 底层原理要点（面试常问）

- 运行时生成字节码（`ProxyGenerator`），并通过类加载器定义该类。
- 代理类的方法体通常会把调用转发到 `InvocationHandler#invoke`。
- 最终调用目标方法时使用反射 `Method.invoke`（JDK 会做一定优化，但仍有反射语义与可访问性检查成本）。

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

- CGLIB 基于 ASM 等字节码框架动态生成子类字节码。
- 常配合 **FastClass** 思想：用“方法索引表”减少反射分派成本（实现与版本相关）。
- 生成的类会占用元空间（Metaspace），频繁生成会带来类加载与内存压力，因此通常会有缓存。

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

## 5. Spring AOP 相关的高频坑（代理必考）

### 5.1 自调用导致 @Transactional 失效

同一个类内部方法 A 调用方法 B（B 上有 `@Transactional`），如果是 `this.b()` 这种自调用，通常不会经过代理，因此事务不会生效。

解决方向：

- 把 B 抽到另一个 Bean；
- 或通过注入自身代理（不推荐滥用）；
- 或使用 `AopContext.currentProxy()`（需要开启暴露代理，谨慎）。

### 5.2 final / private 方法无法被代理增强

- CGLIB 无法覆盖 `final` 方法；
- 代理通常也不会拦截 `private` 方法（调用路径与可见性决定）。

## 6. 总结（面试回答模板）

- JDK 动态代理：基于接口，核心是 `InvocationHandler#invoke`，运行时生成实现接口的代理类。
- CGLIB：基于继承，核心是 `MethodInterceptor#intercept`，运行时生成子类并重写方法拦截。
- 选型：能接口优先 JDK；无接口或需要类代理用 CGLIB；注意 Spring AOP 的自调用与 `final` 限制。

