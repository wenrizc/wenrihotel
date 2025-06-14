
#### 一、背景与目标
- **MySQL 优势**：高性能、低成本、资源丰富，互联网公司首选。
- **优化重要性**：读写比约 10:1，查询性能瓶颈突出，尤其是复杂查询。
- **目标**：通过索引优化慢查询，提升系统效率。

---

#### 二、索引原理
##### 1. 索引目的
- **提高查询效率**：类比字典，通过索引快速定位数据，减少全表扫描。

##### 2. 索引基础
- **原理**：缩小数据范围，随机查询变顺序查询。
- **查询类型**：等值（如 `=`）、范围（如 `>`、`<`）、模糊（如 `LIKE`）、并集（如 `OR`）。

##### 3. 磁盘 IO 与预读
- **磁盘 IO 成本**：
  - 寻道（~5ms）+ 旋转延迟（7200转 ~4.17ms）+ 传输（忽略）≈ 9ms。
  - 相比内存操作（指令级 ns），IO 成本高约 10万倍。
- **预读优化**：
  - 局部预读性：一次 IO 读取整页（4K/8K），降低单次 IO 频率。

##### 4. B+ 树数据结构
- **需求**：控制 IO 次数为常数级别。
- **B+ 树特点**：
  - 多路搜索树，非叶子节点存指引数据，叶子节点存真实数据。
  - 示例：3 层 B+ 树支持百万数据，3 次 IO。
- **性质**：
  - 高度 \( h = \log_{m+1}(N) \)（\( N \) 为数据量，\( m \) 为每块数据项数）。
  - \( m = 磁盘块大小 / 数据项大小 \)，字段越小，\( m \) 越大，\( h \) 越低。
  - **最左前缀匹配**：复合索引按顺序比较，缺失前缀字段无法利用后续索引。

---

#### 三、慢查询优化原则
1. **最左前缀匹配**：
   - 匹配至范围查询（`>`、`<`、`BETWEEN`、`LIKE`）停止。
   - 例：`(a,b,c,d)` 索引，`a=1 and b=2 and c>3 and d=4`，`d` 用不到。
2. **等值查询乱序**：
   - `=` 和 `IN` 可优化顺序，如 `(a,b,c)` 索引支持 `a=1 and c=3 and b=2`。
3. **高区分度列**：
   - 区分度 = `count(distinct col)/count(*)`，越高越好（如唯一键=1，状态字段~0）。
   - 建议：Join 字段区分度 > 0.1。
4. **索引列“干净”**：
   - 避免计算，如 `from_unixtime(create_time)` 改为 `create_time = unix_timestamp()`。
5. **扩展现有索引**：
   - 如已有 `a` 索引，加 `(a,b)` 只需扩展。

##### 优化步骤
1. **验证慢查询**：加 `SQL_NO_CACHE` 测试。
2. **单表分析**：查各字段区分度，锁定最小记录表。
3. **Explain 检查**：验证执行计划与预期一致。
4. **优先排序表**：`ORDER BY LIMIT` 时先查排序表。
5. **了解场景**：业务需求决定优化方向。
6. **建索引**：遵循原则。
7. **验证效果**：迭代优化。

##### Explain 核心指标
- **rows**：扫描行数，核心优化目标。
- **type**：访问类型（`ALL` 全扫、`ref` 索引、`eq_ref` 主键等）。
- **Extra**：附加信息（如 `Using temporary`、`Using filesort`）。

---

#### 四、案例分析
##### 1. 初始慢查询
```sql
SELECT COUNT(*) 
FROM task 
WHERE status=2 AND operator_id=20839 
AND operate_time>1371169729 AND operate_time<1371174603 
AND type=2;
```
- **问题**：工程师建议所有字段加索引。
- **优化**：
  - **联合索引**：`(status, operator_id, type, operate_time)`。
  - **顺序调整**：`operate_time` 放最后（范围查询），其他可乱序。
  - **综合评估**：结合其他查询，如 `status=0 and type=12`。
- **结果**：索引 `(status, type, operator_id, operate_time)` 覆盖多种场景。

