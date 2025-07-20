

Spring AI 旨在简化 Java 应用程序中人工智能功能的开发。它的核心目标是提供一个标准的、可移植的 API，用于与各种大型语言模型（LLM）进行交互，让开发者不必关心底层 AI 提供商的具体实现细节，从而可以轻松地在不同模型之间切换。

### 一、 核心 API 与概念

#### 1. 模型 API (Model APIs)

Spring AI 为不同类型的 AI 任务提供了统一的、可移植的 API 抽象。

- **ChatModel (聊天模型)**: 用于构建对话式 AI 应用的核心接口。它接收一个 `Prompt` 对象（包含一系列消息）并返回一个 `ChatResponse`。这是与 LLM 进行问答和对话的基础。
    
- **ImageModel (图像模型)**: 用于文本到图像的生成。它接收包含图像描述的指令，并返回生成图像的 URL 或基础数据。
    
- **EmbeddingModel (嵌入模型)**: 用于将文本转换为数值向量（即“嵌入”）。这些向量对于语义搜索、聚类和推荐系统至关重要，是实现 RAG（检索增强生成）模式的关键。
    
- **Audio Transcription (音频转录)**: 提供将音频文件转换为文本的功能。
    
- **Text to Speech (文本转语音)**: 提供将文本转换为音频的功能。
    

这些 API 都支持同步和异步流式（Streaming）两种调用方式，以适应不同场景的需求。

#### 2. Prompt 与 Message

`Prompt` 是发送给 AI 模型的完整指令，它是构建有效 AI 交互的核心。

- **Prompt**: 一个 `Prompt` 对象封装了要发送给模型的所有信息，主要包括一个或多个 `Message` 对象和可选的 `ChatOptions`（用于模型特定配置）。
    
- **Message**: `Message` 接口代表对话中的单条信息，包含内容、元数据和角色。
    
- **角色 (Roles)**:
    
    - `SystemMessage`: 用于设定 AI 的行为、个性、角色和响应规则。它为整个对话提供高级指令。
        
    - `UserMessage`: 代表最终用户的输入、问题或指令。`UserMessage` 还支持包含图片、音频等多模态内容。
        
    - `AssistantMessage`: 代表 AI 模型的响应。它可以包含文本内容，也可以是请求调用外部工具（Tool Call）。
        
    - `ToolMessage`: 用于响应模型的工具调用请求，将外部工具的执行结果返回给模型。
        
- **PromptTemplate**: 这是一个强大的模板引擎，允许开发者创建带有占位符的动态提示。它可以在运行时将变量（如用户输入、数据库查询结果等）填充到模板中，生成最终的 `Prompt`。
    

#### 3. ChatClient (流畅 API)

`ChatClient` 是一个高级的、流畅的 API，它构建在 `ChatModel` 之上，极大地简化了与聊天模型的交互。

- **流畅的调用链**: 开发者可以使用 `.prompt()`、`.user()`、`.system()`、`.call()`、`.stream()` 等方法以链式调用的方式构建和发送请求。
    
- **自动配置**: Spring Boot 为 `ChatClient` 提供了强大的自动配置功能，可以轻松注入和使用。
    
- **多模型支持**: 可以轻松配置和使用多个来自不同提供商或具有不同配置的 `ChatClient` 实例。
    
- **集成高级功能**: `ChatClient` 是集成工具调用、聊天记忆和 RAG 等高级功能的入口点。
    

### 二、 核心功能与模式

#### 1. 工具调用 (Tool Calling / Function Calling)

这是 Spring AI 的一项核心功能，允许 AI 模型调用应用程序中定义的外部函数或工具，从而极大地扩展了模型的能力。

- **用途**:
    
    - 信息检索：从 API、数据库或搜索引擎获取实时信息（如查询天气、股价）。
        
    - 执行操作：代表用户执行任务（如发送邮件、创建订单、调用服务）。
        
