
## 问题分析

这是一个典型的分布式事务一致性问题：
- ✅ 库存扣减成功（数据库操作）
- ❌ RabbitMQ消息发送失败（RPC调用）

如不妥善处理，可能导致：
- 用户付款但订单未处理
- 库存状态与实际订单不一致
- 业务流程中断

## 解决方案

### 1. 本地消息表 + 定时任务（推荐）

```
┌───────────┐    ┌───────────┐    ┌───────────┐
│  库存扣减  │ -> │  本地消息  │ -> │   MQ投递  │
└───────────┘    └───────────┘    └───────────┘
                      ↑               │
                      └───────────────┘
                        定时重试机制
```

实现步骤：
1. 创建本地消息表，与业务操作在**同一个数据库事务**中
2. 在表中记录MQ消息内容和状态（待发送）
3. 定时任务扫描表中状态为"待发送"的消息并重试发送
4. 发送成功后将状态更新为"已发送"

代码示例:

```java
@Transactional
public void processSeckill(Long productId, Long userId) {
    // 1. 扣减库存
    boolean deductSuccess = inventoryService.deduct(productId, 1);
    if (!deductSuccess) {
        throw new RuntimeException("库存不足");
    }
    
    // 2. 创建订单
    Order order = orderService.createOrder(userId, productId);
    
    // 3. 在同一事务中写入本地消息表
    LocalMessage message = new LocalMessage();
    message.setMessageBody(buildOrderMessage(order));
    message.setStatus("PENDING");
    message.setRetryCount(0);
    message.setNextRetryTime(new Date());
    localMessageRepository.save(message);
    
    // 4. 事务提交后，消息会被保存，即使MQ发送失败也可以通过重试机制保证
}
```

### 2. 可靠消息最终一致性方案

采用可靠消息服务，确保消息必达：
1. 先发送"预消息"（半消息）到消息服务
2. 执行本地事务（库存扣减）
3. 根据事务结果确认或取消消息
4. 消息服务对未确认的消息进行回查

这种方案可使用RocketMQ的事务消息功能实现。

### 3. TCC补偿模式

将秒杀拆分为多个接口：
- **Try**：资源检查和预留（预扣库存）
- **Confirm**：完成实际操作（确认订单，发MQ）
- **Cancel**：出现异常时回滚操作（恢复库存）

### 4. 最简解决方案：异步重试

如果已上线系统需要紧急修复：
1. 记录MQ发送失败的订单ID
2. 使用独立的恢复程序，定时重试发送这些消息
3. 建立监控预警，及时发现未处理的订单

## 最佳实践建议

1. **设计时的考虑**：
   - 幂等性设计，确保重复消息不会导致重复处理
   - 消息体包含足够信息，便于后续处理

2. **监控告警**：
   - 监控本地消息表积压情况
   - 对长时间未处理的消息设置特别告警

3. **降级处理**：
   - 当MQ长时间不可用时，可考虑临时切换到备用MQ或直接调用下游服务

通过上述方案，即使MQ调用失败，也能保证业务数据的最终一致性，避免库存与订单状态不匹配的问题。