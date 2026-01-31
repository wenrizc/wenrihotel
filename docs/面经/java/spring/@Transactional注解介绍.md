## 1. `@Transactional` 的“级别”在说什么

面试里说的“`@Transactional` 各个级别”，通常指两类可枚举的策略：**事务传播行为（Propagation）** 和 **事务隔离级别（Isolation）**。前者解决“调用链里事务边界怎么接”，后者解决“并发读写时允许看到什么数据”。

需要先明确一个前提：这些参数本质上会被转换成 `TransactionDefinition`，最终由 `PlatformTransactionManager` 在运行期按规则创建/加入事务，并把隔离级别等信息落到具体资源（例如 JDBC `Connection`）上。

## 2. 事务传播行为（Propagation）七种语义

传播行为描述的是：当前线程已存在事务时，本次方法调用是 **加入、挂起、还是新建**。Spring 的 7 种传播行为如下（单数据源场景下最常用）。

| 传播行为            | 是否新建物理事务    | 语义要点                   | 高频使用场景             |
| --------------- | ----------- | ---------------------- | ------------------ |
| `REQUIRED`      | 有则不新建，无则新建  | 默认：优先加入外层事务            | 绝大多数业务写操作          |
| `REQUIRES_NEW`  | 总是新建        | 挂起外层事务，内层独立提交/回滚       | 独立日志、关键副作用需要“必达提交” |
| `NESTED`        | 不新建（同一物理事务） | 基于保存点（Savepoint）实现局部回滚 | 局部失败不影响外层继续        |
| `SUPPORTS`      | 取决于外层       | 有事务则加入，无事务则非事务执行       | 读操作复用外层事务          |
| `NOT_SUPPORTED` | 不新建         | 挂起当前事务，强制非事务执行         | 需要避免长事务持锁的逻辑       |
| `MANDATORY`     | 不新建         | 必须存在事务，否则抛异常           | 强制调用方提供事务边界        |
| `NEVER`         | 不新建         | 必须不存在事务，否则抛异常          | 明确禁止在事务中调用         |

**`REQUIRES_NEW` vs `NESTED`** 是高频点：`REQUIRES_NEW` 是“独立物理事务”，`NESTED` 是“同一物理事务 + 保存点”。传播行为的更完整推导可参考同目录《Spring中事务是如何传播的，事务出错时如何实现回滚》。

## 3. 事务隔离级别

隔离级别解决的是并发读写下的可见性问题。标准口径通常从三类现象切入：

- 脏读：读到其他事务 **未提交** 的数据。
- 不可重复读：同一事务内两次读同一行，结果不一致（中间被他人提交修改）。
- 幻读：同一事务内两次范围查询，结果集行数不一致（中间被他人提交插入/删除）。

### 3.1 并发现象与隔离级别对照（SQL 标准口径）

| 隔离级别               | 脏读  | 不可重复读 | 幻读  | 典型取舍             |
| ------------------ | --- | ----- | --- | ---------------- |
| `READ_UNCOMMITTED` | 可能  | 可能    | 可能  | 并发高，一致性差，几乎不用    |
| `READ_COMMITTED`   | 不会  | 可能    | 可能  | 吞吐与一致性折中，很多数据库默认 |
| `REPEATABLE_READ`  | 不会  | 不会    | 可能  | 更强一致性，可能增加锁冲突    |
| `SERIALIZABLE`     | 不会  | 不会    | 不会  | 最强一致性，并发最低       |

注意：上表是 SQL 标准的抽象定义。不同数据库在 MVCC、间隙锁（gap lock）、谓词锁等实现细节上存在差异，某些实现会“弱化/规避”特定现象，但不改变隔离级别的语义目标。

### 3.2 Spring `Isolation` 枚举逐项解释

Spring 在 `@Transactional` 里通过 `isolation` 指定隔离级别，其枚举来自 `org.springframework.transaction.annotation.Isolation`，并映射到 JDBC `Connection` 的隔离级别常量：

