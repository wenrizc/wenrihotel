
## 发布订阅机制实现实时同步

系统实现了发布订阅机制，确保缓存变更能够在集群内实时同步：

1. **发布变更消息**：
   - 当缓存数据更新或删除时，通过发布消息通知其他节点
   - 消息中包含操作类型(update/delete)和缓存键信息

2. **监听缓存变更**：
   - 每个应用节点都会订阅缓存变更消息
   - 收到消息后立即使本地缓存失效

3. **消息去重机制**：
   - 实现了基于Redis的消息ID记录机制防止重复处理
   - 通过UUID确保消息唯一性

## 写操作策略 - 先更新数据库，再失效缓存

系统采用了经典的"Cache Aside"模式:

1. **先更新数据库**：
   - 确保数据持久化
   - 在数据库事务中实现
   
2. **再删除缓存**：
   - 删除Redis缓存而非更新，避免数据不一致
   - 收到消息后本地缓存立即失效
   
3. **布隆过滤器标记**：
   - 记录删除的键到布隆过滤器删除集合
   - 达到阈值时触发重建

## 延时双删策略解决并发问题

针对高并发场景下的"写后读"不一致问题，系统实现了延时双删：

1. **第一次删除**：
   - 更新数据库后立即删除缓存
   
2. **延时执行**：
   - 使用线程池异步执行第二次删除
   - 根据业务配置设置延迟时间(默认200ms)
   
3. **第二次删除**：
   - 再次删除缓存，防止第一次删除后、其他线程读取旧数据写入缓存的情况

```
// 异步双删缓存实现
CACHE_REBUILD_EXECUTOR.schedule(() -> {
    try {
        deleteCache(key);
    } catch (Exception e) {
        log.error("延迟双删缓存失败: {}", e.getMessage());
    }
}, 200, TimeUnit.MILLISECONDS);
```

## 数据库变更监听

系统通过监听数据库变更进一步增强一致性保证：

1. **Canal监听机制**：
   - 监听数据库binlog
   - 捕获数据变更事件
   
2. **RabbitMQ消息分发**：
   - 数据变更事件发送到消息队列
   - 应用服务订阅并处理相关消息
   
3. **幂等处理**：
   - 通过消息ID记录已处理的消息
   - 防止重复处理导致的问题

```
@RabbitListener(queues = "db.change.queue")
public void handleDatabaseChange(Map<String, Object> message) {
    String messageId = generateMessageId(message);
    if (isMessageProcessed(messageId)) {
        return;
    }
    
    // 处理缓存更新或删除
}
```

## 缓存过期策略

系统实现了多层次的过期策略管理一致性风险：

1. **本地缓存短期过期**：
   - 本地缓存TTL设置为60秒
   - 确保数据变更较快反映在本地缓存中
   
2. **Redis随机过期**：
   - 为Redis缓存设置基础过期时间加随机偏移量
   - 避免缓存雪崩同时也限制数据不一致的最大时间窗口

```
// 添加随机偏移量，防止缓存雪崩
long randomOffset = (long) (Math.random() * 0.2 * seconds);
stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(value), 
    seconds + randomOffset, TimeUnit.SECONDS);
```

1. **热点数据逻辑过期**：
   - 热点数据使用逻辑过期机制
   - 后台线程定期检查并更新
