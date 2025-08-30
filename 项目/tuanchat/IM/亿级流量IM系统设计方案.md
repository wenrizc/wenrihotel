


**核心目标：**

1.  **峰值QPS：** 18M/li (Messages per second, assuming "li" is a typo for "消息" or messages)
2.  **可靠性：** 5个9 (99.999%)
3.  **收发延迟：** 10ms以下 (核心路径)
4.  **消息时序一致性：** 发送与接收端顺序一致，不重不漏。
5.  **功能：** 万人群聊，好友管理，单聊等。
6.  **可运维性：** 易于部署、监控、扩展和故障排除。

**架构总览**

我们将采用分层、微服务化的架构，核心组件包括：客户端、接入层、业务逻辑层、数据存储层以及各类支撑服务。

```
+---------------------+     +----------------------+     +-----------------------+
|      客户端         | --> |   IPConf/DNS服务     | --> |   L4负载均衡 (SLB)    |
| (iOS, Android, Web) |     +----------------------+     +-----------------------+
+---------------------+                                          |
                                                                 v
+-----------------------------------------------------------------------------------+
|                                   接入层 (Gateway)                                  |
| - TCP长连接管理 (Netty/Go原生)                                                      |
| - TLS/SSL加密卸载                                                                  |
| - 用户身份认证 (轻量级)                                                              |
| - FD <-> UID 映射 (可存Redis或本地+协调服务同步)                                       |
| - 心跳维持                                                                        |
| - 上行消息初步解析与转发 (至MQ)                                                        |
| - 下行消息推送 (从MQ消费)                                                            |
| - 设计为尽可能无状态或状态易重建                                                        |
+-----------------------------------------------------------------------------------+
                                         ^     | (上行消息/ACK)
                                         |     v (下行消息)
+-----------------------------------------------------------------------------------+
|                               消息队列 (Message Queue - Kafka/RocketMQ)             |
| - 单聊Topic、群聊Topic、信令Topic、ACK Topic等                                        |
| - 削峰填谷、异步解耦                                                                 |
| - 保证消息可靠投递 (至少一次)                                                          |
+-----------------------------------------------------------------------------------+
                                         ^     | (消费消息)
                                         |     v (生产消息)
+-----------------------------------------------------------------------------------+
|                               业务逻辑层 (IM Server Core)                           |
|  - 用户服务 (UserSvc): 用户信息、好友关系、黑名单                                       |
|  - 消息服务 (MessageSvc):                                                         |
|    - 消息ID生成、消息持久化 (写扩散/读扩散)                                             |
|    - 消息路由与分发逻辑                                                             |
|    - 离线消息处理、消息状态同步 (已送达、已读)                                        |
|    - 维护会话SeqID                                                                 |
|  - 群组服务 (GroupSvc): 群创建/解散、成员管理、群消息处理 (扩散)                           |
|  - 信令服务 (SignalingSvc): 好友申请、入群邀请等                                       |
|  - 离线推送服务 (OfflinePushSvc): 调用APNS/FCM/厂商通道                             |
+-----------------------------------------------------------------------------------+
     |          |          |          |          |
     v          v          v          v          v
+------------------+  +------------------+  +------------------+  +------------------+
|    关系型数据库    |  |  NoSQL数据库     |  |   分布式缓存     |  |   对象存储       |
| (MySQL/PostgreSQL)|  | (HBase/Cassandra)|  |   (Redis Cluster)|  |   (S3/OSS)       |
| - 用户信息、关系   |  | - 消息内容 (海量)|  | - 用户在线状态    |  | - 图片、音视频    |
| - 群组信息        |  | - 会话列表       |  | - FD-UID映射     |  |                  |
| - 少量关键消息索引 |  |                  |  | - 热点数据缓存    |  |                  |
+------------------+  +------------------+  +------------------+  +------------------+

支撑服务：
-   **服务发现与配置中心 (ZooKeeper/Etcd/Nacos):** IPConf依赖它发现接入层节点，各微服务间发现与配置。
-   **分布式ID生成服务 (Snowflake):** 生成全局唯一的消息ID、会话ID等。
-   **监控告警系统 (Prometheus/Grafana/ELK):** 监控系统健康度、性能指标，及时告警。
-   **日志服务 (ELK Stack):** 收集、存储、查询日志。
```

**核心模块详解**

1.  **客户端 (Client)**
    *   **网络库：** 实现高效的TCP长连接，支持断线重连、心跳保活。
    *   **协议：** 自定义二进制协议 (如Protobuf + LengthFieldBasedFrameDecoder) 以提高效率和安全性。
    *   **本地存储：** SQLite/CoreData等，存储消息、会话列表、联系人、SeqID等。
    *   **状态同步：** 维护每个会话的`max_seq_id`，用于增量拉取消息。
    *   **UI交互：** 消息收发、已读未读、好友/群组管理等。

