
## 1. 项目简述
Apache ShenYu 是一个高性能、开源的API Gateway，其核心优势之一在于其灵活的插件系统。通过开发新的插件，可以方便地扩展Gateway的功能。 Apache ShenYu AI Plugin，是 Apache Shenyu 对 AI 模型接口进行代理的插件，此插件使得 ShenYu 能够轻松地与各种公有云AI服务（如OpenAI, DeepSeek等）或私有化部署的AI模型集成。 目前插件的内容比较简单直接，没有集成主流的 Spring AI 框架的能力，在与AI交互调用的过程中还可以再做增强，可以补充如下内容：
1. 支持配置AI调用降级
2. 支持记录会话历史
3. 支持配置代理apiKey，并支持设置apiKey调用限额
4. 支持AI模型调用缓存
## 2. 项目需求分析

为了应对日益复杂的AI应用场景和用户对AI服务管理的精细化需求，需要实现以下需求：

*   **2.1. 需求1：AI调用降级与重试**
    *   **现状**: 当前AI调用失败后处理机制单一，缺乏智能降级。
    *   **需求**: 系统应能根据用户配置的模型优先级列表选择模型，当高级别模型调用失败（如超时、速率限制、服务不可用）时，并根据预设策略执行重试或降级模型。

*   **2.2. 需求2：支持会话历史管理**
    *   **现状**: 缺乏对多轮对话上下文的有效管理。
    *   **需求**: 系统应能为每个AI会话（通过唯一对话ID识别）记录完整的交互历史（用户提问、AI回答）。支持将历史存储在内存或持久化存储，以便在后续调用时构建上下文。

*   **2.3. 需求3：代理API Key与调用限额**
    *   **现状**: 直接暴露真实API Key存在安全风险，且难以对用户进行用量控制。
    *   **需求**: 系统应支持在真实API Key基础上生成代理API Key，管理员可在ShenYu网关配置中为代理Key设置调用配额。

*   **2.4. 需求4：优化AI调用成本与响应速度 
    *   **现状**: 相同的或语义相似的请求会重复调用AI模型，造成Token浪费和延迟增加。
    *   **需求**: 系统应能拦截对AI模型的调用，为请求内容生成语义向量。通过在向量数据库中搜索与当前请求语义最相近的已缓存请求向量，若找到相似度满足预设阈值的缓存条目，则直接返回其对应的缓存响应。

## 3. 技术方案

本项目的核心在于利用 Spring AI 框架的强大功能，对 Apache ShenYu 的 AI 插件进行全面的智能化升级。

### 3.1. 集成 Spring AI

旨在将 Spring AI 框架无缝集成到 ShenYu AI 插件中，充分利用 Spring AI 的模型客户端管理与交互能力，为 ShenYu 网关提供高效、灵活的 AI 支持，是其他升级项目的核心基础。
AiService作为核心服务组件，负责封装对 Spring AI 中ChatClient和EmbeddingClient等 Bean 的调用，并对其进行管理，确保按需获取正确的实例。同时，通过读取网关中相关 AI 模型的配置，AiService负责初始化和管理全局 AI 配置（如 API Key、Endpoint 及默认模型参数）。提供了统一的标准化调用接口（如 chat() 和 embed()方法），有效屏蔽了与具体 AI 服务商客户端交互的复杂细节。

```java
/**
 * AI 服务核心接口，负责封装 Spring AI 框架的调用与管理，提供统一的 AI 服务调用接口。
 */
public interface AiService {

    /**
     * 聊天对话服务调用方法。
     * AiRequest 需包含足够信息（如目标模型名称或服务商标识），以选择正确的 ChatClient。
     */
    ChatResponse chat(AiRequest request);

    /**
     * 向量嵌入服务调用方法。
     * 将文本转换为向量表示，适用于语义搜索、相似度计算等场景。
     */
    EmbeddingResponse embed(EmbeddingRequest request);

    /**
     * 初始化 AiService 配置。
     * AiConfig 包含 AiService 行为逻辑的配置（如默认模型、降级策略等）。
     */
    void initializeConfig(AiConfig config);

    /**
     * 获取ChatClient实例
     */
    ChatClient getChatClient(String providerOrBeanName);

    /**
     * 获取EmbeddingModel实例
     */
    EmbeddingModel getEmbeddingModel(String providerOrBeanName);

}
```

同时需要在 ShenYu AI 插件中引入 Spring AI 相关的 starter 依赖，从而利用 Spring AI 提供的各种 AI 模型客户端（ChatClient和EmbeddingMode）的自动配置能力。AiService的实现类在初始化时，读取 ShenYu 网关中配置的 AI 模型信息，并将其封装成内部的配置对象（AiConfig），从而为不同的 AI 模型提供商创建并注册bean。

### 3.2. AI 调用降级与重试机制

