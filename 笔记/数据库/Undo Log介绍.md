
Undo Log（撤销日志）是 InnoDB 存储引擎的核心组成部分，它在数据库的两个关键领域——**并发控制 (MVCC)** 和 **故障恢复 (Crash Recovery)** 中扮演着不可或缺的角色。InnoDB 中的 Undo Log 具有“亦日志亦数据”的特性，这意味着它既像日志一样记录操作，又像数据一样被管理，其自身的持久性也由 Redo Log 来保障。

#### Undo Log 的核心作用

1.  **实现事务的原子性（回滚）**: 数据库保证事务是原子性的，即事务中的所有操作要么全部成功，要么全部失败。当用户执行 `ROLLBACK`，或因死锁、系统崩溃等原因导致事务中断时，数据库需要撤销该事务已经执行的所有修改。Undo Log 记录了数据被修改前的“历史版本”，通过逆向操作这些记录，可以将数据恢复到事务开始前的状态，从而保证原子性。

2.  **实现多版本并发控制 (MVCC)**: 为了提高并发性能，避免读写操作互相阻塞，InnoDB 采用了 MVCC 机制。当一个事务需要读取某行数据时，如果该行正在被另一个未提交的事务所修改，InnoDB 不会直接加锁等待，而是会利用 Undo Log 中存储的历史版本，为读事务提供一个在该事务开始时不包含未提交修改的数据“快照”，从而实现无锁的“一致性读”。

### Undo Log 的组织结构

要理解 Undo Log 的工作机制，首先必须了解其层级化的组织结构：

*   **Undo Record**: 这是最小的单元，记录了单次行记录修改前的镜像。主要分为 `INSERT` 类型（用于回滚插入操作）和 `UPDATE` 类型（用于回滚更新/删除操作，并服务于 MVCC）。
*   **Undo Log**: 一个事务可能会修改多行数据，产生的多个 Undo Record 会被串联成一个链表，形成该事务的 Undo Log。
*   **Undo Segment**: 这是一个物理存储单元，用于存放一个或多个 Undo Log。每个执行写操作的事务都会独占一个 Undo Segment。为了高效利用空间，多个较小的 Undo Log 可以紧凑地存放在同一个 Undo Page 中。
*   **Rollback Segment**: 它是 Undo Segment 的一个集合，通常包含 1024 个 Undo Segment 的槽位（Slot）。数据库的并发写事务数受 Rollback Segment 总数的限制。
*   **Undo Tablespace**: 这是存放 Rollback Segment 的物理文件。从 MySQL 8.0 开始，Undo Tablespace 可以独立于系统表空间（ibdata1）存在，这极大地改善了空间管理和回收的灵活性。

### Undo Log 的生命周期：写入、落盘与丢弃机制

#### 1. 写入与分配 (Writing)

当一个写事务首次执行修改操作时，InnoDB 会为其分配一个 Undo Segment。事务执行的每一次数据修改（INSERT, UPDATE, DELETE），都会执行以下步骤：

1.  **生成 Undo Record**: 根据修改的类型（插入、更新或删除）生成一个对应的 Undo Record，其中包含了恢复数据所需的主键、被修改字段的历史值、指向前一个历史版本的指针（`Rollptr`）等信息。
2.  **写入 Undo Log**: 将这个新生成的 Undo Record 顺序写入当前事务持有的 Undo Segment 的活动 Undo Log 中。
3.  **更新数据记录**: 将数据页（Data Page）中的记录进行修改，并将该记录内部的隐藏字段 `rollptr` 指向刚刚写入的 Undo Record。这个指针构成了版本链，是 MVCC 实现的关键。

#### 2. 落盘与刷新机制 (Flushing to Disk)

这是理解 Undo Log “亦日志亦数据”特性的关键。**Undo Log 本身没有独立的、类似 Redo Log `fsync` 那样的强制刷盘机制**。它的持久化遵循 InnoDB 对所有数据的通用管理方式：

