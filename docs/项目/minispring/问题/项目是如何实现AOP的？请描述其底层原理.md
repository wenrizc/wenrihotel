
1. `UserController`调用`userService.getUserById(id)`
2. 调用进入ByteBuddy生成的代理类
3. 代理类将调用转发给`LoggingHandler`
4. `LoggingHandler`执行前置逻辑(记录时间、参数)
5. `LoggingHandler`通过反射调用原始`UserServiceImpl.getUserById`方法
6. `LoggingHandler`执行后置逻辑(计算耗时、记录结果)
7. 结果返回给`UserController`
MiniSpring框架实现AOP的核心是通过`BeanPostProcessor`接口，特别是`AnnotationProxyBeanPostProcessor`抽象类来实现的。整个实现过程是一个典型的代理模式应用，下面详细分析其工作原理：

## 1. 核心组件

### 1.1 BeanPostProcessor接口

```java
public interface BeanPostProcessor {
    // Bean实例化后，初始化前调用
    default Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean;
    }

    // Bean初始化后调用
    default Object postProcessAfterInitialization(Object bean, String beanName) {
        return bean;
    }

    // 在依赖注入前调用，解决代理对象注入问题
    default Object postProcessOnSetProperty(Object bean, String beanName) {
        return bean;
    }
}
```

### 1.2 AnnotationProxyBeanPostProcessor抽象类

```java
public abstract class AnnotationProxyBeanPostProcessor<A extends Annotation> implements BeanPostProcessor {
    Map<String, Object> originBeans = new HashMap<>(); // 存储原始Bean
    Class<A> annotationClass; // 要处理的注解类型
    
    public AnnotationProxyBeanPostProcessor() {
        this.annotationClass = getParameterizedType();
    }
    
    // 其他方法...
}
```

### 1.3 具体实现类

```java
// 处理@Around注解
public class AroundProxyBeanPostProcessor extends AnnotationProxyBeanPostProcessor<Around> {
}

// 处理@Transactional注解  
public class TransactionalBeanPostProcessor extends AnnotationProxyBeanPostProcessor<Transactional> {
}
```

## 2. AOP代理创建流程

### 2.1 扫描与注册

1. 应用启动时，`AnnotationConfigApplicationContext`构造函数扫描所有组件：
   ```java
   public AnnotationConfigApplicationContext(Class<?> configClass, PropertyResolver propertyResolver) {
       // 扫描组件
       final Set<String> beanClassNames = scanForClassNames(configClass);
       // 创建Bean定义
       this.beans = createBeanDefinitions(beanClassNames);
       
       // 优先创建BeanPostProcessor类型的Bean
       List<BeanPostProcessor> processors = this.beans.values().stream()
               .filter(this::isBeanPostProcessorDefinition)
               .sorted()
               .map(def -> (BeanPostProcessor) createBeanAsEarlySingleton(def))
               .collect(Collectors.toList());
       this.beanPostProcessors.addAll(processors);
       
       // 创建其他Bean、注入依赖等
       // ...
   }
   ```

### 2.2 代理创建过程

在`postProcessBeforeInitialization`方法中处理需要被代理的Bean：

```java
public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
    Class<?> beanClass = bean.getClass();
    
    // 检查类上是否有对应注解
    A anno = beanClass.getAnnotation(annotationClass);
    if (anno != null) {
        // 获取注解中的处理器名称
        String handlerName = (String) anno.annotationType().getMethod("value").invoke(anno);
        // 创建代理
        Object proxy = createProxy(beanClass, bean, handlerName);
        // 保存原始Bean，用于后续依赖注入
        originBeans.put(beanName, bean);
        return proxy;
    }
    return bean;
}
```

### 2.3 代理对象创建

```java
Object createProxy(Class<?> beanClass, Object bean, String handlerName) {
    // 获取应用上下文
    ConfigurableApplicationContext ctx = ApplicationContextUtils.getRequiredApplicationContext();
    
    // 查找处理器Bean
    BeanDefinition def = ctx.findBeanDefinition(handlerName);
    if (def == null) {
        throw new AopConfigException("处理器未找到");
    }
    
    // 获取处理器实例
    Object handlerBean = def.getInstance();
    if (handlerBean == null) {
        handlerBean = ctx.createBeanAsEarlySingleton(def);
    }
    
    // 创建代理
    if (handlerBean instanceof InvocationHandler handler) {
        return ProxyResolver.getInstance().createProxy(bean, handler);
    }
    
    throw new AopConfigException("处理器类型不正确");
}
```

### 2.4 使用ByteBuddy创建代理

```java
public <T> T createProxy(T bean, InvocationHandler handler) {
    Class<?> targetClass = bean.getClass();
    
    // 使用ByteBuddy创建子类作为代理
    Class<?> proxyClass = this.byteBuddy
            .subclass(targetClass, ConstructorStrategy.Default.DEFAULT_CONSTRUCTOR)
            .method(ElementMatchers.isPublic())
            .intercept(InvocationHandlerAdapter.of(
                    new InvocationHandler() {
                        @Override
                        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                            // 委托给传入的处理器
                            return handler.invoke(bean, method, args);
                        }
                    }))
            .make()
            .load(targetClass.getClassLoader())
            .getLoaded();
            
    // 创建代理实例
    Object proxy = proxyClass.getConstructor().newInstance();
    return (T) proxy;
}
```

## 3. 依赖注入过程中的特殊处理

为了确保在依赖注入时使用原始Bean而非代理对象（避免自我调用时AOP失效），实现了`postProcessOnSetProperty`方法：

```java
@Override
public Object postProcessOnSetProperty(Object bean, String beanName) {
    Object origin = this.originBeans.get(beanName);
    return origin != null ? origin : bean;
}
```

## 4. 调用链过程

当调用被代理对象的方法时：
1. 调用进入代理对象
2. 代理对象将调用委托给`InvocationHandler`处理器
3. 处理器执行前置逻辑
4. 处理器调用原始对象的方法
5. 处理器执行后置逻辑
6. 返回可能被修改的结果

例如`LoggingHandler`实现：

```java
@Component("loggingHandler")
public class LoggingHandler implements InvocationHandler {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 前置处理
        System.out.println("[" + LocalDateTime.now() + "] Before method: " + method.getName());
        
        // 执行原方法
        Object result = method.invoke(proxy, args);
        
        // 后置处理
        System.out.println("[" + LocalDateTime.now() + "] After method: " + method.getName() + ", result: " + result);
        
        // 可以修改返回值
        if (result instanceof String) {
            return "[Intercepted] " + result;
        }
        return result;
    }
}
```

通过这种设计，MiniSpring实现了一个简洁而功能完整的AOP支持，可以对任何具有特定注解的Bean进行方法拦截和增强。

