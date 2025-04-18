
从项目代码来看，MiniSpring框架实现了一个简化版的Spring容器，Bean的生命周期大致分为以下几个阶段：

## 1. Bean定义阶段

- **扫描组件**：通过`scanForClassNames`方法扫描被`@Component`等注解标记的类
- **注册Bean定义**：创建并注册`BeanDefinition`对象，包含Bean的名称、类型、构造方法/工厂方法等信息
- **解析配置类**：处理`@Configuration`类中的`@Bean`方法

## 2. Bean创建阶段

- **实例化**：通过构造方法或工厂方法创建Bean实例
- **依赖注入**：注入构造方法参数依赖的其他Bean
- **前置处理**：应用`BeanPostProcessor.postProcessBeforeInitialization`

## 3. Bean初始化阶段

- **属性注入**：通过`injectBean`方法注入字段和setter方法上的`@Autowired`或`@Value`
- **初始化方法**：调用`@PostConstruct`标注的方法或Bean注解中指定的`initMethod`
- **后置处理**：应用`BeanPostProcessor.postProcessAfterInitialization`

## 4. Bean销毁阶段

- **销毁方法**：应用上下文关闭时，调用`@PreDestroy`标注的方法或指定的`destroyMethod`

# Bean的创建过程

在`AnnotationConfigApplicationContext`构造函数中：

1. 首先扫描获取所有Bean的类型并创建`BeanDefinition`
2. 按优先级顺序创建Bean：
   - 先创建`@Configuration`类型的Bean
   - 再创建`BeanPostProcessor`类型的Bean
   - 最后创建普通Bean
3. 通过`createBeanAsEarlySingleton`方法创建Bean实例
4. 通过`injectBean`方法注入依赖
5. 通过`initBean`方法初始化Bean

# 延迟创建实现

MiniSpring中实现延迟创建的方式：

## 1. 按需创建策略

从代码可以看出，Bean不是一次性全部创建的，而是按需创建：

- 在构造函数中仅主动创建`@Configuration`和`BeanPostProcessor`类型的Bean
- 其他Bean只有在被依赖或显式调用`getBean`时才会创建

## 2. 递归依赖触发

在`createBeanAsEarlySingleton`方法中，当发现一个Bean依赖的其他Bean尚未创建时，会递归创建依赖的Bean：

```
if (autowiredBeanInstance == null && !isConfiguration && !isBeanPostProcessor) {
    // 当前依赖Bean尚未初始化，递归调用初始化该依赖Bean
    autowiredBeanInstance = createBeanAsEarlySingleton(dependsOnDef);
}
```

## 3. 查找与获取的区别

- `findBean`方法仅查找Bean，不存在时返回null，不会触发创建
- `getBean`方法在Bean不存在时会抛出异常，不会延迟创建不存在的Bean
- 只有通过构造方法依赖或字段注入时才会触发递归创建

## 4. 循环依赖检测

MiniSpring使用`creatingBeanNames`集合跟踪正在创建的Bean，防止循环依赖：

```
if (!this.creatingBeanNames.add(def.getName())) {
    throw new UnsatisfiedDependencyException(String.format("Circular dependency detected when create bean '%s'", def.getName()));
}
```

相比Spring框架，MiniSpring实现了基础的延迟创建机制，但缺少显式的`@Lazy`注解支持。Bean的创建主要依赖于依赖关系驱动，只有被其他Bean依赖或显式获取时才会被创建，这实现了一种隐式的延迟加载机制。