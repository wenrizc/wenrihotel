## 1. 为什么需要 Redo Log

Redo Log 的核心目的是解决一个性能和可靠性之间的矛盾，这个技术方案被称为 WAL（Write-Ahead Logging，预写日志）。

1.  性能问题：数据库的数据最终是存储在磁盘上的。如果每次事务提交，我们都必须把所有修改过的数据页（Data Page）都从内存刷回到磁盘，那么会产生大量的随机 I/O。随机 I/O 是非常慢的，这会极大地拖慢数据库的性能。

2.  WAL 解决方案：InnoDB 引入了 Redo Log 来解决这个问题。它的工作流程是：
    - 当事务修改数据时，它首先修改的是内存中的数据页（Buffer Pool）。
    - 同时，它会将这次修改的“物理操作”记录到 Redo Log Buffer（一块内存区域）中。
    - 当事务提交时，它不需要立即将数据页刷盘，而只需要保证将对应的 Redo Log 记录从 Redo Log Buffer 刷到磁盘上的 Redo Log 文件中。
    - 这个 Redo Log 的刷盘操作是顺序 I/O，速度远快于数据页的随机 I/O。

3.  带来的好处：
    - 保证了事务的持久性（Durability）：只要事务提交时 Redo Log 成功写入磁盘，那么即使数据库此时宕机，内存中的数据页修改全部丢失，我们也能在重启后通过重放 Redo Log 来恢复这些修改，保证已提交的事务永不丢失。
    - 提升了性能：将事务提交时的大量随机 I/O 操作，转换成了一次或几次高效的顺序 I/O 操作，极大地提升了数据库的吞吐量。真正的数据页刷盘（称为“刷脏”）可以由后台线程异步、批量地进行。

## 2. Redo Log 的工作流程（两阶段提交）

Redo Log 的写入过程非常精细，涉及到两个关键的指针和一块内存缓冲区：

- `write pos`: 当前记录的位置，一边写一边后移。
- `check point`: 当前要擦除的位置，也是一边擦除一边后移。
- `Log Buffer`: Redo Log 的内存缓冲区。

当事务提交时，为了保证 Binlog 和 Redo Log 之间的一致性（这对于主从复制和数据恢复至关重要），InnoDB 使用了内部的**两阶段提交（Two-phase Commit）**机制：

1.  **Prepare 阶段**：
    - 当执行 `COMMIT` 时，InnoDB 首先会将该事务的 Redo Log 记录写入文件系统缓存，并打上一个“prepare”标记。
    - 然后通知 MySQL Server 层，可以写入 Binlog 了。

2.  **Commit 阶段**：
    - MySQL Server 层接收到通知后，将该事务的 Binlog 写入磁盘。
    - Binlog 写入成功后，Server 层会再调用 InnoDB 的接口，告诉它可以真正提交事务了。
    - InnoDB 收到这个最终确认后，才会在 Redo Log 中写入一个“commit”标记，代表事务的真正完成。

### 2.1 两阶段提交的重要性：

这个机制保证了任何一个事务，要么它的 Redo Log 和 Binlog 都完整存在，要么就都没有。

- 如果数据库在 Redo Log prepare 之后、Binlog 写入前宕机：重启后，InnoDB 发现这个 Redo Log 只有 prepare 标记没有 commit 标记，它会去查询 Binlog 中是否有对应的事务记录。发现没有，就会回滚这个事务。
- 如果数据库在 Binlog 写入之后、Redo Log commit 前宕机：重启后，InnoDB 发现这个 Redo Log 有 prepare 标记，并且在 Binlog 中也能找到对应的事务，它就会认为这个事务是需要被提交的，会继续完成 Redo Log 的 commit 操作。

## 3. Redo Log 与 Binlog 的区别

| 特性 | Redo Log | Binlog |
| :--- | :--- | :--- |
| **所属层次** | InnoDB 引擎层 | MySQL Server 层 |
| **日志格式** | 物理日志（数据页的修改） | 逻辑日志（SQL 语句或行变更） |
| **写入方式** | 循环写入 | 追加写入，写满一个文件换下一个 |
| **核心作用** | 保证事务持久性，实现崩溃恢复 | 实现主从复制和数据恢复 |
| **幂等性** | 是 | 不一定（STATEMENT 格式下） |