| Spring `Isolation` | `TransactionDefinition` | JDBC `Connection` 常量 |
| --- | --- | --- |
| `DEFAULT` | `ISOLATION_DEFAULT`（-1） | 不设置，使用数据库/连接默认值 |
| `READ_UNCOMMITTED` | `ISOLATION_READ_UNCOMMITTED` | `TRANSACTION_READ_UNCOMMITTED` |
| `READ_COMMITTED` | `ISOLATION_READ_COMMITTED` | `TRANSACTION_READ_COMMITTED` |
| `REPEATABLE_READ` | `ISOLATION_REPEATABLE_READ` | `TRANSACTION_REPEATABLE_READ` |
| `SERIALIZABLE` | `ISOLATION_SERIALIZABLE` | `TRANSACTION_SERIALIZABLE` |

#### 3.2.1 `Isolation.DEFAULT`

**默认值不是“某个固定级别”，而是“交给底层数据库/数据源决定”**。例如：MySQL InnoDB 常见默认是 `REPEATABLE_READ`，PostgreSQL 常见默认是 `READ_COMMITTED`。

工程上优先使用 `DEFAULT` 的原因是：隔离级别一旦显式指定，就会变成“业务代码的隐含约束”，迁移数据库或调整实例参数时更难统一收敛。

#### 3.2.2 `Isolation.READ_UNCOMMITTED`

允许读取未提交数据，可能产生脏读。除非你明确知道读到“临时态数据”也不会造成业务错误（并且能接受回滚导致的反直觉结果），否则一般不建议使用。

#### 3.2.3 `Isolation.READ_COMMITTED`

保证不会脏读：只读到已提交版本。代价是同一事务中两次读可能看到不同提交点的数据，因此可能出现不可重复读与幻读。

如果业务希望“读到尽可能新的已提交数据”，并且可通过幂等、校验、乐观锁等手段兜住并发写入带来的变化，`READ_COMMITTED` 通常是更平衡的选择。

#### 3.2.4 `Isolation.REPEATABLE_READ`

保证同一事务内对同一行的重复读取结果一致（标准语义下仍可能幻读）。在 MVCC 数据库中，它通常意味着“事务级快照”：你的普通查询会基于同一个一致性视图（snapshot）。

在 MySQL/InnoDB 中，还要区分快照读与当前读：`SELECT ... FOR UPDATE` 这类当前读会参与加锁，锁冲突与死锁风险通常比快照读更敏感。

#### 3.2.5 `Isolation.SERIALIZABLE`

最强隔离，目标是让并发事务的执行效果等价于串行执行。实现上通常需要更重的锁或谓词/范围锁，**吞吐会显著下降**，并且更容易出现锁等待与超时。

工程上更常见的做法是：在必要的热点路径上使用“短事务 + 明确锁语义（如 `SELECT ... FOR UPDATE`）+ 唯一约束/幂等”替代全局 `SERIALIZABLE`。

## 4. 隔离级别什么时候真正生效：只对“新事务”生效

`isolation`（以及 `timeout`、`readOnly` 等）只有在 **本次调用会创建新的物理事务** 时才会被应用。原因是隔离级别是底层资源（如 JDBC `Connection`）的会话属性，参与已有事务时无法在不破坏一致性的前提下“半路切换”。

下面这个例子是常见误区：内层标了更强隔离，但传播行为是 `REQUIRED`，结果内层只是在复用外层事务，隔离级别并不会变。

```java
import org.springframework.transaction.annotation.Isolation;
import org.springframework.transaction.annotation.Transactional;

@Transactional // 默认：Propagation.REQUIRED + Isolation.DEFAULT
public void outer() {
    inner(); // 复用 outer 的物理事务，inner 的 isolation 不会生效
}

@Transactional(isolation = Isolation.SERIALIZABLE)
public void inner() {}
```

如果你确实需要在调用链中“切换隔离级别”，通常要配合 `Propagation.REQUIRES_NEW` 新开事务；否则请把隔离级别统一放在最外层事务边界上。

源码层面，这一行为由 `AbstractPlatformTransactionManager#getTransaction(...)` 的“已有事务分支”决定：默认不会因为内外层 `isolation` 不一致而报错；如需强校验，可通过 `AbstractPlatformTransactionManager#setValidateExistingTransaction(true)` 在发现不一致时抛出 `IllegalTransactionStateException`。

## 5. Spring 如何把隔离级别落到 JDBC 连接上

以 `DataSourceTransactionManager` 为例，事务开启时（`doBegin`）会从 `DataSource` 取出一个 JDBC `Connection`，并在需要时调用 `Connection#setTransactionIsolation(...)` 设置隔离级别，随后把连接绑定到当前线程（`TransactionSynchronizationManager`）。