- **工作流程**:
    
    1. 开发者通过 `@Tool` 注解或编程式定义工具（Java 方法或 `Function`）。
        
    2. 在与模型交互时，将这些工具提供给 `ChatClient`。
        
    3. 模型根据用户意图，决定是否需要调用某个工具，并生成一个包含工具名称和参数的请求。
        
    4. Spring AI 框架自动拦截此请求，执行相应的 Java 方法。
        
    5. 框架将方法的执行结果返回给模型。
        
    6. 模型利用这个结果生成最终的、更丰富和准确的回复。
        
- **安全性**: 模型本身永远无法直接访问或执行代码，它只能请求调用。实际的执行由您的应用程序在受控环境中完成。
    

#### 2. 检索增强生成 (Retrieval Augmented Generation, RAG)

RAG 是一种强大的模式，通过从外部知识库中检索相关信息来增强 LLM 的响应，以解决模型知识过时和“幻觉”问题。

- **工作流程**:
    
    1. **数据加载 (ETL)**: 使用 Spring AI 的 ETL 框架将您的私有数据（如 PDF、JSON、Markdown 文件）处理、切分并转换为 `Document` 对象。
        
    2. **嵌入与存储**: 使用 `EmbeddingModel` 将 `Document` 对象的文本内容转换为向量嵌入，并存入向量数据库 (`VectorStore`)。
        
    3. **检索 (Retrieval)**: 当用户提问时，首先将用户的问题也转换为向量，然后在向量数据库中进行相似性搜索，找到最相关的文档片段。
        
    4. **增强 (Augmentation)**: 将检索到的文档内容作为上下文信息，与用户的原始问题一起，通过 `PromptTemplate` 组合成一个新的、更丰富的提示。
        
    5. **生成 (Generation)**: 将这个增强后的提示发送给 LLM，模型会基于提供的上下文生成一个更准确、更具事实性的回答。
        
- **模块化 RAG**: Spring AI 提供了高度模块化的 RAG 架构，允许开发者对查询转换、文档检索、后处理等各个环节进行定制和优化。
    

#### 3. 聊天记忆 (Chat Memory)

LLM 本身是无状态的，聊天记忆功能使其能够“记住”之前的对话内容，从而进行有上下文的多轮对话。

- **ChatMemory 接口**: 定义了存储、检索和管理对话消息的规范。
    
- **ChatMemoryRepository**: 消息的存储和检索接口，Spring AI 提供了多种实现：
    
    - `InMemoryChatMemoryRepository`: 内存存储，适用于简单场景和测试。
        
    - `JdbcChatMemoryRepository`: 使用 JDBC 将历史记录存储在关系型数据库中（如 PostgreSQL, MySQL）。
        
    - `CassandraChatMemoryRepository`, `Neo4jChatMemoryRepository` 等：支持多种 NoSQL 数据库。
        
- **Advisors 集成**: 通过 `MessageChatMemoryAdvisor` 等顾问，可以轻松地为 `ChatClient` 添加聊天记忆功能，它会在每次请求时自动从仓库中提取历史记录并附加到提示中。
    

#### 4. 结构化输出转换器 (Structured Output Converters)

此功能用于强制 LLM 以特定的结构化格式（如 JSON）返回响应，并自动将其转换为 Java 对象。

- **用途**: 当你需要模型返回的数据能被程序直接使用时（例如，填充一个 Java Bean 或返回一个列表），此功能非常有用。
    
- **与工具调用的区别**: 工具调用是模型主动请求执行外部函数，而结构化输出是指导模型响应本身的格式。当模型的原生工具调用功能不支持或不适用时，这是一个很好的替代方案。
    
- **实现**:
    
    - `BeanOutputConverter`: 将模型的 JSON 输出直接映射到指定的 Java Bean 或 Record。
        
    - `ListOutputConverter`: 将模型的输出解析为逗号分隔的字符串列表。
        
    - `MapOutputConverter`: 将模型的输出解析为 `Map<String, Object>`。
        
- **工作原理**: 转换器会自动向提示中添加指令，告诉模型应该以何种格式生成响应。然后，它会解析模型的文本输出并转换为目标 Java 类型。
    

#### 5. 多模态 (Multi-Modality)

Spring AI 支持能够处理多种输入类型（如文本和图像）的多模态模型。

