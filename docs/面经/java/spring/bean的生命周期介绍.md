# Bean的生命周期介绍

## 1. 结论

Spring Bean 的生命周期包括实例化、依赖注入、初始化、使用与销毁几个阶段，核心扩展点集中在 `BeanPostProcessor` 与初始化/销毁回调。

## 2. 生命周期流程

1. 解析并注册 `BeanDefinition`。
2. 实例化 Bean（构造器）。
3. 依赖注入与属性填充。
4. 调用 Aware 接口（如 `BeanNameAware`）。
5. 执行 `BeanPostProcessor` 的前置处理。
6. 初始化：`@PostConstruct`、`InitializingBean#afterPropertiesSet`、自定义 init-method。
7. 执行 `BeanPostProcessor` 的后置处理（可能生成代理）。
8. 容器关闭时调用销毁回调（`@PreDestroy`、`DisposableBean`、destroy-method）。

## 3. 关键点/注意事项

- `BeanPostProcessor` 会影响所有 Bean，顺序需关注。
- 单例 Bean 在容器启动时创建，原型 Bean 在每次获取时创建。
- AOP 代理通常在后置处理阶段生成。

## 4. 常见追问/易错点

- 追问：原型 Bean 的销毁回调为什么不会自动触发？
  - 答：核心结论是：Spring Bean 的生命周期包括实例化、依赖注入、初始化、使用与销毁几个阶段，核心扩展点集中在 `BeanPostProcessor` 与初始化/销毁回调。展开时可按“生命周期流程、关键点/注意事项”组织，先概述再逐点展开，保证结构完整。其中生命周期流程侧重解析并注册 `BeanDefinition`，关键点/注意事项侧重`BeanPostProcessor` 会影响所有 Bean，顺序需关注。回答时要体现步骤、关键点与适用场景，必要时补充示例或对比。
- 易错点：在初始化前使用 Bean 会拿到未完成依赖注入的对象。
  - 答：常见易错点是忽略步骤顺序或前置条件、遗漏关键点或注意事项。比如生命周期流程中提到：解析并注册 `BeanDefinition`。关键点/注意事项中还提到：`BeanPostProcessor` 会影响所有 Bean，顺序需关注。这些细节很容易被忽视。回答时应明确边界、关键步骤与适用场景，并用实例或对比验证。