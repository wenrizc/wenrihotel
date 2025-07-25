
一、事务的传播机制 (Transaction Propagation)

事务的传播机制，定义的是当一个带有事务的方法被另一个方法调用时，事务应该如何表现。比如，一个已经处于事务中的方法A，调用了另一个也配置了事务的方法B，那么方法B是应该加入方法A的现有事务，还是开启一个自己的新事务？这就是传播机制要解决的问题。

Spring通过`@Transactional`注解的`propagation`属性来控制，它有以下几种核心的传播行为：

1. REQUIRED (默认值)
    
    这是最常用的传播行为。如果当前已经存在一个事务，那么该方法就会加入到这个事务中执行；如果当前没有事务，则会为自己创建一个新的事务。这确保了方法总是在一个事务内执行。
    
2. REQUIRES_NEW
    
    这个行为表示，无论当前是否存在事务，该方法都会为自己创建一个全新的、独立的事务。如果外部已经存在事务，那么外部事务会被挂起，直到这个新事务执行完成。这个新事务有自己独立的提交和回滚，不受外部事务的影响。它常用于一些需要独立提交的核心日志记录、消息发送等场景，确保即使外部主业务回滚，这些操作也能成功。
    
3. NESTED
    
    如果当前存在一个事务，那么该方法会开启一个“嵌套事务”。这个嵌套事务依赖于外部事务，但它内部可以通过设置保存点（Savepoint）来实现局部回滚，而不会影响到外部事务。如果当前没有事务，它的行为就和REQUIRED一样，创建一个新事务。它与REQUIRES_NEW的主要区别在于，NESTED的提交依赖于外部事务的最终提交，而REQUIRES_NEW是完全独立的。
    
4. SUPPORTS
    
    如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续执行。它表示“支持”事务，但不是强制性的。
    
5. NOT_SUPPORTED
    
    以非事务方式执行操作。如果当前存在事务，则把当前事务挂起。
    
6. MANDATORY
    
    强制要求当前必须存在一个事务，如果不存在，则抛出异常。
    
7. NEVER
    
    强制要求当前不能存在事务，如果存在，则抛出异常。
    

二、事务出错时的回滚实现

Spring事务的回滚是基于AOP和异常机制实现的。当一个被`@Transactional`注解的方法被调用时，实际执行的是Spring通过动态代理生成的代理对象。

回滚的流程大致如下：

1. 开启事务：在目标方法执行之前，Spring的事务拦截器（`TransactionInterceptor`）会介入，根据`@Transactional`的配置，通过事务管理器（`PlatformTransactionManager`）开启一个数据库事务（例如，执行`conn.setAutoCommit(false)`）。
    
2. 执行业务逻辑：接着，代理对象会调用我们自己编写的原始的业务方法。
    
3. 异常捕获与处理：
    
    - 如果业务方法顺利执行完毕，没有抛出任何异常，那么事务拦截器会在方法执行后，通过事务管理器提交事务（执行`conn.commit()`）。
        
    - 如果业务方法在执行过程中抛出了异常，这个异常会被抛出到方法外部，最终被事务拦截器捕获。
        
4. 决定是否回滚：这是关键的一步。事务拦截器捕获到异常后，并不会立即回滚，而是会检查这个异常的类型，并根据`@Transactional`注解的配置来决定是否要执行回滚操作。
    
    - 默认规则：Spring的默认策略是，当捕获到的异常是`RuntimeException`（运行时异常）或`Error`时，它就会标记当前事务为“仅回滚”（rollback-only），并最终执行回滚操作（执行`conn.rollback()`）。
        
    - 对于受检异常（Checked Exception，即非`RuntimeException`的`Exception`子类），Spring默认是不会触发回滚的。这也是一个常见的坑，比如在方法里抛出了`IOException`或自定义的`BusinessException`，默认情况下事务不会回滚。
        
5. 自定义回滚规则：我们可以通过`@Transactional`注解的`rollbackFor`和`noRollbackFor`属性来覆盖默认规则。
    
    - `rollbackFor`：指定一个或多个异常类，当方法抛出这些类型的异常时，即使它们是受检异常，事务也会回滚。例如：`@Transactional(rollbackFor = Exception.class)`表示任何异常都会导致回滚。
        
    - `noRollbackFor`：指定一个或多个异常类，当方法抛出这些类型的异常时，即使它们是运行时异常，事务也不会回滚。
        

总结来说，Spring事务的传播机制决定了事务的边界和上下文，而它的回滚则是一个由AOP代理捕获异常，并根据预设规则（默认或自定义）来决定是否向数据库发出`rollback`指令的过程。理解这两点对于编写健壮的、事务正确的代码至关重要。