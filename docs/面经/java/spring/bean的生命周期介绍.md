

一个Bean的完整生命周期，大致可以分为四个主要阶段：

1. 实例化 (Instantiation)：Spring容器根据Bean的定义（BeanDefinition）创建出Bean的Java对象实例。
    
2. 属性填充 (Population)：Spring容器为Bean实例的属性注入值，即我们常说的依赖注入（DI）。
    
3. 初始化 (Initialization)：执行一系列的回调和处理，使Bean达到一个完全可用的状态。
    
4. 销毁 (Destruction)：在容器关闭或Bean被销毁时，执行清理工作。
    

1.实例化阶段

- 触发：容器启动后，会解析配置元信息，为每个Bean生成一个BeanDefinition。然后，容器会根据BeanDefinition来创建Bean。
    
- 核心操作：Spring通过反射调用Bean的构造函数，来创建一个原始的、未经任何配置的Java对象实例。此时，对象已经分配了内存空间，但其内部的属性都还是默认值。
    

2.属性填充阶段

- 核心操作：Spring检查BeanDefinition中的属性信息，通过依赖注入（DI）为Bean的成员变量赋值。这通常是通过`@Autowired`、`@Resource`等注解，利用反射来完成的。此时，Bean的依赖对象已经被注入进来。
    

3.初始化阶段

这是Bean生命周期中最为复杂，也是扩展点最多的一个阶段。在实例化和属性填充完成之后，Bean会经历一系列的初始化回调。

- 3.1 Aware接口回调
    
    - 如果Bean实现了特定的`Aware`接口，Spring会调用这些接口的方法，将相应的资源注入给Bean。这是一个让Bean能感知到自身在容器中环境的方式。
            
- 3.2 BeanPostProcessor前置处理
    
    - 扩展点：`BeanPostProcessor`接口的`postProcessBeforeInitialization`方法。
        
    - 操作：在执行任何初始化回调（如`@PostConstruct`或`afterPropertiesSet`）之前，Spring会调用所有已注册的`BeanPostProcessor`的这个方法。这给了我们在Bean正式初始化前，对Bean实例进行修改或包装的机会。
        
- 3.3 初始化方法执行
    
    - Spring会检查并执行Bean定义的初始化方法，让Bean执行自定义的启动逻辑。
        
    - 扩展点1：`@PostConstruct`注解。如果Bean的方法被这个JSR-250规范的注解标记，该方法会被调用。
        
    - 扩展点2：`InitializingBean`接口。如果Bean实现了这个接口，其`afterPropertiesSet`方法会被调用。
        
    - 扩展点3：自定义`init-method`。可以在`@Bean`注解或XML配置中指定一个自定义的初始化方法。
        
    - 执行顺序：`@PostConstruct`注解的方法 -> `afterPropertiesSet`方法 -> `init-method`指定的方法。
        
- 3.4 BeanPostProcessor后置处理
    
    - 扩展点：`BeanPostProcessor`接口的`postProcessAfterInitialization`方法。
        
    - 操作：在Bean的所有初始化逻辑执行完毕后，Spring会调用所有`BeanPostProcessor`的这个方法。
        
    - 核心应用：这是Spring AOP实现的关键所在。如果一个Bean需要被AOP代理，那么就是在这个步骤中，Spring会返回一个该Bean的代理对象，来替换掉原始的Bean实例。所以，我们最终从容器中获取到的Bean，可能是一个代理对象。
        

Bean准备就绪

经过以上所有步骤，一个完整、可用、可能已被代理的Bean就创建完成了。它会被放入Spring的单例缓存池（singletonObjects）中，等待被其他Bean注入或被应用程序调用。

4. 销毁阶段

当Spring容器关闭，或者一个Bean不再需要时，会触发销毁流程。

- 触发：容器调用`close()`方法。
    
- 核心操作：Spring会检查并执行Bean定义的销毁方法，让Bean有机会释放资源。
    
    - 扩展点1：`@PreDestroy`注解。如果Bean的方法被这个JSR-250规范的注解标记，该方法会在销毁前被调用。
        
    - 扩展点2：`DisposableBean`接口。如果Bean实现了这个接口，其`destroy`方法会被调用。
        
    - 扩展点3：自定义`destroy-method`。可以在`@Bean`注解或XML配置中指定一个自定义的销毁方法。
        
    - 执行顺序：`@PreDestroy`注解的方法 -> `destroy`方法 -> `destroy-method`指定的方法。
        

Spring Bean的生命周期是一个被精心设计的、充满扩展点的流程。它从实例化开始，经过属性填充，再到一系列复杂的初始化回调（包括Aware接口、BeanPostProcessor前后处理、以及多种初始化方法），最终成为一个可用的Bean。在容器关闭时，又会通过一系列销毁回调来完成清理工作。理解这个生命周期，不仅能帮助我们更好地使用Spring，也为我们理解Spring AOP、事务等高级功能的实现原理打下了坚实的基础。