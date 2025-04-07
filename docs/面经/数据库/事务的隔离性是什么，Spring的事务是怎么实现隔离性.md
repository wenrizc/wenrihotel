
**事务的隔离性（Isolation）** 是数据库事务的四大特性（ACID）之一，指一个事务的执行不受其他并发事务的干扰，未提交的更改对其他事务不可见。它确保事务之间相互独立，避免数据不一致问题。
### 关键事实

1. **定义**：
    - 隔离性保证事务在并发执行时，行为如同串行执行（一个接一个）。
    - 一个事务的中间状态（如未提交的修改）不会影响其他事务。
2. **实现方式**：
    - 通过 **锁机制**（如行锁、表锁）或 **多版本并发控制（MVCC）**实现。
    - 数据库提供不同隔离级别，平衡一致性与性能。
3. **隔离级别**（由低到高）：
    - **读未提交（Read Uncommitted）**：可读取其他事务未提交的数据（脏读）。
    - **读已提交（Read Committed）**：只能读取已提交数据，避免脏读，但可能有不可重复读。
    - **可重复读（Repeatable Read）**：同一事务中多次读取结果一致，避免不可重复读，但可能有幻读。
    - **串行化（Serializable）**：最高级别，完全隔离，事务串行执行，避免所有并发问题。
4. **并发问题**：
    - **脏读**：读取未提交数据，后回滚导致不一致。
    - **不可重复读**：同事务内多次读同一数据，结果不同。
    - **幻读**：同事务内查询范围数据，新增或删除行导致不一致。
### 延伸与面试角度

- **为什么需要隔离性？**
    - 并发事务可能导致数据不一致（如银行余额错误），隔离性保证正确性。
- **隔离级别选择**：
    - **读未提交**：性能高，但一致性差，适合日志系统。
    - **可重复读**：MySQL InnoDB 默认级别，兼顾一致性和并发。
    - **串行化**：金融系统用，但性能低。
- **实现技术**：
    - **锁**：悲观并发控制，阻塞其他事务。
    - **MVCC**：乐观控制，通过版本（如时间戳）隔离数据，Redis 不支持事务隔离，依赖单线程。
- **实际应用**：
    - 电商库存扣减：可重复读防止超卖。
    - 银行转账：串行化确保资金安全。

Spring 的事务通过 **代理模式** 和 **底层数据库支持** 实现隔离性。它并不直接实现隔离逻辑，而是依托数据库的事务隔离机制（如 MySQL 的 InnoDB），通过配置事务属性（如隔离级别）控制隔离行为。Spring 使用 AOP（面向切面编程）和事务管理器（如 DataSourceTransactionManager）将隔离性配置传递给数据库。

---

### 关键事实

1. **实现机制**：
    - **代理模式**：Spring 为目标方法生成代理对象（如 CGLIB 或 JDK 动态代理），在方法执行前后管理事务。
    - **事务管理器**：如 DataSourceTransactionManager，负责与数据库交互，设置隔离级别。
    - **数据库支持**：隔离性由底层数据库（如 MySQL、PostgreSQL）实现，Spring 只负责传递配置。
2. **隔离级别配置**：
    - Spring 支持标准的 JDBC 隔离级别，通过 @Transactional 注解或 XML 配置指定：
        - ISOLATION_DEFAULT：数据库默认（如 MySQL 的可重复读）。
        - ISOLATION_READ_UNCOMMITTED：读未提交。
        - ISOLATION_READ_COMMITTED：读已提交。
        - ISOLATION_REPEATABLE_READ：可重复读。
        - ISOLATION_SERIALIZABLE：串行化。
3. **工作流程**：
    - 开始事务时，Spring 通过 Connection.setTransactionIsolation() 设置隔离级别。
    - 调用数据库的 JDBC API（如 beginTransaction），执行 SQL。
    - 事务提交或回滚时，释放资源并恢复连接状态。
4. **底层依赖**：
    - 数据库引擎（如 InnoDB）使用锁或 MVCC（多版本并发控制）实现隔离。
    - Spring 只协调，不改变数据库的隔离逻辑。

---

### 示例
CollapseWrapCopy

`@Service public class AccountService { @Transactional(isolation = Isolation.REPEATABLE_READ) public void transfer(Account from, Account to, BigDecimal amount) { from.setBalance(from.getBalance().subtract(amount)); to.setBalance(to.getBalance().add(amount)); } }`

- **过程**：
    1. Spring 拦截 transfer 方法，创建代理。
    2. 通过事务管理器获取数据库连接，设置隔离级别为 REPEATABLE_READ。
    3. 执行转账逻辑，InnoDB 用 MVCC 确保事务内余额读取一致。
    4. 提交事务，释放连接。

---

### 延伸与面试角度

- **具体实现细节**：
    - **AOP 拦截**：TransactionInterceptor 在方法前后调用 begin、commit 或 rollback。
    - **隔离传递**：Spring 调用 JDBC 的 Connection.setTransactionIsolation(level)，底层数据库接管。
    - **线程绑定**：通过 TransactionSynchronizationManager 将事务绑定到当前线程。
- **数据库依赖**：
    - MySQL InnoDB 默认可重复读，用 MVCC 隔离（快照读避免不可重复读）。
    - Oracle 默认读已提交，用锁机制。
- **为什么不自己实现隔离？**：
    - 隔离性依赖数据存储，Spring 作为框架只提供抽象，复用数据库能力。
- **注意事项**：
    - 配置隔离级别需考虑性能（如 SERIALIZABLE 锁开销大）。
    - Propagation（如 REQUIRED）影响事务嵌套，但隔离性由最外层事务决定。
- **实际应用**：
    - 电商订单：用 REPEATABLE_READ 防止库存超卖。
    - 银行转账：用 SERIALIZABLE 确保严格一致性。