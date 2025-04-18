
#### MySQL 慢 SQL 优化
- **定义**：
  - 慢 SQL 是指执行时间超过阈值（如 `long_query_time`）的查询，影响性能。
- **优化目标**：
  - 减少执行时间、降低资源消耗、提高并发能力。

#### 优化方法
1. **分析慢查询**：用慢查询日志和 EXPLAIN 定位问题。
2. **优化索引**：添加或调整索引，避免全表扫描。
3. **改写 SQL**：简化查询逻辑，减少复杂操作。
4. **表结构优化**：规范化或分区设计。
5. **配置调优**：调整 MySQL 参数。
6. **分库分表**：处理大数据量。

#### 核心点
- 从索引、SQL 到架构，层层优化。

---

### 1. 分析慢查询
#### (1) 开启慢查询日志
- **配置**：
```sql
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1; -- 阈值 1 秒
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';
```
- **分析工具**：
  - `mysqldumpslow`：汇总慢查询。
```bash
mysqldumpslow -t 10 /var/log/mysql/slow.log
```
  - `pt-query-digest`：更详细分析。

#### (2) 使用 EXPLAIN
- **作用**：
  - 查看执行计划，检查索引、扫描行数等。
- **关注字段**：
  - `type`：`ALL`（全表扫描）需优化。
  - `rows`：扫描行数多需减少。
  - `key`：未用索引（`NULL`）需加。
  - `Extra`：`Using filesort`、`Using temporary` 需优化。
- **示例**：
```sql
EXPLAIN SELECT * FROM user WHERE age > 20 ORDER BY name;
```
  - 结果：
```
id | select_type | table | type | key | rows | Extra
1  | SIMPLE      | user  | ALL  | NULL | 1000 | Using filesort
```
  - 问题：全表扫描、无索引。

---

### 2. 优化索引
#### (1) 添加索引
- **场景**：
  - `WHERE`、`JOIN`、`ORDER BY` 列无索引。
- **示例**：
```sql
CREATE INDEX idx_age ON user(age);
```
  - 优化后：`type: range`，扫描行减少。

#### (2) 复合索引
- **场景**：
  - 多条件查询。
- **示例**：
```sql
CREATE INDEX idx_age_name ON user(age, name);
SELECT * FROM user WHERE age = 25 AND name = 'Alice';
```
  - 遵循最左前缀原则。

#### (3) 覆盖索引
- **场景**：
  - 避免回表。
- **示例**：
```sql
SELECT age, name FROM user WHERE age = 25;
```
  - 用 `idx_age_name` 覆盖查询，`Extra: Using index`。

#### (4) 删除冗余索引
- **场景**：
  - 重复索引浪费空间。
- **检查**：
```sql
SHOW INDEX FROM user;
```
- **删除**：
```sql
DROP INDEX idx_old ON user;
```

---

### 3. 改写 SQL
#### (1) 精简查询
- **问题**：
  - `SELECT *` 返回多余列。
- **优化**：
```sql
-- 差
SELECT * FROM user WHERE age = 25;
-- 好
SELECT id, name FROM user WHERE age = 25;
```

#### (2) 拆分复杂查询
- **问题**：
  - 大 JOIN 或子查询性能差。
- **优化**：
```sql
-- 差
SELECT * FROM user u JOIN order o ON u.id = o.user_id WHERE u.age > 20;
-- 好
SELECT id FROM user WHERE age > 20 INTO @user_ids;
SELECT * FROM order WHERE user_id IN (@user_ids);
```

#### (3) 避免函数和运算
- **问题**：
  - 函数破坏索引。
- **优化**：
```sql
-- 差
SELECT * FROM user WHERE YEAR(create_time) = 2025;
-- 好
SELECT * FROM user WHERE create_time >= '2025-01-01' AND create_time < '2026-01-01';
```

#### (4) 优化排序
- **问题**：
  - `ORDER BY` 无索引，触发 `Using filesort`。
- **优化**：
  - 加索引：
```sql
CREATE INDEX idx_create_time ON user(create_time);
SELECT * FROM user ORDER BY create_time;
```

---

### 4. 表结构优化
#### (1) 字段设计
- **优化**：
  - 用小类型：`INT` 替代 `BIGINT`，`VARCHAR(50)` 替代 `TEXT`。
  - 避免 NULL：用默认值。
- **示例**：
```sql
CREATE TABLE user (
    id INT NOT NULL,
    name VARCHAR(50) NOT NULL DEFAULT ''
);
```

#### (2) 分区表
- **场景**：
  - 大表（如日志）按时间分区。
- **示例**：
```sql
CREATE TABLE log (
    id BIGINT,
    create_time DATETIME
) PARTITION BY RANGE (UNIX_TIMESTAMP(create_time)) (
    PARTITION p0 VALUES LESS THAN (UNIX_TIMESTAMP('2025-01-01')),
    PARTITION p1 VALUES LESS THAN (UNIX_TIMESTAMP('2026-01-01'))
);
```

#### (3) 反范式
- **场景**：
  - 减少 JOIN，用冗余字段。
- **示例**：
  - 订单表存 `user_name`，避免查用户表。

---

### 5. 配置调优
- **参数**：
  - `innodb_buffer_pool_size`：加大缓存（如 70% 内存）。
  - `query_cache_size`（5.7 以下）：启用查询缓存。
  - `tmp_table_size`：增大临时表空间。
- **示例**（my.cnf）：
```ini
[mysqld]
innodb_buffer_pool_size = 2G
tmp_table_size = 64M
```
- **监控**：
  - `SHOW STATUS LIKE 'Innodb_buffer_pool%';`

---

### 6. 分库分表
- **场景**：
  - 单表超千万行，查询慢。
- **方法**：
  - **垂直分表**：拆分字段（如用户信息、扩展信息）。
  - **水平分表**：按 ID 或时间分片。
  - **分库**：按业务（如订单库、用户库）。
- **工具**：
  - MyCat、ShardingSphere。

---

### 7. 其他优化
- **批量操作**：
  - 替换循环插入：
```sql
-- 差
INSERT INTO user (name) VALUES ('Alice');
INSERT INTO user (name) VALUES ('Bob');
-- 好
INSERT INTO user (name) VALUES ('Alice'), ('Bob');
```
- **缓存**：
  - 用 Redis 缓存热点数据。
- **异步处理**：
  - 非核心查询用队列（如 RabbitMQ）。

---

### 8. 延伸与面试角度
- **与 EXPLAIN**：
  - 优化前必用 EXPLAIN 定位问题。
- **实际应用**：
  - 电商：优化订单查询。
  - 日志：分区表加速分析。
- **监控**：
  - `slow_query_log` + Prometheus 跟踪。
- **面试点**：
  - 问“优化”时，提索引和改写。
  - 问“案例”时，提示例 SQL。

---

### 总结
MySQL 慢 SQL 优化需从分析（慢查询日志、EXPLAIN）、索引（复合、覆盖）、SQL 改写、表结构（分区、反范式）、配置到分库分表全面入手。优先解决全表扫描和高扫描行问题。面试时，可提优化流程或示例，展示实战能力。