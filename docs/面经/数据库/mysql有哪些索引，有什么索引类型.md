
#### MySQL 的索引
- **定义**：
  - 索引是数据库中用于加速查询的数据结构，通过减少扫描数据量提高性能。
- **作用**：
  - 快速定位记录，优化 `WHERE`、`JOIN` 等操作。

#### MySQL 中的索引类型
1. **主键索引（Primary Key）**：
   - 唯一标识每行数据的索引。
2. **唯一索引（Unique Index）**：
   - 确保列值唯一。
3. **普通索引（Normal Index）**：
   - 加速查询，无特殊约束。
4. **全文索引（Full-Text Index）**：
   - 用于文本搜索。
5. **复合索引（Composite Index）**：
   - 多列组合索引。
6. **空间索引（Spatial Index）**：
   - 用于地理数据。

#### 存储引擎差异
- **InnoDB**：默认 B+ 树。
- **MyISAM**：支持 B+ 树和全文索引。

#### 核心点
- B+ 树是主流索引结构。

---

### 1. 索引类型详解
#### (1) 主键索引（Primary Key）
- **特点**：
  - 唯一、不为空，自动创建。
  - InnoDB 中是聚簇索引（数据与索引一起存储）。
- **创建**：
```sql
CREATE TABLE user (
    id INT PRIMARY KEY,
    name VARCHAR(50)
);
```
- **场景**：
  - 主键查询（如 `WHERE id = 1`）。

#### (2) 唯一索引（Unique Index）
- **特点**：
  - 列值唯一，可为空。
  - 非聚簇索引（InnoDB）。
- **创建**：
```sql
CREATE UNIQUE INDEX idx_name ON user(name);
```
- **场景**：
  - 用户名、邮箱唯一性检查。

#### (3) 普通索引（Normal Index）
- **特点**：
  - 无唯一性约束，加速查询。
- **创建**：
```sql
CREATE INDEX idx_age ON user(age);
```
- **场景**：
  - 频繁查询的列（如 `WHERE age > 20`）。

#### (4) 全文索引（Full-Text Index）
- **特点**：
  - 用于文本搜索，支持词匹配。
  - MyISAM 默认支持，InnoDB 5.6+ 支持。
- **创建**：
```sql
CREATE FULLTEXT INDEX idx_content ON article(content);
```
- **使用**：
```sql
SELECT * FROM article WHERE MATCH(content) AGAINST('keyword');
```
- **场景**：
  - 搜索文章、评论。

#### (5) 复合索引（Composite Index）
- **特点**：
  - 多列组成一个索引，按最左前缀原则使用。
- **创建**：
```sql
CREATE INDEX idx_name_age ON user(name, age);
```
- **使用**：
  - 有效：`WHERE name = 'Alice' AND age = 25`。
  - 无效：`WHERE age = 25`（无最左列）。
- **场景**：
  - 多条件查询。

#### (6) 空间索引（Spatial Index）
- **特点**：
  - 用于空间数据类型（如 GEOMETRY）。
  - 基于 R 树。
- **创建**：
```sql
CREATE SPATIAL INDEX idx_location ON geo_table(location);
```
- **场景**：
  - 地理位置查询（如附近门店）。

---

### 2. 底层实现
#### (1) B+ 树索引
- **特点**：
  - 非叶子节点存键，叶子节点存数据，链表连接。
- **优点**：
  - 范围查询快，IO 少。
- **使用**：
  - 主键、唯一、普通、复合索引。

#### (2) 哈希索引
- **特点**：
  - 等值查询快（O(1)），不支持范围。
- **支持**：
  - Memory 引擎默认，InnoDB 自适应哈希。
- **场景**：
  - 键值对查询。

#### (3) 全文索引
- **特点**：
  - 倒排索引，记录词和位置。
- **支持**：
  - MyISAM、InnoDB。

#### (4) R 树索引
- **特点**：
  - 多维索引，适合空间数据。
- **支持**：
  - MyISAM、InnoDB。

---

### 3. InnoDB vs MyISAM
| **索引类型**   | **InnoDB**         | **MyISAM**         |
|----------------|--------------------|--------------------|
| **主键索引**   | 聚簇（数据即索引） | 非聚簇            |
| **唯一索引**   | 支持               | 支持               |
| **普通索引**   | 支持               | 支持               |
| **全文索引**   | 5.6+ 支持          | 默认支持           |
| **空间索引**   | 支持               | 支持               |

---

### 4. 使用与优化
- **创建索引**：
```sql
ALTER TABLE user ADD INDEX idx_name (name);
```
- **查看索引**：
```sql
SHOW INDEX FROM user;
```
- **优化**：
  - 覆盖索引：查询只用索引（如 `SELECT name FROM user WHERE name = 'Alice'`）。
  - 避免冗余索引。

---

### 5. 延伸与面试角度
- **与性能**：
  - 索引加速查询，但增删改慢。
- **实际应用**：
  - 订单表：主键（id）、复合（user_id, time）。
- **面试点**：
  - 问“类型”时，提六种及特点。
  - 问“底层”时，提 B+ 树和哈希。

---

### 总结
MySQL 索引包括主键、唯一、普通、全文、复合和空间索引，主要用 B+ 树实现，优化查询效率。面试时，可提类型分类或举 SQL 示例，展示理解深度。