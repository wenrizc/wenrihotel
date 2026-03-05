## 1. 使用 `PING` 命令

这是最简单、最直接的方法，用于检查节点是否在线以及网络连接是否正常。

- **方法**：通过 `redis-cli` 连接到指定节点，然后发送 `PING` 命令。
- **命令**：

    ```bash
    redis-cli -h <节点 IP> -p <节点端口> PING
    ```

- **预期响应**：如果节点正常，它将返回 `PONG`。

## 2. 查看集群节点状态

如果您使用的是 Redis 集群，可以使用 `CLUSTER NODES` 命令来获取所有节点的详细状态。

- **方法**：连接到集群中的任意一个节点，然后执行 `CLUSTER NODES`。
- **命令**：

    ```bash
    redis-cli -h <节点 IP> -p <节点端口> CLUSTER NODES
    ```

- **预期输出**：该命令会列出集群中的所有节点及其状态信息，包括节点 ID、IP 地址、端口、角色（主/从）、连接状态等。
    - **正常状态**：所有节点都应显示为 `connected` 状态。
    - **异常状态**：如果某个节点的状态为 `disconnected` 或 `fail`，则表示该节点可能存在问题。

## 3. 检查主从同步状态

对于主从结构的 Redis 部署，检查主从同步是否正常至关重要。

- **方法**：连接到 Redis 节点并执行 `INFO replication` 命令。
- **命令**：

    ```bash
    redis-cli -h <节点 IP> -p <节点端口> INFO replication
    ```

- **预期输出**：
    - **主节点 (Master)**：会显示 `role:master` 以及连接的从节点 (slave) 的信息。
    - **从节点 (Slave)**：会显示 `role:slave`，并指示其主节点的 IP 和端口，以及 `master_link_status` 是否为 `up`。

## 4. 监控 Redis 日志文件

Redis 的日志文件记录了运行过程中的重要事件和错误信息，是排查问题的重要来源。

- **方法**：查看 Redis 节点的日志文件。
- **日志位置**：通常位于 `/var/log/redis/` 目录下，或在 Redis 配置文件 (`redis.conf`) 中指定的路径。
- **关注点**：检查是否有错误信息、警告或异常日志，例如网络问题、内存不足等。

## 5. 检查端口监听状态

您还可以使用操作系统的网络工具来确认 Redis 进程是否在正常监听端口。

- **方法**：在 Redis 服务器上使用 `netstat` 或 `ss` 命令。
- **命令示例**：

    ```bash
    netstat -tuln | grep <Redis 端口>
    ```

- **预期输出**：如果 Redis 正常运行，您应该能看到指定端口处于 `LISTEN` 状态。

## 6. 从“存活”到“健康”：要看的信号

线上判断“节点是否正常工作”，至少要覆盖三个层次：

- **进程存活**：进程在、端口在、能 `PING`。
- **服务可用**：能在目标延迟内执行读写命令，连接数与阻塞情况可控。
- **拓扑一致**：主从/集群角色清晰，复制链路与故障转移状态正常。

## 7. 常用命令清单

```bash
# 基础信息：版本、运行时、内存、统计
redis-cli -h <节点 IP> -p <节点端口> INFO server
redis-cli -h <节点 IP> -p <节点端口> INFO memory
redis-cli -h <节点 IP> -p <节点端口> INFO stats

# 延迟与慢命令
redis-cli -h <节点 IP> -p <节点端口> LATENCY DOCTOR
redis-cli -h <节点 IP> -p <节点端口> SLOWLOG GET 10

# 客户端连接与阻塞情况
redis-cli -h <节点 IP> -p <节点端口> INFO clients
redis-cli -h <节点 IP> -p <节点端口> CLIENT LIST

# 主从/集群
redis-cli -h <节点 IP> -p <节点端口> ROLE
redis-cli -h <节点 IP> -p <节点端口> INFO replication
redis-cli -h <节点 IP> -p <节点端口> CLUSTER INFO
```

解释要点）：

- `SLOWLOG`：找出“命令执行时间长”的请求，通常与大 key、范围查询、阻塞型命令有关。
- `LATENCY DOCTOR`：观察事件循环延迟与抖动，辅助判断 CPU 抢占、系统抖动或持久化 `fork` 影响。
- `INFO memory/stats/clients`：看内存是否逼近上限、命中率是否异常、连接与阻塞是否持续增长。
