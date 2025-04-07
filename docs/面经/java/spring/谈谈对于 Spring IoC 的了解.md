
- **定义**：
  - IoC 是 Spring 框架的核心特性，指将对象的创建、管理和依赖注入的控制权从代码反转到容器，由 Spring IoC 容器负责。
- **核心思想**：
  - **控制反转**：传统程序由开发者手动创建对象，IoC 则由容器接管。
  - **依赖注入（DI）**：IoC 的实现方式，容器自动将依赖的对象注入到目标对象。
- **作用**：
  - 解耦代码，提高模块化、可测试性和可维护性。

#### 核心组件
1. **IoC 容器**：`BeanFactory` 或 `ApplicationContext`，管理 Bean。
2. **Bean**：容器管理的对象。
3. **依赖注入**：通过构造器、Setter 或字段注入依赖。

---

### 1. IoC 的核心概念
#### (1) 控制反转
- **传统方式**：
  - 开发者通过 `new` 创建对象，主动管理依赖。
```java
class Service {
    private Dao dao = new DaoImpl(); // 手动创建
}
```
- **IoC 方式**：
  - 容器负责创建和注入，代码只需声明依赖。
```java
class Service {
    private Dao dao; // 容器注入
}
```

#### (2) 依赖注入（DI）
- **方式**：
  - **构造器注入**：通过构造方法传入依赖。
  - **Setter 注入**：通过 Setter 方法注入。
  - **字段注入**：通过反射直接注入字段（`@Autowired`）。
- **示例**：
```java
@Component
class Service {
    private final Dao dao;

    @Autowired // 构造器注入
    public Service(Dao dao) {
        this.dao = dao;
    }
}
```

---

### 2. IoC 容器的工作原理
#### (1) 核心容器
- **BeanFactory**：
  - 基础 IoC 容器，提供懒加载（使用时创建 Bean）。
- **ApplicationContext**：
  - 高级容器，扩展 `BeanFactory`，支持预加载、事件发布等。
  - 常用实现：`ClassPathXmlApplicationContext`、`AnnotationConfigApplicationContext`。

#### (2) Bean 的生命周期
1. **实例化**：通过反射创建对象（`newInstance`）。
2. **属性填充**：注入依赖（DI）。
3. **初始化**：
   - 调用 `InitializingBean.afterPropertiesSet()` 或 `@PostConstruct`。
4. **使用**：Bean 可用。
5. **销毁**：
   - 调用 `DisposableBean.destroy()` 或 `@PreDestroy`。
- **示例**：
```java
@Component
class MyBean {
    @PostConstruct
    void init() { System.out.println("Init"); }
    @PreDestroy
    void destroy() { System.out.println("Destroy"); }
}
```

#### (3) 配置方式
- **XML**：
```xml
<bean id="dao" class="com.example.DaoImpl"/>
<bean id="service" class="com.example.Service">
    <property name="dao" ref="dao"/>
</bean>
```
- **注解**：
  - `@Component`、`@Autowired`、`@Configuration`。
```java
@Configuration
class AppConfig {
    @Bean
    public Dao dao() { return new DaoImpl(); }
    @Bean
    public Service service(Dao dao) { return new Service(dao); }
}
```
- **Java Config**：优先级高于 XML。

---

### 3. IoC 的实现机制
- **反射**：
  - 通过 `Class.forName()` 和 `newInstance()` 创建对象。
- **Bean 定义**：
  - `BeanDefinition`：存储 Bean 的元信息（类名、作用域等）。
- **依赖解析**：
  - 容器扫描注解或 XML，构建依赖图，递归注入。
- **作用域**：
  - Singleton（默认单例）、Prototype（多例）、Request 等。

#### 示例流程
```java
ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
Service service = context.getBean(Service.class);
```
1. 扫描 `@Configuration`，注册 `BeanDefinition`。
2. 创建 `Dao` 和 `Service`，注入依赖。
3. 返回 `service` 实例。

---

### 4. IoC 的优点与局限
#### 优点
- **解耦**：降低类之间的直接依赖。
- **灵活性**：通过配置调整依赖关系。
- **测试性**：便于 mock 依赖。

#### 局限
- **复杂性**：配置和注解增加学习成本。
- **性能**：启动时扫描和反射有开销。
- **隐式依赖**：依赖关系不直观。

---

### 5. 延伸与面试角度
- **与 AOP 关系**：
  - IoC 管理 Bean，AOP 增强功能。
- **源码关键类**：
  - `DefaultListableBeanFactory`：核心容器实现。
  - `BeanDefinitionRegistry`：注册 Bean。
- **实际应用**：
  - Spring Boot：自动配置大量 Bean。
  - 微服务：管理服务依赖。
- **面试点**：
  - 问“原理”时，提反射和生命周期。
  - 问“好处”时，提解耦和测试。

---

### 总结
Spring IoC 通过控制反转和依赖注入，将对象管理交给容器，核心是 `ApplicationContext` 和 Bean 生命周期。实现上依赖反射和配置，优点是解耦和灵活性。面试时，可提配置示例或画容器流程，展示理解深度。