利用 Spring AI 的RetryTemplate构建统一的错误处理和降级框架，适配多种 AI 提供商。允许在网关中配置模型优先级和处理策略。通过Spring AI的机制处理可重试的瞬态错误，在重试耗尽后或遇到不可重试的非瞬态错误时，系统将启动降级策略，尝试使用备选AI模型。

*   **Spring AI RetryTemplate**:
    *   每个AI模型客户端（如OpenAiChatModel）在初始化时都会注入一个RetryTemplate实例。 RetryTemplate将按照Spring AI的默认配置或用户自定义配置进行设置，包括最大重试次数、退避策略（如指数退避）、以及重试监听器。当捕获到TransientAiException类型的异常时，RetryTemplate才会激活并执行重试逻辑。

*   **Spring AI ResponseErrorHandler**:
    *  作为HTTP客户端（如RestTemplate）的一部分，它在AI模型客户端内部调用远程API后，首先检查HTTP响应。错误分类: 根据HTTP状态码，将错误进行分类。例如，5xx错误通常被包装为TransientAiException，而4xx客户端错误通常被包装为NonTransientAiException。

以上是spring ai中已经实现的重试策略，在本次插件升级中，会根据网关配置维护一个按优先级排序的AI模型列表。当需要降级时，此组件负责按顺序选择下一个可用的AI模型。
*   **流程**:
    1.  接收到主模型调用失败的信号（因 RetryTemplate耗尽重试次数后抛出TransientAiException，或直接因NonTransientAiException）。
    2.  调用AI Model Provider获取下一个优先级的备用模型。
    3.  如果所有备用模型均尝试失败，或没有备用模型可用：记录最终错误，并返回预定义的错误提示或执行最终的兜底逻辑。


### 3.3. 会话历史管理

此功能旨在为 ShenYu AI 插件引入有状态的 AI 交互能力，通过高效、可扩展的会话历史管理实现多轮对话的上下文记忆，并为语义缓存提供核心支持。

#### 统一存储接口 ChatMemoryStore
提供标准化方法管理会话历史，屏蔽底层存储差异。
```java
/**
 * 聊天记忆统一存储接口
 * 定义管理对话历史的标准化方法。
 */
public interface ChatMemoryStore {

    /**
     * 将一条或多条消息添加到指定对话。
     *
     * @param conversationId 对话唯一标识符。
     * @param messages       要添加的消息列表。
     */
    void add(String conversationId, List<Message> messages);

    /**
     * 获取指定对话中最近 N 条消息。
     *
     * @param conversationId 对话唯一标识符。
     * @param lastN          要获取的最近消息数量。
     * @return 最近 N 条消息列表，若对话不存在或无消息，返回空列表。
     */
    List<Message> get(String conversationId, int lastN);

    /**
     * 获取指定对话中所有消息。
     *
     * @param conversationId 对话唯一标识符。
     * @return 对话中所有消息列表，若对话不存在或无消息，返回空列表。
     */
    List<Message> get(String conversationId);

    /**
     * 清除指定对话的所有历史消息。
     *
     * @param conversationId 要清除的对话唯一标识符。
     */
    void clear(String conversationId);

    /**
     * 从指定对话中检索与查询文本语义相似的消息。
     *
     * @param conversationId      对话唯一标识符。
     * @param queryText           用于查找相似消息的查询文本。
     * @param topK                返回的相似消息最大数量。
     * @param similarityThreshold 消息相关性的相似度阈值（如余弦相似度）。
     * @return 按相似度排序的相似消息列表，若无足够相似消息或操作不受支持，返回空列表。
     * @throws UnsupportedOperationException 若底层存储不支持语义搜索。
     */
    default List<Message> getSimilarMessages(String conversationId, String queryText, int topK, double similarityThreshold) {
        throw new UnsupportedOperationException("此 ChatMemoryStore 实现不支持语义消息检索。");
    }
}
```

#### 存储实现

##### PgVectorChatMemoryStore
- **依赖**：PostgreSQL 数据库 + pgvector扩展。
- **表结构**：
  - **shenyu_ai_chat_messages（消息详情）**：
    - message_id：消息唯一标识。
    - conversation_id：所属会话 ID。
    - content：消息文本内容。
    - timestamp：消息时间戳。
    - metadata：消息元数据（如工具调用 ID、Token 计数）。
    - embedding：消息内容的语义向量，维度依 Embedding 模型确定。
- **写入策略**：消息元数据（除 Embedding 向量外）同步写入 PostgreSQL，确保持久化和数据一致性。消息内容的 Embedding 向量生成与更新采用异步方式，计算结果更新至 shenyu_ai_chat_messages表的embedding字段，完成向量化存储。
- **检索策略**：读取时将请求文本转换为查询向量，在 PostgreSQL 中针对指定conversation_id的消息，使用pgvector扩展查询相似度最高的内容并返回。

