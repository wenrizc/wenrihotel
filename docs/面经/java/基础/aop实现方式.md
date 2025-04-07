
AOP（Aspect-Oriented Programming）的实现方式主要有 **动态代理** 和 **字节码操作** 两种。主流框架如 Spring AOP 使用 **JDK 动态代理**（基于接口）和 **CGLIB 代理**（基于类）实现，通过在运行时生成代理对象，拦截目标方法并织入切面逻辑。此外，AspectJ 通过编译时或加载时字节码增强，提供更强大的实现。

### 关键事实

1. **基本原理**：
    - AOP 通过代理或字节码修改，在不改变目标代码的情况下，插入横切逻辑（如日志、事务）。
    - 核心概念：切面（Aspect）、切点（Pointcut）、通知（Advice）。
2. **实现方式**：
    - **JDK 动态代理**：
        - 基于接口，利用 Java 的 java.lang.reflect.Proxy。
        - 动态生成代理类，拦截接口方法。
    - **CGLIB 代理**：
        - 基于类，通过字节码生成库（CGLIB），继承目标类。
        - 无需接口，适用于普通类。
    - **AspectJ**：
        - 编译时（Compile-Time Weaving）：修改源码生成新字节码。
        - 加载时（Load-Time Weaving）：JVM 加载时增强。
3. **Spring AOP 默认策略**：
    - 有接口时用 JDK 动态代理。
    - 无接口时用 CGLIB。

### 实现方式详解与示例

#### 1. JDK 动态代理

- **原理**：运行时生成代理类，实现目标接口，拦截方法调用。
- **适用**：目标对象必须实现接口。
- **示例**：
``` java
public interface UserService {
    void save();
}

public class UserServiceImpl implements UserService {
    public void save() { System.out.println("Save user"); }
}

public class MyInvocationHandler implements InvocationHandler {
    private Object target;
    public MyInvocationHandler(Object target) { this.target = target; }
    
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("Before save"); // 前置通知
        Object result = method.invoke(target, args);
        System.out.println("After save");  // 后置通知
        return result;
    }
}

public static void main(String[] args) {
    UserService target = new UserServiceImpl();
    UserService proxy = (UserService) Proxy.newProxyInstance(
        target.getClass().getClassLoader(),
        target.getClass().getInterfaces(),
        new MyInvocationHandler(target)
    );
    proxy.save();
}
```
#### 2. CGLIB 代理

- **原理**：运行时生成目标类的子类，重写方法插入逻辑。
- **适用**：无需接口，直接代理类。
- **示例**：
``` java
public class UserService {
    public void save() { System.out.println("Save user"); }
}

public class MyMethodInterceptor implements MethodInterceptor {
    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        System.out.println("Before save");
        Object result = proxy.invokeSuper(obj, args);
        System.out.println("After save");
        return result;
    }
}

public static void main(String[] args) {
    Enhancer enhancer = new Enhancer();
    enhancer.setSuperclass(UserService.class);
    enhancer.setCallback(new MyMethodInterceptor());
    UserService proxy = (UserService) enhancer.create();
    proxy.save();
}
```

### 延伸与面试角度

- **Spring AOP vs. AspectJ**：
    - **Spring AOP**：运行时代理，简单但只支持方法级切面。
    - **AspectJ**：编译时增强，功能强大但配置复杂。
- **优缺点**：
    - **JDK 动态代理**：轻量，依赖接口，性能稍低（反射）。
    - **CGLIB**：灵活，无接口限制，但生成子类有开销。
    - **AspectJ**：高效，支持细粒度切面，学习成本高。
- **选择依据**：
    - 接口明确用 JDK 代理，无接口用 CGLIB。
    - 复杂场景（如字段拦截）用 AspectJ。
- **实际应用**：
    - **Spring 事务**：@Transactional 用 AOP 代理管理事务。
    - **日志**：拦截方法打印调用信息。
- **性能考虑**：
    - 代理创建有开销，高频调用可缓存代理对象。

---

### 总结

AOP 实现方式包括 JDK 动态代理（接口）、CGLIB（类继承）和 AspectJ（字节码增强）。Spring AOP 默认用前两者，运行时织入，AspectJ 提供编译时支持。面试时，可结合 @Transactional 或日志示例，展示对代理机制的理解。