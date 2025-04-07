
在 Spring 中，**Bean 的生命周期**由 **Spring IOC 容器**（如 `ApplicationContext`）管理，从创建到销毁经历多个阶段，包括实例化、属性填充、初始化和销毁。Spring 提供丰富的扩展点（如 `BeanPostProcessor`、`InitializingBean`），允许开发者介入生命周期。

---

### 完整过程
#### 1. Bean 定义加载
- **描述**：Spring 从配置文件（XML）、注解（`@Component`）或 Java 配置（`@Bean`）加载 Bean 定义。
- **扩展点**：
  - **`BeanDefinitionRegistryPostProcessor`**：修改 Bean 定义。
    - 示例：`postProcessBeanDefinitionRegistry` 添加动态 Bean。
- **实现**：`BeanDefinition` 对象存储元信息（如类名、作用域）。

#### 2. Bean 实例化（Instantiation）
- **描述**：容器根据 `BeanDefinition` 创建 Bean 实例。
- **过程**：
  - 调用构造器（默认无参，或通过 `@Autowired` 指定）。
  - 单例 Bean 在容器启动时创建（默认），懒加载 Bean（`@Lazy`）延迟到首次使用。
- **扩展点**：
  - **`InstantiationAwareBeanPostProcessor`**：
    - `postProcessBeforeInstantiation`：可返回代理对象，跳过默认实例化。
- **代码**：
```java
Class<?> beanClass = beanDefinition.getBeanClass();
Object bean = beanClass.getDeclaredConstructor().newInstance();
```

#### 3. 属性填充（Population）
- **描述**：为 Bean 注入依赖（如 `@Autowired` 属性或 Setter）。
- **过程**：
  - 解析依赖（依赖注入，DI）。
  - 通过反射设置字段或调用 Setter。
- **扩展点**：
  - **`InstantiationAwareBeanPostProcessor`**：
    - `postProcessProperties`：修改属性注入。
- **示例**：
```java
@Autowired
private UserDao userDao; // 注入
```

#### 4. 初始化前处理（Pre-Initialization）
- **描述**：实例化和属性填充后，执行初始化前的逻辑。
- **扩展点**：
  - **`BeanPostProcessor`**：
    - `postProcessBeforeInitialization`：修改 Bean（如代理）。
    - 示例：AOP 动态代理在此生成。
- **实现**：
  - Spring 内部（如 `@Autowired` 注解处理）。

#### 5. 初始化（Initialization）
- **描述**：Bean 执行自定义初始化逻辑。
- **方式**：
  1. **`InitializingBean` 接口**：
     - 实现 `afterPropertiesSet()`。
  2. **`@PostConstruct` 注解**：
     - 方法在属性设置后调用。
  3. **`init-method`**：
     - XML 或 `@Bean(initMethod = "init")` 指定。
- **示例**：
```java
@Component
public class UserService implements InitializingBean {
    @PostConstruct
    public void init() {
        System.out.println("PostConstruct");
    }

    @Override
    public void afterPropertiesSet() {
        System.out.println("InitializingBean");
    }
}
```

#### 6. 初始化后处理（Post-Initialization）
- **描述**：初始化后进一步处理，常用于代理或校验。
- **扩展点**：
  - **`BeanPostProcessor`**：
    - `postProcessAfterInitialization`：返回最终 Bean。
    - 示例：AOP 代理完成。
- **结果**：Bean 准备就绪，放入单例池（`singletonObjects`）。

#### 7. 使用阶段
- **描述**：Bean 被应用使用（如通过 `getBean()` 获取）。
- **特点**：
  - 单例 Bean：缓存复用。
  - 原型 Bean：每次新建。

#### 8. 销毁（Destruction）
- **描述**：容器关闭时销毁单例 Bean（原型 Bean 不受管理）。
- **方式**：
  1. **`DisposableBean` 接口**：
     - 实现 `destroy()`。
  2. **`@PreDestroy` 注解**：
     - 方法在销毁前调用。
  3. **`destroy-method`**：
     - XML 或 `@Bean(destroyMethod = "close")` 指定。
- **过程**：
  - `AbstractApplicationContext.close()` 触发。
- **示例**：
```java
@Component
public class UserService implements DisposableBean {
    @PreDestroy
    public void cleanup() {
        System.out.println("PreDestroy");
    }

    @Override
    public void destroy() {
        System.out.println("DisposableBean");
    }
}
```

---

### 生命周期总结
1. **加载定义**：解析 `BeanDefinition`。
2. **实例化**：创建对象。
3. **属性填充**：注入依赖。
4. **初始化前**：`BeanPostProcessor` 前处理。
5. **初始化**：`@PostConstruct` 等。
6. **初始化后**：`BeanPostProcessor` 后处理。
7. **使用**：Bean 可用。
8. **销毁**：`@PreDestroy` 等。

---

### 扩展点详解
1. **`BeanDefinitionRegistryPostProcessor`**：
   - 修改 Bean 定义，如动态注册。
2. **`InstantiationAwareBeanPostProcessor`**：
   - 控制实例化和属性注入。
3. **`BeanPostProcessor`**：
   - 初始化前后修改 Bean（如 AOP）。
4. **`InitializingBean` / `@PostConstruct`**：
   - 自定义初始化。
5. **`DisposableBean` / `@PreDestroy`**：
   - 自定义销毁。

---

### 示例（完整生命周期）
```java
@Component
public class MyBean implements InitializingBean, DisposableBean {
    @Autowired
    private String name;

    @PostConstruct
    public void postConstruct() {
        System.out.println("1. PostConstruct");
    }

    @Override
    public void afterPropertiesSet() {
        System.out.println("2. InitializingBean");
    }

    @Override
    public void destroy() {
        System.out.println("3. DisposableBean");
    }

    @PreDestroy
    public void preDestroy() {
        System.out.println("4. PreDestroy");
    }
}

@Configuration
@ComponentScan
public class AppConfig {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
        context.close();
    }
}
```
- **输出**：
```
1. PostConstruct
2. InitializingBean
3. PreDestroy
4. DisposableBean
```

---

### 延伸与面试角度
- **单例 vs 原型**：
  - 单例：完整生命周期。
  - 原型：不执行销毁。
- **AOP 影响**：
  - `BeanPostProcessor` 生成代理。
- **异常处理**：
  - 初始化失败，Bean 不可用。
- **面试点**：
  - 问“阶段”时，提 8 步流程。
  - 问“扩展”时，提 `BeanPostProcessor`。

---

### 总结
Spring 通过 IOC 容器管理 Bean 生命周期，从定义加载到销毁，提供多阶段扩展点（如 `BeanPostProcessor`、`@PostConstruct`）。面试时，可画流程图或写示例，展示理解深度。