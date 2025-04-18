
基于提供的代码，我可以看到该系统使用了 Canal 来实现 MySQL 与 Elasticsearch 之间的数据同步。这是一个典型的 CDC（变更数据捕获）实现方案，下面详细解析：

## Canal 数据同步核心机制

Canal 工作原理是模拟 MySQL 的从库复制机制，能够解析 MySQL 的 binlog 日志并转换为易处理的数据格式：

1. **基础配置**：
   ```yaml
   canal:
     host: localhost
     port: 11111
     destination: hmdp
     username: 
     password:
   ```

2. **启动和订阅**：`CanalClient.java` 中实现了与 Canal 服务的连接和数据订阅：
   ```java
   connector.connect();
   // 订阅特定表的变更
   connector.subscribe(".*\\.tb_shop,.*\\.tb_blog,.*\\.tb_voucher,hmdp.tb_product");
   ```

## 数据同步流程

从代码中可以看到同步流程非常清晰：

1. **变更事件捕获**：
   - Canal 监听 MySQL binlog
   - 当数据发生变更时触发相应处理

2. **商品数据差异处理**：
   ```java
   // 处理商品表数据同步到ES
   if ("tb_product".equals(tableName)) {
       handleProductChange(rowChange);
   } else {
       // 其他表的变更发送到MQ
       publishRowData(entry);
   }
   ```

3. **ES同步实现**：`ProductSearchServiceImpl.java` 中的核心方法：
   ```java
   public void syncProductToES(Long productId) {
       // 从缓存或数据库获取商品
       Product product = unifiedCache.queryWithHeatAware(
               "product", RedisConstants.CACHE_SHOP_KEY, productId,
               Product.class, id -> productService.getById(id), false);

       if (product != null) {
           // 将Product转换为ProductDocument
           ProductDocument document = ProductDocument.fromProduct(product);
           // 保存到ES
           productRepository.save(document);
           log.info("商品 [{}] 已同步到ES", productId);
       }
   }
   ```

4. **增删改处理差异**：
   - 对于INSERT/UPDATE：同步到ES中
   - 对于DELETE：从ES中删除对应数据

## 异步解耦设计

系统使用了消息队列进行解耦，提高系统可用性：

1. **RabbitMQ配置**：
   ```java
   public static final String DB_CHANGE_EXCHANGE = "db.change.exchange";
   public static final String DB_CHANGE_QUEUE = "db.change.queue";
   public static final String DB_CHANGE_ROUTING_KEY = "db.change";
   ```

2. **变更消息发布**：
   ```java
   // 发送到RabbitMQ
   rabbitTemplate.convertAndSend(EXCHANGE_NAME, ROUTING_KEY, message);
   ```

3. **消息消费**：
   ```java
   @RabbitListener(queues = "db.change.queue")
   public void handleDatabaseChange(Map<String, Object> message) {
       // 处理数据库变更消息
       BusinessType businessType = getBusinessTypeByTable(table);
       // ...后续处理
   }
   ```

## 缓存一致性保障

为保证ES与主库及缓存的一致性，系统实现了：

1. **缓存与ES协同更新**：变更发生时，既更新Redis缓存也同步到ES
2. **布隆过滤器优化**：使用布隆过滤器减少缓存穿透问题
3. **多级缓存策略**：结合本地缓存(Caffeine)和Redis的多级缓存架构

这种基于Canal的同步方案具有低侵入性、实时性好、可靠性高的特点，适合于保持MySQL和Elasticsearch数据的一致性。