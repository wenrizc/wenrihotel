

**后端IM通信核心技术方案**

**1. 架构总览与核心组件**

*   **API 网关 (Spring WebFlux)**:
    *   **职责**: 初步用户认证，封装客户端消息（含元数据如`userId`, `gatewayId`, `client_temp_id`），并将其投递到第一级消息队列。自身不处理复杂业务逻辑。
*   **第一级消息队列 (RabbitMQ - "Ingest Queue")**:
    *   **类型**: `Direct Exchange` (or `Topic` for finer-grained routing if needed).
    *   **职责**: 解耦 API 网关与核心消息处理逻辑，削峰填谷，为消息的初步处理和持久化提供缓冲。
*   **`MessageIngestService` (消息接收与预处理服务)**:
    *   **职责**:
        1.  消费来自第一级 MQ 的消息（用户发送的聊天消息）。
        2.  **生成服务端消息标识与顺序**: 接受客户端生成的uuid，同时生成会话内有序的 `sequence_id` (利用 Redis 原子操作)。
        3.  **消息持久化 (异步)**: 将聊天消息及其元数据异步存入 MySQL（使用另一个rabbit mq消息队列）。
        4.  **生成 C2S 发送回执**: 对于用户发送的聊天消息，构建一个回执，通过目标 `gatewayId` (HTTP 调用网关) 发回给原始发送客户端。
        5.  **准备 S2C 推送**: 将处理好的用户聊天消息封装后投递到第二级消息队列。
        6.  **大厅消息风暴处理**: 识别大厅消息，在检测到高并发时切换策略，发送轻量级通知到第二级 MQ，并将完整消息批量缓存到 Redis。
*   **第二级消息队列 (RabbitMQ - "Push Queue")**:
    *   **类型**: `Fanout Exchange` (广播给所有推送服务实例)。
    *   **职责**: 将待推送的消息（聊天内容、状态更新通知）广播给所有  实例，以实现推送任务的负载分担和目标用户定位。
*   **`MessagePushService` (消息推送服务 )**:
    *   **职责**:
        1.  消费来自第二级 MQ 的消息。
        2.  **确定推送目标**: 根据消息中的接收者 `userId`(s)，查询 Redis 获取其在线状态及所连接的 `gatewayId`。
        3.  **在线推送**: 若用户在线，通过内部 RPC/HTTP 调用目标 `gatewayId` 对应的 API 网关实例，由网关将消息通过 WebSocket 推送给目标客户端。
        4.  **离线处理**: 若用户离线，不做任何处理。
        5. 收到来自客户端的回执确认，对没有成功的客户端会重试，如果多次不成功，认为它已经下线，自动执行踢出登陆的操作。
*   **Controller 层 (MVC)**:
    *   **职责**: 提供 HTTP API 接口。
*   **数据与状态管理**:
    *   **Redis**:
        *   用户在线状态及 `gatewayId` 映射。
        *   会话内消息 `sequence_id` 生成器。
        *   大厅消息风暴模式下的消息缓存。
    *   **MySQL**:
        *   完整的聊天历史记录（包括 `message_id`, `sequence_id`, 内容, 发送方, 时间戳, 会话ID等）。
        *   消息的送达与已读状态（针对每个接收者）。
### **2. 关键优化方案的后端实现**

#### **2.1. 大厅消息风暴应对 (信令+旁路拉取)**

此逻辑在**`MessageIngestService`**中实现。

1.  **流量识别**: 在处理一级MQ的消息时，增加一个判断逻辑，识别出消息是否发往高并发的大厅会话。可以通过配置（硬编码的`conversation_id`）或动态检测（基于Redis中维护的会话活跃度计数器）来实现。
2.  **触发阈值**: 设置一个时间窗口内的消息频率阈值（例如，1秒内超过20条消息）。
3.  **策略切换**:
    *   **正常模式**: 低于阈值时，按标准流程将完整消息投递到二级MQ。
    *   **风暴模式**: 超过阈值时，**改变投递到二级MQ的内容**：
        *   不再发送完整的聊天消息，而是发送一条轻量级的**“新消息通知”信令**，如
        *   同时，将实际的聊天消息**批量缓存**到Redis的一个临时`List`中，并使用`LTRIM`保持列表大小在合理范围（如最近200条）。对于踢出的消息会拆分到另一队列，积累到一定数目后主动送出。在送到客户端前会进行压缩
4.  **提供拉取API**:
    *   在`IM服务的Controller`中，提供一个HTTP API
    *   该API接收客户端请求，从Redis的`lobby:cache`列表中拉取一批消息，可选择使用Protobuf进行序列化并用Gzip压缩后返回，实现高效的数据传输。

#### **2.2. 历史与离线消息 (推拉结合)**

*   **历史消息拉取 (信令模式)**:
    *   **实现**: 在`IM服务的Controller`中，提供一个分页查询的HTTP API
    *   **逻辑**: 该接口直接查询**MySQL数据库**（因为历史数据是冷数据），根据分页参数返回历史消息列表。客户端在进入聊天界面或向上滚动时调用此接口。

*   **离线消息处理 (推拉结合)**:
    *   **用户登录时**:
        1.  `用户服务`在处理登录成功后，除了生成Token，还需要执行一个**同步初始化**的动作。
        2.  **推送“摘要”**: 它会查询Redis中该用户的**所有会话未读数** (`user:unread:counts`) 和 **少量最近的离线消息**（从`offline:msg:{userId}`列表中`LPOP`几条）。
        3.  将这些摘要信息（如`{ unread_counts: {"conv1": 10, "conv2": 5}, recent_messages: [...] }`）通过登录成功的HTTP响应或首次WebSocket连接建立后的一条特殊消息推送给客户端。
        4.  客户端收到后，在UI上渲染未读红点，用户点击进入具体会话时，再通过**历史消息拉取API**按需加载完整聊天记录。

#### **2.3. 可靠消息投递 (ACK + 去重 + 有序)**

*   **服务端保障**:
    *   **全局有序**: 通过`Redis INCR`生成的`sequenceId`，为每个会话内的消息提供了严格的顺序。
    *   **消息不丢失**: RabbitMQ的持久化模式 + 生产者确认/消费者ACK机制，确保消息在从网关到`MessagePushService`的整个后端流程中不会丢失。
    *   **ACK处理**: 客户端发送的送达/已读ACK，同样作为一条特殊消息经由`网关 -> 一级MQ -> MessageIngestService`处理。服务收到后，更新MySQL中对应消息的状态字段。

*   **客户端配合 (后端需提供支持)**:
    *   **去重**: 后端推送给客户端的每条消息都必须包含一个**全局唯一的消息ID**（`message_id`，可以是数据库主键）和**会话内有序的序列号**（`sequence_id`）。客户端维护一个基于`message_id`的近期消息ID缓存池（如一个有固定大小的`Set`），收到新消息时先检查ID是否存在，若存在则直接丢弃，实现幂等性。
    *   **排序**: 客户端始终根据`sequence_id`对收到的消息进行排序展示，即使消息因网络原因乱序到达，也能在UI上正确显示。
