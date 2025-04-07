

通过对minispring代码库的分析，该框架主要实现了**单例(Singleton)模式**，尚未完整实现多作用域管理机制。

### 单例实现原理

MiniSpring框架中的单例管理主要通过以下方式实现：

1. **BeanDefinition存储实例**：每个Bean定义中包含一个instance字段保存Bean实例

```java
public class BeanDefinition {
    // ...其他属性
    private Object instance; // 存储Bean实例
    
    public Object getInstance() {
        return this.instance;
    }
    
    public void setInstance(Object instance) {
        this.instance = instance;
    }
}
```

1. **创建Bean时检查实例**：在创建Bean前检查是否已存在实例

```java
public Object getBean(String name) {
    BeanDefinition def = findBeanDefinition(name);
    if (def == null) {
        throw new NoSuchBeanDefinitionException(name);
    }
    // 如果实例已存在，直接返回
    Object beanInstance = def.getInstance();
    if (beanInstance != null) {
        return beanInstance;
    }
    
    // 如果实例不存在，则创建并保存
    beanInstance = createBeanAsEarlySingleton(def);
    return beanInstance;
}
```

1. **集中存储所有Bean定义**：将所有BeanDefinition统一存储在容器中

```java
public class AnnotationConfigApplicationContext implements ConfigurableApplicationContext {
    private final Map<String, BeanDefinition> beans;
    
    // 查找已注册的BeanDefinition
    @Override
    public BeanDefinition findBeanDefinition(String name) {
        return this.beans.get(name);
    }
}
```

目前框架没有显式的@Scope注解支持，所有Bean默认都是单例模式管理。

## 设计方案：增加作用域支持

以下是为MiniSpring添加作用域支持的设计方案：

### 1. 添加Scope注解

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Scope {
    String value() default "singleton";
}
```

### 2. 扩展BeanDefinition

```java
public class BeanDefinition {
    // 现有字段
    private final String name;
    private final Class<?> beanClass;
    private Object instance;
    
    // 新增作用域字段
    private String scope = "singleton"; // 默认单例
    
    public boolean isSingleton() {
        return "singleton".equals(this.scope);
    }
    
    public boolean isPrototype() {
        return "prototype".equals(this.scope);
    }
    
    public String getScope() {
        return this.scope;
    }
    
    public void setScope(String scope) {
        this.scope = scope;
    }
}
```

### 3. 解析@Scope注解

```java
protected Map<String, BeanDefinition> createBeanDefinitions(Set<String> classNameSet) {
    Map<String, BeanDefinition> defs = new HashMap<>();
    for (String className : classNameSet) {
        Class<?> clazz = Class.forName(className);
        Component component = ClassUtils.findAnnotation(clazz, Component.class);
        if (component != null) {
            // 创建BeanDefinition
            BeanDefinition def = new BeanDefinition(/* 现有参数 */);
            
            // 处理Scope注解
            Scope scope = ClassUtils.findAnnotation(clazz, Scope.class);
            if (scope != null) {
                def.setScope(scope.value());
            }
            
            // 注册BeanDefinition
            defs.put(def.getName(), def);
            
            // 处理@Configuration类...
        }
    }
    return defs;
}
```

### 4. 修改getBean方法支持不同作用域

```java
public Object getBean(String name) {
    BeanDefinition def = findBeanDefinition(name);
    if (def == null) {
        throw new NoSuchBeanDefinitionException(name);
    }
    
    // 对于单例Bean，复用已存在的实例
    if (def.isSingleton()) {
        Object beanInstance = def.getInstance();
        if (beanInstance != null) {
            return beanInstance;
        }
        // 创建单例实例并存储
        beanInstance = createBean(def);
        def.setInstance(beanInstance);
        return beanInstance;
    }
    // 对于原型Bean，每次都创建新实例
    else if (def.isPrototype()) {
        return createBean(def);
    }
    // 其他作用域(如request、session等)
    else {
        return resolveCustomScopedBean(def);
    }
}

// 创建Bean的通用方法
private Object createBean(BeanDefinition def) {
    // 创建Bean实例
    Object instance = createNewInstance(def);
    
    // 依赖注入
    injectBean(def, instance);
    
    // 初始化
    initializeBean(def, instance);
    
    return instance;
}
```

### 5. 支持自定义作用域

```java
public interface ScopeResolver {
    String getScopeName();
    Object resolveBean(BeanDefinition definition);
}

// 容器中注册作用域解析器
private final Map<String, ScopeResolver> scopeResolvers = new HashMap<>();

// 解析自定义作用域Bean
private Object resolveCustomScopedBean(BeanDefinition def) {
    ScopeResolver resolver = this.scopeResolvers.get(def.getScope());
    if (resolver == null) {
        throw new BeanCreationException("Unknown scope: " + def.getScope());
    }
    return resolver.resolveBean(def);
}
```

### 6. 请求和会话作用域实现(Web环境)

```java
public class RequestScopeResolver implements ScopeResolver {
    private final ThreadLocal<Map<String, Object>> requestBeans = ThreadLocal.withInitial(HashMap::new);
    
    @Override
    public String getScopeName() {
        return "request";
    }
    
    @Override
    public Object resolveBean(BeanDefinition definition) {
        Map<String, Object> beans = requestBeans.get();
        return beans.computeIfAbsent(definition.getName(), 
                name -> createBean(definition));
    }
    
    // 请求结束时清理资源
    public void removeRequestScope() {
        requestBeans.remove();
    }
}
```

通过这种设计，MiniSpring框架将能够支持单例、原型及自定义作用域，同时保持代码的简洁性和可扩展性。对于Web环境的请求和会话作用域，则需要与Web容器集成，通过适当的生命周期钩子管理作用域Bean的创建和销毁。