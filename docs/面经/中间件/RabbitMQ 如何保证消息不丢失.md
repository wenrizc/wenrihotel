
RabbitMQ 通过以下机制保证消息不丢失：
1. **生产者确认（Publisher Confirm）**：确保消息送达交换机。
2. **消息持久化**：队列和消息存盘，重启不丢。
3. **消费者确认（Consumer ACK）**：确保消息被处理。
4. **高可用架构**：镜像队列或集群防止节点故障。

#### 核心点
- 从生产到消费全链路保障，需配置配合。

---

### 1. 机制详解
#### (1) 生产者确认（Publisher Confirm）
- **原理**：
  - 生产者发送消息后，RabbitMQ 返回确认（ACK），表示消息已到达交换机。
- **实现**：
  - 开启确认模式，异步或同步接收 ACK。
- **优点**：
  - 确认送达，若失败可重试。
- **示例**（Java）：
```java
channel.confirmSelect(); // 开启确认模式
channel.basicPublish("exchange", "routingKey", null, "msg".getBytes());
channel.waitForConfirmsOrDie(); // 同步等待确认
```
- **注意**：
  - 未到交换机（路由失败）不保证，需持久化。

#### (2) 消息持久化
- **原理**：
  - 队列和消息标记为持久化，写入磁盘。
- **步骤**：
  1. 队列持久化：`durable = true`。
  2. 消息持久化：`deliveryMode = 2`（持久）。
- **优点**：
  - 重启后恢复数据。
- **示例**：
```java
channel.queueDeclare("queue", true, false, false, null); // 持久化队列
channel.basicPublish("", "queue", MessageProperties.PERSISTENT_TEXT_PLAIN, "msg".getBytes());
```
- **局限**：
  - 写入磁盘前故障仍可能丢失。
  - 性能下降。

#### (3) 消费者确认（Consumer ACK）
- **原理**：
  - 消费者处理消息后手动确认（ACK），未确认消息可重发。
- **实现**：
  - 关闭自动 ACK，显式确认。
- **优点**：
  - 确保消息被消费。
- **示例**：
```java
channel.basicQos(1); // 预取 1 条
channel.basicConsume("queue", false, (tag, delivery) -> {
    System.out.println(new String(delivery.getBody()));
    channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false); // 手动 ACK
}, tag -> {});
```
- **注意**：
  - 未 ACK 消息占用内存。

#### (4) 高可用架构
- **镜像队列**：
  - 队列数据同步到多节点，故障切换。
  - 配置：
```bash
rabbitmqctl set_policy HA ".*" '{"ha-mode":"all"}'
```
- **集群**：
  - 多节点分担负载，主节点故障从节点接管。
- **优点**：
  - 防止单点故障丢失。
- **局限**：
  - 网络延迟可能丢少量未同步消息。

---

### 2. 全链路保障流程
1. **生产者**：
   - 开启确认，重试失败消息。
2. **交换机到队列**：
   - 持久化队列，确保路由成功。
3. **消费者**：
   - 手动 ACK，异常重入队列。
4. **服务器**：
   - 镜像队列或集群备份。

#### 图示
```
[Producer] --(Confirm)--> [Exchange] --(Durable Queue)--> [Consumer] --(ACK)--> Done
```

---

### 3. 注意事项
- **性能代价**：
  - 持久化、确认降低吞吐量。
- **极端情况**：
  - 磁盘故障、网络分区仍可能丢。
- **优化**：
  - 批量确认（`multiple = true`）。
  - 异步确认提升性能。

---

### 4. 延伸与面试角度
- **与 Redis 对比**：
  - RabbitMQ：持久化 + ACK 更可靠。
  - Redis：AOF 类似但无交换机。
- **实际应用**：
  - 订单：持久化 + ACK 防丢。
  - 日志：镜像队列高可用。
- **局限**：
  - 未持久化的临时队列易丢。
- **面试点**：
  - 问“机制”时，提确认和持久化。
  - 问“代码”时，提示例。

---

### 总结
RabbitMQ 通过生产者确认、消息持久化、消费者确认和高可用架构保证消息不丢失。全链路配置确保可靠性，但需权衡性能。面试时，可提代码或画流程，展示理解深度。