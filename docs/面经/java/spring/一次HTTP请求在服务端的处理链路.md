## 1. 从浏览器发出请求：到达服务端前发生了什么

浏览器把 HTTP 请求发出去之前，通常会经历：

- URL 解析与缓存决策（强缓存/协商缓存）。
- DNS 解析拿到 IP。
- 建立连接：TCP（三次握手）+（如是 HTTPS）TLS 握手。
- 选择协议栈：HTTP/1.1（多连接）或 HTTP/2（单连接多路复用）等。

## 2. 请求进入机房：CDN / WAF / 负载均衡 / 反向代理

一条典型链路：

1. **CDN**：静态资源优先命中边缘缓存；未命中才回源。
2. **WAF**：拦截常见攻击（SQL 注入、扫描、CC 等）。
3. **负载均衡（4/7 层）**：把流量分发到后端实例（一致性哈希、最小连接数等策略）。
4. **Nginx/Envoy**：反向代理，常做：
   - 路由与灰度；
   - 限流、熔断、重试（需谨慎，避免放大风暴）；
   - gzip/br 压缩；
   - 连接复用与缓冲（`proxy_buffering`）。

## 3. 进入服务器之后：Linux 网络栈与 Socket 层

请求到达机器后，数据大体会经历：

1. 网卡收到包 → 中断/软中断 → 内核协议栈处理（IP/TCP）。
2. TCP 把数据按连接（socket）重组，放入接收缓冲区。
3. 服务端 `listen` 的 socket 维护两个队列（概念上）：
   - 半连接队列（SYN 收到但未完成握手）。
   - 全连接队列（握手完成，等待应用 `accept`）。
4. 应用（Tomcat）调用 `accept()` 拿到已建立连接，再通过 `epoll/kqueue` 等多路复用等待可读事件。

关键风险点：

- `backlog` 太小或负载突刺：导致连接建立失败或排队延迟上升。
- 大量短连接：握手与 TIME_WAIT 放大，端口与 CPU 压力上升（优先连接复用/HTTP/2）。

## 4. Tomcat 接管连接：从 Connector 到 Servlet

以 Tomcat（NIO）为例，一个请求进入容器的主路径可以理解为：

### 4.1 Connector：网络层与线程模型

- **Acceptor**：负责 `accept()` 新连接。
- **Poller**：基于 NIO Selector 监听 socket 可读/可写事件。
- **Worker 线程池**：把“解析请求 + 调用应用”交给工作线程执行。

常见配置项：

- `maxThreads`：最大工作线程数。
- `acceptCount`：连接排队上限（队列满会拒绝/阻塞）。
- `connectionTimeout`：读请求头/体的超时等（版本与协议栈相关）。

### 4.2 解析 HTTP：从字节流到 Request/Response

Tomcat 会把 socket 的字节流解析为：

- 请求行：method、URI、协议版本。
- 请求头：Host、Cookie、Content-Type、Content-Length 等。
- 请求体：表单、JSON、文件上传等。

解析完成后，Tomcat 构造 `HttpServletRequest` / `HttpServletResponse`（门面对象），准备进入 Servlet 规范链路。

### 4.3 FilterChain：Servlet 过滤器链

请求进入应用之前会先经过 Filter：

- 典型基础过滤器：编码、请求上下文、表单内容处理等。
- 如果接入 **Spring Security**，最关键的是 **Security Filter Chain**（鉴权、会话、CSRF 等）。

Filter 的本质是“环绕调用”，最终会走到 Servlet 的 `service()`。

## 5. Spring MVC 处理：DispatcherServlet 的核心链路

Spring MVC 的入口是 `DispatcherServlet`，它是前端控制器。典型处理链路：

1. **HandlerMapping**：根据 URL、HTTP method、Header 等找到匹配的 `Handler`（一般是 `@RequestMapping` 方法）。
2. **HandlerAdapter**：选择合适适配器执行 Handler（常见是 `RequestMappingHandlerAdapter`）。
3. **拦截器（HandlerInterceptor）**：
   - `preHandle`：进入 Controller 前（鉴权、限流、Trace 注入等）。
   - `postHandle`：Controller 后、渲染前。
   - `afterCompletion`：请求结束清理资源。
4. **参数解析**：`HandlerMethodArgumentResolver` 把请求解析成方法参数：
   - `@RequestParam`、`@PathVariable`、`@RequestBody` 等。
5. **数据绑定与校验**：`WebDataBinder` + `javax.validation`（`@Valid`）。
6. **调用 Controller**：执行你的业务逻辑（可能进入 Service、DAO、RPC）。
7. **返回值处理**：
   - `@ResponseBody` / `ResponseEntity`：走 `HttpMessageConverter`（Jackson 序列化 JSON）。
   - 视图渲染：返回 `ModelAndView`，走 `ViewResolver`（前后端分离场景较少）。
8. **异常处理**：
   - `HandlerExceptionResolver` 链（`@ControllerAdvice` / `@ExceptionHandler` 等）。

## 6. “业务逻辑执行阶段”常见关键环节

### 6.1 事务：@Transactional

如果使用 `@Transactional`：

- Spring AOP 会在方法调用前后织入事务边界（开启、提交、回滚）。
- 真正生效的前提是“通过代理调用”（自调用常见失效点）。

### 6.2 下游依赖：Redis / MySQL / MQ / RPC

请求的总耗时往往不在 Controller，而在下游依赖：

- MySQL 慢查询、锁等待；
- Redis 超时与雪崩；
- RPC 链路抖动；
- MQ 堆积导致同步等待。

因此排查链路时，必须把“应用耗时”拆成：应用自身 + 下游依赖耗时。

## 7. 响应返回：从 Spring 回到浏览器

1. Spring 写入响应体与响应头（状态码、`Content-Type`、缓存头等）。
2. Tomcat 把响应序列化为 HTTP 报文，写回 socket（可能分块传输）。
3. 上游（Nginx/LB）可能做缓冲、压缩、连接复用等，再返回给客户端。
4. 浏览器接收响应，决定是否缓存并触发渲染/JS 执行。

## 8. 一条链路里常见的观测点（定位慢在哪里）

- 浏览器侧：TTFB、DNS、TLS、下载耗时。
- Nginx：`request_time`、`upstream_response_time`、状态码分布。
- Tomcat：访问日志、线程池耗尽、GC 停顿。
- 应用：接口耗时分布、Trace（TraceId）、关键下游 span。
- MySQL：慢查询日志、锁等待、死锁信息。

