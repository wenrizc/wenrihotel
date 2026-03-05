## 1. `@Transactional` 解决什么问题

`@Transactional` 用来声明事务边界：在方法执行前开启事务，在方法正常返回时提交，在满足回滚规则时回滚。它解决的是“把事务控制从业务代码中抽离出来”，避免手写 `Connection`、`commit()`、`rollback()` 造成的重复与遗漏。

- **事务只对同一线程内的数据库操作生效**。Spring 默认把事务上下文绑定到当前线程（`ThreadLocal`），跨线程（`@Async`、手动新建线程）不会自动继承。
- **`@Transactional` 本质是 Spring AOP 代理拦截**。只有“通过代理对象调用”才会触发事务拦截器，自调用（`this.xxx()`）通常会绕过代理。

## 2. `@Transactional` 生效的前提与边界

### 2.1 必要前提

- 目标对象必须是 Spring 容器管理的 Bean。
- 调用必须经过代理对象（JDK 动态代理或 CGLIB 代理）。
- 容器中存在可用的 `PlatformTransactionManager`（例如 `DataSourceTransactionManager`、`JpaTransactionManager`）。

### 2.2 典型边界

- `private` 方法通常无法被代理增强；`final` 类或 `final` 方法在 CGLIB 下也无法增强。
- 事务对外部系统（RPC、MQ、Redis）不提供强一致保证，跨系统一致性需要依赖 `Saga`、`Outbox`、幂等、补偿等方案。

## 3. 注解参数

`@Transactional` 主要参数：

| 参数 | 类型 | 默认值 | 作用 |
| --- | --- | --- | --- |
| `propagation` | `Propagation` | `REQUIRED` | **传播行为**：有无外层事务时如何参与或新建事务 |
| `isolation` | `Isolation` | `DEFAULT` | 隔离级别：由具体数据库与驱动决定最终行为 |
| `timeout` | `int` | `-1` | 超时时间（秒），由具体事务管理器实现解释 |
| `readOnly` | `boolean` | `false` | 只读提示：可能影响路由、`flush`、优化器策略（不等于“禁止写入”） |
| `rollbackFor` / `rollbackForClassName` | `Class[]` / `String[]` | 空 | 指定哪些异常需要回滚（补充默认规则） |
| `noRollbackFor` / `noRollbackForClassName` | `Class[]` / `String[]` | 空 | 指定哪些异常不回滚（覆盖默认规则） |
| `transactionManager` | `String` | 空 | 指定使用哪个 `PlatformTransactionManager` Bean（多数据源常用） |

## 4. 传播行为（Propagation）

传播行为决定两件事：

- 当前方法被调用时，如果调用方已经在事务中，**是否加入该事务**。
- 如果调用方没有事务，**是否创建新事务**。

### 4.1 传播行为矩阵

| 传播行为            | 外层有事务                        | 外层无事务                | 典型用途                       |
| --------------- | ---------------------------- | -------------------- | -------------------------- |
| `REQUIRED`      | 加入外层事务                       | 新建事务                 | 绝大多数业务写操作（默认值）             |
| `SUPPORTS`      | 加入外层事务                       | 非事务执行                | 读操作可用，想“有事务就加入”            |
| `MANDATORY`     | 加入外层事务                       | 抛异常                  | 强制要求必须在事务中被调用              |
| `REQUIRES_NEW`  | **挂起外层事务**，新建事务              | 新建事务                 | 独立提交的日志、审计、补偿记录            |
| `NOT_SUPPORTED` | **挂起外层事务**，非事务执行             | 非事务执行                | 明确要求不在事务内执行的操作             |
| `NEVER`         | 抛异常                          | 非事务执行                | 强制要求不能在事务中被调用              |
| `NESTED`        | 在外层事务内创建嵌套事务（通常基于 Savepoint） | 新建事务（等价于 `REQUIRED`） | 局部回滚但不影响外层的场景（依赖具体事务管理器能力） |

### 4.2 `REQUIRED`：默认且最常用

`REQUIRED` 的关键点是“**参与**”：有外层事务就复用同一个物理事务边界，没有外层事务就创建新的。

常见误区：

- 你以为“内部方法标注了 `REQUIRED` 会开启新事务”，实际上它只是加入外层事务。
- 外层抛异常导致回滚时，内部逻辑也会一起回滚，因为它们属于同一个事务。

### 4.3 `REQUIRES_NEW`：强制独立事务