事务结束时（`doCleanupAfterCompletion`），Spring 会把连接的 `autoCommit`、隔离级别、`readOnly` 等属性尽量恢复成原值，再把连接归还给连接池。**这一步很关键**：如果属性未被恢复，连接池复用连接时就会出现“脏隔离级别”串扰。

如果你用的是分布式事务（如 JTA），或 ORM 框架自行管理连接/会话，隔离级别是否能按 `@Transactional` 精确落地，取决于具体事务管理器与资源适配方式，不能一概而论。

## 6. 实战选型建议

- 能用 `DEFAULT` 就先用 `DEFAULT`，把隔离级别当作“数据库层面默认策略”，不要轻易在大量业务方法上散落显式配置。
- 需要更强一致性时，优先缩短事务、优化索引与访问顺序，必要时使用显式锁（如 `SELECT ... FOR UPDATE`）或乐观锁，而不是直接把隔离级别拉满。
- 要求“内层独立提交/回滚”时用 `REQUIRES_NEW`；要“同一事务内局部回滚”且底层支持保存点时用 `NESTED`。
- 遇到“配了 `isolation` 但没效果”，第一反应检查：是否真的新开了物理事务（传播行为、是否已有外层事务）。

## 7. `@Transactional` 的底层实现

`@Transactional` 的实现可以一句话概括：**用 AOP 把 `TransactionInterceptor` 包到目标方法外层，在运行期把注解解析成 `TransactionAttribute`，再委托 `PlatformTransactionManager` 进行事务创建、挂起、提交与回滚**。

它不是“编译期魔法”，而是 Spring 容器启动阶段注册的一套基础设施（Advisor/Interceptor/AttributeSource）在运行期协作完成。

### 7.1 开启事务能力：`@EnableTransactionManagement` 做了什么

Spring 通过 `@EnableTransactionManagement`（或 Spring Boot 的自动配置）启用声明式事务。其核心是导入 `ProxyTransactionManagementConfiguration`（代理模式），注册三类基础组件：

- `TransactionAttributeSource`：从方法/类上解析 `@Transactional`，产出事务规则。
- `TransactionInterceptor`：环绕增强，负责开启事务并在方法返回/抛异常时提交或回滚。
- `BeanFactoryTransactionAttributeSourceAdvisor`：把“哪些方法需要事务”和“用哪个拦截器”绑定到一起，让 AOP 自动创建代理并织入调用链。

如果选择 AspectJ 模式（较少见），事务织入不是通过代理完成，拦截范围与限制也会不同（例如非 `public` 方法）。

### 7.2 注解如何被解析：`@Transactional` → `TransactionAttribute`

默认解析链路是：

- `AnnotationTransactionAttributeSource`：入口，定位方法/类上的事务注解。
- `SpringTransactionAnnotationParser`：把注解属性转成 Spring 的事务定义。
- `RuleBasedTransactionAttribute`：承载传播、隔离、超时、只读、回滚规则等信息。

这里有两个容易忽略的点：

- `TransactionAttributeSource` 内部会做缓存（基于 `AbstractFallbackTransactionAttributeSource`），避免每次调用都反射解析注解。
- 规则合并顺序一般是“方法优先于类”，更具体的声明覆盖更抽象的声明。

### 7.3 事务是怎么织入的：Advisor 匹配 + 代理拦截

当一个 Bean 初始化完成后，Spring AOP 会根据 `BeanFactoryTransactionAttributeSourceAdvisor` 判断它是否需要创建代理：只要某个方法能从 `TransactionAttributeSource` 解析到事务属性，就会命中。

运行期调用链（典型同步场景）可以理解为：

- 业务方调用代理对象方法。
- `TransactionInterceptor#invoke(...)` 执行，进入 `TransactionAspectSupport#invokeWithinTransaction(...)`。
- 解析 `TransactionAttribute`，确定 `PlatformTransactionManager`，然后调用 `getTransaction(...)`。
- 执行目标方法：正常返回则 `commit(...)`；抛异常则按回滚规则 `rollback(...)` 或 `commit(...)`。

因此，事务的“边界”本质上就是 AOP 代理能否拦到这次方法调用。

