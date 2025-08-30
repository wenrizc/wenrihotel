
#### undo log (回滚日志)

1.  为什么需要 undo log？
    *   为了实现事务的原子性。当事务需要回滚（如发生错误或用户执行ROLLBACK）或MySQL崩溃时，可以利用undo log将数据恢复到事务开始前的状态。

2.  undo log 如何工作？
    *   在进行增、删、改操作前，MySQL会先将回滚所需的信息记录到undo log。
    *   插入操作：记录插入记录的主键，回滚时删除该主键对应的记录。
    *   删除操作：记录被删除记录的完整内容，回滚时重新插入这些内容。
    *   更新操作：记录被更新列的旧值，回滚时将列恢复为旧值。
    *   注意：delete操作并非立即删除，而是打上删除标记，由purge线程后续清理。更新主键列的操作会分解为“先删除，再插入”。

3.  undo log 的两大作用：
    *   事务回滚：保障事务的原子性。
    *   实现MVCC (多版本并发控制)：
        *   记录的每次更新都会产生一个undo log，并通过`roll_pointer`指针将这些log串联起来，形成版本链。
        *   在“读提交”和“可重复读”隔离级别下，快照读（普通SELECT）通过ReadView结合undo log版本链，找到对当前事务可见的记录版本，从而实现非阻塞的并发读取。

4.  undo log 的持久化：
    *   undo log 本身也需要持久化保护。它和数据页一样，先被写入Buffer Pool中的Undo页面，对Undo页面的修改同样会记录到redo log中，其持久化由redo log来保证。

---

#### Buffer Pool (缓冲池)

1.  为什么需要 Buffer Pool？
    *   为了弥补磁盘和CPU之间的巨大速度差异。MySQL将磁盘中的数据按“页”（默认16KB）加载到内存中的Buffer Pool进行读写操作，以提高性能。

2.  Buffer Pool 的工作机制：
    *   读取：数据在Buffer Pool中则直接返回，否则从磁盘加载到Buffer Pool再返回。
    *   修改：直接修改Buffer Pool中的页，并将该页标记为“脏页”（内存数据与磁盘不一致）。脏页由后台线程在合适的时机刷回磁盘，而不是每次修改都立即刷盘。

3.  Buffer Pool 缓存内容：
    *   主要缓存数据页和索引页。
    *   此外还包括Undo页、插入缓存、自适应哈希索引、锁信息等。

---

#### redo log (重做日志)

1.  为什么需要 redo log？
    *   解决因MySQL宕机、断电等导致Buffer Pool中的脏页数据丢失的问题，从而保证事务的持久性。
    *   引入了WAL（Write-Ahead Logging，预写日志）技术：写操作先写日志，再在合适时机写磁盘。

2.  redo log 如何工作？
    *   当记录需要更新时，InnoDB会先更新内存中的页（标记为脏页），然后将“对哪个页做了什么修改”的物理日志记录到redo log中。事务提交时，只需保证redo log已落盘即可。
    *   即使系统崩溃，脏页数据丢失，重启后MySQL也可以根据redo log的内容，将数据恢复到崩溃前的最新状态，这个能力称为crash-safe（崩溃恢复）。

3.  为什么不直接写数据盘而要多一步 redo log？
    *   性能提升：写redo log是顺序写磁盘，而更新数据文件是随机写磁盘。顺序写的效率远高于随机写。WAL技术将随机写转换为了顺序写，大大提升了写入性能。

4.  redo log 和 undo log 的区别：
    *   redo log 记录的是事务“修改后”的数据状态，用于崩溃恢复，保证持久性。
    *   undo log 记录的是事务“修改前”的数据状态，用于事务回滚，保证原子性。

5.  redo log 的写入过程：
    *   产生的redo log会先写入内存中的`redo log buffer`。
    *   `redo log buffer`中的日志会在特定时机刷盘。

6.  redo log 的刷盘时机：
    *   MySQL正常关闭时。
    *   `redo log buffer`使用量超过一半时。
    *   后台线程每秒定时刷盘。
    *   每次事务提交时（由`innodb_flush_log_at_trx_commit`参数控制）。

7.  `innodb_flush_log_at_trx_commit`参数详解：
    *   `1` (默认值): 每次事务提交时，都将redo log从buffer持久化到磁盘。数据安全性最高，性能开销最大。
    *   `0`: 每次事务提交时，redo log仍留在buffer中，由后台线程每秒将其写入Page Cache并刷盘。MySQL崩溃会丢失最后一秒的事务。性能最好，安全性最低。
    *   `2`: 每次事务提交时，仅将redo log从buffer写入操作系统的Page Cache。后台线程每秒将其从Page Cache刷盘。MySQL崩溃不丢数据，但操作系统宕机或断电会丢失最后一秒的事务。性能和安全性的折中。

8.  redo log 文件写满了怎么办？
    *   redo log文件组（如`ib_logfile0`, `ib_logfile1`）以循环方式写入。
    *   `write pos`是当前记录位置，`checkpoint`是要擦除的位置（代表该位置前的日志对应的脏页都已刷盘）。
    *   当`write pos`追上`checkpoint`时，表示日志文件已满，MySQL会阻塞，强制将一些脏页刷盘，推动`checkpoint`前进，腾出空间后才能继续执行更新。

---

#### binlog (归档日志)

1.  为什么需要 binlog？
    *   历史原因：MySQL最初的MyISAM引擎没有crash-safe能力，binlog用于归档。后来InnoDB作为插件引入，自带redo log实现crash-safe。
    *   核心用途：用于数据备份恢复和主从复制。

