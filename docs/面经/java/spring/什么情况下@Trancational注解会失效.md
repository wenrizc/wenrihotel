
`@Transactional` 注解失效是一个在实际开发中非常常见、也极易出错的问题。几乎所有失效的场景，其根本原因都可以归结为一点：调用`@Transactional`注解标注的方法时，没有经过Spring生成的代理对象，而是直接通过原始对象调用的。

`@Transactional`是基于Spring AOP实现的，而AOP的实现又是基于动态代理。Spring在运行时会为标注了`@Transactional`的Bean生成一个代理对象。当外部调用这个Bean的public方法时，实际上是调用了代理对象的方法，代理对象会在调用真实方法前后开启、提交或回滚事务。如果调用链绕过了这个代理对象，那么事务自然就不会生效。


---

### 1. 注解用在非public方法上

这是最常见的一种错误。Spring的AOP代理只会拦截public方法的调用。如果你将`@Transactional`注解用在`protected`、`private`或包级私有的方法上，Spring不会为这些方法应用事务切面，事务也就不会生效。

原因：在Spring AOP的设计中，无论是JDK动态代理还是CGLIB代理，其拦截逻辑都是围绕公共方法展开的，这是为了避免违反Java语言的访问控制规则。

### 2. 类内部方法调用（自调用问题）

这是最典型也是最隐蔽的一种失效场景。当一个类中的非事务方法A，调用了同一个类中被`@Transactional`注解的事务方法B时，方法B的事务会失效。

Java

```
@Service
public class UserServiceImpl implements UserService {

    public void methodA() {
        // ... some logic ...
        this.methodB(); // 这里的事务会失效
    }

    @Transactional
    public void methodB() {
        // ... transactional logic ...
    }
}
```

原因：这里的调用是通过`this`引用进行的。`this`指向的是真实的、未被代理的原始对象，而不是Spring生成的代理对象。因此，这个调用直接发生在原始对象内部，完全绕过了AOP代理的拦截，事务切面自然也就无法织入。

如何解决自调用问题？

- 将事务方法B移到另一个独立的Service中，然后在方法A中注入并调用这个新的Service。这是最推荐、最符合设计原则的做法。
    
- 在该类中注入自身。通过`@Autowired`注入一个代理对象，然后通过代理对象来调用方法B。
    
- 使用`AopContext.currentProxy()`获取当前的代理对象，然后用它来调用方法B。
    

### 3. 异常被`try-catch`捕获且没有手动抛出

如果事务方法内部发生了异常，但这个异常被`try-catch`块完全捕获了，没有再向上抛出，那么Spring的事务管理器就无法感知到异常的发生，也就不会触发回滚。

Java

```
@Transactional
public void methodC() {
    try {
        // ... database operations that throw an exception ...
    } catch (Exception e) {
        // 异常被“吃掉”了，没有继续抛出
        // 事务不会回滚
    }
}
```

解决方法：在`catch`块中手动抛出一个运行时异常，或者通过`TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();`来手动标记事务需要回滚。

### 4. 默认的回滚规则不匹配

Spring的声明式事务默认只对`RuntimeException`（运行时异常）和`Error`进行回滚。如果你的方法抛出的是一个受检异常（Checked Exception，如`IOException`），默认情况下事务是不会回滚的。

原因：这是Spring团队基于一个普遍的观点做的设计：受检异常通常是业务逻辑中可预期的、可处理的异常，不一定需要回滚事务。

解决方法：在@Transactional注解中使用rollbackFor属性来明确指定需要回滚的异常类型。

@Transactional(rollbackFor = Exception.class)

### 5. 目标类或方法是`final`或`static`的

- 如果一个类被声明为`final`，CGLIB代理将无法为其创建子类，如果该类又没有实现接口，那么AOP代理将无法创建，事务失效。
    
- 如果一个方法被声明为`final`或`static`，CGLIB同样无法重写它，代理逻辑也就无法织入。静态方法属于类而不是对象，也无法被AOP代理。
    

### 6. 数据库引擎不支持事务

这是一个环境层面的问题，但非常关键。如果你的数据库表使用的是不支持事务的存储引擎，比如MySQL的MyISAM引擎，那么无论Spring层面如何配置，数据库本身都不会执行事务操作。

解决方法：确保所有需要事务的表都使用支持事务的存储引擎，如InnoDB。

### 7. Spring相关配置问题

- Bean没有被Spring容器管理：如果你的Service类没有被`@Service`、`@Component`等注解标记，或者没有被XML配置声明为Bean，Spring容器中根本就没有这个对象，自然也谈不上为它创建代理。
    
- 没有开启事务管理：在非Spring Boot的传统Spring项目中，需要通过`@EnableTransactionManagement`注解来显式地开启声明式事务支持。Spring Boot通常会自动配置好。
    
- AOP相关依赖缺失或配置错误。
    
