

MiniSpring的AOP实现是基于动态代理和Bean后处理器的组合设计，下面结合`ProxyResolver.java`代码详细说明整个流程：

## 1. AOP核心组件

### ProxyResolver类（代理创建器）
```java
public class ProxyResolver {
    // 单例实现
    public static ProxyResolver INSTANCE = null;
    
    public static ProxyResolver getInstance() {
        if (INSTANCE == null) {
            INSTANCE = new ProxyResolver();
        }
        return INSTANCE;
    }
    
    // 代理创建核心方法
    @SuppressWarnings("unchecked")
    public <T> T createProxy(T bean, InvocationHandler handler) {
        Class<?> targetClass = bean.getClass();
        // ...使用ByteBuddy创建代理
    }
}
```

## 2. AOP实现流程

### 阶段一：代理创建前准备
1. **Bean扫描**：容器启动时扫描带有`@Component`等注解的类
2. **注解检测**：`BeanPostProcessor`实现类检测带有AOP相关注解的Bean
3. **拦截器获取**：根据注解值找到对应的拦截处理器

### 阶段二：代理对象创建
从代码中可以看到关键流程：
```java
Class<?> proxyClass = this.byteBuddy
    .subclass(targetClass, ConstructorStrategy.Default.DEFAULT_CONSTRUCTOR)
    .method(ElementMatchers.isPublic()).intercept(InvocationHandlerAdapter.of(
        new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                // 委托给外部handler处理，传入原始bean
                return handler.invoke(bean, method, args);
            }
        }))
    .make().load(targetClass.getClassLoader()).getLoaded();
```

1. **使用ByteBuddy**：创建目标类的子类
2. **方法筛选**：匹配所有public方法进行拦截
3. **设置拦截器**：使用适配器将方法调用委托给传入的handler
4. **实例化代理**：创建并返回代理对象

### 阶段三：代理对象替换原Bean
```java
// 伪代码 - 在BeanPostProcessor实现中：
public Object postProcessAfterInitialization(Object bean, String beanName) {
    if (需要代理) {
        InvocationHandler handler = getHandler(bean); // 获取处理器
        Object proxy = ProxyResolver.getInstance().createProxy(bean, handler);
        return proxy; // 返回代理替换原bean
    }
    return bean;
}
```

### 阶段四：运行时方法拦截
当调用代理对象的方法时：
1. 调用被拦截并路由到`InvocationHandler.invoke()`方法
2. 在`invoke()`方法中可以执行前置、后置处理
3. 通过反射调用原始对象方法

## 3. 整体工作机制

1. **代理模式**：使用ByteBuddy创建子类代理
2. **装饰器模式**：在方法调用前后添加额外逻辑
3. **责任链模式**：多个拦截器可能组成调用链
4. **策略模式**：不同的`InvocationHandler`实现可以提供不同的拦截策略

这种设计让AOP与IoC容器紧密集成，在Bean生命周期中自然地进行切面增强，而无需开发者手动干预。