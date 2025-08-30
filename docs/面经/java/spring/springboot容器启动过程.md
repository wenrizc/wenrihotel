
### 1. 启动入口：`main` 方法

Spring Boot 应用的启动通常从一个标准的 Java `main` 方法开始。这个方法会调用 `SpringApplication.run()` 静态方法。
```java
@SpringBootApplication
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```
`@SpringBootApplication` 是一个组合注解，包含了 `@Configuration`, `@EnableAutoConfiguration` 和 `@ComponentScan`，它们是 Spring Boot 核心功能的基石。

### 2. `SpringApplication.run()` 方法的执行

`SpringApplication.run()` 方法是 Spring Boot 启动的核心，它执行了以下主要任务：

#### 2.1. `SpringApplication` 对象的创建和初始化
*   **推断应用类型**: `SpringApplication` 首先会根据 classpath 中的类（例如是否存在 `Spring Web` 相关的类如 `DispatcherServlet`）来推断应用程序的类型（`WebApplicationType.SERVLET`, `WebApplicationType.REACTIVE` 或 `WebApplicationType.NONE`）。这决定了后续创建哪种类型的 `ApplicationContext` 和嵌入式 Web 服务器。
*   **寻找和加载 `SpringApplicationRunListeners`**: `SpringApplication` 会在 `META-INF/spring.factories` 文件中查找并加载所有的 `SpringApplicationRunListeners`。这些监听器会在 Spring Boot 应用程序的整个生命周期中发布事件。
*   **加载 `ApplicationContextInitializers` 和 `ApplicationListeners`**: 同样通过 `META-INF/spring.factories` 加载预定义的 `ApplicationContextInitializer` 和 `ApplicationListener`。`ApplicationContextInitializer` 允许在 `ApplicationContext` 刷新之前对其进行编程性配置。

#### 2.2. 运行监听器事件（`SpringApplicationRunListeners`）

`SpringApplicationRunListeners` 在应用程序启动过程中发布一系列事件，允许开发者或框架在关键时刻介入：
*   **`starting` 事件**: 在 `run` 方法开始执行，但 `Environment` 和 `ApplicationContext` 尚未创建时发布。
*   **`environmentPrepared` 事件**: `Environment` 准备就绪，但 `ApplicationContext` 尚未创建时发布。此时可以访问和修改配置属性。
*   **`contextPrepared` 事件**: `ApplicationContext` 已创建，但尚未刷新（即 bean 定义已加载但尚未初始化）时发布。
*   **`contextLoaded` 事件**: `ApplicationContext` 已加载所有 bean 定义，但尚未刷新时发布。
*   **`started` 事件**: `ApplicationContext` 已刷新，所有 `CommandLineRunner` 和 `ApplicationRunner` 已执行完毕，表示应用程序已经启动并正在运行。
*   **`running` 事件**: 在 `started` 事件之后，表示应用程序已经完全启动，并且可以接收请求（如果是 Web 应用）。
*   **`failed` 事件**: 在启动过程中出现任何异常时发布。

#### 2.3. 准备 `Environment`

*   `SpringApplication` 会创建并配置 `ConfigurableEnvironment` 对象。
*   **加载配置**: 从各种源（如 `application.properties`/`application.yml`、命令行参数、系统属性、环境变量等）加载配置属性，并将其添加到 `Environment` 中。
*   **激活 `profiles`**: 根据配置激活相应的 Spring Profile。

#### 2.4. 创建 `ApplicationContext`

*   根据推断出的应用类型（Servlet、Reactive 或 None），创建相应的 `ApplicationContext` 实例。
    *   **Servlet 应用**: 通常是 `AnnotationConfigServletWebServerApplicationContext`。
    *   **Reactive 应用**: 通常是 `AnnotationConfigReactiveWebServerApplicationContext`。
    *   **非 Web 应用**: 通常是 `AnnotationConfigApplicationContext`。
*   **应用 `ApplicationContextInitializers`**: 在 `ApplicationContext` 刷新之前，调用所有已注册的 `ApplicationContextInitializer` 来对 `ApplicationContext` 进行进一步的配置。