2.  redo log 和 binlog 的区别：
    *   适用对象：redo log是InnoDB特有的；binlog是MySQL Server层的，所有引擎通用。
    *   文件格式：redo log是物理日志（记录页修改）；binlog是逻辑日志，有三种格式：
        *   STATEMENT：记录SQL原文，可能因动态函数（如NOW()）导致主从不一致。
        *   ROW：记录行的变更内容，准确但文件体积大。
        *   MIXED：混合模式，根据情况自动选择。
    *   写入方式：redo log是循环写，空间固定；binlog是追加写，写满一个文件后会创建新文件。
    *   用途：redo log用于崩溃恢复；binlog用于备份恢复和主从复制。如果数据库被误删，只能用binlog恢复。

3.  主从复制的实现原理：
    *   依赖binlog。过程分为三个阶段：
        1.  主库写入Binlog：主库执行事务，写入binlog。
        2.  从库同步Binlog：从库的I/O线程连接主库，将binlog复制到自己的中继日志（relay log）。
        3.  从库回放Binlog：从库的SQL线程读取relay log，重放其中的操作，更新自己的数据。
    *   复制模型：
        *   异步复制（默认）：主库不等从库响应，性能好但主库宕机可能丢数据。
        *   同步复制：主库等所有从库响应，性能和可用性极差。
        *   半同步复制：主库等待至少一个从库响应即可，兼顾性能与数据安全。

4.  binlog 的刷盘时机：
    *   事务执行中，binlog先写入`binlog cache`（每个线程一份）。
    *   事务提交时，`binlog cache`中的内容被写入binlog文件。
    *   `sync_binlog`参数控制刷盘频率：
        *   `0`: 只写入Page Cache，由操作系统决定何时刷盘。性能最好，风险最高。
        *   `1`: 每次事务提交都刷盘。性能最差，安全性最高。
        *   `N` (>1): 每N个事务提交后刷盘一次。性能和安全的折中。

---

#### 两阶段提交 (Two-Phase Commit, 2PC)

1.  为什么需要两阶段提交？
    *   为了保证redo log（影响主库数据）和binlog（影响从库数据）这两个独立日志之间的数据一致性。
    *   防止在写完一个日志后、写另一个日志前发生崩溃，导致主从数据不一致。

2.  两阶段提交的过程：
    *   MySQL内部使用XA事务，以binlog为协调者，存储引擎为参与者。
    *   Prepare阶段：InnoDB将redo log写入磁盘，并将其中的事务状态标记为`prepare`。
    *   Commit阶段：Server层将binlog写入磁盘，然后通知InnoDB提交事务，InnoDB将redo log中的事务状态标记为`commit`。

3.  异常重启后的恢复逻辑：
    *   MySQL重启后，扫描redo log，找到处于`prepare`状态的事务。
    *   用该事务的XID去binlog中查找。
    *   如果binlog中有此XID：说明binlog已成功写入，则提交该事务。
    *   如果binlog中没有此XID：说明binlog未成功写入，则回滚该事务。
    *   核心思想：以binlog是否成功写入作为事务是否提交的最终标志。

4.  两阶段提交的问题及优化（组提交）：
    *   问题：
        *   磁盘I/O次数高：在“双1”配置下，每个事务至少2次刷盘（redo log一次，binlog一次）。
        *   锁竞争激烈：早期版本用一把大锁（`prepare_commit_mutex`）串行化提交过程，并发性能差。
    *   组提交(Group Commit)优化：将多个并发提交的事务打包，一次性完成刷盘，减少I/O次数和锁粒度。
    *   组提交阶段（以“双1”配置为例）：
        1.  Flush阶段：多个事务的redo log被一次性`fsync`刷盘；然后将这些事务的binlog一次性`write`到Page Cache。
        2.  Sync阶段：等待一个短暂延时（`binlog_group_commit_sync_delay`）或凑够一定数量的事务（`binlog_group_commit_sync_no_delay_count`）后，将Page Cache中的多个事务的binlog一次性`fsync`刷盘。
        3.  Commit阶段：通知InnoDB引擎层，完成各个事务的最终提交。

---

#### MySQL磁盘I/O高优化方法

当磁盘I/O成为瓶颈时，可以通过牺牲一定的数据安全性来换取性能：
1.  调整组提交参数：设置`binlog_group_commit_sync_delay`和`binlog_group_commit_sync_no_delay_count`，通过适当延迟来聚合更多的事务，减少binlog刷盘次数。
2.  调整`sync_binlog > 1`：允许多个事务的binlog聚合后再刷盘，主机掉电时可能丢失最近N个事务的binlog。
3.  调整`innodb_flush_log_at_trx_commit = 2`：让redo log的刷盘交由操作系统调度，主机掉电时可能丢失最后一秒的事务数据。

---

#### 一条UPDATE语句的完整流程

1.  执行器通过索引找到记录，若记录不在Buffer Pool，则从磁盘加载。
2.  执行器判断记录更新前后是否一致，不一致则继续。
3.  InnoDB开启事务，先记录undo log到Buffer Pool的Undo页（此操作也会产生redo log）。
4.  InnoDB更新Buffer Pool中的数据页（成为脏页），并记录相应的redo log。
5.  Server层记录该语句的binlog到binlog cache。
6.  事务提交，开始两阶段提交：
    a.  Prepare阶段：将redo log刷盘，状态置为`prepare`。
    b.  Commit阶段：将binlog刷盘，然后通知引擎将redo log状态置为`commit`。
7.  一条更新语句执行完毕。