- **实现**: 通过 `UserMessage` 的 `Media` 字段实现。开发者可以在发送用户消息时，除了提供文本 `content`，还可以附加一个或多个 `Media` 对象（如图像）。
    
- **用法**: `new UserMessage("请描述这张图片", new Media(MimeTypeUtils.IMAGE_PNG, imageResource))`。
    
- **支持的模型**: OpenAI (GPT-4o), Anthropic Claude 3, Google Gemini 等主流多模态模型都受支持。
    

### 三、 数据与存储

#### Vector Store API (向量数据库 API)

这是实现 RAG 的基石。Spring AI 提供了统一的 `VectorStore` 接口来与各种向量数据库进行交互。

- **核心功能**:
    
    - `add(List<Document> documents)`: 添加文档（及其向量嵌入）。
        
    - `delete(List<String> ids)`: 删除文档。
        
    - `similaritySearch(SearchRequest request)`: 执行核心的相似性搜索，查找与查询最相关的文档。
        
- **元数据过滤**: 可以在相似性搜索的同时，根据文档的元数据进行过滤，实现更精确的检索。支持类 SQL 的字符串表达式和流畅的 `Filter.Expression` API。
    
- **广泛的支持**: 支持超过 14 种向量数据库，包括 PgVector, Chroma, Milvus, Weaviate, Pinecone, Elasticsearch 等。
    

### 四、 高级定制与扩展

#### Advisors API

Advisors 提供了一种类似 AOP（面向切面编程）的机制，用于拦截和增强 `ChatClient` 的请求和响应。它是实现 RAG、聊天记忆等可重用功能的强大工具。

- **功能**: 可以在模型调用前后执行逻辑，例如修改提示、处理响应、添加上下文等。
    
- **内置 Advisors**:
    
    - `QuestionAnswerAdvisor`: 实现 RAG 的核心顾问，自动执行向量检索并增强提示。
        
    - `MessageChatMemoryAdvisor`: 实现聊天记忆的核心顾问。
        
    - `SimpleLoggerAdvisor`: 用于记录请求和响应，方便调试。
        
- **自定义**: 开发者可以创建自己的 Advisor 来封装特定的、可重用的 AI 交互逻辑。
    

### 五、 评估、测试与运维

#### 1. 模型评估 (Model Evaluation)

此功能旨在通过编程方式评估 AI 模型的响应质量，以检测“幻觉”并确保响应的准确性。

- **RelevancyEvaluator**: 评估模型的响应是否与提供的上下文信息相关。这对于测试 RAG 流程的有效性至关重要。
    
- **FactCheckingEvaluator**: 评估模型的声明是否在逻辑上得到所提供上下文的支持，用于事实核查。
    

#### 2. 可观测性 (Observability)

Spring AI 与 Spring Boot 的可观测性功能深度集成，为 AI 操作提供指标（Metrics）和分布式跟踪（Tracing）。

- **覆盖范围**: `ChatClient`、`ChatModel`、`EmbeddingModel`、`VectorStore` 等核心组件都埋有观测点。
    
- **核心指标**: 记录操作耗时、令牌（Token）使用量等关键性能指标。
    
- **分布式跟踪**: 在复杂的 AI 调用链（如 RAG + Tool Calling）中，可以清晰地看到每个环节的耗时和依赖关系，极大地简化了调试和性能分析。
    
- **日志控制**: 考虑到提示和响应内容可能包含敏感信息，默认情况下不记录。开发者可以通过配置属性显式开启，以便于调试。
    

#### 3. Testcontainers & Docker Compose 集成

Spring AI 提供了与 Testcontainers 和 Docker Compose 的无缝集成，极大地简化了开发和测试环境的搭建。

- **功能**: 当项目中包含相应依赖时，Spring Boot 会自动配置并启动所需的 AI 服务容器（如 Ollama、ChromaDB），并自动将连接信息注入到应用程序中，实现零配置连接。
    

---

总而言之，Spring AI 为 Java 开发者提供了一个全面、强大且易于使用的工具集，涵盖了从与模型基础交互到构建复杂 RAG 和 Agent 应用的全过程，并通过与 Spring 生态系统的深度集成，提供了企业级的可观测性、可测试性和可扩展性。