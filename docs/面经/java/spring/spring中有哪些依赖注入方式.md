
依赖注入的核心思想是一个组件（比如一个Service类）不再由自己去创建和管理它所依赖的另一个组件（比如一个Repository类），而是将被依赖的组件的控制权交给外部容器（即Spring IoC容器）。组件只需要声明它需要什么，容器就会在运行时将对应的实例注入给它。这样做的好处是极大地降低了组件之间的耦合度，使得代码更易于测试和维护。

Spring主要提供了以下三种依赖注入的方式：

1.构造函数注入 (Constructor Injection)

这是Spring团队目前最推荐的注入方式。

- 实现方式：
    
    在类的构造函数中声明需要依赖的组件作为参数。Spring容器在创建这个类的Bean实例时，会自动查找并传入相应的依赖Bean。
    

```Java
@Service
public class MyService {

    private final UserRepository userRepository;

    // 如果只有一个构造函数，@Autowired注解可以省略
    public MyService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    // ... 业务方法
}
```

- 优点：
    
    1. 保证依赖不可变：可以将依赖字段声明为 final，确保一旦注入就不会被修改，增强了程序的稳定性和线程安全。
        
    2. 保证对象完整性：依赖在对象构造时必须被提供，这意味着对象在创建后一定是处于一个完整、可用的状态，不会出现依赖为null的情况。
        
    3. 清晰地暴露依赖关系：类的所有必要依赖都在构造函数中一目了然。如果构造函数参数过多，也提醒我们这个类可能违反了单一职责原则，需要重构。
        
- 缺点：
    
    1. 不支持循环依赖：如果类A的构造函数需要B，同时类B的构造函数也需要A，Spring无法解决这种构造器循环依赖，会抛出异常。但这通常被认为是一个优点，因为它能帮助我们及早发现设计上的问题。
        
    2. 代码可能稍显冗长。
        

2.Setter方法注入 (Setter Injection)

- 实现方式：
    
    在类中为需要注入的依赖提供一个公开的setter方法，并通常在该方法上标注@Autowired。Spring容器会先调用类的无参构造函数创建实例，然后再调用setter方法来注入依赖。
    


```Java
@Service
public class MyService {

    private UserRepository userRepository;

    @Autowired
    public void setUserRepository(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    // ... 业务方法
}
```

- 优点：
    
    1. 适用于可选依赖：对于非必需的依赖，可以使用setter注入，即使没有提供这个依赖，对象本身也能被创建。
        
    2. 可以解决循环依赖：Spring可以通过先实例化Bean（此时依赖为null），然后再通过setter方法相互注入，从而解决循环依赖问题。
        
- 缺点：
    
    1. 无法保证对象完整性：对象在被创建后，其依赖可能仍是null，只有在setter方法被调用后才完整。
        
    2. 依赖可变：setter方法可以被多次调用，导致对象的依赖在运行时被改变，这可能会引入不确定性。
        

3.字段注入 (Field Injection)

这是过去非常流行，但现在已不被推荐的注入方式。

- 实现方式：
    
    直接在类的成员字段上使用@Autowired注解。
    

``` Java
@Service
public class MyService {

    @Autowired
    private UserRepository userRepository;

    // ... 业务方法
}
```

- 优点：
    
    1. 代码最简洁：写法非常简单，省去了构造函数或setter方法的代码。
        
- 缺点：
    
    1. 与IoC容器强耦合：被注入的字段通常是私有的，这意味着如果不通过Spring容器，就无法方便地创建这个类的实例并为其设置依赖（尤其是在单元测试中），必须借助反射。
        
    2. 隐藏依赖关系：类的依赖关系被隐藏在实现细节中，从类的公开契约（构造函数和方法）上看不出它需要哪些依赖。
        
    3. 无法保证依赖不可变：不能将字段声明为 final。
        
    4. 容易掩盖设计问题：它同样可以解决循环依赖，但这往往会掩盖代码中存在的坏味道。
        
