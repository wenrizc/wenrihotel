
Spring的启动过程，本质上就是其IoC容器（`ApplicationContext`）的初始化和其中所有单例Bean的创建、配置和装配的过程。我可以将其分解为以下几个核心阶段来阐述。

整个过程的核心是围绕`ApplicationContext`的`refresh()`方法展开的，我们以一个常见的`AnnotationConfigApplicationContext`为例。

第一阶段：容器的准备与初始化

当执行`new AnnotationConfigApplicationContext(AppConfig.class)`时，构造函数内部会做两件关键的事情：

1. 创建一个`DefaultListableBeanFactory`：这是Spring IoC容器最底层的实现，是真正存储和管理Bean的地方。它内部包含了用于存储Bean定义（`BeanDefinition`）和单例Bean实例的Map等结构。
    
2. 初始化`BeanDefinitionReader`和`ClassPathBeanDefinitionScanner`：前者用于读取XML等配置，后者用于扫描指定包路径下的注解（如`@Component`, `@Service`等）。
    

在这一阶段，容器的基本骨架已经搭建好了，但里面还没有任何我们自己定义的Bean。

第二阶段：加载与解析BeanDefinition

这是Spring搜集所有需要管理的Bean信息的阶段。

1. 扫描与解析：Spring会使用`ClassPathBeanDefinitionScanner`去扫描我们在配置类（如`AppConfig`）中指定的包路径。
    
2. 生成BeanDefinition：当扫描器找到带有`@Component`、`@Configuration`等注解的类时，它并不会立刻创建这个类的实例。相反，它会为每个符合条件的类生成一个对应的`BeanDefinition`对象。
    
3. 什么是`BeanDefinition`：这是一个非常重要的概念。它相当于一个Bean的“图纸”或“配方”，里面包含了创建这个Bean所需要的所有元数据，比如：
    
    - Bean的类名（`beanClassName`）。
        
    - 作用域（`scope`），如singleton或prototype。
        
    - 是否懒加载（`lazyInit`）。
        
    - 依赖的其他Bean（`dependsOn`）。
        
    - 属性值（`propertyValues`）等等。
        
4. 注册`BeanDefinition`：所有生成的`BeanDefinition`都会被注册到一个`DefaultListableBeanFactory`内部一个名为`beanDefinitionMap`的`Map<String, BeanDefinition>`中。
    

完成这一阶段后，Spring就知道了它需要管理哪些Bean，以及如何去创建它们，但此时还没有任何一个我们自己定义的Bean被实例化。

第三阶段：BeanFactory的后处理

在实例化任何Bean之前，Spring提供了一个扩展点，允许我们对`BeanDefinition`进行修改或补充，这就是`BeanFactoryPostProcessor`。

1. 执行`BeanFactoryPostProcessor`：Spring会找出容器中所有实现了`BeanFactoryPostProcessor`接口的Bean，并执行它们的`postProcessBeanFactory`方法。
    
2. 典型应用：最经典的例子就是`PropertySourcesPlaceholderConfigurer`。它的作用就是扫描所有`BeanDefinition`，并将属性值中的占位符（如`${jdbc.url}`）替换成配置文件中真实的值。Spring Boot中的`@ConfigurationProperties`的绑定也是在这个阶段附近完成的。
    

这个阶段好比是在“图纸”最终确定前，进行最后一次的修改和完善。

第四阶段：Bean的实例化与生命周期管理

这是整个启动过程中最核心、最复杂的部分。Spring会遍历`beanDefinitionMap`，开始实例化所有非懒加载的单例Bean。这个过程遵循一个严格的生命周期：

1. 实例化（Instantiation）：Spring通过反射调用Bean的构造函数，创建出一个原始的、不包含任何依赖的对象实例。此时，如果发生循环依赖，三级缓存机制会介入。
    
2. 属性填充（Populate）：Spring会分析这个Bean依赖的其他Bean（即带有`@Autowired`等注解的属性），然后从容器中获取这些依赖Bean的实例，通过反射注入到当前对象实例中。
    
3. 初始化（Initialization）：这是Bean变得可用的最后一步，也是一个包含多个子步骤的过程：
    
    - 执行Aware接口：如果Bean实现了`BeanNameAware`、`BeanFactoryAware`、`ApplicationContextAware`等接口，Spring会调用相应的方法，将Bean的ID、BeanFactory、ApplicationContext等注入进来。
        
    - 执行`BeanPostProcessor`的`postProcessBeforeInitialization`方法：这是Bean初始化回调（如`@PostConstruct`）之前的一个扩展点。
        
    - 执行初始化回调：如果Bean定义了`@PostConstruct`注解的方法，或者实现了`InitializingBean`接口的`afterPropertiesSet`方法，它们会在此被调用。
        
    - 执行`BeanPostProcessor`的`postProcessAfterInitialization`方法：这是初始化之后最重要的一个扩展点。AOP就是在这里实现的。如果一个Bean需要被代理（比如它有`@Transactional`或`@Async`注解），一个名为`AnnotationAwareAspectJAutoProxyCreator`的`BeanPostProcessor`会在这里将原始的Bean对象包装成一个代理对象，并返回这个代理对象。因此，最终存入单例池的可能是这个代理对象。
        
4. 注册到单例池：经过以上所有步骤后，一个完整的、可用的Bean就诞生了。它会被放入第一级缓存`singletonObjects`中，我们称之为“单例池”。后续任何对该Bean的请求都将直接从这个缓存中获取。
    

第五阶段：启动完成与后续工作

当所有非懒加载的单例Bean都成功创建并初始化后，Spring容器的启动过程就基本完成了。

1. 发布`ContextRefreshedEvent`事件：Spring会发布一个容器刷新完成的事件。一些需要等整个容器启动完毕后再执行逻辑的Bean（比如实现了`ApplicationListener<ContextRefreshedEvent>`的Bean）会监听到这个事件并开始执行。
    
2. 执行`CommandLineRunner`和`ApplicationRunner`：在Spring Boot环境中，还会查找并执行所有实现了`CommandLineRunner`或`ApplicationRunner`接口的Bean。这通常用于执行一些应用启动后的初始化任务。
    

Spring的启动可以概括为：准备容器 -> 扫描并加载Bean的定义信息（`BeanDefinition`） -> 执行后处理器对定义信息进行加工 -> 依次实例化、填充、初始化所有单例Bean（这个过程会应用AOP等）-> 最终完成并通知应用可以开始工作。