##### RedisChatMemoryStore
- **依赖**：Redis。
- **内部结构**：
  - 每个会话（conversationId）对应 Redis 中的一个 List，Key 格式：shenyu:ai:memory:chat:{conversationId}。
  - Message对象序列化后存储。

### 3.4. 代理 API Key 与调用限额

通过在 HTTP 请求中携带特定 Header 传入标志位（Flag），ShenYu AI 插件根据标志位查询网关配置中的限额规则，动态生成相应的代理 API Key 并传递给 服务。同时，代理 API Key 及其元数据存储至 Redis，支持后续的校验、限额管理。

**Header 字段**：X-AI-Proxy-Key

- 内容：标志位（Flag），一个唯一标识用户、调用方或租户的字符串，用于从网关配置中查找对应的限额规则。
- 标志位需预先在网关配置中注册 

**代理 API Key 生成格式**：proxy_<标志位><随机字符串>

标志位：从 X-AI-Proxy-Key Header 提取，与配置中的标志位保持一致。
随机字符串：采用uuid生成

**Redis Key 格式**：ai:proxy:key:<代理API Key>

**Redis Value 格式**：
```json
{
  "flag": "<标志位>",
  "proxy_api_key": "<代理API Key>",
  "real_api_key": "<真实API Key>",
  "quota_total": <总限额>,
  "quota_remaining": <剩余限额>,
  "expires_at": <过期时间戳>,
  "created_at": <创建时间戳>,
}
```

### 3.5. AI 模型调用语义缓存

AI 模型调用语义缓存通过存储和重用语义相似的 AI 模型响应，减少重复调用，从而显著降低运营成本和响应延迟。

语义缓存系统采用分层设计以提高缓存效率和灵活性：
1. **缓存层 (Cache Provider)**  使用 Redis 等缓存实现快速精确匹配，存储 AI 响应结果。
2. **向量层 (Vector Provider)**  使用向量数据库（如 DashVector）存储查询文本的嵌入向量，进行语义相似度搜索。  
3. **嵌入层 (Embedding Provider)**  通过文本向量化服务将用户查询转化为嵌入向量。  

**处理流程**：
1. **请求拦截与预处理**：提取用户查询文本，进行规范化处理（如去除多余空格、标点符号归一化、转换为小写），确保查询一致性。
2. **语义相似性搜索**：使用 Embedding 模型将规范化文本转换为高维向量，先在 Redis 缓存中查找精确缓存，若未找到，则在向量数据库中执行 ANN 搜索，检索相似度高的候选向量。同时使用余弦相似度计算文本与候选向量的相似度。
3. **缓存命中处理**：若找到满足阈值的相似结果，直接返回缓存结果，避免实际 AI 模型调用。
4. **缓存未命中处理**：若未命中缓存，则调用实际 AI 模型，获取响应后返回同时存入redis缓存。

Redis 缓存项：
```json
Key: "<hash_value>" // 查询哈希值，对规划化后的文本进行哈希
Value: {
  "query": "<normalized_query>", // 规范化查询文本
  "vector": [<float>, <float>, ...], // 查询文本的嵌入向量
  "response": <response_object>, // AI 模型响应结果
  "metadata": {
    "created_at": "<timestamp>", // 缓存创建时间
    "access_count": <integer>, // 缓存访问次数
    "threshold": <float> // 语义相似度阈值
  }
}
```

### 3.6 一次AI调用的完整生命周期

1. **请求接入与路由**: 用户请求经ShenYu网关路由至AI插件。
2. **代理Key与限额**: 插件校验请求头 X-AI-Proxy-Key 标志位，查询网关配置生成代理Key，并在Redis中校验其限额与有效期；对已包含代理key，判断限额。
3. **语义缓存查询**: 用户查询文本规范化、向量化后，先查Redis，再查向量数据库（语义相似搜索）。若命中且相似度达标，直接返回缓存响应。
4. **会话历史构建**: 若为多轮对话 (有conversationId)，从 ChatMemoryStore (如PgVector/Redis) 获取历史消息，与当前提问构成上下文Prompt。
5. **响应与数据持久化**:AI响应成功后，将用户提问和AI回答存入redis和向量数据库，同时在Redis中扣减代理API Key调用次数，并将处理结果封装后，由ShenYu网关返回给客户端。

## 4. 时间规划

第一阶段 7.1 - 8.20 基本实现项目各项需求，完成各种功能的实现

第二阶段 8.21 - 9.30 对各项功能进行测试，并编写文档，进行修正和提交

## 5. 期望

希望能借助此次开源之夏的机会，参与进入开源项目的开发中，积累经验、学习知识，为社区做一份贡献，同时提升自己的编程能力。