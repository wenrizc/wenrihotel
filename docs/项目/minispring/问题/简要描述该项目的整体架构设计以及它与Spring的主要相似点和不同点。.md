
## 整体架构

主要由以下核心模块组成：

- context：核心 IoC 容器，支持基于注解的依赖注入
- aop：AOP 支持，基于 ByteBuddy 实现注解驱动的代理
- jdbc：提供 JdbcTemplate 和声明式事务管理
- web：支持基于 Jakarta Servlet 6.0 的 Web 应用
- boot：简化 Summer 应用的启动和运行
- parent：管理依赖版本和构建配置

## 与 Spring Framework 的相似点

1. **模块化架构**：采用与 Spring 类似的模块划分
2. **IoC 容器**：基于注解的依赖注入机制
3. **AOP 支持**：提供面向切面编程能力
4. **数据访问**：JdbcTemplate 和事务管理
5. **MVC 架构**：Web 应用支持
6. **快速启动**：类似 Spring Boot 的便捷启动方式

## 与 Spring Framework 的不同点

1. **规模更小**：是一个极简实现，代码量远小于 Spring
2. **功能精简**：专注于核心功能，不包含 Spring 的众多扩展模块
3. **实现方式**：在某些实现细节上可能采用了不同的技术选型（如使用 ByteBuddy 实现 AOP）

