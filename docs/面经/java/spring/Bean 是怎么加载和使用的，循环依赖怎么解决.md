
### Bean的加载与使用：Bean的生命周期

Spring Bean的生命周期描述了一个Bean从一个简单的Java类定义，到成为一个功能完备、可供应用程序使用的对象所经历的全部过程。这个过程可以概括为以下几个关键阶段：

#### 1. Bean的定义 (Bean Definition)

这个阶段是容器的“准备阶段”。Spring容器启动时，会读取应用程序提供的配置元数据。这些元数据可以来自于XML配置文件、Java注解（如`@Component`, `@Service`, `@Repository`）或Java Config（`@Configuration`类中的`@Bean`方法）。Spring会解析这些元数据，为每个Bean创建一个对应的`BeanDefinition`对象。你可以把`BeanDefinition`看作是创建Bean的“配方”或“蓝图”，它包含了Bean的类名、作用域（singleton、prototype等）、依赖关系、初始化方法等所有信息。

#### 2. Bean的实例化 (Instantiation)

当容器准备好`BeanDefinition`后，在某个时刻（对于单例Bean通常是容器启动时，对于原型Bean则是每次请求时），Spring会根据`BeanDefinition`来创建Bean的实例。这个过程通常是通过Java的反射机制调用类的构造函数来完成的。此时得到的只是一个“裸露”的对象，它的属性都还是默认值，依赖也尚未注入。

#### 3. Bean的属性填充 (Population)

实例化完成后，Spring会检查`BeanDefinition`中的依赖信息，并对Bean的属性进行填充。这就是我们熟知的“依赖注入”（DI）。Spring会根据`@Autowired`、`@Resource`等注解，或者XML配置，去容器中查找并注入该Bean所依赖的其他Bean。

#### 4. Bean的初始化 (Initialization)

这是Bean生命周期中最为复杂、扩展点最多的一个阶段。在属性填充完毕后，Bean还不能立即使用，需要经过一系列的初始化操作，使其达到可用的最终状态。这个过程大致包含以下步骤：

- 执行Aware接口：如果Bean实现了如`BeanNameAware`、`BeanFactoryAware`、`ApplicationContextAware`等接口，Spring会调用相应的方法，将Bean的ID、Bean工厂、应用上下文等容器自身的信息注入给Bean。
    
- BeanPostProcessor前置处理：Spring会调用所有已注册的`BeanPostProcessor`的`postProcessBeforeInitialization`方法。这是一个非常重要的扩展点，允许我们对Bean进行自定义的加工。
    
- 执行初始化方法：如果Bean实现了`InitializingBean`接口，Spring会调用其`afterPropertiesSet`方法。如果Bean定义了`init-method`（或使用`@PostConstruct`注解），该方法也会在此时被调用。
    
- BeanPostProcessor后置处理：最后，Spring会调用所有`BeanPostProcessor`的`postProcessAfterInitialization`方法。Spring的AOP功能，即为Bean创建代理对象，通常就是在这个步骤中完成的。经过这一步后返回的对象，可能已经不是原始实例，而是被代理过的对象了。
    

#### 5. Bean的就绪与使用 (Ready for Use)

经过完整的初始化阶段后，这个Bean就是一个功能完备的对象了。它被存放在Spring的单例缓存池中（对于单例Bean而言），随时可以被应用程序的其他部分注入和使用。

#### 6. Bean的销毁 (Destruction)

当Spring容器关闭时，它会负责销毁其管理的所有单例Bean。销毁过程同样提供了回调机制：

- 如果Bean实现了`DisposableBean`接口，Spring会调用其`destroy`方法。
    
- 如果Bean定义了`destroy-method`（或使用`@PreDestroy`注解），该方法会被调用。
    

---

### 二、循环依赖的解决方案

循环依赖是指两个或多个Bean互相持有对方的引用，形成了一个闭环，比如A依赖B，同时B又依赖A。这是一个在大型项目中可能遇到的棘手问题。

