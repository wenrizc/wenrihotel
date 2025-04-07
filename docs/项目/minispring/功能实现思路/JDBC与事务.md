
## JDBC功能实现 - JdbcTemplate模式

MiniSpring的JDBC实现采用了**模板方法模式**，通过`JdbcTemplate`类封装了所有JDBC操作的共性部分：

```java
public class JdbcTemplate {
    final DataSource dataSource;
    
    // 查询方法
    public <T> T queryForObject(String sql, Class<T> clazz, Object... args)
    public <T> List<T> queryForList(String sql, RowMapper<T> rowMapper, Object... args)
    
    // 更新方法
    public int update(String sql, Object... args)
    public Number updateAndReturnGeneratedKey(String sql, Object... args)
    
    // 核心执行方法
    public <T> T execute(ConnectionCallback<T> action)
    public <T> T execute(PreparedStatementCreator psc, PreparedStatementCallback<T> action)
}
```

### 工厂模式应用

**行映射工厂**：通过不同的`RowMapper`实现，将结果集映射到不同类型对象：

```java
// 基于Class类型动态选择适合的RowMapper
public <T> T queryForObject(String sql, Class<T> clazz, Object... args) {
    if (clazz == String.class) {
        return (T) queryForObject(sql, StringRowMapper.instance, args);
    }
    if (clazz == Boolean.class || clazz == boolean.class) {
        return (T) queryForObject(sql, BooleanRowMapper.instance, args);
    }
    // 默认使用Bean映射器
    return queryForObject(sql, new BeanRowMapper<>(clazz), args);
}
```

### 单例模式应用

针对简单类型的`RowMapper`采用了单例模式实现，避免重复创建：

```java
class StringRowMapper implements RowMapper<String> {
    static StringRowMapper instance = new StringRowMapper();
    
    @Override
    public String mapRow(ResultSet rs, int rowNum) throws SQLException {
        return rs.getString(1);
    }
}
```

## 事务管理实现

### 核心架构设计

MiniSpring事务管理基于AOP和ThreadLocal实现：

```java
public class DataSourceTransactionManager implements PlatformTransactionManager, InvocationHandler {
    // 使用ThreadLocal存储当前线程事务状态
    static final ThreadLocal<TransactionStatus> transactionStatus = new ThreadLocal<>();
    final DataSource dataSource;
    
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 事务处理逻辑
    }
}
```

### 观察者模式应用

事务管理通过`TransactionalBeanPostProcessor`实现对`@Transactional`注解的拦截处理：

```java
public class TransactionalBeanPostProcessor extends AnnotationProxyBeanPostProcessor<Transactional> {
    // 继承自AOP处理器，自动为@Transactional注解的Bean创建代理
}

// 在配置中注册
@Bean
TransactionalBeanPostProcessor transactionalBeanPostProcessor() {
    return new TransactionalBeanPostProcessor();
}
```

事务处理流程：
1. 拦截方法调用
2. 检查当前线程是否有事务上下文
3. 若无，创建新事务，设置autoCommit=false
4. 执行目标方法
5. 根据结果决定提交或回滚
6. 恢复连接状态

### 事务传播实现

```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    TransactionStatus ts = transactionStatus.get();
    if (ts == null) {
        // 无事务上下文，创建新事务
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            transactionStatus.set(new TransactionStatus(connection));
            // 执行业务逻辑
            Object r = method.invoke(proxy, args);
            connection.commit();
            return r;
        } catch (Exception e) {
            // 回滚处理
        } finally {
            transactionStatus.remove();
        }
    } else {
        // 已有事务上下文，复用当前事务
        return method.invoke(proxy, args);
    }
}
```

## 设计模式提高扩展性

1. **工厂模式**：通过`JdbcConfiguration`创建数据库组件，允许灵活替换实现（如更换数据源或事务管理器）
2. **单例模式**：通过容器管理组件生命周期，特定组件（如RowMapper）使用单例优化性能
3. **观察者模式**：通过AOP拦截和通知，实现横切关注点（如事务管理）不侵入业务代码

这些设计使得MiniSpring的JDBC模块具有高度可扩展性，例如可以轻松添加新的结果映射器、事务传播行为或不同类型的数据源支持，而无需修改核心代码。
