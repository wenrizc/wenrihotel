## 1. Spring 事务是什么

Spring 事务本质上是对底层事务能力的一层**统一抽象**。

底层真正执行事务的，仍然是具体资源或框架，例如：

- JDBC 连接的本地事务。
- JPA / Hibernate 的事务。
- JTA 分布式事务。

Spring 做的事情主要有两类：

1. **统一事务编程模型**：开发者不需要直接操作各种底层 API，而是通过统一的事务接口和注解使用事务。
2. **把事务边界从业务代码里抽出来**：让“开启事务、提交事务、异常回滚”这些横切逻辑交给 Spring AOP 和事务管理器处理。

所以一句话总结：

**Spring 事务不是自己发明了一套数据库事务，而是把不同事务资源统一封装成一致的使用方式。**

## 2. Spring 为什么要做事务抽象

如果没有 Spring，业务代码通常要显式写：

```java
Connection conn = dataSource.getConnection();
try {
    conn.setAutoCommit(false);
    // 执行业务 SQL
    conn.commit();
} catch (Exception e) {
    conn.rollback();
    throw e;
} finally {
    conn.close();
}
```

这类代码有几个明显问题：

- 模板代码重复。
- 容易漏掉 `rollback`、`close`。
- 不同技术栈的事务写法不统一。
- 事务边界散落在业务代码中，可维护性差。

Spring 事务抽象的核心价值，就是把这些模板逻辑收口到框架层。

## 3. Spring 事务的两种使用方式

### 3.1 编程式事务

编程式事务是指业务代码显式控制事务。

常见方式：

- 直接使用 `PlatformTransactionManager`
- 使用 `TransactionTemplate`

示意：

```java
transactionTemplate.execute(status -> {
    orderDao.insert(order);
    inventoryDao.deduct(order.getSkuId(), order.getCount());
    return null;
});
```

优点：

- 事务边界非常明确。
- 对复杂分支控制更灵活。

缺点：

- 侵入业务代码。
- 不如声明式事务简洁。

### 3.2 声明式事务

声明式事务是 Spring 中最常见的方式，典型就是 `@Transactional`。

开发者只需要声明：

- 哪个类或方法需要事务。
- 事务传播行为是什么。
- 隔离级别是什么。
- 出现哪些异常要回滚。

至于“何时开启、何时提交、何时回滚”，交给 Spring 处理。

这也是大多数面试题里说的“Spring 事务”。

## 4. Spring 事务体系中的核心接口

### 4.1 `PlatformTransactionManager`

这是 Spring 事务抽象的核心接口。

它主要负责三件事：

- 获取事务。
- 提交事务。
- 回滚事务。

典型实现包括：

- `DataSourceTransactionManager`
- `JpaTransactionManager`
- `JtaTransactionManager`

不同实现对应不同底层资源，但业务代码上层使用方式可以一致。

### 4.2 `TransactionDefinition`

它描述事务的“定义”，主要包括：

- 传播行为 `propagation`
- 隔离级别 `isolation`
- 超时时间 `timeout`
- 是否只读 `readOnly`

### 4.3 `TransactionStatus`

它表示当前事务运行状态，例如：

- 当前事务是否是新事务。
- 是否已标记为回滚。
- 是否已经完成。

### 4.4 `TransactionAttribute`

它是在 `TransactionDefinition` 基础上扩展出来的，额外包含：

- 回滚规则，例如 `rollbackFor`
- 不回滚规则，例如 `noRollbackFor`

`@Transactional` 最终会被解析成这类事务属性对象。

## 5. `@Transactional` 注解详细介绍

根据 Spring 官方文档，`@Transactional` 只是**事务元数据**，真正让它生效的是运行时的事务基础设施。

### 5.1 注解可以标在哪里

`@Transactional` 可以标在：

- 类上。
- 方法上。

通常建议标在**具体类的方法上**，而不是只依赖接口声明。

这是因为 Spring 官方也明确建议：

- 优先标注在 concrete class methods 上。
- 仅依赖接口上的注解，在某些代理或织入模式下可能被忽略。

### 5.2 类级别和方法级别的优先级

如果类上和方法上同时标了 `@Transactional`，通常是：

- **方法级别优先于类级别**

也就是说，类上可以提供默认事务策略，方法上再局部覆盖。

### 5.3 默认配置

Spring 官方文档给出的 `@Transactional` 默认语义主要是：

