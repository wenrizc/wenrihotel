## 1. IoC 与 DI：Spring 容器在“反转”什么

IoC（Inversion of Control，控制反转）不是一种具体技术，而是一种设计思想：**对象的创建与依赖关系不再由业务代码手动 new/拼装，而是交给容器统一管理**。

DI（Dependency Injection，依赖注入）是 IoC 的具体实现方式：容器在创建对象时，把它所依赖的对象“注入”进去。

工程收益：

- 降低耦合：依赖通过接口与配置组合。
- 生命周期统一管理：初始化、销毁、代理增强。
- 可测试性更好：依赖可替换、可 mock。

## 2. 核心抽象：`BeanDefinition` 与容器接口

### 2.1 `BeanDefinition`：Bean 的“配方”

`BeanDefinition` 描述了如何创建一个 Bean，核心信息包括：

- Bean 的类型（class）
- 作用域（singleton/prototype/...）
- 构造参数与属性依赖
- 初始化/销毁回调
- 是否懒加载、是否 primary 等

### 2.2 `BeanFactory` 与 `ApplicationContext`

- `BeanFactory`：容器的最小核心能力（获取 Bean、创建 Bean）。
- `ApplicationContext`：在 `BeanFactory` 之上提供更多能力（事件、国际化、资源加载等），并提供 `refresh()` 启动流程。

## 3. 容器启动主流程：`refresh()` 做了什么

面试讲“底层实现思路”，最加分的是能把 Spring 启动流程说清楚（无需背源码细节，但要成体系）。

典型流程可以概括为：

1. 读取配置，构建并注册 `BeanDefinition`。
2. 执行 `BeanFactoryPostProcessor`：允许修改 `BeanDefinition`（“改配方”）。
3. 注册 `BeanPostProcessor`：允许在 Bean 初始化前后增强（“改成品”）。
4. 初始化单例：按需创建 Bean，并完成依赖注入与生命周期回调。

## 4. Bean 创建流程（源码级主干）

以 `AbstractAutowireCapableBeanFactory` 的创建逻辑为主线，Bean 创建大致是：

1. 实例化（构造器反射/工厂方法）
2. 属性填充（依赖注入）
3. 初始化（Aware 回调、`@PostConstruct`、`InitializingBean`、init-method）
4. 后置处理（`BeanPostProcessor`，可能生成 AOP 代理）

可以用伪代码表达主干（更利于面试复述）：

```java
createBean(name):
  instance = instantiateBean(name)
  populateProperties(instance)
  instance = initializeBean(instance) // Aware + init callbacks
  instance = applyPostProcessorsAfterInitialization(instance) // 可能代理
  return instance
```

## 5. IoC 与 AOP/事务的关系

Spring AOP 与 `@Transactional` 往往依赖 `BeanPostProcessor` 创建代理：

- 容器创建 Bean 后，在后置处理阶段判断是否需要增强。
- 需要增强则返回代理对象，后续注入与使用的都是代理。

因此很多事务“失效”的根因并不在事务本身，而在于**是否经过代理调用**。

