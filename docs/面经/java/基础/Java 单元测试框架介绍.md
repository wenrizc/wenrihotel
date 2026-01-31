## 1. 单元测试与集成测试的边界

单元测试（Unit Test）关注“一个类/一个函数”的行为，特点是**快、稳定、定位准**。集成测试（Integration Test）关注模块间集成（容器、数据库、MQ 等），更贴近真实，但通常更慢、更容易受环境影响。

工程上常见策略是：

- 单元测试覆盖大部分逻辑分支。
- 少量集成测试覆盖关键链路，避免全量依赖导致测试雪崩变慢。

## 2. 主流框架选型

- JUnit 5（Jupiter）：当前事实标准，扩展模型完善，生态成熟。
- TestNG：分组、依赖管理更强，老项目或大型工程中仍常见。
- JUnit 4：存量项目常见，可通过 Vintage 兼容层迁移到 JUnit 5。

## 3. JUnit 5 的组成与常用能力

JUnit 5 可拆成三层理解：

- Platform：测试发现与执行平台（IDE、构建工具对接入口）。
- Jupiter：JUnit 5 的编程模型（`@Test` 等）与扩展模型。
- Vintage：兼容运行 JUnit 4（迁移期使用）。

### 3.1 常用注解

- 生命周期：`@BeforeEach`、`@AfterEach`、`@BeforeAll`、`@AfterAll`
- 用例：`@Test`
- 参数化：`@ParameterizedTest` + `@ValueSource` / `@MethodSource`

### 3.2 最小示例：JUnit 5 + Mockito + AssertJ

```java
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;

import static org.assertj.core.api.Assertions.assertThat;

class OrderServiceTest {
    @Test
    void should_calculate_total_price() {
        PriceClient client = Mockito.mock(PriceClient.class);
        Mockito.when(client.priceOf("sku")).thenReturn(100);

        OrderService service = new OrderService(client);
        int total = service.totalPrice("sku", 2);

        assertThat(total).isEqualTo(200);
    }
}
```

## 4. 断言与 Mock 生态

围绕测试框架通常搭配断言库与 Mock 工具：

- AssertJ / Hamcrest：断言表达更可读。
- Mockito：主流 Mock 框架，支持行为校验、参数捕获等。

避免“为 mock 而 mock”：单元测试更应该验证对外行为与业务规则，而不是绑定实现细节。

## 5. Spring 测试

- `@SpringBootTest`：全量启动 Spring 容器，最贴近真实，但最慢。
- Slice Test：如 `@WebMvcTest`、`@DataJpaTest`，只加载局部组件，速度与稳定性更好。

建议优先单测，必要时再上集成测试，并控制数量。

## 6. 工程实践：让测试“快、稳、准”

- 结构遵循 AAA（Arrange/Act/Assert），可读性更强。
- 避免共享状态与顺序依赖，确保可并行执行。
- 少用 `Thread.sleep()`：用可控时钟、轮询等待或同步工具替代。
- 测试数据要最小化，避免引入大量无关依赖与初始化成本。

## 7. 常见坑

- 测试之间共享静态变量，导致用例顺序依赖与偶现失败。
- 过度 Mock 导致测试只验证实现细节，重构成本极高。
- 把集成测试当单测写，启动容器与外部依赖导致执行慢且不稳定。
