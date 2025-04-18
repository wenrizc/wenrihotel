
#### 判断 Redis 节点是否正常
要判断 Redis 节点是否正常工作，可通过以下方法：
1. **客户端连接测试**：检查是否可连接并执行命令。
2. **INFO 命令**：查看运行状态和关键指标。
3. **PING 命令**：检测响应性。
4. **主从状态检查**：确认同步是否正常。
5. **哨兵/集群监控**：查看节点角色和健康状态。
6. **外部工具**：使用监控系统（如 Zabbix、Prometheus）。

#### 核心点
- 从连接性、响应性、数据一致性多角度验证。

---

### 1. 判断方法详解
#### (1) 客户端连接测试
- **方法**：
  - 用 `redis-cli` 或程序尝试连接。
- **命令**：
```bash
redis-cli -h 127.0.0.1 -p 6379
```
- **判断**：
  - 连接成功且可执行 `SET`/`GET`，基本正常。
  - 失败（如 `Connection refused`），可能宕机或网络问题。
- **适用**：
  - 单节点、集群。

#### (2) INFO 命令
- **方法**：
  - 执行 `INFO ALL` 或指定模块，获取运行信息。
- **关键指标**：
  - **uptime_in_seconds**：运行时间，异常重启为小值。
  - **connected_clients**：客户端连接数，异常可能为 0。
  - **used_memory**：内存使用，过高可能 OOM。
  - **rejected_connections**：拒绝连接数，非 0 表示满载。
- **示例**：
```redis
INFO SERVER
# Server
redis_version:6.2.5
uptime_in_seconds:3600
```
- **判断**：
  - 参数正常且无错误，节点健康。

#### (3) PING 命令
- **方法**：
  - 发送 `PING`，期待 `PONG` 响应。
- **示例**：
```bash
redis-cli -h 127.0.0.1 -p 6379 PING
# 返回: PONG
```
- **判断**：
  - 返回 `PONG`：响应正常。
  - 无响应或超时：节点异常。
- **适用**：
  - 快速检测存活。

#### (4) 主从状态检查
- **方法**：
  - 检查主从同步状态。
- **命令**：
```redis
INFO REPLICATION
# Replication
role:master
connected_slaves:1
slave0:ip=127.0.0.1,port=6380,state=online
```
- **判断**：
  - 主节点：`role:master`，从节点数正常。
  - 从节点：`role:slave`，`master_link_status:up`。
  - 异常：从节点 `state:offline` 或延迟大。

#### (5) 哨兵/集群监控
- **哨兵**：
  - 检查哨兵状态：
```redis
SENTINEL get-master-addr-by-name mymaster
# 返回: 127.0.0.1 6379
```
  - 判断：主节点地址正常，哨兵未切换。
- **集群**：
  - 检查节点状态：
```redis
CLUSTER NODES
# 返回: node_id ip:port master/slave slots flags
```
  - 判断：节点 `flags` 无 `fail`，槽分配完整。

#### (6) 外部工具
- **工具**：
  - **Prometheus + Redis Exporter**：监控延迟、内存、连接数。
  - **Zabbix**：自定义脚本检测。
- **指标**：
  - 延迟（latency）、QPS、错误率。
- **判断**：
  - 指标异常（如延迟 > 100ms），节点可能有问题。

---

### 2. 综合判断流程
1. **连接性**：`PING` 或 `redis-cli` 测试。
2. **状态**：`INFO` 查看运行参数。
3. **角色**：主从/集群确认职责。
4. **监控**：外部工具长期观察。
- **示例脚本**：
```bash
#!/bin/bash
HOST="127.0.0.1"
PORT="6379"
if redis-cli -h $HOST -p $PORT PING | grep -q "PONG"; then
    echo "Redis is running"
else
    echo "Redis is down"
fi
```

---

### 3. 异常场景分析
- **无响应**：
  - 进程挂了（`ps -ef | grep redis`）。
  - 网络不通（`telnet 127.0.0.1 6379`）。
- **内存满**：
  - `used_memory` 接近 `maxmemory`，触发淘汰。
- **主从断连**：
  - `master_link_status:down`，需查网络或配置。

---

### 4. 延伸与面试角度
- **与持久化**：
  - AOF/RDB 文件损坏影响恢复，需检查。
- **实际应用**：
  - 电商：库存节点需实时监控。
- **优化**：
  - 哨兵自动切换，减少人工判断。
- **面试点**：
  - 问“方法”时，提 PING 和 INFO。
  - 问“集群”时，提 `CLUSTER NODES`。

---

### 总结
判断 Redis 节点是否正常靠连接测试（PING）、状态检查（INFO）和角色验证（主从/集群），结合外部监控全面评估。面试时，可提脚本或集群命令，展示实践能力。