2.  **IPConf/DNS服务**
    *   **功能：** 客户端启动时请求该服务，获取负载较低、网络延迟最优的接入层服务器IP列表。
    *   **实现：**
        *   可以是智能DNS，根据客户端来源IP返回就近的SLB IP。
        *   也可以是一个HTTP服务，后端通过服务发现（如ZooKeeper）获取所有健康的接入层SLB的IP，并结合接入层节点的实时负载信息（由接入层节点上报给协调服务或一个中心化的调度服务），进行智能调度。
    *   **高可用：** 本身需要多实例部署和负载均衡。

3.  **L4负载均衡 (SLB/NLB)**
    *   **功能：** 将客户端的TCP连接请求分发到后端的多个接入层(Gateway)实例。
    *   **策略：** 基于源IP的哈希或最少连接数。
    *   **健康检查：** 定期检查后端接入层实例的健康状况，自动剔除故障节点。

4.  **接入层 (Gateway)**
    *   **长连接管理：**
        *   使用Netty (Java) 或 Go 原生 `net` 包等高性能网络框架。
        *   维护客户端连接 (FD)，处理TCP粘包、半包问题。
        *   管理 `FD <-> UID` 的映射关系。此映射关系可以：
            *   存储在当前Gateway节点的内存中，并通过服务发现/路由组件确保特定UID的消息总是被路由到正确的Gateway。
            *   或者，存储在外部共享的分布式缓存（如Redis）中：`UID -> GatewayID`，`GatewayID:FD -> UID`。这样Gateway重启时，客户端重连后可以快速恢复。
    *   **心跳机制：** 客户端定时发送心跳包，服务器响应；服务器长时间未收到心跳则断开连接。
    *   **认证与安全：**
        *   TLS/SSL加密通信。SLB可以做TLS卸载，也可以由Gateway自身处理。
        *   客户端连接后，发送认证Token (如JWT)。Gateway通过调用用户服务或本地校验（如果Token可自解释）来验证用户身份。认证成功后才建立UID与FD的映射。
    *   **消息上下行：**
        *   **上行：** 接收客户端消息 -> 解析协议 -> 附加上下文信息 (如UID, DeviceID) -> 投递到消息队列 (MQ) 的指定Topic。
        *   **下行：** 从MQ消费属于本Gateway所连接用户的消息 -> 按协议封装 -> 通过对应FD推送给客户端。
    *   **"变与不变的隔离" 和 "无状态化"：**
        *   **不变的逻辑：** 连接管理、心跳、基础协议解析、消息的原始收发。这部分逻辑相对稳定，在Gateway层实现。
        *   **变化的逻辑：** 复杂的业务处理、需要持久化的状态（如好友关系变更、群成员变更等）。这些逻辑不应在Gateway层处理，而是通过MQ将事件/消息传递给后端的业务逻辑层。
        *   Gateway尽量做到“无业务状态”。连接状态（FD-UID映射）是其核心状态，但这部分状态可以通过客户端重连和外部缓存（Redis）辅助快速重建。频繁迭代的业务信令，其解析和处理逻辑应在后端业务服务中，Gateway仅做透传或简单分发。
        *   如果Gateway需要处理一些轻量级的、与连接本身相关的业务信令（如在线状态上报），这些信令应设计得简单且不依赖复杂DB操作。

5.  **消息队列 (MQ - 如Kafka, RocketMQ)**
    *   **核心作用：** 异步化、解耦、削峰填谷。
    *   **Topic规划：**
        *   `P2P_MESSAGE_TOPIC`: 单聊消息。
        *   `GROUP_MESSAGE_TOPIC`: 群聊消息。
        *   `SIGNALING_TOPIC`: 好友申请、加群邀请等信令。
        *   `ACK_TOPIC`: 消息送达、已读回执。
        *   `ONLINE_STATUS_TOPIC`: 用户上下线状态。
        *   `PUSH_MESSAGE_TOPIC_PREFIX_GATEWAYID` (或者每个Gateway监听自己负责的UID范围的消息): 用于业务层向特定Gateway推送消息。
    *   **分区 (Partitioning):**
        *   单聊消息可按 `sender_id` 或 `receiver_id` 或 `conversation_id` 进行分区，确保同一会话消息有序。
        *   群聊消息可按 `group_id` 分区。
    *   **可靠性：** 配置为至少一次投递，配合下游业务服务的幂等处理，实现不重不漏。