1.  **写入 Buffer Pool**: Undo Log 的写入操作与数据页的修改一样，首先是在内存中的 **Buffer Pool** 中完成的。写入 Undo Record 实际上是修改了 Buffer Pool 中某个 Undo Page 的内容。
2.  **生成 Redo Log**: 对 Undo Page 的所有修改（例如，写入一个新的 Undo Record、更新 Undo Page Header 等）都会被记录为 Redo Log。例如，`MLOG_UNDO_INSERT` 类型的 Redo Log 会记录下向 Undo Page 添加一条 Undo Record 的这一物理操作。
3.  **持久化依赖 Redo Log**: Undo Log 的持久性由 Redo Log 来保证。当事务提交时，其产生的 Redo Log 必须被刷新到磁盘（即 `innodb_flush_log_at_trx_commit` 控制的策略）。只要 Redo Log 落盘了，即使此时数据库崩溃，对应的 Undo Page 还没有从 Buffer Pool 刷到磁盘，在恢复时 InnoDB 也可以通过重放 Redo Log 来重建这个 Undo Page 的修改，从而确保 Undo Log 的完整性。
4.  **异步刷盘**: 变脏的 Undo Page（即在 Buffer Pool 中被修改但尚未写入磁盘的 Undo Page）会像普通的数据页一样，由后台的刷脏线程（Flush List 或 LRU List Flush）在合适的时机异步地写入到对应的 Undo Tablespace 文件中。

总结来说，Undo Log 的“落盘”是一个间接过程：其操作的持久性由同步刷新的 Redo Log 保证，而其本身所在的 Undo Page 则由后台线程异步刷入磁盘。

#### 3. 丢弃与清理机制 (Discarding / Purging)

当一个事务提交后，它产生的 Undo Log 并不会立即被删除。因为可能还有其他正在运行的读事务需要访问这些 Undo Log 来构建数据快照（MVCC）。只有当系统确认某个 Undo Log 记录的历史版本不再被任何事务所需要时，才能进行清理。这个清理过程由后台的 **Purge 线程** 负责，分为以下几个阶段：

1.  **移入 History List**: 当一个写事务提交后，如果其 Undo Log 包含被更新或删除的记录（即 `UPDATE` 类型的 Undo Log），这个 Undo Log 会被移动到一个名为 **History List** 的链表中，并按事务提交的顺序（`trx_no`）排列。

2.  **判断是否可清理**: Purge 线程会检查当前所有活跃的只读事务。每个读事务都有一个 ReadView，其中记录了它启动时系统中最老的活跃事务ID。一个 Undo Log 可以被清理的前提是，创建这个 Undo Log 的事务的提交序号（`trx_no`）比当前所有活跃读事务 ReadView 中记录的最老事务ID还要小。这意味着，所有正在运行的读事务都“看得到”这个 Undo Log 对应的事务的修改，因此不再需要访问这个历史版本了。

3.  **执行 Purge 操作 (Undo Purge)**:
    *   Purge 线程从 History List 中找到符合清理条件的 Undo Log。
    *   它会遍历这些 Undo Log 中的每一条 Undo Record。
    *   对于 `TRX_UNDO_DEL_MARK_REC` 类型的记录（即逻辑删除的记录），Purge 线程会执行真正的物理删除操作，即从主键索引和二级索引中彻底移除该记录。这是因为在执行 `DELETE` 操作时，InnoDB 为了 MVCC 只是给记录打上了一个删除标记，而不是立即物理删除。

4.  **回收 Undo Log 空间 (Undo Truncate)**:
    *   当一个 Undo Log 中的所有 Undo Record 都被处理完毕后，这个 Undo Log 本身就可以被回收了。
    *   Purge 线程会释放这些 Undo Log 所占用的 Undo Page，并将其所属的 Undo Segment 标记为可重用（加入到 Cached List）或直接释放。这个过程就是 **Undo Segment 的截断 (Truncate)**，它会回收 Undo Tablespace 内部的空间。
    *   这个操作通常是批量进行的，其频率由 `innodb_rseg_truncate_frequency` 参数控制。

5.  **收缩 Undo Tablespace 文件 (Undo Tablespace Truncate)**:
    *   如果启用了独立的 Undo Tablespace，并且配置了 `innodb_undo_log_truncate` 选项，当一个 Undo Tablespace 文件中的所有 Undo Segment 都变为空闲状态后，InnoDB 可以将这个文件标记为 `inactive`。
    *   在没有新事务使用这个文件后，InnoDB 会在后台重建这个 Undo Tablespace，将其大小收缩到初始值，从而将文件系统空间返还给操作系统。这解决了以往 Undo Log 空间只增不减的问题。
