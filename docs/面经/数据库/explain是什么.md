
`EXPLAIN` 是 SQL 中一个非常强大的诊断工具，当你在一个 SQL 查询语句（如 `SELECT`, `INSERT`, `UPDATE`, `DELETE` 等）前加上 `EXPLAIN` 关键字时，数据库并不会真正执行这条语句，而是会返回它将如何执行这条语句的详细计划。 这个执行计划揭示了数据库访问数据的方式，例如表的连接顺序、使用了哪些索引、数据是如何被扫描和排序的等等。

### EXPLAIN 能得到什么结果

`EXPLAIN` 的输出通常是一个表格，其中包含了多个列，不同的数据库（如 MySQL, PostgreSQL）输出的列会略有不同，但核心信息是相似的。以下是 MySQL 中 `EXPLAIN` 输出的主要列及其含义：

| 列名 (Column)       | 描述                                                                                                                                                  |
| :---------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------- |
| **id**            | 查询中每个 `SELECT` 子句的唯一标识符。id 越大，执行优先级越高。                                                                                                              |
| **select_type**   | `SELECT` 查询的类型（例如 `SIMPLE` 简单查询, `SUBQUERY` 子查询, `UNION` 联合查询等）。                                                                                    |
| **table**         | 当前行正在访问的表的名称。                                                                                                                                       |
| **partitions**    | 查询将匹配到的分区。对于未分区的表，此值为 `NULL`。                                                                                                                       |
| **type**          | **[重要]** 显示了数据库如何查找表中的行，是评估查询性能的关键指标。 常见的类型从最优到最差依次为：`system` > `const` > `eq_ref` > `ref` > `range` > `index` > **`ALL`**。 `ALL` 表示全表扫描，通常意味着性能问题。 |
| **possible_keys** | 指出数据库可以用来查找行的索引。                                                                                                                                    |
| **key**           | 数据库实际决定使用的索引。如果为 `NULL`，则表示没有使用索引。                                                                                                                  |
| **key_len**       | 使用的索引的长度（字节数）。这个值可以帮助判断复合索引是否被完全利用。                                                                                                                 |
| **ref**           | 显示哪些列或常量被用于与 `key` 列中的索引进行比较。                                                                                                                       |
| **rows**          | **[重要]** 估算的为了找到所需行而需要读取的行数。 这个数值越小越好。                                                                                                              |
| **filtered**      | 表示按表条件过滤后，剩下的行数的百分比。`rows` * `filtered` / 100 可以估算出将与下一张表连接的行数。                                                                                     |
| **Extra**         | **[重要]** 包含了不适合在其他列中显示但非常重要的额外信息。 例如 `Using filesort`（表示需要进行外部排序，效率低）、`Using temporary`（表示使用了临时表）、`Using index`（表示使用了覆盖索引，性能好）。                     |

### 使用 `EXPLAIN` 排查问题

假设我们有一个用户表 `users`，表结构和数据如下：

**表结构:**
```sql
CREATE TABLE `users` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(50) DEFAULT NULL,
  `email` varchar(50) DEFAULT NULL,
  `status` tinyint(4) DEFAULT '0',
  PRIMARY KEY (`id`)
);
```

现在，我们需要查找所有状态为 "活跃"（假设 `status = 1`）的用户。我们执行以下查询：

```sql
SELECT * FROM users WHERE status = 1;
```

在数据量很大的情况下，我们发现这个查询非常慢。这时就可以使用 `EXPLAIN` 来诊断问题。

#### 第一步：执行 `EXPLAIN`

在查询前加上 `EXPLAIN` 关键字：

```sql
EXPLAIN SELECT * FROM users WHERE status = 1;
```

#### 第二步：分析 `EXPLAIN` 的输出结果

你可能会得到类似下面的结果（不同版本的数据库和数据量，结果可能略有差异）：

| id | select_type | table | type | possible_keys | key  | key_len | ref  | rows   | Extra |
|----|-------------|-------|------|---------------|------|---------|------|--------|-------|
| 1  | SIMPLE      | users | **ALL**  | NULL          | NULL | NULL    | NULL | 1000000 | Using where |

**问题诊断：**

1.  **`type` 列为 `ALL`**：这是一个非常明确的危险信号，表示数据库正在进行**全表扫描**。 也就是说，即使我们只想要几条 `status = 1` 的记录，数据库也必须检查表中的每一行（估算的 `rows` 为 100 万行）。 这是导致查询缓慢的根本原因。
2.  **`key` 列为 `NULL`**：这证实了数据库没有使用任何索引来执行这个查询。

#### 第三步：提出优化方案

为了避免全表扫描，最直接有效的办法是在查询条件涉及的列（这里是 `status` 列）上创建一个索引。

**优化方案：** 在 `status` 列上添加索引。

```sql
CREATE INDEX idx_status ON users(status);
```

#### 第四步：验证优化效果

现在我们再次对相同的查询执行 `EXPLAIN`：

```sql
EXPLAIN SELECT * FROM users WHERE status = 1;
```

输出结果可能会变成这样：

| id | select_type | table | type  | possible_keys | key          | key_len | ref   | rows   | Extra |
|----|-------------|-------|-------|---------------|--------------|---------|-------|--------|------------------|
| 1  | SIMPLE      | users | **ref**   | idx_status    | idx_status   | 1       | const | 50000  | Using index condition |

**效果分析：**

1.  **`type` 列变为 `ref`**：查询类型从 `ALL`（全表扫描）变成了 `ref`。这表示数据库通过索引找到了所有匹配的行，性能远高于 `ALL`。
2.  **`key` 列显示 `idx_status`**：明确地告诉我们，新创建的索引 `idx_status` 已经被成功使用。
3.  **`rows` 列显著减少**：预估扫描的行数从 100 万急剧下降到了 5 万（假设有 5 万活跃用户）。数据库不再需要遍历整张表，而是直接通过索引定位到需要的数据，查询速度会得到数量级的提升。

通过这个简单的例子，我们可以看到 `EXPLAIN` 是如何帮助我们一步步发现问题、制定策略并验证优化的。在处理复杂的 `JOIN` 查询或子查询时，`EXPLAIN` 同样能提供关键的洞察，帮助你理解查询的每一个步骤，从而进行精确的性能调优。