- 传播行为：`Propagation.REQUIRED`
- 隔离级别：`Isolation.DEFAULT`
- 读写事务：`readOnly = false`
- 超时：底层事务系统默认值
- 回滚规则：`RuntimeException` 和 `Error` 默认回滚，受检异常默认不回滚

这一点是面试高频考点。

## 6. `@Transactional` 常用属性详解

### 6.1 `propagation`

事务传播行为，决定“当前方法被调用时，如果上下文里已经有事务，该怎么办”。

Spring 常见的七种传播行为如下：

| 传播行为 | 含义 | 常见使用场景 |
| --- | --- | --- |
| `REQUIRED` | 有事务就加入，没有就新建 | 最常用，业务主流程默认选择 |
| `REQUIRES_NEW` | 无论外部是否有事务，都新建一个事务，并挂起外部事务 | 审计日志、补偿记录、独立提交 |
| `SUPPORTS` | 有事务就加入，没有就以非事务方式执行 | 可有可无的读操作 |
| `NOT_SUPPORTED` | 以非事务方式执行，挂起当前事务 | 明确不希望被事务包裹的逻辑 |
| `MANDATORY` | 必须运行在已有事务中，否则抛异常 | 强依赖上层事务的方法 |
| `NEVER` | 必须在无事务环境执行，否则抛异常 | 明确禁止进入事务的方法 |
| `NESTED` | 如果存在事务，则在当前事务中创建嵌套事务；否则行为类似 `REQUIRED` | 需要部分回滚的 JDBC 场景 |

#### 6.1.1 `REQUIRED`

这是默认值。

含义是：

- 当前有事务，就加入当前事务。
- 当前没有事务，就新建一个事务。

因此它最适合作为大多数业务方法的默认传播行为。

#### 6.1.2 `REQUIRES_NEW`

它会：

- 挂起当前事务。
- 开一个全新的事务。
- 新事务执行完之后，再恢复外部事务。

常见用途：

- 主业务失败要回滚，但日志、审计、补偿记录希望独立提交。

但要注意：

- 它会增加数据库连接占用。
- 嵌套过多时容易加大系统复杂度。

#### 6.1.3 `NESTED`

`NESTED` 和 `REQUIRES_NEW` 容易混淆。

区别在于：

- `REQUIRES_NEW` 是**真正开启独立新事务**。
- `NESTED` 通常依赖 **savepoint（保存点）**，是在同一个外层事务内部做部分回滚。

所以：

- `NESTED` 更像“事务里的子作用域”。
- `REQUIRES_NEW` 更像“完全独立的新事务”。

### 6.2 `isolation`

事务隔离级别，控制并发事务之间的可见性和一致性。

常见值包括：

- `DEFAULT`
- `READ_UNCOMMITTED`
- `READ_COMMITTED`
- `REPEATABLE_READ`
- `SERIALIZABLE`

这些隔离级别本质上仍依赖底层数据库实现。

常见问题：

- 脏读
- 不可重复读
- 幻读

Spring 只是把隔离级别声明出来，真正执行的是数据库。

### 6.3 `timeout`

指定事务超时时间。

超时的作用是：

- 防止事务长时间占锁。
- 避免慢 SQL 或外部调用导致事务一直不结束。

但需要注意：

- 它的生效方式依赖具体事务管理器和底层资源实现。

### 6.4 `readOnly`

`readOnly = true` 表示当前事务逻辑上是只读事务。

它的作用更像是一个**提示（hint）**，Spring 官方文档也明确说了：

- 它允许底层事务系统做相应优化。
- 但并不保证一定禁止写操作。

所以：

- 它不是数据库层面的强硬只读开关。
- 更常见的价值是性能优化和语义表达。

### 6.5 `rollbackFor`

指定哪些异常类型出现时必须回滚。

例如：

```java
@Transactional(rollbackFor = Exception.class)
```

常见用途：

- 希望受检异常也回滚。

因为默认情况下：

- `RuntimeException` 和 `Error` 回滚。
- checked exception 默认不回滚。

### 6.6 `noRollbackFor`

指定哪些异常出现时**不要回滚**。

例如某些业务异常虽然要抛给上层，但希望事务仍提交。

### 6.7 `transactionManager`

当系统里有多个事务管理器时，可以显式指定使用哪个事务管理器。

例如：

```java
@Transactional(transactionManager = "orderTxManager")
```

这在多数据源场景里非常重要。

### 6.8 `label`

