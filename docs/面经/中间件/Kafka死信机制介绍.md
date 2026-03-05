## 1. Kafka 的“死信”是什么，为什么需要它

Kafka 没有像 RabbitMQ 那样的 **死信交换机（DLX）** 与 Broker 侧自动路由。Kafka 的消费模型是 pull + offset，因此工程上所谓 Kafka 的“死信机制”，通常指：

- **DLQ / DLT（Dead Letter Queue / Dead Letter Topic）**：用一个独立的 `Topic` 存放“处理失败且不应继续阻塞主链路”的消息。

死信机制解决的不是“让消息永远不丢”，而是把系统做成可治理的闭环：

- **把失败隔离**：毒丸消息不阻塞分区推进。
- **把失败可观测**：告警、指标、可定位原因。
- **把失败可修复**：人工或离线修复后可回放。

## 2. 什么时候应该进 DLT：失败分类

把失败分成两类更符合工程事实：

### 2.1 不可恢复错误（直接进 DLT）

典型例子：

- 反序列化失败或 schema 不兼容，无法解析出业务对象。
- 业务校验失败（必填字段为空、状态机非法迁移），重试不会变好。
- 数据库唯一约束冲突且业务上不允许覆盖，属于数据问题而非瞬时问题。

### 2.2 可恢复错误（重试后仍失败才进 DLT）

典型例子：

- 网络抖动、下游瞬时超时、连接池短暂耗尽。
- 依赖服务短时间不可用。

工程上常用做法是“有限次快速重试 + 延迟重试 + 最终 DLT”。

## 3. DLT 设计的关键决策：offset 怎么办

Kafka 的“确认”语义体现在 offset 上，因此你必须先回答：

- 处理失败时，**这条消息对应的 offset 要不要提交**？

常见两种策略：

### 3.1 提交 offset + 投递到 DLT（主链路不阻塞）

适用：

- 你明确选择“主链路继续推进”，失败消息走旁路治理。

代价：

- 如果 DLT 投递失败且你已提交 offset，就会出现“主链路丢失该消息”的风险。

落地要求：

- DLT 投递必须有可靠保障（例如重试、告警、落库兜底），否则这条路不成立。

### 3.2 不提交 offset（让消息重读）

适用：

- 你更强调“主链路不允许跳过”，宁可阻塞。

代价：

- 毒丸消息会卡住分区，导致该分区后续消息无法推进，最终可能引发堆积与雪崩。

实践上通常会再加一层“超过阈值后转入 DLT”，避免无限阻塞。

## 4. 推荐架构：主 Topic + 重试 Topic + DLT

### 4.1 基础拓扑

建议把三类 Topic 明确隔离：

1. 主 `Topic`：正常业务消费。
2. 重试 `Topic`：承载可恢复错误的延迟重试。
3. `DLT`：承载最终失败消息。

### 4.2 延迟重试怎么做：不要在分区里 sleep

Kafka 没有内建“按消息 TTL 延迟投递”的交换机能力，延迟常见实现方式是“多级重试 Topic”：

- `xxx.retry.5s`
- `xxx.retry.1m`
- `xxx.retry.10m`

实现要点：

- 每个重试 Topic 都由独立的消费者组消费。
- 重试消费者拿到消息后判断是否到期，到期则转发回主 Topic 或下一级重试 Topic。
- 不要在同一分区里“原地 sleep”，否则会阻塞该分区后续消息。

### 4.3 重试次数与可追溯性：用 headers 建模

建议统一写入 headers：

- `x-retry-count`：当前重试次数。
- `x-next-at`：下一次可执行时间（毫秒时间戳）。
- `x-origin-topic`、`x-origin-partition`、`x-origin-offset`：原始位置。
- `x-exception-class`、`x-exception-message`：失败原因摘要（注意脱敏）。
- `x-biz-key`：业务主键（用于幂等与定位）。

## 5. 反序列化失败的特殊性

反序列化失败往往发生在“业务逻辑之前”，这类失败有两个特点：

- 业务代码拿不到完整对象，难以走同一套重试逻辑。
- 同一条坏消息会持续失败，极易形成分区阻塞。

因此更常见的策略是：

- 对反序列化失败直接进入 DLT，并保留原始 `value`（或原始字节）与元数据，便于排查。

## 6. Spring Kafka 的一个最小实现片段（示意）

下面示例表达核心意图：失败后按规则写入 DLT，避免阻塞主分区推进。

```java
import java.util.function.BiFunction;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.context.annotation.Bean;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.listener.DeadLetterPublishingRecoverer;
import org.springframework.kafka.listener.DefaultErrorHandler;
import org.springframework.util.backoff.FixedBackOff;

class KafkaDlqConfig {
    @Bean
    DefaultErrorHandler errorHandler(KafkaTemplate<Object, Object> template) {
        DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(
                template,
                (rec, ex) -> new org.apache.kafka.common.TopicPartition(rec.topic() + ".DLT", rec.partition())
        );

        // 示例：本地快速重试 3 次，每次间隔 1 秒，之后写入 DLT
        DefaultErrorHandler handler = new DefaultErrorHandler(recoverer, new FixedBackOff(1000L, 3L));

        // 不可恢复异常可直接跳过重试（示意：实际应按业务分类）
        handler.addNotRetryableExceptions(IllegalArgumentException.class);
        return handler;
    }
}
```

注意：

- “写入 DLT”属于旁路写入，仍需配套告警与监控，避免 DLT 写入失败成为 silent failure。
- 以上代码以“表达机制”为目的，具体类名与可用能力受 Spring Kafka 版本影响，但底层思路一致。

## 7. DLT 消费、回放与治理闭环

### 7.1 DLT 应该怎么消费

DLT 不是“继续按原逻辑消费一次”，而是“专门的治理通道”。常见做法：

- DLT 消费者只做：落库、告警、生成工单、归档原始消息与异常原因。
- 对确实需要回放的消息，通过“人工确认 + 回放工具”把消息重新投递回主 Topic。

### 7.2 DLT 的保留策略

DLT 往往是排障证据链，建议：

- DLT 的 retention 不要太短，避免问题还没定位消息就过期。
- 体量大的 DLT 可以配合落库归档，Kafka 里只保留一段时间的窗口数据。

