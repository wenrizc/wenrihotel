
### 1. 启动入口：`SpringApplication.run()`

一切始于应用程序主类中的 `main` 方法，其中调用了 `SpringApplication.run()`。 这个静态方法是整个 Spring Boot 应用程序的入口点，它负责引导整个启动过程。 在这个阶段，Spring Boot 会：

*   **创建一个 `SpringApplication` 实例**: 这个实例用于配置和启动应用程序。
*   **推断应用程序类型**: Spring Boot 会根据类路径下的依赖来判断应用程序的类型，例如，如果存在 `spring-boot-starter-web`，则会创建适用于 Web 应用程序的上下文。
*   **加载 `ApplicationContextInitializer` 和 `ApplicationListener`**: 从 `META-INF/spring.factories` 文件中加载并注册初始器和监听器，用于在启动过程的不同阶段进行回调。

### 2. 创建和准备 `ApplicationContext`

`ApplicationContext` 是 Spring 框架的核心，它是一个控制反转 (IoC) 容器，负责管理应用程序中的所有对象（称为 Bean）。 Spring Boot 在启动过程中会创建一个 `ApplicationContext` 实例。

*   **创建应用上下文**: 根据应用程序的类型（Web、响应式或非 Web），Spring Boot 会创建不同类型的 `ApplicationContext`。
*   **环境准备**: Spring Boot 会准备应用程序的环境，包括加载配置文件（如 `application.properties` 或 `application.yml`）、系统属性和环境变量。

### 3. 自动配置与 Bean 的定义

Spring Boot 的一个核心特性是自动配置。 在这个阶段，Spring Boot 会：

*   **扫描类路径**: 默认情况下，Spring Boot 会扫描主应用程序类所在包及其子包下的组件（用 `@Component`, `@Service`, `@Repository`, `@Controller` 等注解的类）。
*   **触发自动配置**: 基于类路径中的依赖，Spring Boot 会自动配置各种组件。 例如，如果类路径下有 `spring-boot-starter-data-jpa`，它会自动配置数据源和 JPA 相关的 Bean。 这是通过 `@EnableAutoConfiguration` 注解实现的，它会评估各种自动配置类，并应用满足 `@Conditional` 注解条件的配置。

### 4. Bean 的生命周期：实例化、依赖注入和初始化

在 `ApplicationContext` 刷新之后，Spring IoC 容器开始管理 Bean 的生命周期。这个过程包括：

*   **实例化 Bean**: Spring 容器根据扫描到的类定义创建 Bean 的实例。
*   **依赖注入**: Spring 容器会自动将 Bean 所依赖的其他 Bean 注入进来。
*   **初始化**: 执行 Bean 的初始化方法，例如调用 `@PostConstruct` 注解的方法。

### 5. 启动内嵌的 Web 服务器（针对 Web 应用）

对于 Web 应用程序，Spring Boot 会启动一个内嵌的 Web 服务器，例如 Tomcat、Jetty 或 Undertow。 这个阶段还包括配置和启动 `DispatcherServlet`，它是 Spring MVC 框架的核心，负责处理所有传入的 HTTP 请求。

### 6. 运行 `CommandLineRunner` 和 `ApplicationRunner`

在应用程序上下文完全初始化并且内嵌服务器启动之后，Spring Boot 会执行所有实现了 `CommandLineRunner` 或 `ApplicationRunner` 接口的 Bean。 这为开发者提供了一个在应用程序启动完成前执行自定义逻辑的机会，例如预加载数据或执行一次性任务。

### 7. 应用程序就绪

至此，Spring Boot 应用程序已经完全启动并准备好接收请求。 Spring Boot 会发布一个 `ApplicationReadyEvent` 事件，表示应用程序已成功启动。

整个启动过程中，Spring Boot 会发布一系列的事件，允许开发者通过 `ApplicationListener` 接口在不同的生命周期阶段插入自定义逻辑。 这些事件包括 `ContextRefreshedEvent`、`ContextStartedEvent`、`ContextClosedEvent` 等。

总而言之，Spring Boot 的启动阶段是一个高度自动化和可扩展的过程，它通过自动配置、内嵌服务器和事件驱动的生命周期管理，极大地简化了 Java 应用程序的开发和部署。