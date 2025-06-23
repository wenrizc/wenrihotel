## 核心服务模块

### 1. **spring-cloud-gateway-server**
- **作用**：传统的网关核心实现模块（已标记为废弃）
- **功能**：包含最初的网关路由、谓词、过滤器等核心功能
- **地位**：项目的原始核心，现在已被新的WebFlux和WebMVC版本替代

### 2. **spring-cloud-gateway-server-webflux**
- **作用**：基于WebFlux的响应式网关核心实现
- **技术栈**：Spring WebFlux + Netty
- **特点**：异步非阻塞、高并发处理能力
- **适用场景**：高并发、低延迟的网关场景

### 3. **spring-cloud-gateway-server-webmvc**
- **作用**：基于WebMVC的网关核心实现
- **技术栈**：Spring WebMVC + Servlet容器
- **特点**：传统同步处理模式，使用函数式编程风格
- **适用场景**：传统企业级应用，对响应式编程不熟悉的团队

### 4. **spring-cloud-gateway-server-mvc**
- **作用**：WebMVC版网关的具体实现模块
- **功能**：包含MVC版本的路由处理、过滤器、谓词等
- **关系**：与spring-cloud-gateway-server-webmvc配合使用

## 兼容层模块

### 5. **spring-cloud-gateway-webflux**
- **作用**：WebFlux相关的通用组件和工具类
- **功能**：提供WebFlux环境下的代理交换、配置等支持
- **地位**：WebFlux技术栈的基础支撑模块

### 6. **spring-cloud-gateway-mvc**
- **作用**：MVC相关的通用组件和工具类
- **功能**：提供MVC环境下的基础设施支持
- **特点**：包含.jdk8标识，表示对JDK8的特殊支持

## 代理交换模块

### 7. **spring-cloud-gateway-proxyexchange-webflux**
- **作用**：WebFlux版本的代理交换功能
- **功能**：提供简化的HTTP代理转发能力
- **使用场景**：简单的请求转发，不需要复杂路由逻辑时

### 8. **spring-cloud-gateway-proxyexchange-webmvc**
- **作用**：WebMVC版本的代理交换功能
- **功能**：为MVC环境提供代理转发能力
- **特点**：同步处理模式的代理转发

## 启动器模块

### 9. **spring-cloud-starter-gateway**
- **作用**：传统的网关启动器（已废弃）
- **功能**：一键引入网关所需的所有依赖
- **状态**：向后兼容保留，建议使用新版本

### 10. **spring-cloud-starter-gateway-mvc**
- **作用**：MVC版网关的Spring Boot启动器
- **功能**：自动配置MVC版网关环境
- **优势**：开箱即用，自动装配所需组件

### 11. **spring-cloud-starter-gateway-server-webflux**
- **作用**：WebFlux版网关的启动器
- **功能**：配置WebFlux网关运行环境
- **特点**：响应式技术栈的完整配置

### 12. **spring-cloud-starter-gateway-server-webmvc**
- **作用**：WebMVC版网关的启动器
- **功能**：配置WebMVC网关运行环境
- **特点**：传统技术栈的完整配置

## 依赖管理模块

### 13. **spring-cloud-gateway-dependencies**
- **作用**：统一管理所有网关相关依赖的版本
- **功能**：版本控制、依赖传递管理
- **重要性**：确保所有模块使用一致的依赖版本

## 测试与集成模块

### 14. **spring-cloud-gateway-integration-tests**
- **作用**：集成测试模块的父容器
- **内容**：包含多个子测试模块
- **功能**：端到端测试、兼容性测试、性能测试等

### 15. **spring-cloud-gateway-sample**
- **作用**：完整的使用示例项目
- **内容**：演示各种配置和使用方式
- **价值**：学习参考、快速上手的最佳实践
- **技术栈**：支持Java和Kotlin双语言示例

## 文档模块

### 16. **docs**
- **作用**：项目文档生成和管理
- **技术**：基于Antora的文档系统
- **内容**：使用指南、API文档、配置参考等
- **输出**：生成HTML格式的完整文档站点

## 架构设计理念

这种模块划分体现了几个重要的设计原则：

1. **技术栈隔离**：WebFlux和WebMVC完全分离，避免冲突
2. **渐进式升级**：保留废弃模块，支持平滑迁移
3. **功能分层**：核心功能、启动器、工具类分层设计
4. **可选依赖**：用户可以根据需要选择特定的技术栈
5. **完整生态**：从核心实现到示例文档一应俱全

这种架构使得Spring Cloud Gateway能够同时支持响应式和传统的开发模式，满足不同团队和项目的需求。