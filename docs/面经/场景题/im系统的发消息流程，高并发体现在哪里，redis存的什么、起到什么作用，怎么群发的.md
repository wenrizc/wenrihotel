
#### 发消息流程
IM（即时消息）系统的发消息流程一般包括：客户端发送消息 -> 服务器接收并处理 -> 存储消息 -> 推送给接收方。具体步骤如下：
1. 客户端发送消息（JSON/WebSocket）。
2. 服务器验证用户身份和权限。
3. 消息入库（存 MySQL 或其他持久化存储）。
4. 推送消息（单聊推给个体，群聊推给群成员）。

#### 高并发体现
- **大量用户同时在线**：发送和接收消息的高频并发。
- **实时性要求**：消息需快速分发，延迟低。
- **群聊广播**：一个群消息需推送给数百或千用户。

#### Redis 作用
- **存储内容**：
  - 用户在线状态（如 `user:online:<userId>`）。
  - 会话列表（如 `session:<userId>`）。
  - 群成员列表（如 `group:members:<groupId>`）。
- **作用**：
  - 快速查询在线用户和群成员。
  - 缓存热点数据，减少数据库压力。

#### 群发实现
- 获取群成员列表（Redis）。
- 遍历在线成员，推送消息（WebSocket 或 MQ）。

---

### 1. 发消息流程详解
#### 单聊流程
1. **客户端发送**：
   - 用户 A 通过 WebSocket 发送 `{ "to": "userB", "msg": "hello" }`。
2. **服务器处理**：
   - 验证 A 的 Token，确认合法性。
   - 检查 B 是否在线（Redis 查询）。
3. **存储**：
   - 消息存入 MySQL（如 `messages` 表：`id`, `from`, `to`, `content`, `time`）。
4. **推送**：
   - 若 B 在线，通过 WebSocket 推给 B。
   - 若 B 离线，存离线消息，待上线推送。

#### 群聊流程
1. **客户端发送**：
   - 用户 A 发送 `{ "groupId": "g1", "msg": "hi all" }`。
2. **服务器处理**：
   - 验证 A 是否群成员。
   - 获取群成员列表（Redis）。
3. **存储**：
   - 存入群消息表（如 `group_messages`）。
4. **推送**：
   - 遍历在线成员，推送消息。

#### 图示
```
客户端A       服务器         Redis         MySQL         客户端B
  |  msg ----->|             |             |             |
  |            | 验证身份    |             |             |
  |            | 查询在线 --->| user:online |             |
  |            | 存消息 ----->|             | messages   |
  |            | 推送 ------>|             |            |----> msg
```

---

### 2. 高并发体现在哪里
#### 高并发点
1. **连接管理**：
   - 数万用户通过 WebSocket 长连接，服务器需维持大量连接。
2. **消息分发**：
   - 单聊：一对一推送。
   - 群聊：一对多广播，高并发下放大请求量。
3. **数据库压力**：
   - 高频消息存取，QPS 可能达万级。
4. **实时性**：
   - 要求毫秒级延迟，处理速度需极快。

#### 挑战
- **吞吐量**：每秒处理大量消息。
- **扩展性**：用户增长需水平扩展。
- **一致性**：消息顺序和可靠性。

---

### 3. Redis 存储内容及作用
#### 存储内容
1. **用户在线状态**：
   - Key：`user:online:<userId>`。
   - Value：布尔值或时间戳。
   - 作用：快速判断推送目标。
2. **会话列表**：
   - Key：`session:<userId>`。
   - Value：Set 或 List（如最近联系人）。
   - 作用：缓存会话，提升响应。
3. **群成员列表**：
   - Key：`group:members:<groupId>`。
   - Value：Set（成员 ID 集合）。
   - 作用：快速获取推送目标。
4. **离线消息**（可选）：
   - Key：`offline:<userId>`。
   - Value：List（未读消息）。
   - 作用：暂存离线用户消息。

#### 作用
- **加速查询**：O(1) 时间复杂度，替代数据库慢查询。
- **减轻压力**：热点数据缓存，减少 MySQL 负载。
- **分布式支持**：集群化 Redis 适应高并发。

#### 示例
```java
Jedis jedis = new Jedis("localhost", 6379);
// 检查在线
boolean isOnline = jedis.exists("user:online:userB");
// 获取群成员
Set<String> members = jedis.smembers("group:members:g1");
```

---

### 4. 群发实现
#### 步骤
1. **获取群成员**：
   - Redis：`SMEMBERS group:members:<groupId>`。
2. **过滤在线用户**：
   - 遍历成员，检查 `user:online:<userId>`。
3. **推送消息**：
   - 在线用户：通过 WebSocket 推送。
   - 离线用户：存入离线队列（如 Redis 或 MQ）。

#### 示例代码
```java
public void sendGroupMessage(String groupId, String msg) {
    Jedis jedis = new Jedis("localhost", 6379);
    Set<String> members = jedis.smembers("group:members:" + groupId);
    
    // 存储群消息
    saveGroupMessage(groupId, msg);
    
    // 推送
    for (String userId : members) {
        if (jedis.exists("user:online:" + userId)) {
            webSocket.send(userId, msg); // WebSocket 推送
        } else {
            jedis.lpush("offline:" + userId, msg); // 离线存储
        }
    }
}
```

#### 高并发优化
- **异步推送**：用线程池或 MQ（如 Kafka）分发。
- **批量处理**：批量推送减少 IO。
- **分布式架构**：多节点分担推送。

---

### 5. 延伸与面试角度
- **高并发优化**：
  - **负载均衡**：Nginx 分发 WebSocket。
  - **MQ**：Kafka 解耦推送。
  - **分布式锁**：Redis 保证消息顺序。
- **一致性**：
  - 事务存储消息，推送失败可重试。
- **实际应用**：
  - 微信：群聊消息广播。
  - Slack：实时通知。
- **面试点**：
  - 问“流程”时，提存储和推送。
  - 问“高并发”时，提 Redis 和 MQ。

---

### 总结
IM 发消息流程包括验证、存储、推送，高并发体现在连接和分发，Redis 存在线状态和群成员加速查询，群发靠遍历推送。面试时，可画流程图或提 Kafka 优化，展示设计能力。