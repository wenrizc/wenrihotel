
#### Spring 事务传播与回滚概述
- **事务传播**：
  - Spring 事务传播（Transaction Propagation）定义了在方法调用链中，事务如何在不同方法之间传播或创建。Spring 通过 `@Transactional` 注解的 `propagation` 属性配置传播行为。
- **事务回滚**：
  - Spring 事务回滚是指当事务执行失败（如抛出异常）时，撤销所有数据库操作，恢复到事务开始前的状态。回滚由事务管理器（`PlatformTransactionManager`）处理，默认针对特定异常触发。
- **核心机制**：
  - 基于 **AOP 动态代理**（JDK 或 CGLIB）和 **事务管理器**实现。

#### 核心点
- 事务传播决定事务的创建或复用，回滚依赖异常类型和配置。

---

### 1. Spring 事务传播机制
Spring 定义了七种事务传播行为（`Propagation` 枚举），通过 `@Transactional(propagation = Propagation.XXX)` 配置。

#### (1) 传播行为详解
| **传播行为**          | **描述**                                                                 | **示例场景**                     |
|-----------------------|-------------------------------------------------------------------------|-------------------------------|
| **REQUIRED** (默认)   | 如果当前有事务，加入该事务；否则创建新事务。                              | 订单创建和库存扣减共用事务。       |
| **SUPPORTS**          | 如果当前有事务，加入该事务；否则以非事务方式运行。                        | 查询操作可选事务。               |
| **MANDATORY**         | 如果当前有事务，加入该事务；否则抛出异常（`IllegalTransactionStateException`）。 | 必须在事务中执行的操作。         |
| **REQUIRES_NEW**      | 总是创建新事务，挂起当前事务（若存在）。                                  | 独立日志记录，需单独事务。        |
| **NOT_SUPPORTED**     | 以非事务方式运行，若当前有事务，挂起该事务。                              | 非事务操作，如日志打印。          |
| **NEVER**             | 以非事务方式运行，若当前有事务，抛出异常。                                | 禁止事务的操作。                 |
| **NESTED**            | 如果当前有事务，创建嵌套事务（子事务）；否则创建新事务。                  | 部分操作需独立回滚，如批量处理。   |

#### (2) 传播机制实现
- **核心组件**：
  - **TransactionInterceptor**：AOP 拦截器，处理事务逻辑。
  - **PlatformTransactionManager**：事务管理器（如 `DataSourceTransactionManager`），管理事务的开启、提交、回滚。
  - **TransactionStatus**：记录事务状态（如是否新事务、嵌套级别）。
- **底层原理**：
  - **AOP 代理**：
    - `@Transactional` 方法通过代理拦截，调用 `TransactionInterceptor.invoke`。
    - 代理决定是否创建新事务、加入现有事务或挂起事务。
  - **Connection 管理**：
    - Spring 使用 `ThreadLocal`（`TransactionSynchronizationManager`）存储当前线程的事务信息（如 `Connection`）。
    - 传播行为通过 `Connection` 的状态（`autoCommit`、保存点）实现。
  - **传播逻辑**：
    - **REQUIRED**：检查 `ThreadLocal` 是否有事务，若无则调用 `getTransaction` 创建。
    - **REQUIRES_NEW**：挂起当前事务（保存 `Connection`），创建新 `Connection`。
    - **NESTED**：使用数据库的 **保存点（Savepoint）** 实现子事务。
- **源码关键**：
  - `AbstractPlatformTransactionManager.getTransaction`：决定传播行为。
  - `DataSourceTransactionManager.doBegin`：开启事务。
  - `TransactionAspectSupport.invokeWithinTransaction`：执行事务逻辑。

#### (3) 示例
```java
@Service
public class OrderService {
    @Autowired
    private InventoryService inventoryService;

    @Transactional(propagation = Propagation.REQUIRED)
    public void createOrder() {
        // 保存订单
        inventoryService.updateInventory(); // 加入当前事务
    }
}

@Service
public class InventoryService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void updateInventory() {
        // 独立事务，挂起 OrderService 事务
        // 更新库存
    }
}
```
- **流程**：
  - `createOrder` 开启事务 T1。
  - 调用 `updateInventory`，挂起 T1，创建新事务 T2。
  - T2 提交或回滚不影响 T1。

---

### 2. 事务出错时的回滚机制
Spring 事务回滚由事务管理器和异常处理机制共同实现。

