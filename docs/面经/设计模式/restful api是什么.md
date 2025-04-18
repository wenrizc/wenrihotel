
#### RESTful API 概述
- **定义**：
  - RESTful API（Representational State Transfer API）是一种基于 **REST** 架构风格的 Web API 设计规范，通过标准化的 HTTP 方法、URI 和状态码操作资源，提供无状态、客户端-服务器通信。
- **核心思想**：
  - 将系统功能抽象为**资源**（如用户、订单），通过统一的接口（URI + HTTP 方法）进行增删改查。
- **特点**：
  - 无状态、可缓存、层次化、统一接口。

#### 核心点
- RESTful API 以资源为中心，使用 HTTP 标准协议，易于扩展和集成。

---

### 1. RESTful API 核心概念
#### (1) REST 架构风格
- **起源**：
  - 由 Roy Fielding 在 2000 年提出，定义了 Web 分布式系统的设计约束。
- **六大约束**：
  1. **客户端-服务器**：
     - 分离关注点，客户端处理 UI，服务器处理逻辑和数据。
  2. **无状态**：
     - 每次请求包含完整信息（如 JWT 令牌），服务器不存会话。
  3. **可缓存**：
     - 响应可缓存（通过 `Cache-Control` 头），提升性能。
  4. **分层系统**：
     - 支持中间层（如代理、网关），客户端无需感知。
  5. **统一接口**：
     - 标准化的资源操作方式（URI、HTTP 方法）。
  6. **按需代码**（可选）：
     - 服务器可发送可执行代码（如 JavaScript），少用。

#### (2) 资源（Resource）
- **定义**：
  - 系统的核心实体（如 `/users`、`orders/123`）。
- **表示**：
  - 资源通过 JSON、XML 等格式传输。
  - 例：`{"id": 1, "name": "Alice"}`。
- **URI**：
  - 唯一标识资源，命名用名词。
  - 例：`/users/1`（单个用户），`/users`（用户集合）。

#### (3) HTTP 方法
- **常用方法**：
  - **GET**：查询资源（如 `GET /users/1`）。
  - **POST**：创建资源（如 `POST /users`）。
  - **PUT**：更新资源（整体替换，如 `PUT /users/1`）。
  - **PATCH**：部分更新（如 `PATCH /users/1`）。
  - **DELETE**：删除资源（如 `DELETE /users/1`）。
- **幂等性**：
  - GET、PUT、DELETE 幂等，POST 非幂等。

#### (4) 状态码
- **常见状态码**：
  - `200 OK`：成功。
  - `201 Created`：创建成功。
  - `204 No Content`：删除成功。
  - `400 Bad Request`：请求错误。
  - `401 Unauthorized`：未授权。
  - `404 Not Found`：资源不存在。
  - `500 Internal Server Error`：服务器错误。

---

### 2. RESTful API 设计原则
#### (1) 资源导向
- 用名词表示资源，避免动词。
- **差**：`/getUser`、`/createOrder`。
- **好**：`/users`、`/orders`。

#### (2) 层次化 URI
- 清晰反映资源关系。
- 例：
  - `/users/1/orders`：用户 1 的订单。
  - `/orders/123/items`：订单 123 的条目。

#### (3) HTTP 方法语义
- 正确使用方法：
  - 创建：`POST /users`。
  - 查询：`GET /users?name=Alice`。
  - 更新：`PUT /users/1` 或 `PATCH /users/1`。

#### (4) 无状态
- 每次请求自包含（如携带 JWT）。
- 例：
```
GET /users/1
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
```

#### (5) HATEOAS（超媒体）
- 响应包含相关资源链接，增强导航。
- 例：
```json
{
  "id": 1,
  "name": "Alice",
  "links": [
    { "rel": "self", "href": "/users/1" },
    { "rel": "orders", "href": "/users/1/orders" }
  ]
}
```
- **少用**：实际项目中复杂。

#### (6) 版本控制
- 防止 API 变更破坏客户端。
- 例：
  - URI 版本：`/v1/users`。
  - 头版本：`Accept: application/vnd.api.v1+json`。

---

### 3. RESTful API 实现（Spring 示例）
结合您提到过的 Spring AOP，展示如何用 **Spring Boot** 实现 RESTful API。

#### (1) 依赖
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

#### (2) 实体
```java
public class User {
    private Long id;
    private String name;
    // Getters/Setters
}
```