`REQUIRES_NEW` 的关键点是“**挂起外层** + **开启新事务**”。新事务的提交与回滚不受外层影响。

工程注意点：

- 对 JDBC（`DataSourceTransactionManager`）来说，外层事务占用的 `Connection` 会被挂起，内层通常需要从连接池再拿一条新 `Connection`。连接池过小可能导致内层拿不到连接而卡死。
- 外层事务最终回滚也不会影响内层已提交的数据，这既是能力也是风险，尤其是审计日志与业务数据的一致性需要你在设计上接受。

### 4.4 `NESTED`：嵌套事务与 Savepoint

`NESTED` 的语义是“外层事务仍是一个大事务”，但内层可以通过 Savepoint 做局部回滚。

关键点：

- 对 JDBC 来说，嵌套事务通常依赖数据库 Savepoint（同一条连接内的保存点），回滚到保存点不会回滚外层已执行但未提交的其他操作。
- 并非所有事务管理器都支持真正的嵌套事务。比如 JTA 场景可能退化为 `REQUIRED` 或直接不支持 Savepoint（取决于实现）。

### 4.5 传播行为最小示例

#### 4.5.1 `REQUIRES_NEW`：审计日志独立提交

```java
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

@Service
public class OrderService {
    private final AuditService auditService;
    private final OrderRepository orderRepository;

    public OrderService(AuditService auditService, OrderRepository orderRepository) {
        this.auditService = auditService;
        this.orderRepository = orderRepository;
    }

    @Transactional // REQUIRED
    public void createOrder() {
        orderRepository.insertOrder();
        auditService.writeAuditLog(); // 独立事务，外层回滚也不会影响它
        throw new RuntimeException("fail");
    }
}

@Service
class AuditService {
    private final AuditRepository auditRepository;

    AuditService(AuditRepository auditRepository) {
        this.auditRepository = auditRepository;
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void writeAuditLog() {
        auditRepository.insertAudit();
    }
}
```

#### 4.5.2 `NESTED`：局部失败不影响外层继续

```java
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

@Service
class PaymentService {
    private final CouponService couponService;

    PaymentService(CouponService couponService) {
        this.couponService = couponService;
    }

    @Transactional
    public void pay() {
        // 外层逻辑
        reserve();
        try {
            couponService.couponInNested(); // 失败只回滚到保存点，不影响 reserve()
        } catch (Exception ignore) {
            // 注意：吞异常会让外层事务继续提交，是否合理取决于业务语义
        }
        confirm();
    }

    private void reserve() {}

    private void confirm() {}
}

@Service
class CouponService {
    @Transactional(propagation = Propagation.NESTED)
    public void couponInNested() {
        // 内层逻辑
        throw new IllegalStateException("coupon error");
    }
}
```

## 5. 底层实现原理：从注解到事务提交/回滚

`@Transactional` 的底层实现可以按“三段式”理解：**解析注解**、**AOP 拦截**、**事务管理器执行**。

### 5.1 解析注解：`TransactionAttributeSource`

Spring 会把 `@Transactional` 解析成事务属性（传播、隔离、回滚规则等），核心抽象是：

- `TransactionAttributeSource`：给定 `Method` 与 `Class`，解析出 `TransactionAttribute`。
- 常见实现：`AnnotationTransactionAttributeSource`。
- 常见属性实现：`RuleBasedTransactionAttribute`，它内部持有回滚规则（`RollbackRuleAttribute`）。

结论是：`@Transactional` 不是“开关”，它会被解析为一份可执行的事务定义。

### 5.2 AOP 拦截：`TransactionInterceptor`

事务是通过 AOP Advisor 织入的，核心拦截器是 `TransactionInterceptor`（它实现了 AOP Alliance 的 `MethodInterceptor`）。

在方法调用时，拦截器的逻辑可以抽象为：

```text
1. 根据方法与类解析 TransactionAttribute（含 propagation、isolation、rollback rules）
2. 选择 PlatformTransactionManager（默认按类型注入，多数据源可按 transactionManager 指定）
3. 调用 tm.getTransaction(...) 获取 TransactionStatus
4. 执行目标方法
5. 正常返回：tm.commit(status)
6. 异常返回：根据 rollback rules 决定 tm.rollback(status) 或 tm.commit(status)
```

这也是为什么“自调用”会导致事务失效：没有走代理就不会进入 `TransactionInterceptor`。

### 5.3 事务管理器模板：`AbstractPlatformTransactionManager`

`PlatformTransactionManager` 定义了事务的三大操作：

