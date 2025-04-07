
## 1. 核心组件

MiniSpring Web MVC 主要包含以下核心组件：

- **DispatcherServlet**: 中央处理器，接收所有请求并分发
- **Controller/RestController**: 处理请求的控制器
- **ViewResolver**: 视图解析接口
- **FreeMarkerViewResolver**: 基于 FreeMarker 的视图解析器实现
- **ModelAndView**: 封装模型数据和视图信息

## 2. Web 应用启动流程

当 Web 应用启动时：

1. Servlet 容器初始化，调用 `ContextLoaderListener.contextInitialized`
2. 创建 `PropertyResolver` 解析配置属性
3. 设置请求/响应编码
4. 创建应用上下文 `ApplicationContext`
5. 注册自定义过滤器 `WebUtils.registerFilters`
6. 注册 `DispatcherServlet` 处理所有 URL 请求
7. 将应用上下文作为属性存储在 `ServletContext` 中

```java
public void contextInitialized(ServletContextEvent sce) {
    var servletContext = sce.getServletContext();
    // 设置 ServletContext
    WebMvcConfiguration.setServletContext(servletContext);
    
    // 创建属性解析器和设置编码
    var propertyResolver = WebUtils.createPropertyResolver();
    String encoding = propertyResolver.getProperty("${summer.web.character-encoding:UTF-8}");
    servletContext.setRequestCharacterEncoding(encoding);
    servletContext.setResponseCharacterEncoding(encoding);
    
    // 创建应用上下文
    var applicationContext = createApplicationContext(...);
    
    // 注册过滤器和DispatcherServlet
    WebUtils.registerFilters(servletContext);
    WebUtils.registerDispatcherServlet(servletContext, propertyResolver);
}
```

## 3. DispatcherServlet 初始化

`DispatcherServlet` 初始化时：

1. 扫描应用上下文中所有带有 `@Controller` 或 `@RestController` 注解的类
2. 为每个控制器添加处理器映射：
   - 解析 `@GetMapping` 和 `@PostMapping` 注解
   - 创建对应的 `Dispatcher` 对象处理不同的 URL 模式
   - 支持 URL 路径变量、请求参数等不同类型的参数注入

```java
@Override
public void init() throws ServletException {
    // 扫描所有控制器
    for (var def : applicationContext.findBeanDefinitions(Object.class)) {
        Class<?> beanClass = def.getBeanClass();
        Object bean = def.getRequiredInstance();
        
        // 检查注解并添加控制器
        Controller controller = beanClass.getAnnotation(Controller.class);
        RestController restController = beanClass.getAnnotation(RestController.class);
        
        if (controller != null) {
            addController(false, def.getName(), bean);
        }
        if (restController != null) {
            addController(true, def.getName(), bean);
        }
    }
}
```

## 4. 请求处理流程

当请求到达时：

1. 根据 HTTP 方法调用 `doGet` 或 `doPost`
2. 对于静态资源请求（如 favicon.ico 或 /static/ 下的资源），调用 `doResource` 处理
3. 其他请求通过 `doService` 方法处理

```java
@Override
protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
    String url = req.getRequestURI();
    if (url.equals(this.faviconPath) || url.startsWith(this.resourcePath)) {
        doResource(url, req, resp);
    } else {
        doService(req, resp, this.getDispatchers);
    }
}
```

## 5. URL 匹配和参数解析

`Dispatcher` 负责匹配 URL 和解析参数：

1. 使用正则表达式匹配 URL 模式
2. 提取 URL 中的路径变量 (`@PathVariable`)
3. 获取请求参数 (`@RequestParam`)
4. 解析请求体 (`@RequestBody`)
5. 注入 Servlet 相关对象 (如 `HttpServletRequest`、`HttpServletResponse` 等)

```java
Result process(String url, HttpServletRequest request, HttpServletResponse response) throws Exception {
    Matcher matcher = urlPattern.matcher(url);
    if (matcher.matches()) {
        Object[] arguments = new Object[this.methodParameters.length];
        for (int i = 0; i < arguments.length; i++) {
            Param param = methodParameters[i];
            arguments[i] = switch (param.paramType) {
                case PATH_VARIABLE -> {
                    String s = matcher.group(param.name);
                    yield convertToType(param.classType, s);
                }
                case REQUEST_PARAM -> {
                    String s = getOrDefault(request, param.name, param.defaultValue);
                    yield convertToType(param.classType, s);
                }
                case REQUEST_BODY -> {
                    BufferedReader reader = request.getReader();
                    yield JsonUtils.readJson(reader, param.classType);
                }
                case SERVLET_VARIABLE -> {
                    // 注入 HttpServletRequest, HttpServletResponse 等
                    // ...
                }
            };
        }
        // 调用控制器方法
        Object result = this.handlerMethod.invoke(this.controller, arguments);
        return new Result(true, result);
    }
    return NOT_PROCESSED;
}
```

## 6. 响应处理

根据控制器方法返回类型和注解，有不同处理方式：

1. **REST 控制器**：
   - 将返回结果转换为 JSON 格式
   - 设置 `Content-Type: application/json`
   - 直接写入响应流

2. **普通控制器**：
   - 如果返回 `String` 类型：
     - 带有 `@ResponseBody`：直接作为响应体
     - 以 `redirect:` 开头：执行重定向
   - 如果返回 `ModelAndView` 对象：
     - 通过 `ViewResolver` 渲染指定的视图
     - 将模型数据传递给视图

## 7. 视图渲染

视图渲染由 `ViewResolver` 完成：

```java
public void render(String viewName, Map<String, Object> model, HttpServletRequest req, HttpServletResponse resp) 
    throws ServletException, IOException {
    
    Template templ = this.config.getTemplate(viewName);
    PrintWriter pw = resp.getWriter();
    try {
        templ.process(model, pw);
    } catch (TemplateException e) {
        throw new ServerErrorException(e);
    }
    pw.flush();
}
```

默认实现是 `FreeMarkerViewResolver`，支持：
- 指定模板路径和编码
- HTML 模板渲染
- 模型数据注入

## 8. 异常处理

请求处理过程中的异常会被捕获并适当处理：

- `ErrorResponseException`：转换为 HTTP 错误状态码
- `ServerWebInputException`：处理输入验证错误
- `ServerErrorException`：处理服务器内部错误
- 其他异常：记录日志并包装为 `NestedRuntimeException`

## 总结

MiniSpring Web MVC 模块通过模仿 Spring MVC 的设计，实现了一个轻量级但功能完备的 MVC 框架。它支持注解驱动的控制器、URL 映射、参数解析、视图渲染等核心功能，为 Web 应用提供了简洁高效的开发方式。

通过这种设计，开发者可以使用类似 Spring MVC 的方式定义控制器和处理方法，享受注解驱动开发带来的便利。