#### (3) Controller
```java
@RestController
@RequestMapping("/users")
public class UserController {
    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    // 查询所有用户
    @GetMapping
    public List<User> getAllUsers() {
        return userService.findAll();
    }

    // 查询单个用户
    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        return userService.findById(id)
            .map(ResponseEntity::ok)
            .orElseGet(() -> ResponseEntity.notFound().build());
    }

    // 创建用户
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public User createUser(@RequestBody User user) {
        return userService.save(user);
    }

    // 更新用户
    @PutMapping("/{id}")
    public User updateUser(@PathVariable Long id, @RequestBody User user) {
        user.setId(id);
        return userService.save(user);
    }

    // 删除用户
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteUser(@PathVariable Long id) {
        userService.delete(id);
    }
}
```

#### (4) 与 Spring AOP 集成
- **日志切面**（结合您提问的 Spring AOP）：
```java
@Aspect
@Component
public class LogAspect {
    @Around("execution(* com.example.controller.*.*(..))")
    public Object log(ProceedingJoinPoint joinPoint) throws Throwable {
        System.out.println("API: " + joinPoint.getSignature().getName());
        return joinPoint.proceed();
    }
}
```

#### (5) 测试
- **创建用户**：
```bash
curl -X POST http://localhost:8080/users -H "Content-Type: application/json" -d '{"name": "Alice"}'
```
- **响应**：
```json
{
  "id": 1,
  "name": "Alice"
}
```

---

### 4. RESTful API 使用场景
- **微服务**：
  - 服务间通过 RESTful API 通信。
  - 例：订单服务调用用户服务。
- **前后端分离**：
  - 前端通过 API 获取数据。
  - 例：Vue 调用 `/users`。
- **第三方集成**：
  - 开放 API（如支付、地图）。
- **移动端**：
  - APP 调用后端 API。

#### 与您的问题
- **JWT 单点登录**：
  - RESTful API 常用于登录接口：
```bash
POST /auth/login
{
  "username": "alice",
  "password": "123"
}
```
  - 返回 JWT：
```json
{
  "token": "eyJhbGciOiJIUzI1NiJ9..."
}
```
  - 后续请求携带：
```
GET /users/1
Authorization: Bearer <token>
```

---

### 5. 优缺点
#### 优点
- **标准化**：基于 HTTP，易于理解和实现。
- **无状态**：便于扩展和高可用。
- **跨平台**：支持多种客户端（Web、移动端）。
- **可缓存**：提升性能。

#### 缺点
- **复杂性**：
  - HATEOAS 等高级特性实现复杂。
- **性能**：
  - HTTP 相比 RPC 稍慢。
- **过度获取**：
  - 可能返回多余数据（可用 GraphQL 优化）。

---

### 6. 最佳实践
- **命名规范**：
  - 用名词：`/users` 而非 `/getUsers`。
  - 复数形式：`/orders` 表示集合。
- **状态码**：
  - 精确使用，如 `201` 表示创建。
- **错误处理**：
```json
{
  "error": "User not found",
  "code": 404
}
```
- **分页**：
  - 例：`GET /users?page=2&size=10`。
- **认证**：
  - 用 JWT 或 OAuth（结合您提问的 SSO）。
- **版本化**：
  - 例：`/v1/users`。

---

### 7. 延伸与面试角度
- **与 Spring AOP**：
  - AOP 可为 API 加日志、权限验证。
- **与单点登录**：
  - RESTful API 是 SSO 的传输层，JWT 验证通过拦截器或 AOP。
- **与设计模式**：
  - **代理模式**：拦截 API 请求。
  - **外观模式**：封装复杂服务为简单 API。
- **实际应用**：
  - 电商：`/products`、`orders`。
  - 微服务：服务间 REST 调用。
- **面试点**：
  - 问“定义”时，提六大约束。
  - 问“设计”时，提命名和状态码。

---

### 8. 总结
RESTful API 是基于 REST 架构的 Web API，以资源为中心，通过 HTTP 方法（GET、POST 等）和 URI（如 `/users`）操作，遵循无状态、统一接口等原则。Spring Boot 简化实现，结合 AOP 可增强功能（如日志、JWT 验证）。面试时，可提设计规范、代码示例或 SSO 场景，结合代理模式，展示理解深度。

---

如果您想深入某部分（如具体实现、认证集成）或有其他场景，请告诉我，我可以进一步优化！