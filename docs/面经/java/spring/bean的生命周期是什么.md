
#### Spring Bean 生命周期概述
- **定义**：
  - Spring Bean 生命周期是指 Spring IoC 容器管理 Bean 从创建到销毁的完整过程，涵盖实例化、配置、初始化、使用和销毁等阶段。
- **目的**：
  - 自动化管理对象，确保依赖注入、初始化和清理的正确执行。
- **适用**：
  - 主要针对单例 Bean（`scope="singleton"`），原型 Bean（`scope="prototype"`）部分阶段不同。

#### 核心点
- 生命周期由容器驱动，提供多个扩展点，支持自定义逻辑。

---

### 1. Bean 生命周期详解
以下是 Spring 单例 Bean 的完整生命周期步骤：

#### (1) 实例化（Instantiation）
- **描述**：
  - 容器通过构造方法创建 Bean 实例。
- **实现**：
  - 使用反射调用默认或带参构造器。
- **方式**：
  - `@Component`、`@Bean`、XML `<bean>` 定义。
- **示例**：
```java
@Component
public class UserService {
    public UserService() {
        System.out.println("Instantiating");
    }
}
```

#### (2) 属性注入（Populate Properties）
- **描述**：
  - 为 Bean 填充属性和依赖（如其他 Bean）。
- **方式**：
  - **字段注入**：`@Autowired`。
  - **Setter 注入**：`setXxx`。
  - **构造器注入**：构造方法。
- **示例**：
```java
@Component
public class UserService {
    @Autowired
    private OrderService orderService;
}
```

#### (3) 前置处理（BeanPostProcessor Before Initialization）
- **描述**：
  - 执行 `BeanPostProcessor` 的 `postProcessBeforeInitialization` 方法。
- **作用**：
  - 修改 Bean 或准备代理（如 AOP）。
- **示例**：
```java
@Component
public class CustomBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        System.out.println("Before init: " + beanName);
        return bean;
    }
}
```

#### (4) 初始化（Initialization）
- **子步骤**：
  1. **Aware 接口**：
     - 注入容器信息，如 `BeanNameAware`、`ApplicationContextAware`.

#### (5) 后置处理（BeanPostProcessor After Initialization）
- **描述**：
  - 执行 `BeanPostProcessor` 的 `postProcessAfterInitialization` 方法。
- **作用**：
  - 完成代理或最终增强。
- **示例**：
```java
@Override
public Object postProcessAfterInitialization(Object bean, String beanName) {
    System.out.println("After init: " + beanName);
    return bean;
}
```

#### (6) 使用（Ready for Use）
- **描述**：
  - Bean 可供应用程序使用。
- **方式**：
  - 通过 `context.getBean()` 或 `@Autowired`。
- **示例**：
```java
@Autowired
private UserService userService;
```

#### (7) 销毁（Destruction）
- **描述**：
  - 容器关闭时销毁单例 Bean。
- **子步骤**：
  1. **@PreDestroy**：
```java
@PreDestroy
public void destroy() {
    System.out.println("PreDestroy");
}
```
  1. **DisposableBean**：
```java
@Component
public class MyBean implements DisposableBean {
    @Override
    public void destroy() {
        System.out.println("DisposableBean");
    }
}
```
  1. **destroy-method**：
```xml
<bean id="myBean" class="com.example.MyBean" destroy-method="customDestroy"/>
```
```java
public void customDestroy() {
    System.out.println("Custom destroy");
}
```
- **触发**：
  - `ApplicationContext.close()`。

---

### 2. 流程图
```
实例化 --> 属性注入 --> BeanPostProcessor Before --> 初始化 --> BeanPostProcessor After --> 使用 --> 销毁
```

---

### 3. 单例 vs 原型
- **单例**：
  - 完整生命周期，容器管理销毁。
- **原型**：
  - 到初始化为止，容器不负责销毁。

---

### 4. 扩展点
- **接口**：
  - `BeanPostProcessor`、`InitializingBean`、`DisposableBean`。
  - `Aware`：如 `ApplicationContextAware`。
- **注解**：
  - `@PostConstruct`、 `@PreDestroy`.
- **配置**：
  - `init-method`、 `destroy-method`.

---

### 5. 场景
- **初始化**：
  - 加载配置、连接数据库。
- **销毁**：
  - 关闭资源（如连接池）。
- **增强**：
  - AOP 代理、事务管理。

---

### 6. 面试角度
- **问“步骤”**：
  - 提 7 大阶段，细化初始化。
- **问“扩展”**：
  - 提注解和接口。
- **问“问题”**：
  - 提循环依赖或销毁未触发。

---
### 7. 总结
Spring Bean 生命周期包括实例化、属性注入、前后置处理、初始化、使用和销毁，提供丰富的扩展点（如 `@PostConstruct`）。单例 Bean 全程管理，原型到初始化为止。面试可提流程或示例，清晰展示理解。

---
如果您想深入某部分（如源码或问题排查），请告诉我！