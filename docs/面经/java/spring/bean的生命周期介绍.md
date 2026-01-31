## 1. 生命周期主线：从“配方”到“成品”

Bean 生命周期可以按“配方 → 创建 → 注入 → 初始化 → 使用 → 销毁”来理解：

1. 注册 `BeanDefinition`（Bean 的配方）。
2. 创建 Bean 实例（实例化）。
3. 依赖注入（属性填充）。
4. 初始化回调与后置处理（可能产生代理）。
5. 容器关闭时执行销毁回调（单例）。

## 2. 创建与初始化阶段的关键步骤

下面的顺序以常见单例 Bean 为主，能覆盖面试绝大多数问题。

### 2.1 实例化（Instantiation）

- 构造器实例化（反射）
- 工厂方法实例化（`@Bean`、FactoryBean 等场景）

### 2.2 属性填充（Populate Properties）

解析依赖并注入：

- `@Autowired`、`@Resource` 等
- 构造器参数、setter 方法、字段等

### 2.3 Aware 回调（让 Bean “感知容器”）

常见如：

- `BeanNameAware`
- `BeanFactoryAware`
- `ApplicationContextAware`

### 2.4 初始化（Initialization）

初始化常见触发点（顺序可概括为）：

1. `@PostConstruct`
2. `InitializingBean#afterPropertiesSet`
3. 自定义 `init-method`

### 2.5 `BeanPostProcessor`（生命周期最重要的扩展点）

`BeanPostProcessor` 会在初始化前后对 Bean 做增强，是 AOP、事务、`@Async` 等能力的关键入口。

常见影响：

- 返回代理对象替换原对象（容器里保存的是代理）。
- 修改属性、注入额外依赖、做校验等。

## 3. 销毁阶段（Destruction）

容器关闭时，单例 Bean 会执行销毁回调：

- `@PreDestroy`
- `DisposableBean#destroy`
- 自定义 `destroy-method`

prototype Bean 通常不由容器统一销毁，资源释放需要业务方自行管理。

## 4. 作用域差异（Singleton vs Prototype）

- singleton：默认，容器级单例，通常在容器启动或首次使用时创建。
- prototype：每次 `getBean` 都新建，生命周期管理不完整（销毁不托管）。

Web 场景还可能有 request/session 等作用域，生命周期与请求上下文绑定。

## 5. 总结

- Bean 生命周期主干：实例化 → 注入 → 初始化（含 BPP）→ 使用 → 销毁。
- `BeanPostProcessor` 是核心扩展点，AOP/事务代理多在这里产生。
- prototype 的销毁不由容器托管，外部资源释放要显式处理。
