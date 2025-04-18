
## 1. Bean 定义阶段

- **组件扫描**：通过 `@ComponentScan` 扫描特定包下的所有带有 `@Component`（及其派生注解）的类
- **Bean 定义注册**：解析注解元数据，创建 `BeanDefinition` 对象并注册到 `BeanDefinitionRegistry` 中
- **处理条件注解**：处理 `@Conditional` 等条件注解，决定是否创建此 Bean

## 2. Bean 实例化阶段

- **实例化前处理**：调用 `InstantiationAwareBeanPostProcessor` 的 `postProcessBeforeInstantiation` 方法
- **实例化**：通过反射调用构造方法创建 Bean 实例
- **实例化后处理**：调用 `InstantiationAwareBeanPostProcessor` 的 `postProcessAfterInstantiation` 方法

## 3. 依赖注入阶段

- **属性处理前**：调用 `InstantiationAwareBeanPostProcessor` 的 `postProcessProperties` 方法
- **依赖注入**：处理 `@Autowired`、`@Value` 等注解，注入依赖对象和值

## 4. 初始化阶段

- **初始化前**：调用 `BeanPostProcessor` 的 `postProcessBeforeInitialization` 方法
- **Aware 接口回调**：处理 `BeanNameAware`、`BeanFactoryAware`、`ApplicationContextAware` 等接口
- **初始化方法调用**：
  - 如果 Bean 实现了 `InitializingBean` 接口，调用 `afterPropertiesSet()` 方法
  - 调用 `@PostConstruct` 注解标记的方法
  - 调用在 Bean 定义中指定的自定义初始化方法
- **初始化后**：调用 `BeanPostProcessor` 的 `postProcessAfterInitialization` 方法（通常用于 AOP 代理生成）

## 5. 就绪阶段

- Bean 可以被应用程序使用
- 单例 Bean 会被缓存在容器中
- 原型 Bean 每次请求都会创建新实例

## 6. 销毁阶段

- **销毁前**：容器关闭时触发销毁流程
- **销毁方法调用**：
  - 如果 Bean 实现了 `DisposableBean` 接口，调用 `destroy()` 方法
  - 调用 `@PreDestroy` 注解标记的方法
  - 调用在 Bean 定义中指定的自定义销毁方法

## 主要扩展点

1. **BeanPostProcessor**：Bean 初始化前后的处理
2. **InstantiationAwareBeanPostProcessor**：Bean 实例化前后的处理
3. **BeanFactoryPostProcessor**：容器加载 Bean 定义后，实例化前的处理
4. **Aware 接口族**：允许 Bean 感知容器特定资源
5. **生命周期回调**：`@PostConstruct`、`@PreDestroy`、`InitializingBean`、`DisposableBean`
