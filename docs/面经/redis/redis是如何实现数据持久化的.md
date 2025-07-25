
Redis为了保证数据在服务器重启或意外宕机后不丢失，提供了两种主要的持久化机制：RDB（Redis DataBase）和AOF（Append Only File）。这两种机制可以单独使用，也可以同时使用。

1.  RDB (Redis DataBase) 持久化：
    *   原理：RDB持久化是在指定的时间间隔内，将内存中的数据集快照（Snapshot）写入到磁盘上的一个二进制文件（默认为 `dump.rdb`）。它记录的是某一时刻Redis数据库的全量数据。
    *   触发方式：
        *   手动触发：通过客户端执行 `SAVE` 或 `BGSAVE` 命令。
            *   `SAVE`：阻塞Redis服务器进程，直到RDB文件创建完毕。在服务器阻塞期间，不能处理客户端请求。不推荐在生产环境频繁使用。
            *   `BGSAVE`：Redis会fork出一个子进程，由子进程负责将数据写入临时的RDB文件。父进程继续处理客户端请求。当子进程完成写入后，会用新的RDB文件替换旧的RDB文件。这是推荐的在线生成RDB快照的方式。
        *   自动触发：通过配置文件中的 `save` 选项设置。例如 `save 900 1` 表示如果在900秒内，至少有1个key发生了改变，则自动触发 `BGSAVE`。可以配置多条 `save` 规则。
        *   执行 `FLUSHALL` 命令时，也会产生一个空的RDB文件。
        *   执行复制（Replication）时，如果主节点没有正在执行的 `BGSAVE`，或者RDB文件不够新，主节点也会执行 `BGSAVE` 并将RDB文件发送给从节点。
    *   优点：
        *   RDB文件是一个紧凑的二进制文件，非常适合用于备份、灾难恢复以及数据迁移。
        *   恢复速度快：在Redis重启时，加载RDB文件恢复数据通常比加载AOF文件快，因为它直接将内存映像加载回来。
        *   对性能影响小（使用`BGSAVE`）：父进程fork子进程后，可以继续处理请求，实际的I/O操作由子进程完成。
    *   缺点：
        *   数据丢失风险较高：RDB是间隔性地进行快照。如果在两次快照之间发生服务器宕机，那么最后一次快照之后的所有数据修改都会丢失。这个间隔时间越长，丢失的数据就越多。
        *   fork子进程的开销：虽然父进程不阻塞，但fork子进程本身在数据量较大时可能会消耗一定的CPU时间和内存（写时复制Copy-On-Write机制），可能导致服务短暂卡顿。

2.  AOF (Append Only File) 持久化：
    *   原理：AOF持久化记录的是服务器接收到的每一个写命令（如SET, SADD, LPUSH等），这些命令以Redis协议的格式追加到一个文件（默认为 `appendonly.aof`）的末尾。当Redis重启时，会重新执行AOF文件中保存的写命令来恢复数据集。
    *   写入策略（`appendfsync`配置）：
        *   `always`：每个写命令都立即同步到AOF文件磁盘，最安全，但性能最低。
        *   `everysec`（默认）：每秒钟同步一次，即异步将写命令缓冲区的数据写入AOF文件。这是性能和安全性的一个较好折中。即使发生宕机，最多只会丢失1秒内的数据。
        *   `no`：由操作系统决定何时同步，性能最好，但数据丢失风险最高。
    *   AOF重写（Rewrite）：
        *   由于AOF文件会随着写命令的增多而不断变大，Redis提供了AOF重写机制来压缩AOF文件体积。
        *   重写原理：Redis会fork一个子进程，这个子进程会读取当前数据库中的键值对状态，然后用最少的命令（例如，多个INCR命令可以合并成一个SET命令）来重新生成一个新的、更紧凑的AOF文件。在重写期间，父进程接收到的新的写命令会同时写入旧的AOF文件和AOF重写缓冲区。当子进程完成新AOF文件的生成后，父进程会将重写缓冲区中的增量命令追加到新AOF文件中，然后用新的AOF文件替换旧的。
        *   触发方式：手动执行 `BGREWRITEAOF` 命令，或者通过配置自动触发（如 `auto-aof-rewrite-percentage` 和 `auto-aof-rewrite-min-size`）。
    *   优点：
        *   数据完整性更好：根据`appendfsync`策略，可以做到丢失数据较少（甚至不丢失，如果设置为`always`）。
        *   AOF文件是可读的（虽然是协议格式），可以方便地进行分析和修复（如果出现错误）。
    *   缺点：
        *   文件体积通常比RDB文件大（除非经过了高效的重写）。
        *   恢复速度通常比RDB慢，因为需要重新执行所有写命令。
        *   根据`appendfsync`策略，对性能有一定影响，特别是`always`模式。

选择与组合：

*   只用RDB：如果能容忍一段时间的数据丢失，并且对恢复速度有要求。
*   只用AOF：如果对数据完整性要求非常高，不能容忍太多数据丢失。
*   同时使用RDB和AOF（推荐）：
    *   Redis 4.0及以后版本，引入了混合持久化（AOF an RDB preamble）。当开启混合持久化时，AOF重写会生成一个以RDB格式开头的AOF文件，其中RDB部分包含了重写开始时的数据库快照，后续的增量写命令仍然以AOF格式追加。
    *   这样做的好处是：
        *   在恢复时，可以先加载RDB部分快速恢复大部分数据，然后再加载AOF增量部分，使得恢复速度接近RDB，同时数据完整性又接近AOF（只丢失AOF最后一次同步到重写完成之间的增量命令）。
        *   AOF文件本身仍然是增量追加的，保证了数据安全性。
    *   这种方式结合了RDB和AOF的优点。

在Redis重启加载数据时，如果同时开启了RDB和AOF持久化，Redis会优先加载AOF文件来恢复数据，因为它通常能保证数据的最新状态。

总结来说，Redis通过RDB（快照）和AOF（命令日志）两种方式实现数据持久化，各有优缺点。现代Redis版本推荐同时使用两者，并开启混合持久化模式，以兼顾恢复速度和数据完整性。开发者可以根据业务对数据安全、性能和运维成本的需求来配置合适的持久化策略。