6.  **业务逻辑层 (IM Server Core - 微服务集群)**
    *   **用户服务 (UserSvc):**
        *   用户注册、登录、个人资料管理。
        *   好友关系管理 (添加、删除、列表、备注)。
        *   黑名单管理。
    *   **消息服务 (MessageSvc):**
        *   **消费MQ消息：** 从 `P2P_MESSAGE_TOPIC`, `GROUP_MESSAGE_TOPIC` 消费。
        *   **ID生成：** 调用分布式ID服务为每条消息生成全局唯一的 `server_msg_id`。
        *   **SeqID管理：**
            *   为每个会话（单聊、群聊）维护一个递增的 `seq_id`。当新消息到达时，分配新的`seq_id`。
            *   可以基于Redis的INCR命令实现会话级别的`seq_id`生成器。
        *   **消息持久化：**
            *   **写扩散 (适用于读多写少，如用户消息列表)：** 一条消息写入发件人库、收件人库（或多个群成员的库）。
            *   **读扩散 (适用于写多读少，如微博Feed流)：** 一条消息只写一份，读取时聚合。IM中，单聊常用“写扩散到收件箱”，群聊则可能是一份消息体+多份指向该消息的索引。
            *   存储消息内容、发送者、接收者/群组ID、`server_msg_id`、`client_msg_id` (用于客户端去重)、`seq_id`、时间戳、消息类型等。
        *   **消息路由与分发：**
            *   **单聊：** 查找接收者UID当前连接的GatewayID (从Redis查询 `UID -> GatewayID` 映射)。将消息投递到对应Gateway的MQ Topic (如 `PUSH_MESSAGE_TOPIC_GATEWAYID_X`) 或一个通用的推送Topic，由各Gateway按需消费。
            *   **群聊：** 从群组服务获取群成员列表。对每个在线成员，执行类似单聊的推送逻辑。对离线成员，标记为离线消息。
        *   **离线消息：** 消息持久化后，如果用户不在线，则存储为离线消息。用户上线时，客户端携带本地`max_seq_id`请求，MessageSvc拉取大于此`seq_id`的消息。
        *   **消息回执处理：** 消费 `ACK_TOPIC`。
            *   **送达回执：** Gateway成功推送给客户端后，发ACK给MessageSvc，更新消息状态为“已送达”。
            *   **已读回执：** 客户端上报已读，MessageSvc更新消息状态为“已读”，并通知消息发送方（通过其Gateway）。
    *   **群组服务 (GroupSvc):**
        *   创建群、解散群、邀请/踢出成员、修改群信息、获取群成员列表。
        *   处理群消息的扩散逻辑（配合MessageSvc）。
        *   万人群聊优化：
            *   消息扩散：避免遍历所有成员推送，而是利用MQ的发布订阅特性，或者在消息服务中做高效扇出。
            *   在线状态：对于超大群，不一定实时同步所有成员的在线状态给每个客户端。
            *   读扩散可能更适合超大群消息的存储和拉取。
    *   **信令服务 (SignalingSvc):**
        *   处理好友申请、同意/拒绝、入群申请等非消息类指令。
        *   通常也通过MQ流转，更新相关数据（如用户关系、群成员）。
    *   **离线推送服务 (OfflinePushSvc):**
        *   当MessageSvc判断用户离线且需要推送时，调用此服务。
        *   此服务集成APNS (Apple), FCM (Google), 各大手机厂商推送通道。
        *   管理用户的Device Token。

7.  **数据存储层**
    *   **关系型数据库 (MySQL Cluster, PostgreSQL + Citus):**
        *   存储结构化数据：用户信息、好友关系、群信息、群成员关系。
        *   采用分库分表策略应对高并发和大数据量 (如按UID HASH分库)。
    *   **NoSQL数据库 (HBase, Cassandra):**
        *   **消息内容存储：** 针对亿级甚至百亿级消息，使用HBase等列式存储，RowKey精心设计（如 `user_id_conversation_id_seq_id` 或 `conversation_id_seq_id`）以支持高效查询和范围扫描。
        *   **会话列表 (Session/Conversation List):** `user_id` 为RowKey，存储该用户的所有会话，每个会话包含 `last_msg_id`, `unread_count`, `timestamp` 等。
    *   **分布式缓存 (Redis Cluster):**
        *   `UID -> GatewayID` 映射：用户连接的Gateway节点信息。
        *   `UID -> OnlineStatus`：用户在线状态。
        *   `ConversationID -> Current_Seq_ID`: 会话的最新序列号生成器。
        *   热点数据缓存：如用户信息、群信息、频繁访问的消息。
        *   分布式锁：用于并发控制某些操作。
    *   **对象存储 (S3, OSS):**
        *   存储图片、语音、短视频等多媒体文件。客户端上传文件到对象存储，IM消息中只包含文件URL。

