
## IOC容器整体设计

MiniSpring项目中的IOC容器核心是通过`AnnotationConfigApplicationContext`类实现的，它实现了`ConfigurableApplicationContext`接口，提供了依赖注入和控制反转的基本功能。整个容器的设计思路与Spring框架类似，但实现更加精简。

## 核心组件结构

1. **接口层**:
   - `ApplicationContext`: 顶层接口，定义获取Bean等基本方法
   - `ConfigurableApplicationContext`: 可配置的上下文接口，提供更多控制功能

2. **实现类**:
   - `AnnotationConfigApplicationContext`: 基于注解的应用上下文实现类

3. **Bean定义**:
   - `BeanDefinition`: 存储Bean的元数据，包括名称、类型、构造方法、初始化/销毁方法等

## Bean定义管理

在`AnnotationConfigApplicationContext`中，Bean定义通过以下方式管理:

```java
protected final Map<String, BeanDefinition> beans;
```

这个映射表存储了所有Bean的定义信息，key是Bean的名称，value是对应的BeanDefinition对象。

## IOC容器初始化流程

1. **配置类扫描**:
   ```java
   public AnnotationConfigApplicationContext(Class<?> configClass, PropertyResolver propertyResolver) {
       // 设置应用上下文引用
       ApplicationContextUtils.setApplicationContext(this);
       this.propertyResolver = propertyResolver;
       
       // 扫描获取所有Bean的Class类型
       final Set<String> beanClassNames = scanForClassNames(configClass);
       
       // 创建Bean的定义
       this.beans = createBeanDefinitions(beanClassNames);
       //...
   }
   ```

2. **组件扫描**:
   通过`@ComponentScan`注解指定的包路径进行扫描，查找所有被`@Component`及其派生注解标注的类。

3. **Bean定义创建**:
   对扫描到的类创建`BeanDefinition`对象，包括:
   - 为`@Component`注解的类创建BeanDefinition
   - 为`@Configuration`类中的`@Bean`方法创建BeanDefinition

## Bean的实例化与初始化

Bean的创建过程主要通过`createBeanAsEarlySingleton`方法实现:

1. **检测循环依赖**:
   ```java
   if (!this.creatingBeanNames.add(def.getName())) {
       throw new UnsatisfiedDependencyException(/*...*/);
   }
   ```

2. **选择构造方法或工厂方法**:
   ```java
   if (def.getFactoryName() == null) {
       // 通过构造方法创建
       createFn = def.getConstructor();
   } else {
       // 通过工厂方法创建
       createFn = def.getFactoryMethod();
   }
   ```

3. **处理构造参数**:
   - 支持`@Value`注入属性值
   - 支持`@Autowired`注入依赖Bean

4. **Bean实例化**:
   ```java
   if (def.getFactoryName() == null) {
       // 用构造方法创建
       instance = def.getConstructor().newInstance(args);
   } else {
       // 用@Bean方法创建
       Object configInstance = getBean(def.getFactoryName());
       instance = def.getFactoryMethod().invoke(configInstance, args);
   }
   ```

5. **应用BeanPostProcessor**:
   ```java
   for (BeanPostProcessor processor : beanPostProcessors) {
       Object processed = processor.postProcessBeforeInitialization(def.getInstance(), def.getName());
       //...
   }
   ```

## 依赖注入实现

依赖注入通过`injectBean`和`injectProperties`方法实现:

1. **获取Bean实例**:
   ```java
   final Object beanInstance = getProxiedInstance(def);
   ```

2. **注入属性**:
   ```java
   void injectProperties(BeanDefinition def, Class<?> clazz, Object bean) {
       // 在当前类查找Field和Method并注入
       for (Field f : clazz.getDeclaredFields()) {
           tryInjectProperties(def, clazz, bean, f);
       }
       for (Method m : clazz.getDeclaredMethods()) {
           tryInjectProperties(def, clazz, bean, m);
       }
       // 在父类查找Field和Method并注入
       Class<?> superClazz = clazz.getSuperclass();
       if (superClazz != null) {
           injectProperties(def, superClazz, bean);
       }
   }
   ```

3. **注入单个属性**:
   - 支持`@Value`注解注入属性值 
   - 支持`@Autowired`注解注入依赖Bean

## Bean的生命周期管理

1. **创建阶段**:
   - 首先创建`@Configuration`类
   - 其次创建`BeanPostProcessor`类
   - 最后创建普通Bean

2. **初始化阶段**:
   ```java
   void initBean(BeanDefinition def) {
       // 获取Bean实例或被代理的原始实例
       final Object beanInstance = getProxiedInstance(def);
       
       // 调用init方法
       callMethod(beanInstance, def.getInitMethod(), def.getInitMethodName());
       
       // 调用BeanPostProcessor.postProcessAfterInitialization()
       //...
   }
   ```

3. **销毁阶段**:
   ```java
   public void close() {
       logger.info("Closing {}...", this.getClass().getName());
       this.beans.values().forEach(def -> {
           final Object beanInstance = getProxiedInstance(def);
           callMethod(beanInstance, def.getDestroyMethod(), def.getDestroyMethodName());
       });
       this.beans.clear();
       logger.info("{} closed.", this.getClass().getName());
       ApplicationContextUtils.setApplicationContext(null);
   }
   ```

## 特殊功能实现

1. **@Primary支持**:
   当存在多个同类型的Bean时，优先选择标注了@Primary的Bean:
   ```java
   public BeanDefinition findBeanDefinition(Class<?> type) {
       //...
       if (primaryDefs.size() == 1) {
           return primaryDefs.get(0);
       }
       //...
   }
   ```

2. **@Order支持**:
   通过`@Order`注解控制Bean的优先级顺序:
   ```java
   int getOrder(Class<?> clazz) {
       Order order = clazz.getAnnotation(Order.class);
       return order == null ? Integer.MAX_VALUE : order.value();
   }
   ```

3. **BeanPostProcessor扩展点**:
   提供了Bean处理器机制，支持对Bean进行自定义扩展处理，如AOP实现。

## 总结

MiniSpring的IOC容器实现了Spring核心功能的简化版本，主要包括：
1. 基于注解的Bean定义和扫描
2. 多种依赖注入方式的支持
3. Bean完整生命周期的管理
4. 特殊功能如@Primary、@Order等的支持
5. 通过BeanPostProcessor实现的扩展点机制