##### 2. 复杂语句优化
```sql
SELECT DISTINCT cert.emp_id 
FROM cm_log cl 
INNER JOIN (
    SELECT emp.id AS emp_id, emp_cert.id AS cert_id 
    FROM employee emp 
    LEFT JOIN emp_certificate emp_cert ON emp.id = emp_cert.emp_id 
    WHERE emp.is_deleted=0
) cert 
ON (cl.ref_table='Employee' AND cl.ref_oid=cert.emp_id) 
OR (cl.ref_table='EmpCertificate' AND cl.ref_oid=cert.cert_id) 
WHERE cl.last_upd_date>='2013-11-07 15:03:00' AND cl.last_upd_date<='2013-11-08 16:00:00';
```
- **问题**：1.87s，`derived2` 扫描 63727 行，`cm_log` 只需 379 行。
- **优化**：
  - 拆为两段 `UNION`，避免无效 Join：
    ```sql
    SELECT emp.id FROM cm_log cl 
    INNER JOIN employee emp ON cl.ref_table='Employee' AND cl.ref_oid=emp.id 
    WHERE cl.last_upd_date>='2013-11-07 15:03:00' AND cl.last_upd_date<='2013-11-08 16:00:00' 
    AND emp.is_deleted=0 
    UNION 
    SELECT emp.id FROM cm_log cl 
    INNER JOIN emp_certificate ec ON cl.ref_table='EmpCertificate' AND cl.ref_oid=ec.id 
    INNER JOIN employee emp ON emp.id=ec.emp_id 
    WHERE cl.last_upd_date>='2013-11-07 15:03:00' AND cl.last_upd_date<='2013-11-08 16:00:00' 
    AND emp.is_deleted=0;
    ```
- **结果**：10ms，提升 200 倍。

##### 3. 区分度低的场景
```sql
SELECT * FROM stage_poi sp 
WHERE sp.accurate_result=1 AND sp.sync_status IN (0, 2, 4);
```
- **问题**：6.22s，全表扫描 361万行。
- **分析**：
  - `accurate_result`：3 值（-1, 0, 1），区分度低。
  - `sync_status`：2 值（0, 3），区分度低。
- **场景**：业务每 5 分钟扫描 ~1000 条，数据不平衡。
- **优化**：加索引 `idx_acc_status(accurate_result, sync_status)`。
- **结果**：0.20s，提升 30 倍。

##### 4. 无法优化的语句
```sql
SELECT c.id, ... FROM contact c 
INNER JOIN contact_branch cb ON c.id=cb.contact_id 
INNER JOIN branch_user bu ON cb.branch_id=bu.branch_id AND bu.status IN (1, 2) 
INNER JOIN org_emp_info oei ON oei.data_id=bu.user_id 
AND oei.node_left>=2875 AND oei.node_right<=10802 AND oei.org_category=-1 
ORDER BY c.created_time DESC LIMIT 0, 10;
```
- **问题**：13.06s，排序前 77.8 万行。
- **尝试**：
  - 先排序后 Join：
    ```sql
    SELECT c.id, ... FROM contact c 
    WHERE EXISTS (
        SELECT 1 FROM contact_branch cb 
        INNER JOIN branch_user bu ON cb.branch_id=bu.branch_id AND bu.status IN (1, 2) 
        INNER JOIN org_emp_info oei ON oei.data_id=bu.user_id 
        AND oei.node_left>=2875 AND oei.node_right<=10802 AND oei.org_category=-1 
        WHERE c.id=cb.contact_id
    ) ORDER BY c.created_time DESC LIMIT 0, 10;
    ```
  - 结果：0s（理想），但极端条件（如 `node_left=2875 AND node_right=2875`）2分18秒。
- **原因**：Nested Loop + Limit 机制，极端情况遍历全表。
- **结论**：无法 SQL 优化，需业务逻辑调整。

---

#### 五、总结与经验
- **索引关键**：B+ 树降低 IO，遵循最左匹配、高区分度原则。
- **优化核心**：减少 `rows`，结合业务场景。
- **注意事项**：
  - 并非所有查询可优化。
  - 避免盲目索引，考虑更新成本。
  - 测试极端场景，避免遗漏。
- **复杂案例**：多表 Join、垃圾 SQL 需深入原理分析。
