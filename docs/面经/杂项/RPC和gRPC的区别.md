
**RPC（Remote Procedure Call）** 是一种通用的远程过程调用概念，通过网络调用远程服务像本地函数一样，而 **gRPC** 是基于 RPC 的现代实现，使用 **HTTP/2** 作为传输协议，**Protocol Buffers（Protobuf）** 作为序列化格式。gRPC 相比传统 RPC 提供了更高的性能、更强的功能（如双向流）和跨语言支持。

### 关键区别

1. **协议基础**：
    - **RPC**：传统上基于 TCP 或 HTTP/1.1，无统一标准，协议因实现而异（如 Java RMI 用 TCP）。
    - **gRPC**：基于 HTTP/2，支持多路复用、头部压缩和持久连接。
2. **序列化格式**：
    - **RPC**：通常用 JSON、XML 或自定义格式，序列化效率较低。
    - **gRPC**：用 Protobuf，二进制格式，体积小、解析快。
3. **通信模式**：
    - **RPC**：大多仅支持请求-响应模式。
    - **gRPC**：支持四种模式：
        - 单向请求-响应（Unary）。
        - 服务器流（Server Streaming）。
        - 客户端流（Client Streaming）。
        - 双向流（Bidirectional Streaming）。
4. **性能**：
    - **RPC**：HTTP/1.1 每次请求新建连接，序列化开销大，性能较低。
    - **gRPC**：HTTP/2 多路复用减少连接开销，Protobuf 提升序列化效率，性能更高。
5. **跨语言支持**：
    - **RPC**：依赖具体实现（如 Java RMI 仅限 Java）。
    - **gRPC**：官方支持多种语言（Java、Go、Python 等），通过 Protobuf 生成代码。
6. **生态与工具**：
    - **RPC**：无统一生态，需自行实现负载均衡、重试等。
    - **gRPC**：内置负载均衡、超时、重试、拦截器等，开箱即用。

### 延伸与面试角度

- **为什么 gRPC 更快？**：
    - HTTP/2 多路复用减少 TCP 连接数，Protobuf 比 JSON/XML 小 3-10 倍。
    - 示例：传输 1MB 数据，gRPC 延迟和带宽优于传统 RPC。
- **适用场景**：
    - **RPC**：简单内部系统，语言单一（如 Java RMI）。
    - **gRPC**：微服务、高并发、跨语言场景（如分布式系统）。
- **优缺点**：
    - **RPC**：实现简单，但扩展性差。
    - **gRPC**：性能高、功能强，但学习曲线陡（需掌握 Protobuf）。
- **实际应用**：
    - gRPC 在 Google 内部广泛使用，如 Kubernetes、Istio。
- **面试点**：
    - 问“HTTP/2 如何提升性能”时，提多路复用和头部压缩。
    - 问“序列化差异”时，比较 Protobuf 和 JSON。