Spring 5.3 之后，`@Transactional` 增加了 `label` 属性。

它本质上是给事务打标签，供具体事务管理器做实现相关的处理或描述性标记。

工程里用得不如前面几个属性多，但面试时可以知道它的存在。

## 7. Spring 事务的常见注解和相关配置

除了 `@Transactional` 本身，Spring 声明式事务常见还会涉及下面几个注解或配置。

### 7.1 `@EnableTransactionManagement`

这个注解用于开启基于注解的事务管理。

它的作用是把 Spring 事务相关的基础设施注册进容器，例如：

- `TransactionAttributeSource`
- `TransactionInterceptor`
- `BeanFactoryTransactionAttributeSourceAdvisor`

也就是说，没有开启事务管理，仅仅写了 `@Transactional` 并不会自动生效。

### 7.2 `@TransactionalEventListener`

这个注解不是用来开启事务，而是让事件监听器绑定到事务阶段。

例如：

- `AFTER_COMMIT`
- `AFTER_ROLLBACK`
- `AFTER_COMPLETION`

常见用途：

- 只有事务提交成功之后，才发消息、发通知、更新缓存。

### 7.3 `@Async`

`@Async` 不是事务注解，但它经常和事务一起出问题。

因为 Spring 事务通常绑定在当前线程，而 `@Async` 会切到其他线程执行，所以：

- 事务上下文通常不会自动传过去。

这也是事务失效场景的高频来源之一。

## 8. Spring 事务常见失效原因

很多人以为“加了 `@Transactional` 就一定有事务”，实际完全不是。

### 8.1 没有经过代理对象调用

Spring 官方文档明确说明：

- 默认模式是 `proxy`
- 只有通过代理进入的方法调用才会被拦截

因此：

- **同类内部自调用**不会生效

例如：

```java
@Service
public class OrderService {

    public void create() {
        this.saveOrder(); // 不经过代理
    }

    @Transactional
    public void saveOrder() {
        // ...
    }
}
```

这里的 `this.saveOrder()` 没有经过 Spring 代理，事务不会生效。

这也是最常见失效原因。

### 8.2 方法不是 Spring Bean 上的方法

如果对象不是 Spring 容器管理的 Bean，例如：

- 自己 `new` 出来的对象
- 没有被组件扫描到的类

那么它当然没有事务代理，也就不会生效。

### 8.3 方法签名不满足代理增强条件

常见问题包括：

- `private` 方法不能被代理增强。
- `static` 方法不能被代理增强。
- `final` 方法在 CGLIB 场景下不能被覆盖增强。
- 如果是 JDK 动态代理，只能拦截接口方法调用。

工程实践里最好把事务放在：

- **对外暴露的 `public` 业务方法**

这样最稳妥。

### 8.4 异常被吞掉了

Spring 事务的回滚判断，核心看“方法是否把异常抛出了代理边界”。

例如：

```java
@Transactional
public void createOrder() {
    try {
        orderDao.insert(...);
        int x = 1 / 0;
    } catch (Exception e) {
        log.error("error", e);
    }
}
```

这里异常被 `catch` 掉后，代理层看到的是“正常返回”，事务通常会提交，而不是回滚。

### 8.5 抛出的是受检异常，但没有配置 `rollbackFor`

默认规则是：

- `RuntimeException` / `Error` 回滚。
- checked exception 不回滚。

所以如果你抛的是：

- `Exception`
- `IOException`
- 自定义受检异常

又没有配置 `rollbackFor`，那么事务可能不会回滚。

### 8.6 多线程或 `@Async` 场景

Spring 事务上下文通常绑定在当前线程。

官方 `TransactionSynchronizationManager` 文档也明确说明了：

- 它管理的是 **per thread** 的资源和事务同步

所以：

- 主线程里的事务，不会自动传播到手动创建的新线程
- `@Async` 切换线程后，通常也拿不到原事务上下文

这类场景不能想当然地认为“外层有事务，异步线程里也有事务”。

### 8.7 在 `@PostConstruct` 等初始化阶段使用事务

Spring 官方文档明确提醒：

- 代理对象必须完成初始化后，事务拦截才能正常工作

因此在 `@PostConstruct` 这种初始化阶段依赖 `@Transactional`，经常不符合预期。

### 8.8 多事务管理器或多数据源配置错误

如果系统里有多个数据源或多个 `PlatformTransactionManager`：

- 没有指定正确的事务管理器
- 配置的 bean 名称不匹配

