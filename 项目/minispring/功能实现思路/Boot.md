
## 1. 整体架构

Boot模块依赖于其他核心模块，形成完整的启动链：
- boot模块：提供应用入口和嵌入式容器
- web模块：提供Web MVC框架和请求处理
- context模块：提供IoC容器和Bean生命周期管理
- aop模块：提供代理和切面支持
- jdbc模块：提供数据库访问和事务管理

## 2. 核心启动类

`minispringApplication`类是整个启动过程的核心，提供静态`run`方法作为应用入口点：

```java
public static void run(String webDir, String baseDir, Class<?> configClass, String... args) throws Exception {
    new minispringApplication().start(webDir, baseDir, configClass, args);
}
```

## 3. 启动流程

启动过程主要包括以下步骤：

### 3.1 打印应用横幅
```java
protected void printBanner() {
    String banner = ClassPathUtils.readString("/banner.txt");
    banner.lines().forEach(System.out::println);
}
```

### 3.2 收集启动环境信息
记录Java版本、进程ID、用户名和工作目录等信息，便于诊断问题。

### 3.3 加载配置
通过`WebUtils.createPropertyResolver()`方法加载配置：
- 首先尝试加载`/application.yml`
- 如果未找到则尝试加载`/application.properties`
- 将配置封装为`PropertyResolver`对象

### 3.4 启动嵌入式Tomcat
```java
protected Server startTomcat(String webDir, String baseDir, Class<?> configClass, PropertyResolver propertyResolver) throws Exception {
    int port = propertyResolver.getProperty("${server.port:8080}", int.class);
    Tomcat tomcat = new Tomcat();
    tomcat.setPort(port);
    // 配置Web应用上下文
    Context ctx = (Context) tomcat.addWebapp("", new File(webDir).getAbsolutePath());
    WebResourceRoot resources = new StandardRoot((org.apache.catalina.Context) ctx);
    resources.addPreResources(new DirResourceSet(resources, "/WEB-INF/classes", new File(baseDir).getAbsolutePath(), "/"));
    ((org.apache.catalina.Context) ctx).setResources(resources);
    // 添加Servlet容器初始化器
    ((org.apache.catalina.Context) ctx).addServletContainerInitializer(new ContextLoaderInitializer(configClass, propertyResolver), Set.of());
    tomcat.start();
    return tomcat.getServer();
}
```

## 4. Servlet容器初始化

`ContextLoaderInitializer`类实现`ServletContainerInitializer`接口，在Tomcat启动时被调用：

```java
public void onStartup(Set<Class<?>> c, ServletContext ctx) throws ServletException {
    // 设置字符编码
    String encoding = propertyResolver.getProperty("${summer.web.character-encoding:UTF-8}");
    ctx.setRequestCharacterEncoding(encoding);
    ctx.setResponseCharacterEncoding(encoding);
    
    // 设置ServletContext
    WebMvcConfiguration.setServletContext(ctx);
    
    // 创建应用上下文
    ApplicationContext applicationContext = new AnnotationConfigApplicationContext(this.configClass, this.propertyResolver);
    
    // 注册过滤器和DispatcherServlet
    WebUtils.registerFilters(ctx);
    WebUtils.registerDispatcherServlet(ctx, this.propertyResolver);
}
```

## 5. 应用上下文创建

`AnnotationConfigApplicationContext`是IoC容器的核心实现，负责组件扫描和Bean生命周期管理：

1. **组件扫描**：扫描被`@Component`及其派生注解标记的类
2. **Bean定义创建**：为扫描到的类创建`BeanDefinition`
3. **依赖注入**：通过`@Autowired`、`@Value`等注解注入依赖
4. **生命周期管理**：调用初始化和销毁方法

## 6. Web环境配置

通过`WebUtils.registerDispatcherServlet`方法注册`DispatcherServlet`，用于处理HTTP请求：

```java
public static void registerDispatcherServlet(ServletContext servletContext, PropertyResolver propertyResolver) {
    // 创建DispatcherServlet实例
    // 添加到ServletContext
    // 配置URL映射
}
```

`DispatcherServlet`在初始化时扫描`@Controller`和`@RestController`，建立请求处理映射。

## 7. Bean后处理器机制

minispring通过`BeanPostProcessor`接口提供对Bean的扩展处理：

```java
public interface BeanPostProcessor {
    default Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean;
    }
    
    default Object postProcessAfterInitialization(Object bean, String beanName) {
        return bean;
    }
    
    default Object postProcessOnSetProperty(Object bean, String beanName) {
        return bean;
    }
}
```

特殊功能（如事务管理）通过实现这个接口来提供增强。

## 8. 优雅关闭

当应用关闭时，会调用容器的`close`方法，执行Bean的销毁流程：

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

## 总结

minispring的boot实现通过模仿Spring Boot的设计理念，提供了一套完整的应用启动和自动配置解决方案，主要特点包括：

- 嵌入式Tomcat容器提供HTTP服务
- 基于注解的IoC容器
- 自动配置加载机制
- 生命周期管理和依赖注入
- 面向Web的MVC框架

这种设计让开发者可以专注于业务逻辑开发，而不必关心底层基础设施的搭建和配置。

找到具有 1 个许可证类型的类似代码