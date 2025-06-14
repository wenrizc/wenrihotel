
### 一、 问题现象

*   使用 `DELETE` 命令删除了 InnoDB 表中的大量数据（甚至一半），但对应的 `.ibd` 表文件大小并未减小。

### 二、 InnoDB 存储基础

1.  **`innodb_file_per_table` 参数**：
    *   `ON` (默认自 MySQL 5.6.6 起)：每个 InnoDB 表的数据和索引存储在独立的 `.ibd` 文件中。**推荐设置**，便于管理和空间回收（`DROP TABLE` 直接删除文件）。
    *   `OFF`：表数据存储在共享表空间 (ibdata1) 中，即使 `DROP TABLE`，空间也不会自动回收。
    *   **本文讨论基于 `innodb_file_per_table=ON`**。

2.  **表结构与数据**：
    *   表结构定义（`.frm` 文件或系统表）占用空间小，主要关注数据文件 (`.ibd`)。

### 三、 `DELETE` 命令的工作原理与“空洞”

1.  **记录删除**：执行 `DELETE` 时，InnoDB 仅仅是将记录**标记为已删除**状态。该空间后续**可能**被符合范围条件的新插入记录复用。
2.  **页删除**：当一个数据页上的**所有记录**都被删除后，该数据页会被标记为**可复用**，可以被后续任何需要新页的操作使用。
3.  **空间不回收**：无论是标记记录删除还是标记页可复用，**都不会导致磁盘上的 `.ibd` 文件自动收缩**。这些被标记但未使用的空间形成了“**空洞**”。
4.  **“空洞”的产生**：
    *   `DELETE` 操作。
    *   **随机 `INSERT`** 导致的**页分裂** (新数据插入已满页面，需分裂出新页，原页末尾留空)。
    *   `UPDATE` 操作（可理解为删除旧值+插入新值）。
5.  **结论**：大量增删改操作后，表中普遍存在空洞，导致文件大小虚高。仅靠 `DELETE` 无法回收这部分空间。

### 四、 解决方案：重建表 (消除空洞，收缩空间)

*   **核心思想**：创建一个新的临时表/文件，将原表中的有效数据紧凑地复制过去，然后替换掉原表。

1.  **方法：`ALTER TABLE table_name ENGINE=InnoDB;`** (等同于 `recreate`)
    *   **MySQL 5.5 及之前 (非 Online DDL)**：
        1.  创建临时表 (Server 层)。
        2.  锁定原表 A，禁止写入。
        3.  将表 A 数据逐行读出并插入临时表 B。
        4.  用表 B 替换表 A。
        5.  删除表 A。
        *   **缺点**：整个过程**长时间锁表**，阻塞 DML，**非 Online**。
    *   **MySQL 5.6 及之后 (Online DDL)**：
        1.  获取 MDL 写锁 (短暂)。
        2.  创建临时文件 (`tmp_file`，InnoDB 内部)。
        3.  **降级为 MDL 读锁** (允许 DML)。
        4.  扫描原表 A 主键，将数据页记录构建成 B+ 树存入临时文件。
        5.  **同时**将此期间对表 A 的 DML 操作记录在**日志文件 (row log)** 中。
        6.  临时文件生成后，获取 MDL 写锁 (短暂)。
        7.  将 row log 中的操作应用到临时文件，使其数据与表 A 一致。
        8.  用临时文件替换表 A 的数据文件。
        9.  释放 MDL 写锁。
        *   **优点**：主要的数据拷贝阶段**允许 DML**，锁表时间极短，**基本 Online**。

2.  **Online DDL 的 MDL 锁说明**：
    *   虽然开始和结束需要短暂的 MDL 写锁，但耗时最长的数据拷贝阶段持有的是 MDL 读锁，不阻塞 DML，因此称为 Online。MDL 读锁用于防止其他 DDL 操作干扰重建过程。

3.  **`inplace` vs `copy` 算法**：
    *   `ALTER TABLE ... ENGINE=InnoDB` 默认使用 `ALGORITHM=inplace`。
    *   **`inplace`**：DDL 主要在 InnoDB 内部完成，使用内部临时文件 (`tmp_file`)，Server 层看来是“原地”操作。**注意：仍需要额外的临时磁盘空间**。
    *   **`copy`**：强制使用旧的、非 Online 的拷贝表方式 (Server 层临时表)。
    *   **关系**：Online DDL **一定是** `inplace` 的；但 `inplace` 的 DDL **不一定** 是 Online 的（例如添加 FULLTEXT 或 SPATIAL 索引）。

4.  **其他相关命令**：
    *   `OPTIMIZE TABLE table_name;`：在 InnoDB 中，基本等同于 `ALTER TABLE ... ENGINE=InnoDB` (recreate) + `ANALYZE TABLE`。用于回收空间和更新统计信息。
    *   `ANALYZE TABLE table_name;`：**仅**重新统计索引信息，不修改数据，不回收空间。加 MDL 读锁。

5.  **第三方工具**：对于非常大的表，为更安全地执行在线 DDL，可考虑使用 `gh-ost` 或 `pt-online-schema-change`。

### 五、 潜在问题

*   重建表后，文件大小**可能略微增大**。原因可能包括页填充率的变化、InnoDB 版本内部结构差异等，导致数据在新文件中组织得不那么“极致”紧凑。

### 六、 总结

*   `DELETE` 只标记空间可复用，不缩小文件。
*   回收 InnoDB 表空间（消除空洞）需要**重建表**。
*   推荐使用 `ALTER TABLE ... ENGINE=InnoDB` 或 `OPTIMIZE TABLE ...` 命令，利用 MySQL 5.6+ 的 **Online DDL** 功能，在业务低峰期执行。
*   重建表是 I/O 和 CPU 密集型操作，需要评估资源消耗和操作时间。
*   确保 `innodb_file_per_table=ON` 是进行表空间管理的前提。