就可能出现：

- 事务根本没作用到你预期的数据源上

### 8.9 标在接口上但运行模式不匹配

Spring 官方建议优先标注在具体类的方法上，而不是只标在接口上。

因为：

- 接口上的注解在某些代理或 AspectJ 织入场景下可能不被识别
- 看起来“平时能跑”，到回滚场景才暴露问题

## 9. Spring 事务底层实现原理

这是面试里最核心的部分之一。

### 9.1 整体思路：AOP + 事务管理器

Spring 声明式事务的底层可以概括成一句话：

**Spring 用 AOP 在方法调用前后织入事务逻辑，真正的事务开关则交给 `PlatformTransactionManager`。**

所以它不是魔法，而是一条非常清晰的调用链。

### 9.2 从 `@EnableTransactionManagement` 开始

启用事务管理后，Spring 会导入事务相关配置。

官方文档和相关配置类表明，代理模式下会注册关键基础设施：

- `TransactionAttributeSource`
- `TransactionInterceptor`
- `BeanFactoryTransactionAttributeSourceAdvisor`

它们分别负责：

- 解析事务元数据。
- 执行事务拦截逻辑。
- 把“哪些方法需要事务拦截”织入 AOP advisor。

### 9.3 `TransactionAttributeSource`：解析 `@Transactional`

当 Spring 扫描到 `@Transactional` 时，会把注解属性解析成事务属性对象。

这些属性包括：

- 传播行为
- 隔离级别
- 只读标记
- 超时
- 回滚规则
- 指定事务管理器

这也是为什么说：

**`@Transactional` 本身只是元数据。**

### 9.4 `TransactionInterceptor`：真正执行事务增强

`TransactionInterceptor` 是声明式事务的核心拦截器。

它本质上是一个 `MethodInterceptor`。

调用流程可以简化成：

1. 拦截目标方法。
2. 读取该方法对应的事务属性。
3. 找到对应的 `PlatformTransactionManager`。
4. 调用事务管理器开启或加入事务。
5. 执行业务方法。
6. 如果正常返回，则提交事务。
7. 如果抛异常，则根据回滚规则决定回滚还是提交。

简化理解就是：

```text
进入代理
-> 解析事务属性
-> 开启 / 加入事务
-> 执行业务方法
-> 正常则提交
-> 异常则回滚
```

### 9.5 `PlatformTransactionManager`：真正控制事务生命周期

事务拦截器不会自己去 `commit` 数据库，它会委托给具体事务管理器。

例如 JDBC 场景下，`DataSourceTransactionManager` 会：

1. 从 `DataSource` 获取连接。
2. 关闭自动提交。
3. 把连接绑定到当前线程。
4. 方法执行完成后执行 `commit` 或 `rollback`。
5. 最后解绑并释放连接。

### 9.6 `TransactionSynchronizationManager`：通过 `ThreadLocal` 绑定上下文

Spring 事务里一个非常关键的底层类是：

- `TransactionSynchronizationManager`

Spring 官方文档对它的描述很明确：

- 它管理 **当前线程** 上绑定的资源和事务同步器

这意味着：

- 当前线程可以绑定 JDBC Connection、Hibernate Session 等资源
- 同一个事务调用链里的下层 DAO，可以复用同一线程绑定的连接

这也是为什么：

- 事务通常天然和线程绑定
- 跨线程事务上下文不会自动传递

### 9.7 代理模式为什么会导致自调用失效

因为默认事务模式是 **proxy mode**。

代理模式下只有：

- **外部通过代理对象发起的方法调用**

才会被拦截。

而类内部 `this.xxx()` 调用，本质是：

- 目标对象直接调用自己的方法
- 完全绕过了代理

所以事务切面根本进不去。

### 9.8 JDK 动态代理和 CGLIB

Spring 事务底层代理通常有两种：

- **JDK 动态代理**：基于接口代理
- **CGLIB 代理**：基于子类继承代理

一般规律：

- 有接口时，默认优先 JDK 动态代理
- 没有接口或强制指定时，使用 CGLIB

它们和事务的关系主要体现在：

- JDK 代理只能拦截接口方法调用
- CGLIB 通过生成子类覆盖方法实现拦截，因此 `final` 类 / `final` 方法无法增强

### 9.9 AspectJ 模式

Spring 官方文档还提到，除了默认的 `proxy` 模式，还可以使用 `aspectj` 模式。

区别在于：