#### (1) 回滚触发条件
- **默认规则**：
  - Spring 默认只对 **运行时异常（RuntimeException）** 和 **Error** 回滚。
  - 受检异常（`Checked Exception`，如 `IOException`）不会触发回滚。
- **自定义规则**：
  - 使用 `@Transactional` 的 `rollbackOn` 和 `noRollbackOn` 属性：
```java
@Transactional(rollbackOn = Exception.class)
public void doSomething() throws IOException {
    // 即使抛出 IOException 也会回滚
}
```

#### (2) 回滚实现原理
- **流程**：
  1. **异常捕获**：
     - `TransactionInterceptor` 捕获方法抛出的异常。
  2. **回滚判断**：
     - 检查异常是否符合回滚规则（`rollbackOn` 或默认）。
  3. **执行回滚**：
     - 调用 `PlatformTransactionManager.rollback`。
     - 数据库 `Connection` 执行 `rollback`（撤销所有操作）。
  4. **清理**：
     - 清除 `ThreadLocal` 中的事务状态。
- **嵌套事务**：
  - 使用 **保存点**（`Connection.setSavepoint`）记录子事务状态。
  - 子事务出错只回滚到保存点，不影响父事务。
- **源码关键**：
  - `AbstractPlatformTransactionManager.processRollback`：执行回滚。
  - `DataSourceTransactionManager.doRollback`：调用 `Connection.rollback`。

#### (3) 示例
```java
@Service
public class OrderService {
    @Transactional
    public void createOrder() {
        saveOrder(); // 数据库操作
        throw new RuntimeException("Error"); // 触发回滚
    }
}
```
- **结果**：
  - `RuntimeException` 触发回滚，`saveOrder` 的操作撤销。

#### (4) 常见回滚失效场景
- **异常被捕获**：
```java
@Transactional
public void doSomething() {
    try {
        // 数据库操作
        throw new RuntimeException();
    } catch (Exception e) {
        // 异常被吞没，不回滚
    }
}
```
  - **解决**：抛出异常或手动回滚：
```java
@Autowired
private TransactionTemplate transactionTemplate;

public void doSomething() {
    transactionTemplate.execute(status -> {
        try {
            // 操作
        } catch (Exception e) {
            status.setRollbackOnly(); // 手动回滚
            throw e;
        }
        return null;
    });
}
```
- **方法内部调用**：
  - 同一类中非 `@Transactional` 方法调用 `@Transactional` 方法，代理失效。
  - **解决**：注入自身代理或拆分类。
- **数据库不支持**：
  - 如 MySQL 的 MyISAM 引擎不支持事务。
  - **解决**：使用 InnoDB。

---

### 3. 传播与回滚结合
- **REQUIRED**：
  - 所有方法共享事务，出错全部回滚。
- **REQUIRES_NEW**：
  - 独立事务，内层回滚不影响外层。
- **NESTED**：
  - 子事务出错回滚到保存点，父事务可继续。
- **示例**：
```java
@Service
public class OuterService {
    @Autowired
    private InnerService innerService;

    @Transactional(propagation = Propagation.REQUIRED)
    public void outerMethod() {
        // 操作 1
        innerService.innerMethod(); // 嵌套事务
        // 操作 2
    }
}

@Service
public class InnerService {
    @Transactional(propagation = Propagation.NESTED)
    public void innerMethod() {
        // 操作 3
        throw new RuntimeException(); // 回滚到保存点
    }
}
```
- **结果**：
  - `innerMethod` 回滚操作 3，`outerMethod` 的操作 1 可继续。

---

### 4. 面试角度
- **问“传播”**：
  - 提七种行为，重点 REQUIRED、REQUIRES_NEW、NESTED。
- **问“回滚”**：
  - 提默认规则（RuntimeException）、rollbackOn。
- **问“失效”**：
  - 提异常捕获、内部调用。
- **问“实现”**：
  - 提 AOP 代理、ThreadLocal、保存点。

---

### 5. 总结
Spring 事务传播通过 `@Transactional` 的 `propagation` 属性控制，七种行为决定事务的创建或复用，基于 AOP 代理和 `ThreadLocal` 实现。回滚默认针对运行时异常，由事务管理器执行 `Connection.rollback`，嵌套事务用保存点。失效场景需注意异常捕获和代理调用。面试可提传播示例或回滚规则，清晰展示理解。

---

如果您想深入某部分（如源码或嵌套事务），请告诉我，我可以进一步优化！