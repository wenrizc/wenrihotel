## 1. 测试在工程中的定位

测试的目标不是“写更多用例”，而是用可重复、可度量的方式降低发布风险。工程上更关心的是：关键路径是否被保护、缺陷能否尽早暴露、问题能否被快速定位与复现。

常见质量目标可以拆成三类：

- **正确性**：业务规则与边界条件正确，异常路径可控。
- **稳定性**：在抖动、重试、超时、并发等情况下系统行为可预期。
- **可演进性**：重构与升级不会引入回归，测试能给出及时反馈。

### 1.1 测试金字塔与分层思想

业界常用“测试金字塔”组织测试投入：

- **单元测试**：数量最多，执行最快，覆盖核心业务逻辑与边界。
- **集成测试**：数量适中，验证模块间协作（DB、MQ、Cache、HTTP 客户端等）。
- **端到端测试（E2E）**：数量最少，验证关键业务流程，成本最高。

## 2. 业界常用测试概念与术语

### 2.1 单元测试、集成测试、E2E 的边界

区分边界的核心是“依赖是否真实”：

- **单元测试（Unit Test）**：只测一个单元（类、函数、领域服务），外部依赖用 `mock`、`stub`、`fake` 隔离，要求毫秒级。
- **集成测试（Integration Test）**：至少包含一个真实依赖（真实 DB、真实 Redis、真实消息中间件，或真实 `ApplicationContext`），验证集成契约与配置。
- **E2E**：从入口到出口贯通，包含更多真实组件与网络边界，验证用户级关键流程。

### 2.2 黑盒、白盒、灰盒

- **黑盒测试**：不关心内部实现，强调输入输出与行为契约，适合接口与流程验证。
- **白盒测试**：理解实现细节，强调分支、异常路径、边界条件覆盖，适合复杂逻辑。
- **灰盒测试**：知道一些内部结构，但验证点仍以外部行为为主，适合工程化集成测试。

### 2.3 冒烟、回归、验收

- **冒烟测试（Smoke）**：最小集合，用于快速判断系统是否“基本可用”，通常用于部署后。
- **回归测试（Regression）**：防止历史问题回归，强调稳定、可重复、覆盖关键路径。
- **验收测试（UAT）**：面向业务验收标准，通常由业务或 QA 主导，强调场景覆盖。

### 2.4 非功能测试：性能、稳定性、安全

- **性能测试**：吞吐、延迟、资源占用，常见工具如 `JMeter`、`Gatling`。
- **稳定性测试**：长时间运行、故障注入、抖动模拟，强调可恢复性与降级策略。
- **安全测试**：鉴权绕过、注入、反序列化风险、依赖漏洞扫描等。

### 2.5 覆盖率与“有效性”

覆盖率（行覆盖、分支覆盖）是“测试规模”的指标，不是“测试质量”的指标。工程上常见误区是把覆盖率当作唯一门禁，导致大量无断言、弱断言、只跑通的用例。

更接近“有效性”的方法包括：

- **基于风险的测试设计**：围绕高价值路径、历史缺陷区、复杂状态机做用例。
- **变异测试（Mutation Testing）**：故意把代码做小改动，观察测试是否能失败，衡量测试的“杀伤力”。

## 3. Java 项目测试分层的落地原则

### 3.1 以依赖边界为分层标准

面试和工程实践都可以用一条规则统一口径：只要用例依赖真实 I/O（网络、磁盘、真实中间件、真实容器），它就不是纯单元测试。

一张常用对照表：

| 层级 | 目标 | 真实依赖 | 典型框架/工具 |
| --- | --- | --- | --- |
| 单元测试 | 逻辑正确、边界覆盖 | 无 | JUnit 5、Mockito、AssertJ |
| 组件测试 | 单个组件可运行 | 可选（少量） | Spring Test（轻量）、WireMock |
| 集成测试 | 真实依赖契约正确 | 有 | Spring Boot Test、Testcontainers |
| E2E | 关键链路正确 | 多 | RestAssured、Playwright（前端） |

