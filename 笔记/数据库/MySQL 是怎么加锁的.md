
#### 哪些SQL语句会加行级锁

1. 锁定读语句 (必须在事务中)
    
    - `select ... lock in share mode;` (加共享锁，S锁)
        
    - `select ... for update;` (加独占锁，X锁)
        
2. 数据修改语句
    
    - `update ... where ...;` (加独占锁，X锁)
        
    - `delete from ... where ...;` (加独占锁，X锁)
        

注意：普通的`select`语句是快照读，通过MVCC实现，不加锁（串行化隔离级别除外）。

#### 行级锁的种类（可重复读隔离级别下）

1. Record Lock (记录锁)
    
    - 锁定单条索引记录。
        
2. Gap Lock (间隙锁)
    
    - 锁定一个索引范围，但不包含记录本身（开区间）。
        
    - 主要目的：防止幻读。
        
    - 间隙锁之间是互相兼容的。
        
3. Next-Key Lock (临键锁)
    
    - Record Lock + Gap Lock 的组合。
        
    - 锁定一个范围，并锁定记录本身（前开后闭区间）。
        
    - 是InnoDB加锁的基本单位。
        

#### 行级锁加锁规则

基本原则：

- 加锁的对象是索引。
    
- 加锁的基本单位是 Next-Key Lock。
    
- 在某些场景下，Next-Key Lock 会退化为 Record Lock 或 Gap Lock，以达到在能避免幻读的前提下，提高并发度的目的。
    

分析工具：`select * from performance_schema.data_locks\G;`

- LOCK_MODE 为 `X`：表示 Next-Key Lock。
    
- LOCK_MODE 为 `X, REC_NOT_GAP`：表示 Record Lock。
    
- LOCK_MODE 为 `X, GAP`：表示 Gap Lock。
    

（一）唯一索引（主键索引）等值查询

1. 查询记录存在的情况
    
    - 规则：在索引上定位到该记录后，Next-Key Lock 退化为 Record Lock。
        
    - 示例：`select * from user where id = 1 for update;`
        
    - 加锁：对 `id=1` 的主键索引加一条 Record Lock。
        
    - 原因：Record Lock 阻止了删除和更新。主键的唯一性约束阻止了插入相同ID的记录。这样已经足以避免幻读，无需锁定间隙。
        
2. 查询记录不存在的情况
    
    - 规则：在索引上找到第一个大于查询值的记录，将其 Next-Key Lock 退化为 Gap Lock。
        
    - 示例：`select * from user where id = 2 for update;` (表中无id=2, 下一条记录是id=5)
        
    - 加锁：对 `id=5` 的主键索引加一个范围为 `(1, 5)` 的 Gap Lock。
        
    - 原因：锁定这个间隙可以防止 `id=2,3,4` 的记录被插入，避免了幻读。无需锁定 `id=5` 这条记录本身。
        

（二）唯一索引（主键索引）范围查询

1. 大于 (>) 或 大于等于 (>=) 查询
    
    - 规则：对扫描到的每个索引项都加 Next-Key Lock。但对于`>=`查询，如果等值匹配的记录存在，该记录上的 Next-Key Lock 会退化为 Record Lock。
        
    - 示例 (`id >= 15`)：
        
        - 对 `id=15` 加 Record Lock。
            
        - 对 `id=20` 加 Next-Key Lock `(15, 20]`。
            
        - 对最后的`supremum`伪记录加 Next-Key Lock `(20, +∞]`。
            
2. 小于 (<) 或 小于等于 (<=) 查询
    
    - 规则：对扫描到的满足条件的索引项加 Next-Key Lock。对于第一个不满足条件的索引项，其 Next-Key Lock 是否退化要分情况。
        
    - 当条件值记录不存在时 (如 `id < 6`): 对第一个不满足条件的记录(`id=10`)的锁，会退化为 Gap Lock `(5, 10)`。
        
    - 当条件值记录存在时 (如 `id=5`):
        
        - 若是 `<=` 查询 (`id <= 5`): 对 `id=5` 的锁是 Next-Key Lock `(1, 5]`，不会退化。
            
        - 若是 `<` 查询 (`id < 5`): 对第一个不满足条件的记录(`id=5`)的锁，会退化为 Gap Lock `(1, 5)`。
            

