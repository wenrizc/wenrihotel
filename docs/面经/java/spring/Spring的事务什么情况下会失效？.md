
Spring的声明式事务（`@Transactional`）是基于AOP（面向切面编程）实现的，它通过动态代理技术在目标方法的执行前后动态地织入事务处理的逻辑（如开启事务、提交或回滚）。因此，所有导致AOP代理失效或者绕过代理的场景，都会导致事务失效。

下面我将具体分析几种常见的事务失效场景：

一、方法的访问权限和修饰符问题

这是最基础也最容易被忽略的一点。

1. 方法不是public的：
    
    Spring的AOP代理，无论是基于JDK动态代理还是CGLIB，默认都只会拦截public方法的调用。如果@Transactional注解用在protected、private或包可见性（default）的方法上，事务是不会生效的。这是因为代理对象在外部调用时无法直接访问这些非public方法，也就无法应用事务增强。
    
2. 方法被final修饰：
    
    被final修饰的方法无法被子类重写。对于使用CGLIB生成代理的类来说，它需要通过继承目标类并重写其方法来实现代理。如果方法是final的，CGLIB就无法重写它，自然也就无法加入事务逻辑。
    
3. 方法是static的：
    
    @Transactional注解是作用于对象实例的，而static方法属于类，不属于任何实例。AOP代理是作用于实例级别的，它无法代理一个静态方法。因此，静态方法上的@Transactional注解是无效的。
    

二、方法调用方式问题（绕过了代理）

这是最常见也最隐蔽的一类失效场景。

1. 同一个类中的方法内部调用：
    
    假设在同一个类中有methodA和methodB，methodA没有事务，而methodB有@Transactional注解。如果在methodA中直接通过this.methodB()的方式来调用methodB，那么methodB的事务会失效。
    
    原因是，this引用指向的是原始的目标对象，而不是Spring生成的代理对象。这个调用是对象内部的直接调用，完全没有经过AOP代理，Spring也就没有机会介入并开启事务。
    
    解决方案通常有三种：
    
    - 将`methodB`移到另一个Bean中，通过依赖注入的方式调用。
        
    - 在该类中注入自身的代理对象，通过代理对象来调用`methodB`。
        
    - 使用`AopContext.currentProxy()`获取当前代理对象，再用它来调用`methodB`。
        

三、异常处理问题

事务的回滚是基于异常的，如果异常处理不当，会导致事务无法按预期回滚。

1. 异常被try-catch捕获且没有重新抛出：
    
    如果在带有@Transactional注解的方法内部，使用try-catch捕获了可能导致回滚的异常，但catch块中没有将异常重新抛出，那么Spring的事务AOP切面就无法感知到异常的发生。在它看来，方法是正常执行完成的，因此它会提交事务而不是回滚。
    
2. 抛出的异常类型不正确：
    
    Spring的声明式事务默认只对RuntimeException（运行时异常）和Error（错误）进行回滚。对于受检异常（Checked Exception，即Exception的子类但非RuntimeException的子类），它默认是不会回滚的。
    
    如果你的业务逻辑中抛出的是一个自定义的受检异常，但又希望事务回滚，就必须在@Transactional注解中明确指定rollbackFor属性，例如 @Transactional(rollbackFor = Exception.class)。
    

四、数据库和事务传播行为问题

1. 数据库引擎不支持事务：
    
    这是一个底层原因。如果你的MySQL数据库表的存储引擎是MyISAM，而不是InnoDB，那么事务是不会生效的。因为MyISAM引擎本身就不支持事务。在排查问题时，这也是需要检查的一点。
    
2. 事务的传播行为（Propagation）配置不当：
    
    @Transactional注解有一个propagation属性，用来定义事务的传播行为。如果配置不当，也可能导致事务“看起来”失效了。
    
    例如，一个方法A调用了另一个方法B，如果B的传播行为设置为Propagation.NOT_SUPPORTED（以非事务方式运行，如果当前存在事务，则把当前事务挂起），那么即使A在一个事务中，B的数据库操作也不会被纳入A的事务管理。同理，Propagation.NEVER（以非事务方式运行，如果当前存在事务，则抛出异常）也会阻止事务的执行。
    

总结来说，排查Spring事务失效的问题，我通常会遵循一个思路：首先确认底层（数据库引擎）是否支持，然后检查代码层面，从方法的签名（`public`, 非`final`, 非`static`）到方法的调用方式（是否内部调用），再到异常处理逻辑（是否捕获、异常类型是否正确），最后再看事务的传播配置是否符合预期。理解AOP代理是定位这类问题的核心。