首先，需要明确Spring解决循环依赖的**两个前提条件**：

1. 只针对**单例（Singleton）作用域**的Bean。对于原型（Prototype）作用域的Bean，Spring无法解决，会直接抛出异常。
    
2. 只针对**字段注入或Setter方法注入**。对于构造函数注入，Spring也无法解决，同样会抛出异常，因为对象的创建和依赖的注入耦合在了一起，无法解开。
    

Spring解决循环依赖的核心，在于它内部精巧设计的**三级缓存**机制。

- 一级缓存 singletonObjects：
    
    也叫“单例池”，用于存放已经完全初始化好的Bean。这是一个Map<String, Object>，里面的Bean可以直接使用。
    
- 二级缓存 earlySingletonObjects：
    
    用于存放早期暴露的Bean。这些Bean已经被实例化，但尚未完成属性填充和初始化。这个缓存的存在，是为了打破循环。一旦某个Bean被创建，它的一个早期引用就会被放入这个缓存，即使它还不完整。
    
- 三级缓存 singletonFactories：
    
    这是一个工厂缓存，存放的是用于创建早期Bean的ObjectFactory。这个ObjectFactory是一个函数式接口，当被调用时，它会返回一个Bean对象。三级缓存是解决AOP代理下循环依赖的关键。如果一个Bean需要被AOP代理，那么这个工厂返回的就是代理后的对象，否则返回原始对象。
    

#### 解决流程（以A依赖B，B依赖A为例）：

1. **创建A**：`getSingleton("A")`被调用。首先检查一级缓存，没有；检查二级缓存，没有。
    
2. **实例化A**：Spring发现无法从缓存获取，于是开始创建A。它先通过反射实例化一个“裸露”的A对象。
    
3. **早期暴露A**：为了解决可能的循环依赖，Spring并不会立即去初始化A，而是将一个能够获取A对象（或者A的代理对象）的工厂（`ObjectFactory`）放入**三级缓存**`singletonFactories`中。
    
4. **填充A的属性**：Spring开始为A注入属性，此时发现A依赖于B。于是，Spring会去获取B，即调用`getSingleton("B")`。
    
5. **创建B**：获取B的流程和A一样。检查一级、二级缓存，都没有。于是Spring开始创建B，实例化一个“裸露”的B对象，并将其对应的`ObjectFactory`也放入**三级缓存**。
    
6. **填充B的属性**：Spring为B注入属性，此时发现B依赖于A。于是，Spring再次去获取A，即调用`getSingleton("A")`。
    
7. **打破循环的关键点**：
    
    - 这一次获取A，Spring在一级缓存中找不到（因为A还没完全初始化）。
        
    - 在二级缓存中也找不到。
        
    - 但是，它在**三级缓存**中找到了用于创建A的`ObjectFactory`。
        
    - Spring会调用这个工厂的`getObject()`方法。此时，如果A需要AOP代理，这个工厂就会返回A的代理对象；如果不需要，就返回原始的A对象。
        
    - 然后，这个早期暴露的A对象被放入**二级缓存**`earlySingletonObjects`中，并从三级缓存中移除对应的工厂。
        
    - 这个早期暴露的A对象被返回，并成功注入到B的属性中。
        
8. **完成B的创建**：B的依赖被成功注入，接着B完成后续的初始化过程。初始化完成后，完整的B对象被放入**一级缓存**`singletonObjects`中，并从二级、三级缓存中移除。
    
9. **完成A的创建**：现在，B对象已经可以被获取了。Spring将这个完整的B对象注入到之前等待的A对象中。A的依赖也全部注入完毕，接着A完成后续的初始化过程。最终，完整的A对象也被放入**一级缓存**。
    

至此，A和B的循环依赖被成功解决，两者都成为了功能完备的Bean。这个过程的核心就是通过二级和三级缓存，提前暴露了一个“半成品”的Bean引用，从而打破了循环等待的僵局。
