
#### Spring 事务失效场景
Spring 事务基于 AOP（动态代理）实现，依赖 `@Transactional` 注解或 XML 配置。在以下情况下事务会失效：
1. **方法非 public**：代理无法拦截。
2. **内部调用**：同一类中方法直接调用，绕过代理。
3. **异常被捕获未抛出**：事务默认只回滚运行时异常。
4. **不支持事务的存储引擎**：如 MyISAM 不支持事务。
5. **未启用事务管理**：缺少 `@EnableTransactionManagement` 或配置。
6. **传播行为配置错误**：如 `NOT_SUPPORTED` 不支持事务。
7. **多线程调用**：事务不跨线程生效。

#### 核心点
- 失效多因代理机制或配置不当。

---

### 1. 失效场景详解
#### (1) 方法非 public
- **原因**：
  - Spring 使用 JDK 动态代理（基于接口）或 CGLIB（基于类），只拦截 `public` 方法。
- **示例**：
```java
@Service
public class MyService {
    @Transactional
    void doSomething() { // 非 public，事务失效
        // 操作数据库
    }
}
```
- **解决**：
  - 改为 `public`，或用 CGLIB 强制代理（`@EnableAspectJAutoProxy(proxyTargetClass = true)`）。

#### (2) 内部调用
- **原因**：
  - 同一类中方法直接调用，绕过代理对象，未触发事务切面。
- **示例**：
```java
@Service
public class MyService {
    public void methodA() {
        methodB(); // 直接调用，事务失效
    }

    @Transactional
    public void methodB() {
        // 操作数据库
    }
}
```
- **解决**：
  - 通过代理调用：
```java
@Service
public class MyService {
    @Autowired
    private MyService self; // 注入自身代理

    public void methodA() {
        self.methodB(); // 通过代理调用
    }

    @Transactional
    public void methodB() {}
}
```
- 或拆分到不同类。

#### (3) 异常被捕获未抛出
- **原因**：
  - Spring 默认只回滚 `RuntimeException` 和 `Error`，若异常被捕获未抛出，事务不回滚。
- **示例**：
```java
@Transactional
public void doSomething() {
    try {
        // 数据库操作抛出异常
    } catch (Exception e) {
        // 捕获未抛出，事务不回滚
    }
}
```
- **解决**：
  - 抛出异常，或指定回滚异常：
```java
@Transactional(rollbackOn = Exception.class)
public void doSomething() throws Exception {
    // 抛出异常
}
```

#### (4) 不支持事务的存储引擎
- **原因**：
  - 数据库（如 MySQL）使用非事务引擎（如 MyISAM），事务无效。
- **示例**：
  - MyISAM 表执行 `@Transactional` 操作，不回滚。
- **解决**：
  - 改为 InnoDB（支持事务）。

#### (5) 未启用事务管理
- **原因**：
  - Spring Boot 默认启用，但纯 Spring 项目需手动配置。
- **示例**：
  - 缺少 `<tx:annotation-driven/>` 或 `@EnableTransactionManagement`。
- **解决**：
```java
@Configuration
@EnableTransactionManagement
public class AppConfig {
    @Bean
    public PlatformTransactionManager txManager() {
        return new DataSourceTransactionManager(dataSource());
    }
}
```

#### (6) 传播行为配置错误
- **原因**：
  - `@Transactional(propagation = Propagation.NOT_SUPPORTED)` 不支持事务。
- **示例**：
```java
@Transactional(propagation = Propagation.NOT_SUPPORTED)
public void doSomething() {
    // 无事务
}
```
- **解决**：
  - 使用默认 `REQUIRED` 或其他支持事务的传播行为。

#### (7) 多线程调用
- **原因**：
  - Spring 事务基于 `ThreadLocal`，线程间隔离，新线程无事务上下文。
- **示例**：
```java
@Transactional
public void doSomething() {
    new Thread(() -> {
        // 数据库操作，新线程无事务
    }).start();
}
```
- **解决**：
  - 在主线程完成操作，或用 `TransactionTemplate` 手动管理。

---

### 2. 原理分析
- **Spring 事务实现**：
  - 通过 AOP 动态代理（JDK 或 CGLIB）增强方法。
  - `TransactionInterceptor` 拦截，管理事务开启、提交、回滚。
- **失效根源**：
  - 代理未生效（如内部调用）。
  - 配置或环境不匹配。

---

### 3. 延伸与面试角度
- **与 AOP 关系**：
  - 事务失效多因 AOP 代理限制。
- **源码关键**：
  - `TransactionAspectSupport`：事务逻辑。
  - `PlatformTransactionManager`：事务管理器。
- **实际应用**：
  - 订单服务：确保扣库存和支付一致。
- **面试点**：
  - 问“失效”时，提内部调用和异常。
  - 问“解决”时，提代理注入。

---

### 总结
Spring 事务失效常见于代理未触发（如非 public、内部调用）、异常处理不当或配置错误。理解其 AOP 原理是关键，解决靠调整调用方式或配置。面试时，可提示例或原理，展示理解深度。