### 3.2 可测性设计：让“难测”变“可测”

可测性（Testability）通常来自结构设计，而不是来自更强的测试框架。常用手段：

- 依赖注入：外部依赖通过构造器注入，便于替换为 `mock` 或 `fake`。
- 时间与随机数外置：用 `Clock` 或封装的 `TimeProvider`，避免直接调用 `System.currentTimeMillis()`。
- 纯逻辑与 I/O 分离：把规则计算下沉到无副作用的领域服务，I/O 只做编排。

### 3.3 测试数据管理：可读性与稳定性优先

业界常见做法：

- 单元测试：使用 Builder 或 Fixture 生成对象，避免手写一堆字段。
- 集成测试：优先用 Testcontainers 启真实依赖，再用迁移脚本（Flyway、Liquibase）初始化结构。
- 避免依赖测试执行顺序，避免共用可变的静态数据。

### 3.4 用例结构：AAA 与 Given-When-Then

两种常用结构都可以，关键是让读者快速看懂断言意图：

- AAA：Arrange（准备）- Act（执行）- Assert（断言）。
- Given-When-Then：强调场景语义，适合业务规则与状态机。

## 4. Java 单元测试框架：JUnit 5 与 TestNG

### 4.1 JUnit 5 的组成与执行模型

JUnit 5 由三部分组成：

- `JUnit Platform`：测试发现与执行的基础设施，IDE 与构建工具对接它。
- `JUnit Jupiter`：JUnit 5 的编程模型（`@Test`、扩展机制等）。
- `JUnit Vintage`：兼容运行 JUnit 4 用例（过渡期用）。

这套分层的意义是：执行与编程模型解耦，扩展能力（Extension）更强，便于集成 Mockito、Spring、Testcontainers 等生态。

### 4.2 JUnit 5 常用能力

```java
import org.junit.jupiter.api.*;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;

class CalculatorTest {
  @BeforeEach
  void setUp() {}

  @ParameterizedTest
  @ValueSource(ints = {0, 1, 2})
  void abs_should_return_non_negative(int x) {
    int y = Math.abs(x);
    Assertions.assertTrue(y >= 0);
  }
}
```

常见用法要点：

- 参数化测试：减少重复用例，提升边界覆盖密度。
- Tag 与条件执行：用 `@Tag`、`@EnabledIf` 之类能力区分慢测试与快测试。
- 扩展机制：通过 `@ExtendWith` 注入资源或生命周期钩子。

### 4.3 TestNG 的定位

TestNG 在一些遗留项目或需要更强测试分组、依赖控制的场景仍有使用。新项目通常优先 JUnit 5，生态与 IDE 支持更统一。

### 4.4 断言库：AssertJ 与 Hamcrest

断言库的价值在于可读性与错误信息质量。常见选择：

- `AssertJ`：链式断言表达力强，错误信息更友好。
- `Hamcrest`：历史更久，搭配 JUnit 4 较常见。

## 5. Mock、Stub、Fake：隔离外部依赖的正确方式

### 5.1 概念区分

- **Mock**：验证交互行为（是否调用、调用次数、参数），常用于“编排类”或副作用边界。
- **Stub**：提供固定返回值，让被测代码走到目标分支。
- **Fake**：可运行的简化实现（如内存仓库），比 `mock` 更贴近真实行为。
- **Spy**：对真实对象包一层，部分方法走真实实现，部分方法被替换。

工程上的经验结论：优先 `fake` 或小型内存实现，其次才是重度 `mock`。大量 `mock` 容易把测试绑死在实现细节上，重构成本高。

### 5.2 Mockito 常用写法

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import static org.mockito.Mockito.*;
import static org.assertj.core.api.Assertions.*;

@ExtendWith(MockitoExtension.class)
class UserServiceTest {
  @Mock UserRepository repo;
  @InjectMocks UserService service;

