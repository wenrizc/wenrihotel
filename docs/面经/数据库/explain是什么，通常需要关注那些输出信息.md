
#### EXPLAIN 是什么
- **定义**：
  - EXPLAIN 是 MySQL 提供的一个工具，用于分析 SQL 查询的执行计划，展示查询如何使用索引、扫描多少行等信息。
- **作用**：
  - 帮助开发者理解查询性能，优化慢查询。
- **使用**：
```sql
EXPLAIN SELECT * FROM user WHERE age > 20;
```

#### 通常需要关注的输出信息
1. **id**：查询执行顺序。
2. **select_type**：查询类型。
3. **table**：涉及的表。
4. **type**：访问类型（索引使用情况）。
5. **possible_keys**：可能使用的索引。
6. **key**：实际使用的索引。
7. **rows**：扫描的行数。（重点关注）
8. **Extra**：额外信息。

#### 核心点
- EXPLAIN 是慢查询优化的关键入口。

---

### 1. EXPLAIN 详解
#### (1) 基本概念
- **输出格式**：
  - 表格形式，描述查询的每一步。
- **支持语句**：
  - `SELECT`、`INSERT`、`UPDATE`、`DELETE`。
- **扩展**：
  - `EXPLAIN FORMAT=JSON` 提供更详细信息。

#### (2) 示例输出
```sql
EXPLAIN SELECT name FROM user WHERE age > 20;
```
```
id | select_type | table | type  | possible_keys | key     | rows  | Extra
1  | SIMPLE      | user  | range | idx_age       | idx_age | 500   | Using index condition
```

---

### 2. 需要关注的输出信息
#### (1) id
- **含义**：
  - 查询的执行顺序，子查询或 UNION 有多个 id。
- **关注点**：
  - 值越大越先执行，相同 id 从上到下。

#### (2) select_type
- **含义**：
  - 查询类型。
- **常见值**：
  - **SIMPLE**：简单查询。
  - **PRIMARY**：主查询。
  - **SUBQUERY**：子查询。
  - **DERIVED**：派生表（如 FROM 子句的子查询）。
- **关注点**：
  - 子查询多可能性能差。

#### (3) table
- **含义**：
  - 查询涉及的表名。
- **关注点**：
  - 多表 JOIN 时检查顺序。

#### (4) type
- **含义**：
  - 访问类型，反映索引使用效率。
- **常见值**（从好到差）：
  - **const**：主键或唯一索引等值查询。
  - **eq_ref**：JOIN 时主键或唯一索引匹配。
  - **ref**：非唯一索引等值查询。
  - **range**：范围查询（如 `>`、`<`）。
  - **index**：扫描整个索引。
  - **ALL**：全表扫描。
- **关注点**：
  - `ALL` 表示无索引，需优化。

#### (5) possible_keys
- **含义**：
  - 查询可能用到的索引。
- **关注点**：
  - 若为空，说明无可用索引。

#### (6) key
- **含义**：
  - 实际使用的索引。
- **关注点**：
  - 与 `possible_keys` 对比，若为 NULL，说明未用索引。

#### (7) rows
- **含义**：
  - 预计扫描的行数（估算值）。
- **关注点**：
  - 行数越多，性能越差。

#### (8) Extra
- **含义**：
  - 额外信息，提供优化线索。
- **常见值**：
  - **Using index**：覆盖索引，无需回表。
  - **Using where**：过滤条件。
  - **Using temporary**：用临时表，性能差。
  - **Using filesort**：文件排序，需优化。
- **关注点**：
  - 避免 `Using temporary` 和 `Using filesort`。

---

### 3. 分析示例
#### 查询
```sql
EXPLAIN SELECT * FROM user WHERE age > 20 ORDER BY name;
```
#### 输出
```
id | select_type | table | type  | possible_keys | key     | rows  | Extra
1  | SIMPLE      | user  | range | idx_age       | idx_age | 1000  | Using where; Using filesort
```
- **分析**：
  - `type: range`：用 `idx_age` 范围查询。
  - `rows: 1000`：扫描 1000 行。
  - `Extra: Using filesort`：排序未用索引。
- **优化**：
  - 加复合索引：`CREATE INDEX idx_age_name ON user(age, name)`。

---

### 4. 优化建议
- **优先级**：
  - `type` 从 `ALL` 优化到 `range` 或 `ref`。
  - 减少 `rows`，加索引。
  - 消除 `Using filesort` 和 `Using temporary`。
- **覆盖索引**：
  - 改 `SELECT *` 为 `SELECT age, name`。

---

### 5. 延伸与面试角度
- **与慢查询**：
  - EXPLAIN 分析慢查询原因。
- **实际应用**：
  - 电商：优化订单查询。
- **工具**：
  - `EXPLAIN ANALYZE`（MySQL 8.0+）给实际执行时间。
- **面试点**：
  - 问“关注”时，提 `type` 和 `Extra`。
  - 问“优化”时，提索引调整。

---

### 总结
EXPLAIN 是 MySQL 查询执行计划工具，关注 `id`、`type`、`key`、`rows` 和 `Extra` 等字段，帮助定位性能问题。优化时优先改进访问类型和减少扫描行数。面试时，可提示例分析，展示实战能力。