### 7.4 `PlatformTransactionManager` 如何决定“加入/新建/挂起”

`AbstractPlatformTransactionManager#getTransaction(...)` 是传播行为的核心入口：

- 没有当前事务：按传播行为选择“新建事务 / 直接非事务执行 / 抛异常”。
- 已有当前事务：按传播行为选择“加入 / 挂起再新建 / 建保存点 / 抛异常”。

以 `REQUIRES_NEW` 为例：先 `suspend(...)` 挂起外层资源（连接、同步回调等），再 `doBegin(...)` 新建事务；内层结束后 `resume(...)` 恢复外层事务继续执行。`NESTED` 则通常通过 `SavepointManager` 创建保存点来实现局部回滚。

### 7.5 事务资源如何绑定到线程：`TransactionSynchronizationManager`

Spring 用 `TransactionSynchronizationManager`（一组 `ThreadLocal`）把“当前线程的事务上下文”串起来，里面典型会存：

- 资源句柄：例如 `DataSource` → `ConnectionHolder`，用来复用同一个 JDBC 连接。
- 事务同步回调：`TransactionSynchronization`，用于 `afterCommit`、`afterCompletion` 等时机的回调。

这也解释了一个工程事实：**事务上下文默认不跨线程传播**。一旦你把工作切到线程池（`@Async`、`CompletableFuture` 等），新的线程没有旧线程的 `ThreadLocal`，自然也就“没有事务”。

### 7.6 以 JDBC 为例：`DataSourceTransactionManager` 的关键动作

在单数据源 JDBC 场景，`DataSourceTransactionManager` 会在 `doBegin(...)` 阶段做几件关键事：

- 从连接池取 `Connection`，必要时设置隔离级别、只读标记等连接属性。
- 关闭 `autoCommit`，让后续 SQL 处在同一个事务里。
- 把 `ConnectionHolder` 绑定到 `TransactionSynchronizationManager`，保证同线程内的 DAO/ORM 拿到的是同一连接。

可以用伪代码抓住主线（方法名以 Spring 源码为准，细节略有简化）：

```java
Connection con = DataSourceUtils.getConnection(dataSource);
Integer oldLevel = DataSourceUtils.prepareConnectionForTransaction(con, txAttr);
con.setAutoCommit(false);
TransactionSynchronizationManager.bindResource(dataSource, new ConnectionHolder(con));

try {
    invocation.proceed();
    con.commit();
} catch (Throwable ex) {
    con.rollback();
    throw ex;
} finally {
    TransactionSynchronizationManager.unbindResource(dataSource);
    DataSourceUtils.resetConnectionAfterTransaction(con, oldLevel);
    DataSourceUtils.releaseConnection(con, dataSource);
}
```

这里的工程要点是：**连接属性必须在事务结束后被恢复**。否则连接池复用连接时，会把隔离级别、只读标记等“串”到下一次请求，出现非常隐蔽的一致性问题。

### 7.7 回滚规则在哪里判定：`rollbackOn` 与 `setRollbackOnly`

Spring 并不是看到“有异常”就一定回滚，它会调用 `TransactionAttribute#rollbackOn(Throwable)` 判断是否回滚：

- 默认实现（`DefaultTransactionAttribute`）：遇到 `RuntimeException` 或 `Error` 才回滚。
- 解析了 `rollbackFor` / `noRollbackFor` 后（`RuleBasedTransactionAttribute`）：按规则匹配异常类型，最接近的规则生效。

还有一个常见误区：如果你在事务方法内部把异常吞掉，代理层看起来就是“正常返回”，会走提交分支。此时要么重新抛出异常，要么显式标记回滚：

```java
import org.springframework.transaction.interceptor.TransactionAspectSupport;

TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
```

### 7.8 事务为什么会“失效”：你以为在用，其实没拦到

只要把握住“事务靠代理拦截”这个事实，就能解释大多数失效场景：

- 类内部自调用：`this.xxx()` 绕过代理，`@Transactional` 不生效。
- 非 `public` 方法：代理模式下默认只对 `public` 方法应用事务属性（避免语义与可见性冲突）。
- 初始化阶段调用：例如 `@PostConstruct` 中调用事务方法，此时代理通常尚未生效，属于典型坑点。

更完整的失效清单与修复方式可对照同目录《什么情况下@Transactional注解会失效》。