  @Test
  void find_should_throw_when_missing() {
    when(repo.findById(1L)).thenReturn(null);

    assertThatThrownBy(() -> service.find(1L))
        .isInstanceOf(IllegalStateException.class);

    verify(repo).findById(1L);
    verifyNoMoreInteractions(repo);
  }
}
```

- 不要用 `any()` 过度放宽匹配，否则用例对行为变化不敏感。
- 避免对私有方法、内部实现细节做验证，测试更应该验证对外可观察行为。
- `spy` 易引入不稳定性，除非你明确理解被 spy 对象的副作用与线程安全。

## 6. Spring 项目测试：Spring Test 与 Spring Boot Test

### 6.1 `@SpringBootTest` 的定位

`@SpringBootTest` 会启动完整 `ApplicationContext`，接近真实运行环境，适合集成测试与配置契约验证，但成本高、速度慢。使用时要有意识地控制数量，并把单元测试留给业务逻辑层。

### 6.2 Slice Test：按层切片

Spring Boot 提供了一组切片测试注解，用更小的上下文加速测试并减少干扰：

- `@WebMvcTest`：只加载 MVC 相关组件，适合 Controller 层。
- `@DataJpaTest`：只加载 JPA 相关组件，适合 Repository 层。

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@WebMvcTest(controllers = HealthController.class)
class HealthControllerTest {
  @Autowired MockMvc mvc;

  @Test
  void health_should_200() throws Exception {
    mvc.perform(get("/health"))
        .andExpect(status().isOk());
  }
}
```

### 6.3 外部 HTTP 依赖：WireMock 与契约

当服务要调用外部 HTTP API 时，单元测试不应直连真实外部服务。常见做法是 WireMock 模拟服务端响应，验证：

- 客户端请求路径、Header、Body 是否符合预期。
- 超时、重试、降级等失败策略是否正确。

### 6.4 数据库测试：`@Transactional` 与容器化依赖

常见策略：

- 需要真实方言与真实行为时，用 Testcontainers 起 MySQL、PostgreSQL。
- 需要回滚隔离时用 `@Transactional`，但注意它只对同一事务边界内有效，异步与跨线程不受影响。

## 7. 集成测试利器：Testcontainers

Testcontainers 的核心价值是把“真实依赖”变成可在本地与 CI 重复运行的基础设施，避免：

- H2 等内存数据库与真实 MySQL 行为不一致导致的线上问题。
- 共享测试环境的数据污染与并发冲突。

```java
import org.junit.jupiter.api.Test;
import org.testcontainers.containers.MySQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@Testcontainers
class MySqlIT {
  @Container
  static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0");

  @Test
  void should_start_mysql() {
    // mysql.getJdbcUrl() 用于配置 DataSource
  }
}
```

常见落地要点：

- 容器生命周期：用静态容器减少启动开销，但要控制测试隔离与数据清理方式。
- 依赖组合：MySQL、PostgreSQL、Redis、Kafka、RabbitMQ 都有成熟镜像与容器封装。

## 8. 工具链与 CI：让测试“跑得快、跑得稳”

### 8.1 Maven 的单测与集测分离

业界常把单元测试交给 `maven-surefire-plugin`，集成测试交给 `maven-failsafe-plugin`，通过命名约定区分，例如 `*Test` 与 `*IT`。

```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-surefire-plugin</artifactId>
      <version>3.5.2</version>
    </plugin>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-failsafe-plugin</artifactId>
      <version>3.5.2</version>
      <executions>
        <execution>
          <goals>
            <goal>integration-test</goal>
            <goal>verify</goal>
          </goals>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

### 8.2 覆盖率：JaCoCo 的工程化用法

覆盖率更适合作为“回归基线”而非 KPI。常见门禁策略：

- 关键模块要求一定分支覆盖率。
- 新增代码覆盖率不低于某个阈值（避免历史包袱拖累）。

### 8.3 Flaky Test（不稳定测试）治理

不稳定测试通常来自：

- 时间与并发：依赖 `Thread.sleep`、依赖系统时钟、未等待异步完成。
- 环境依赖：依赖共享外部资源、端口冲突、随机数据未固定种子。
- 全局状态污染：静态变量、单例缓存、未清理的数据库数据。
