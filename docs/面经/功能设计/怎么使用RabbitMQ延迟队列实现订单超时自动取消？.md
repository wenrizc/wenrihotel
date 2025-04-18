
#### 使用 RabbitMQ 延迟队列实现订单超时自动取消
- **目标**：
  - 在订单创建后，若超时（如 30 分钟）未支付，自动取消订单。
- **核心思路**：
  - 使用 RabbitMQ 的**延迟队列**（通过死信交换机或插件）将订单消息延迟投递，超时后触发取消逻辑。
- **实现方式**：
  - 利用 **死信交换机（DLX）** 和 **消息 TTL（Time To Live）** 创建延迟队列。
  - 或使用 **RabbitMQ 延迟消息插件**（更直接）。

#### 核心点
- 延迟队列通过 TTL 和死信机制实现，消费超时消息执行取消。

---

### 1. 实现步骤（基于死信交换机）
#### (1) 定义队列和交换机
- **延迟队列**：
  - 名为 `order.delay.queue`，设置 TTL（如 30 分钟）。
  - 配置死信交换机（DLX）和死信路由键。
- **目标队列**：
  - 名为 `order.cancel.queue`，绑定死信交换机，处理超时消息。
- **交换机**：
  - 延迟交换机：`order.delay.exchange`（类型 `direct`）。
  - 死信交换机：`order.dlx.exchange`（类型 `direct`）。

#### 配置示例
```java
// Spring AMQP 配置
@Bean
public Queue delayQueue() {
    return QueueBuilder.durable("order.delay.queue")
        .withArgument("x-message-ttl", 30 * 60 * 1000) // TTL 30 分钟
        .withArgument("x-dead-letter-exchange", "order.dlx.exchange")
        .withArgument("x-dead-letter-routing-key", "order.cancel")
        .build();
}

@Bean
public Queue cancelQueue() {
    return new Queue("order.cancel.queue", true);
}

@Bean
public DirectExchange delayExchange() {
    return new DirectExchange("order.delay.exchange");
}

@Bean
public DirectExchange dlxExchange() {
    return new DirectExchange("order.dlx.exchange");
}

@Bean
public Binding delayBinding() {
    return BindingBuilder.bind(delayQueue())
        .to(delayExchange()).with("order.delay");
}

@Bean
public Binding cancelBinding() {
    return BindingBuilder.bind(cancelQueue())
        .to(dlxExchange()).with("order.cancel");
}
```

#### (2) 发送订单消息
- **场景**：
  - 用户下单后，发送订单 ID 到延迟队列，设置超时时间。
- **代码**：
```java
@Autowired
private RabbitTemplate rabbitTemplate;

public void createOrder(String orderId) {
    rabbitTemplate.convertAndSend(
        "order.delay.exchange",
        "order.delay",
        orderId
    );
}
```

#### (3) 消费超时消息
- **逻辑**：
  - 监听目标队列，检查订单状态，若未支付则取消。
- **代码**：
```java
@RabbitListener(queues = "order.cancel.queue")
public void cancelOrder(String orderId) {
    // 查询订单状态
    Order order = orderService.getOrder(orderId);
    if (order != null && "UNPAID".equals(order.getStatus())) {
        orderService.cancelOrder(orderId); // 更新状态为取消
        log.info("Order {} cancelled due to timeout", orderId);
    }
}
```

#### (4) 支付成功处理
- **场景**：
  - 用户支付成功后，移除延迟消息或标记订单。
- **方法**：
  - 直接更新订单状态，消费时跳过已支付订单。
- **代码**：
```java
public void payOrder(String orderId) {
    orderService.updateStatus(orderId, "PAID");
}
```

---

### 2. 基于延迟消息插件实现
- **插件**：
  - RabbitMQ Delayed Message Plugin（需安装）。
- **优势**：
  - 直接设置消息延迟，无需死信队列。
- **配置**：
```java
@Bean
public CustomExchange delayExchange() {
    Map<String, Object> args = new HashMap<>();
    args.put("x-delayed-type", "direct");
    return new CustomExchange("order.delay.exchange", "x-delayed-message", true, false, args);
}
```
- **发送消息**：
```java
public void createOrder(String orderId) {
    rabbitTemplate.convertAndSend(
        "order.delay.exchange",
        "order.cancel",
        orderId,
        message -> {
            message.getMessageProperties().setDelay(30 * 60 * 1000); // 30 分钟
            return message;
        }
    );
}
```
- **消费**：
  - 同死信方式，监听 `order.cancel.queue`。

---

### 3. 关键点
- **TTL 设置**：
  - 死信方式：队列 TTL 或消息 TTL。
  - 插件方式：动态设置 `x-delay`。
- **幂等性**：
  - 消费时检查订单状态，避免重复取消。
- **可靠性**：
  - 启用确认机制（`publisher-confirms`）确保消息送达。
  - 持久化队列和消息（`durable=true`）。
- **扩展性**：
  - 多消费者并行处理高并发订单。

---

### 4. 优缺点
#### 优点
- **解耦**：订单创建和取消异步处理。
- **灵活**：延迟时间可配置。
- **可靠**：RabbitMQ 保障消息不丢。

#### 缺点
- **复杂性**：需维护队列和交换机。
- **延迟精度**：插件更精确，死信略有偏差。
- **资源**：大量订单占用队列空间。

---

### 5. 替代方案
- **定时任务**：
  - 扫描订单表，检查超时订单。
  - 缺点：数据库压力大，实时性差。
- **Redis 过期**：
  - 用 `SETEX` 存储订单，过期触发回调。
  - 缺点：回调可靠性需额外保障。
- **数据库触发器**：
  - 定时触发 SQL 取消。
  - 缺点：扩展性差。

---

### 6. 延伸与面试角度
- **与 Kafka**：
  - Kafka 适合高吞吐，RabbitMQ 适合延迟场景。
- **实际应用**：
  - 电商：订单 30 分钟未支付取消。
  - 票务：预订超时释放座位。
- **优化**：
  - 动态 TTL：不同订单不同超时。
  - 监控：用 `rabbitmqctl` 查队列积压。
- **面试点**：
  - 问“实现”时，提死信和插件。
  - 问“幂等”时，提状态检查。

---

### 总结
使用 RabbitMQ 延迟队列实现订单超时取消，可通过死信交换机（TTL）或延迟消息插件，创建订单时发送消息，超时后消费取消。需确保幂等和可靠性。面试时，可提代码或画队列图，展示理解深度。