**关键流程实现思路**

1.  **客户端连接与认证：**
    1.  Client -> IPConf: 获取Gateway IP列表。
    2.  Client -> SLB -> Gateway: 建立TCP长连接。
    3.  Client -> Gateway: 发送Auth Token。
    4.  Gateway -> UserSvc (或本地校验): 验证Token。
    5.  成功后，Gateway 记录 `FD <-> UID` 映射 (存本地并/或更新Redis: `UID -> GatewayID`)，并通知UserSvc或MessageSvc用户上线（发消息到MQ）。
    6.  Client: 发送心跳包。

2.  **单聊消息发送与接收：**
    1.  Client A -> Gateway A: 发送消息 (含`to_uid`, `content`, `client_msg_id`)。
    2.  Gateway A: 校验、封装 -> MQ (`P2P_MESSAGE_TOPIC`，消息体含`from_uid`, `to_uid`, `content`, `client_msg_id`, `timestamp`)。
    3.  MessageSvc: 消费消息。
        a.  生成`server_msg_id`, `seq_id` (针对A和B的会话)。
        b.  持久化消息到DB (如A的Outbox，B的Inbox)。
        c.  查询B的在线状态和Gateway B ID (from Redis)。
        d.  **If B在线:** 将消息 (含`server_msg_id`, `seq_id`) 投递到 MQ (如`PUSH_MESSAGE_TOPIC_GATEWAYB_ID` 或通用推送Topic)。
        e.  **If B离线:** 消息已存DB。可选：MessageSvc -> OfflinePushSvc -> APNS/FCM。
    4.  Gateway B: 消费到推送给B的消息。
    5.  Gateway B -> Client B: 推送消息。
    6.  Client B: 收到消息，显示，发送“已送达ACK” (含`server_msg_id`) -> Gateway B。
    7.  Gateway B -> MQ (`ACK_TOPIC`)。
    8.  MessageSvc: 消费ACK，更新DB中消息状态为“已送达”，并可通知Client A。
    9.  Client B: 用户阅读消息，发送“已读ACK” -> Gateway B -> MQ (`ACK_TOPIC`)。
    10. MessageSvc: 消费ACK，更新DB中消息状态为“已读”，通知Client A。

3.  **消息时序一致性与不重不漏：**
    *   **SeqID:** 服务端为每个会话维护严格递增的`seq_id`。客户端也保存已收到的`max_seq_id`。
    *   **ACK机制：**
        *   C2S ACK: 客户端发送消息后，Gateway会给一个快速ACK，表明Gateway已收到。业务ACK由MessageSvc在持久化后通过MQ返回给客户端。
        *   S2C ACK: 客户端收到消息后，回复ACK给服务端，服务端据此更新消息“已送达”状态。
    *   **去重：**
        *   客户端生成`client_msg_id`，服务端记录此ID。若因重传收到相同`client_msg_id`，可识别为重复。
        *   MQ消费者处理消息时需保证幂等性。
    *   **消息拉取：** 用户上线或进入会话时，携带本地`max_seq_id`，从服务端拉取`seq_id > max_seq_id`的消息。
    *   **MQ可靠性：** MQ配置为持久化、多副本、同步刷盘（可选，影响性能）、生产者确认、消费者手动ACK。

4.  **万人群聊：**
    1.  Client A -> Gateway A: 发送群消息 (含`group_id`, `content`, `client_msg_id`)。
    2.  Gateway A -> MQ (`GROUP_MESSAGE_TOPIC`)。
    3.  MessageSvc/GroupSvc: 消费消息。
        a.  生成`server_msg_id`, `group_seq_id` (针对该群)。
        b.  持久化群消息 (通常一份主体，加成员的索引或状态)。
        c.  GroupSvc: 获取该群所有成员UID列表。
        d.  对于每个在线成员: 查询其GatewayID，将消息投递到对应Gateway的MQ Topic。
        e.  对于离线成员: 标记为离线。
    4.  后续流程类似单聊消息接收。
    *   **优化：** 对于超大群，消息扩散可采用更复杂的扇出策略，或客户端主动订阅群消息更新。读扩散模型（消息只存一份，客户端拉取）可能更优。

5.  **离线消息：**
    1.  用户上线，Client向Gateway发送同步请求，携带各会话的`max_seq_id`。
    2.  Gateway -> MessageSvc (via MQ or direct RPC if needed for sync scenario, though async is preferred)。
    3.  MessageSvc: 根据`max_seq_id`从DB查询各会话的未读消息。
    4.  MessageSvc -> Gateway -> Client: 分批推送离线消息。

6.  **可运维性：**
    *   **物理隔离：** 接入层与业务逻辑层物理分离，独立扩缩容。
    *   **无