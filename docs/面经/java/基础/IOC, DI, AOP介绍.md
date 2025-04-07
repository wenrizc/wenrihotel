
### 1. IOC（Inversion of Control，控制反转）
#### 是什么
IOC 是一种设计思想，将对象的创建和管理权从代码中剥离，交给外部容器（如 Spring IOC 容器），实现控制权的反转。程序不再主动创建对象，而是被动接收容器提供的实例。

#### 核心原理
- **传统方式**：代码直接 `new` 对象，主动控制依赖。
- **IOC 方式**：容器负责实例化并注入依赖，程序只声明需求。
- **实现**：通过配置文件（如 XML）或注解（如 `@Component`）。

#### 示例（Spring）
```java
@Component
public class UserService {
    // 不直接 new
}

// 容器管理
ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
UserService service = context.getBean(UserService.class);
```

#### 特点
- **解耦**：降低类间依赖。
- **灵活**：修改依赖无需改代码。

---

### 2. DI（Dependency Injection，依赖注入）
#### 是什么
DI 是 IOC 的一种实现方式，通过将依赖对象注入到目标对象中，完成控制反转。Spring 中 DI 是 IOC 容器的核心机制。

#### 注入方式
1. **构造器注入**：
```java
@Component
public class UserService {
    private final UserDao dao;

    @Autowired
    public UserService(UserDao dao) {
        this.dao = dao;
    }
}
```
1. **Setter 注入**：
```java
@Component
public class UserService {
    private UserDao dao;

    @Autowired
    public void setUserDao(UserDao dao) {
        this.dao = dao;
    }
}
```
1. **字段注入**（不推荐）：
```java
@Component
public class UserService {
    @Autowired
    private UserDao dao;
}
```

#### 特点
- **明确依赖**：通过构造器或 Setter 显示声明。
- **动态替换**：运行时注入不同实现。

---

### 3. AOP（Aspect-Oriented Programming，面向切面编程）
#### 是什么
AOP 是一种编程范式，通过将横切关注点（如日志、事务）与业务逻辑分离，提高代码模块化。它在不修改源代码的情况下，动态织入额外功能。

#### 核心概念
- **切面（Aspect）**：封装横切逻辑的模块。
- **切点（Pointcut）**：定义拦截的执行点（如方法）。
- **通知（Advice）**：定义拦截后执行的动作（如前置、后置）。
- **织入（Weaving）**：将切面应用到目标对象。

#### 示例（Spring AOP）
```java
@Aspect
@Component
public class LogAspect {
    @Before("execution(* com.example.service.*.*(..))")
    public void logBefore() {
        System.out.println("Method start");
    }
}

@Service
public class UserService {
    public void getUser() {
        System.out.println("Get user");
    }
}
```
- **运行**：调用 `getUser()` 前打印 "Method start"。

#### 特点
- **解耦**：业务与非业务逻辑分离。
- **复用**：切面可应用于多处。

---

### 三者联系
1. **IOC 是思想，DI 是实现**：
   - IOC 提出控制反转，DI 通过注入实现。
   - Spring IOC 容器用 DI 管理 Bean。
2. **AOP 依赖 IOC**：
   - AOP 的切面（如 `LogAspect`）由 IOC 容器管理。
   - DI 注入切面到目标对象。
3. **协同作用**：
   - IOC/DI 解耦对象创建，AOP 解耦横切逻辑。
   - 示例：Spring 中 DI 注入 `UserService`，AOP 添加日志。

---

### 延伸与面试角度
- **IOC/DI 优点**：
  - 降低耦合，便于测试（如 Mock）。
- **AOP 应用**：
  - 日志、事务、权限控制。
- **实现细节**：
  - **DI**：Spring 用反射创建对象。
  - **AOP**：Spring 用动态代理（JDK 或 CGLIB）。
- **面试点**：
  - 问“IOC 好处”时，提解耦和测试。
  - 问“AOP 原理”时，提代理和织入。

---

### 总结
IOC 是控制反转思想，DI 是其实现，通过注入管理依赖；AOP 面向切面，分离横切逻辑。三者协同解耦代码，提升可维护性。面试时，可写 Spring 示例或提代理原理，展示理解深度。