- `proxy` 模式：只拦截经过代理的外部调用
- `aspectj` 模式：通过字节码织入，理论上可以覆盖自调用等场景

但 AspectJ 模式：

- 配置更复杂
- 需要额外织入支持

所以大多数业务系统里，依旧是默认的代理模式。

## 10. 一次典型的事务调用链

下面用一个常见的下单流程串一下整个过程：

```java
@Service
public class OrderService {

    @Transactional(rollbackFor = Exception.class)
    public void createOrder(OrderDTO dto) {
        orderDao.insert(dto);
        inventoryDao.deduct(dto.getSkuId(), dto.getCount());
    }
}
```

执行时大致流程是：

1. 调用方拿到的是 `OrderService` 的代理对象。
2. 调用 `createOrder()` 时先进入事务拦截器。
3. 拦截器解析出：
   - 传播行为 `REQUIRED`
   - 隔离级别 `DEFAULT`
   - 回滚规则 `rollbackFor = Exception.class`
4. 事务管理器开启事务，并把数据库连接绑定到当前线程。
5. `orderDao` 和 `inventoryDao` 在同一线程内复用同一事务资源。
6. 如果方法正常结束，则提交。
7. 如果抛出满足回滚规则的异常，则回滚。
8. 最后清理线程绑定资源。

## 11. 开发中的实践建议

### 11.1 事务方法尽量放在 Service 层

原因是：

- Service 层更适合表达业务边界。
- Controller 层通常更偏请求编排，不适合承载复杂事务逻辑。
- DAO 层过细，容易把事务切得太碎。

### 11.2 事务范围不要过大

事务不是越大越安全。

事务范围太大，会导致：

- 锁持有时间长。
- 并发下降。
- 死锁概率上升。
- 回滚成本变高。

尤其不要在事务里做：

- 远程 RPC
- 大文件 I/O
- 长时间计算
- 调第三方接口

### 11.3 读写分离场景不要滥用 `readOnly`

`readOnly = true` 更像优化提示，不等于所有中间件和数据库都会严格强制只读。

所以它可以作为：

- 语义表达
- 优化 hint

但不要误以为它一定能“物理阻止写入”。

### 11.4 慎用 `REQUIRES_NEW`

`REQUIRES_NEW` 虽然能把事务边界隔离开，但用多了会：

- 增加连接占用
- 让调用链更难推理
- 让问题排查更复杂

它适合小范围、明确语义的独立事务，不适合作为默认方案。

### 11.5 跨线程、异步、分布式场景不要指望本地事务全搞定

Spring 本地事务主要解决的是：

- 单线程
- 单资源或少数本地资源

如果场景变成：

- `@Async`
- MQ
- 分布式服务调用
- 多系统一致性

就要考虑：

- 事务消息
- 本地消息表
- 补偿
- Saga / TCC

不能指望一个 `@Transactional` 兜住全部一致性问题。

## 12. 面试里怎么回答“Spring 事务原理”

可以这样概括：

- Spring 事务本质上是对底层事务能力的统一抽象，核心接口是 `PlatformTransactionManager`。
- 实际开发中最常见的是声明式事务，也就是 `@Transactional`。
- `@Transactional` 本身只是元数据，开启事务管理后，Spring 会通过 AOP 代理把事务拦截器织入到目标方法上。
- 调用进入代理后，`TransactionInterceptor` 会读取事务属性，找到对应的事务管理器，执行开启事务、提交事务、异常回滚等逻辑。
- Spring 事务上下文通常通过 `TransactionSynchronizationManager` 绑定到当前线程，所以自调用、异步调用、多线程切换都容易导致事务不生效。

## 13. 面试里怎么回答“Spring 事务为什么会失效”

可以按下面几个高频原因回答：

1. 方法调用没有经过代理，比如同类内部自调用。
2. 目标对象不是 Spring Bean。
3. `private`、`static`、`final` 等方法或类不满足代理增强条件。
4. 异常被吞掉了，代理层看不到异常。
5. 抛出的是 checked exception，但没有配置 `rollbackFor`。
6. 使用了 `@Async` 或手动开线程，事务上下文没有传过去。
7. 多数据源场景下用错了事务管理器。

## 14. 一句话总结

**Spring 事务 = 事务元数据（`@Transactional`）+ AOP 代理拦截 + `PlatformTransactionManager` 控制提交回滚 + `ThreadLocal` 绑定当前线程资源。**