#### 2.5. 准备 `ApplicationContext`

*   **将 `Environment` 设置到 `ApplicationContext`**: `Environment` 对象被绑定到 `ApplicationContext`。
*   **注册 `SpringApplication` 实例**: 将 `SpringApplication` 实例本身注册为一个 `bean`。
*   **注册 `Source` 类**: 将 `main` 方法所在的启动类（例如 `MyApplication.class`）注册为 `bean` 定义的来源。

#### 2.6. 刷新 `ApplicationContext` (核心阶段)

这是 Spring IoC 容器执行大部分工作的阶段，其过程包括：
*   **加载 Bean 定义**:
    *   **`@ComponentScan`**: 扫描 `main` 方法所在包及其子包，查找 `@Component`, `@Service`, `@Repository`, `@Controller` 等注解的类，并将其注册为 Bean 定义。
    *   **`@EnableAutoConfiguration`**: 这是 Spring Boot 的魔力所在。它会查找 `META-INF/spring.factories` 文件中定义的 `AutoConfiguration` 类。这些类根据 classpath 中的条件（`@ConditionalOnClass`, `@ConditionalOnMissingBean` 等）有选择性地创建 Bean 定义。例如，如果检测到 `Tomcat` 类，就会自动配置 Tomcat 服务器相关的 Bean。
    *   **用户自定义配置**: 加载通过 `@Configuration` 类定义的 Bean。
*   **实例化单例 Bean**: 按照依赖关系，实例化所有单例 Bean，并执行依赖注入（DI）。
*   **执行 `BeanPostProcessors`**: 在 Bean 初始化前后执行各种 `BeanPostProcessor`，例如处理 `@Autowired`, `@Value`, `@Transactional` 等注解。
*   **执行 `Aware` 接口**: 如果 Bean 实现了 `BeanNameAware`, `BeanFactoryAware`, `ApplicationContextAware` 等接口，对应的回调方法会被调用。
*   **调用 `InitializingBean` 和 `@PostConstruct`**: 执行 Bean 的初始化回调方法。

#### 2.7. 启动嵌入式 Web 服务器 (针对 Web 应用)

如果应用程序类型是 `SERVLET` 或 `REACTIVE`，Spring Boot 会启动一个嵌入式的 Web 服务器（如 Tomcat, Jetty, Undertow）。
*   **创建 `WebServerFactory`**: Spring Boot 会根据 classpath 中的依赖自动配置一个 `ServletWebServerFactory` 或 `ReactiveWebServerFactory` 的实现类（如 `TomcatServletWebServerFactory`）。
*   **启动服务器**: `WebServerFactory` 会负责创建并启动实际的 Web 服务器实例，监听配置的端口。
*   **注册 Servlet/Filter**: 将 Spring MVC 的 `DispatcherServlet` 或 Spring WebFlux 的 `HttpHandler` 注册到 Web 服务器中，以便处理传入的 HTTP 请求。

#### 2.8. 调用 `CommandLineRunner` 和 `ApplicationRunner`

在 `ApplicationContext` 刷新完毕，并且 Web 服务器已启动（如果存在）后，Spring Boot 会查找并执行所有实现了 `CommandLineRunner` 或 `ApplicationRunner` 接口的 Bean。这些接口允许您在应用程序启动完成时执行一些特定的业务逻辑，例如加载初始化数据、执行批处理任务等。

#### 2.9. 发布 `started` 和 `running` 事件

*   **`started` 事件**: 表示应用程序的 `ApplicationContext` 已经刷新，并且所有 `CommandLineRunner` 和 `ApplicationRunner` 都已执行完毕。
*   **`running` 事件**: 这是应用程序启动的最后一个事件，表示应用程序已完全启动并准备好接收和处理请求。

至此，Spring Boot 容器启动完成，应用程序进入运行状态。整个过程通过大量的约定优于配置、自动配置和事件监听机制，大大简化了 Java 应用程序的开发和部署。