（三）非唯一索引等值查询

注意：非唯一索引查询会同时对二级索引和主键索引加锁。

1. 查询记录不存在的情况
    
    - 规则：在二级索引上找到第一个大于查询值的记录，将其 Next-Key Lock 退化为 Gap Lock。不加主键锁。
        
    - 示例：`select * from user where age = 25 for update;` (表中无age=25, 下一条是age=39)
        
    - 加锁：对 `age=39` 的二级索引加一个范围为 `(22, 39)` 的 Gap Lock。
        
2. 查询记录存在的情况
    
    - 规则：这是一个扫描过程。
        
        - 对所有扫描到的符合条件的二级索引记录，加 Next-Key Lock。
            
        - 对这些符合条件的记录，在它们的主键索引上加 Record Lock。
            
        - 对扫描到的第一个不符合条件的二级索引记录，其 Next-Key Lock 退化为 Gap Lock。
            
    - 示例：`select * from user where age = 22 for update;`
        
    - 加锁：
        
        - 二级索引：对 `age=22` 加 Next-Key Lock `(21, 22]`；对第一个不满足条件的 `age=39` 加 Gap Lock `(22, 39)`。
            
        - 主键索引：对匹配记录 `id=10` 加 Record Lock。
            
    - 原因：需要加额外的Gap Lock `(22, 39)` 来防止其他事务插入新的 `age=22` 的记录（如 id=12, age=22），从而避免幻读。
        

（四）非唯一索引范围查询

- 规则：与唯一索引范围查询不同，非唯一索引范围查询过程中，所有扫描到的二级索引项上的 Next-Key Lock 都不会退化。
    
- 示例：`select * from user where age >= 22 for update;`
    
- 加锁：
    
    - 二级索引：对 `age=22` 加 Next-Key Lock `(21, 22]`；对 `age=39` 加 Next-Key Lock `(22, 39]`；对`supremum`伪记录加 Next-Key Lock `(39, +∞]`。
        
    - 主键索引：对匹配记录 `id=10` 和 `id=20` 分别加 Record Lock。
        
- 原因：因为索引值不唯一，如果退化为记录锁，将无法阻止其他事务插入相同索引值的新记录，导致幻读。
    

（五）没有加索引的查询

- 规则：如果`update`, `delete`, `select for update` 等语句的 `where` 条件没有使用索引，会进行全表扫描。
    
- 加锁：会对表中每一条记录的聚簇索引（主键索引）都加上 Next-Key Lock。
    
- 效果：这相当于锁住了整张表，会严重阻塞其他所有事务的增、删、改操作。
    
- 结论：线上环境执行加锁性质的语句，务必确保 `where` 条件走了索引。
    

#### 加锁规则流程

（一）唯一索引加锁流程

1. 进行等值查询还是范围查询？
    
    - 等值查询：
        
        - 查询的记录是否存在？
            
            - 存在：加 Record Lock。
                
            - 不存在：找到下一个记录，对其加 Gap Lock。
                
    - 范围查询：
        
        - 扫描到的每一条记录，都加上 Next-Key Lock。
            
        - 然后根据特定规则判断是否退化：
            
            - 对于 `>=`，等值部分退化为 Record Lock。
                
            - 对于 `<` 或 `<=`，在扫描到终止记录时，根据终止记录是否存在以及是 `<` 还是 `<=`，可能退化为 Gap Lock。
                

（二）非唯一索引加锁流程

1. 进行等值查询还是范围查询？
    
    - 等值查询：
        
        - 查询的记录是否存在？
            
            - 存在：对符合条件的二级索引加 Next-Key Lock，对其对应的主键加 Record Lock。对第一个不符合条件的二级索引记录加 Gap Lock。
                
            - 不存在：找到下一个二级索引记录，对其加 Gap Lock。
                
    - 范围查询：
        
        - 扫描到的所有二级索引记录，全部加 Next-Key Lock。
            
        - 对所有符合条件的记录，在它们的主键上加 Record Lock。
            
        - （注意：非唯一索引范围查询不会发生锁退化）。