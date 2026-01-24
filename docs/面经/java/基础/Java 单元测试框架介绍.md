# Java 单元测试框架介绍

## 1. 主流框架
主流选择主要集中在 JUnit 与 TestNG 生态。

- **JUnit 5（Jupiter）**：事实标准，支持参数化测试、动态测试与扩展模型。
- **TestNG**：更强的测试分组与依赖管理，常用于大型工程或兼容旧项目。
- **JUnit 4**：老项目常见，可通过 `vintage` 兼容层迁移到 JUnit 5。

## 2. 断言与 Mock 生态
围绕主框架通常搭配断言库与 Mock 工具。

- **AssertJ / Hamcrest**：提供更易读的断言表达。
- **Mockito**：主流 Mock 框架，支持行为校验与参数捕获。
- **Spring Test**：与 Spring Boot 集成测试搭配使用，支持 `@SpringBootTest`。

## 3. 常用注解与用法
JUnit 5 常用 `@Test`、`@BeforeEach`、`@AfterEach`、`@ParameterizedTest`。TestNG 常用 `@Test`、`@BeforeMethod`、`@DataProvider`。单元测试一般遵循 AAA 模式（Arrange、Act、Assert），把准备、执行、断言分开，便于阅读与维护。

## 4. 选型建议
新项目优先选择 JUnit 5 + Mockito + AssertJ，覆盖单元测试与 Mock 需求。需要 Spring 上下文或数据库的场景再引入 `@SpringBootTest`，尽量减少全量容器启动以缩短测试时间。

## 5. 常见坑
以下问题会让单元测试变慢或变脆弱。

- 测试之间共享状态，导致用例顺序依赖与偶现失败。
- 过度 Mock 让测试只验证实现细节，缺少行为验证。
- 把集成测试当成单元测试，导致执行慢且不稳定。