- `getTransaction(TransactionDefinition definition)`：根据传播行为决定加入、挂起、创建新事务。
- `commit(TransactionStatus status)`：提交或在标记回滚时回滚。
- `rollback(TransactionStatus status)`：回滚。

常见实现会继承 `AbstractPlatformTransactionManager`，把“传播行为处理、挂起与恢复、同步回调”等通用流程模板化，子类只需要实现与具体资源相关的部分：

- `DataSourceTransactionManager`：管理 JDBC `Connection`。
- `JpaTransactionManager`：管理 JPA `EntityManager`。

传播行为的关键处理点就在 `AbstractPlatformTransactionManager#getTransaction(...)`：它会判断当前线程是否已经绑定资源，从而决定加入、挂起或新建事务。

### 5.4 JDBC 场景的关键细节：`DataSourceTransactionManager`

以 JDBC 为例，事务的“资源绑定”与“线程绑定”是两条主线：

- 资源绑定：开启事务时会拿到 `Connection`，设置 `autoCommit=false`，按需设置隔离级别与只读属性。
- 线程绑定：通过 `TransactionSynchronizationManager` 把 `ConnectionHolder` 绑定到当前线程，DAO 层通过 `DataSourceUtils.getConnection(...)` 取到同一条连接，从而保证同一事务内的多次 SQL 在同一物理事务中执行。

`REQUIRES_NEW` 的“挂起”通常意味着：

- 把当前线程绑定的连接与同步信息暂存起来（suspend）。
- 为新事务重新获取并绑定另一条连接（begin）。
- 新事务结束后恢复外层绑定（resume）。

### 5.5 回滚判定：为什么默认只回滚运行时异常

Spring 的默认回滚规则是：

- 遇到 `RuntimeException` 或 `Error`：回滚。
- 遇到受检异常（`Exception` 但非 `RuntimeException`）：默认提交。

原因是：受检异常通常被用来表达“业务可预期分支”，Spring 选择了“默认不回滚”的保守策略，避免把业务分支误判为系统失败。

当你配置 `rollbackFor`、`noRollbackFor` 时，本质是在影响 `TransactionAttribute#rollbackOn(Throwable ex)` 的判断结果。

## 6. 隔离级别（Isolation）与一致性直觉

`isolation` 描述的是数据库层面的并发可见性，常见枚举包括：

- `READ_UNCOMMITTED`：可能脏读。
- `READ_COMMITTED`：避免脏读，仍可能不可重复读与幻读（取决于数据库实现）。
- `REPEATABLE_READ`：保证同一事务内重复读一致性（InnoDB 还会用 MVCC 与间隙锁影响幻读表现）。
- `SERIALIZABLE`：最强隔离，吞吐最差。
- `DEFAULT`：使用数据库默认隔离级别。

工程建议：

- 优先使用数据库默认隔离级别并结合业务锁设计，除非你能解释清楚“为什么需要提升隔离级别”以及它的吞吐代价。
- 不要把隔离级别当成“万能一致性开关”，唯一性约束、幂等键、乐观锁仍然是必要工具。

## 7. 回滚规则与异常处理的坑

### 7.1 `try/catch` 吞异常导致提交

事务拦截器通常以“方法是否抛出异常”作为回滚触发条件之一。你把异常吞掉，拦截器看到的是“正常返回”，就会提交。

```java
import org.springframework.transaction.annotation.Transactional;
import org.springframework.transaction.interceptor.TransactionAspectSupport;

class FooService {
    @Transactional
    public void doBiz() {
        try {
            risky();
        } catch (Exception e) {
            // 如果这里必须吞异常，但又要回滚，需要显式标记回滚
            TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
        }
    }

    private void risky() {}
}
```

### 7.2 受检异常默认不回滚

如果你抛的是受检异常（例如 `IOException`），默认可能不会触发回滚。需要显式配置：

```java
import java.io.IOException;
import org.springframework.transaction.annotation.Transactional;

class BarService {
    @Transactional(rollbackFor = IOException.class)
    public void f() throws IOException {
        throw new IOException("io");
    }
}
```

## 8. 常见失效场景

常见“看起来加了注解但不生效”的原因：

- 同类内部自调用绕过代理。
- 方法可见性或 `final` 限制导致无法增强。
- 异常被吞掉或被转换成了不触发回滚的类型。
- 多数据源场景事务管理器选错（`transactionManager` 未指定或 bean 装配不符合预期）。
- 异步、跨线程导致事务上下文丢失。
