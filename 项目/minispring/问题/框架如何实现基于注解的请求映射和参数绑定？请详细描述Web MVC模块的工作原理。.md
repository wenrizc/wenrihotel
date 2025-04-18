
## 核心组件概述

MiniSpring框架的Web MVC模块主要由以下核心组件构成：

1. **DispatcherServlet**: 前端控制器，处理所有HTTP请求的入口
2. **ControllerRegistry**: 控制器注册表，保存URL路径到处理方法的映射
3. **Dispatcher**: 单个URL映射的分发器，负责参数解析和方法调用
4. **Param**: 方法参数的解析器，根据不同注解类型绑定参数

## 注解定义

MiniSpring框架使用以下关键注解来实现Web MVC功能：

```java
// 标记控制器类
@RestController / @Controller

// 请求映射注解
@GetMapping / @PostMapping / @PutMapping / @DeleteMapping / @RequestMapping

// 参数绑定注解
@PathVariable - 绑定路径参数
@RequestParam - 绑定查询参数
@RequestBody - 绑定请求体
```

## 请求映射实现原理

### 1. 控制器扫描与注册

在应用启动阶段，`DispatcherServlet`会扫描并注册所有的控制器：

```java
public class DispatcherServlet extends HttpServlet {
    private final ControllerRegistry registry;
    
    public DispatcherServlet() {
        this.registry = new ControllerRegistry();
    }
    
    // 在Servlet初始化时调用
    @Override
    public void init() throws ServletException {
        // 获取ApplicationContext
        ApplicationContext applicationContext = ApplicationContextUtils.getRequiredApplicationContext();
        
        // 获取所有Controller类型的Bean
        Map<String, Object> controllers = applicationContext.getBeansWithAnnotation(RestController.class);
        controllers.putAll(applicationContext.getBeansWithAnnotation(Controller.class));
        
        // 注册所有控制器
        for (Object controller : controllers.values()) {
            registerController(controller);
        }
    }
    
    private void registerController(Object controller) {
        // 为每个带@RequestMapping等注解的方法创建Dispatcher
        Class<?> controllerClass = controller.getClass();
        
        // 处理类级别的RequestMapping
        String classPath = "";
        RequestMapping classRequestMapping = controllerClass.getAnnotation(RequestMapping.class);
        if (classRequestMapping != null) {
            classPath = classRequestMapping.value();
        }
        
        // 遍历所有公共方法
        for (Method method : controllerClass.getMethods()) {
            // 处理方法级别的各种HTTP方法注解
            GetMapping getMapping = method.getAnnotation(GetMapping.class);
            if (getMapping != null) {
                addGetDispatcher(controller, classPath + getMapping.value(), method);
            }
            
            PostMapping postMapping = method.getAnnotation(PostMapping.class);
            if (postMapping != null) {
                addPostDispatcher(controller, classPath + postMapping.value(), method);
            }
            
            // 其他HTTP方法的注解处理...
        }
    }
}
```

### 2. URL映射关系构建

`ControllerRegistry`负责维护URL模式到分发器的映射：

```java
public class ControllerRegistry {
    // 不同HTTP方法的URL映射表
    private final Map<String, Dispatcher> getDispatchers = new HashMap<>();
    private final Map<String, Dispatcher> postDispatchers = new HashMap<>();
    private final Map<String, Dispatcher> putDispatchers = new HashMap<>();
    private final Map<String, Dispatcher> deleteDispatchers = new HashMap<>();
    
    // 添加GET方法映射
    public void addGetDispatcher(Object controller, String urlPattern, Method method) {
        Dispatcher dispatcher = new Dispatcher(HttpMethod.GET, isRestController(controller), controller, method, urlPattern);
        this.getDispatchers.put(urlPattern, dispatcher);
    }
    
    // 添加POST方法映射
    public void addPostDispatcher(Object controller, String urlPattern, Method method) {
        Dispatcher dispatcher = new Dispatcher(HttpMethod.POST, isRestController(controller), controller, method, urlPattern);
        this.postDispatchers.put(urlPattern, dispatcher);
    }
    
    // 根据HTTP方法和URL找到对应的分发器
    public Dispatcher findDispatcher(String httpMethod, String url) {
        Map<String, Dispatcher> mappings = switch (httpMethod.toUpperCase()) {
            case "GET" -> this.getDispatchers;
            case "POST" -> this.postDispatchers;
            case "PUT" -> this.putDispatchers;
            case "DELETE" -> this.deleteDispatchers;
            default -> null;
        };
        
        if (mappings == null) {
            return null;
        }
        
        // 先尝试精确匹配
        Dispatcher dispatcher = mappings.get(url);
        if (dispatcher != null) {
            return dispatcher;
        }
        
        // 再尝试模式匹配（支持路径变量）
        for (String pattern : mappings.keySet()) {
            if (pathMatch(pattern, url)) {
                return mappings.get(pattern);
            }
        }
        
        return null;
    }
    
    // 路径匹配算法，支持路径变量
    private boolean pathMatch(String pattern, String path) {
        // 路径匹配逻辑，例如 /users/{id} 匹配 /users/123
        // ...
    }
}
```

## 参数绑定实现原理

### 1. 方法参数解析

`Dispatcher` 类负责具体的参数解析和方法调用：

```java
public class Dispatcher {
    private final Object controller;
    private final Method method;
    private final String urlPattern;
    private final boolean isRest;
    private final Param[] methodParameters;
    
    public Dispatcher(String httpMethod, boolean isRest, Object controller, Method method, String urlPattern) {
        this.controller = controller;
        this.method = method;
        this.urlPattern = urlPattern;
        this.isRest = isRest;
        
        // 解析方法参数
        Parameter[] params = method.getParameters();
        Annotation[][] paramsAnnos = method.getParameterAnnotations();
        this.methodParameters = new Param[params.length];
        for (int i = 0; i < params.length; i++) {
            this.methodParameters[i] = new Param(httpMethod, method, params[i], paramsAnnos[i]);
        }
    }
    
    // 处理请求并调用控制器方法
    public Result process(String url, HttpServletRequest request, HttpServletResponse response) throws Exception {
        // 提取路径变量
        Map<String, String> pathVariables = extractPathVariables(this.urlPattern, url);
        
        // 准备调用参数
        Object[] args = new Object[this.methodParameters.length];
        for (int i = 0; i < args.length; i++) {
            Param param = this.methodParameters[i];
            args[i] = param.resolve(pathVariables, request, response);
        }
        
        // 调用控制器方法
        Object result = this.method.invoke(this.controller, args);
        
        // 处理返回值
        return processResult(result);
    }
}
```

### 2. 参数解析器实现

`Param` 类负责根据注解类型解析不同类型的参数：

```java
public class Param {
    private final Class<?> type;
    private final String name;
    private final ParameterType parameterType;
    
    public Param(String httpMethod, Method method, Parameter parameter, Annotation[] annotations) {
        this.type = parameter.getType();
        this.name = parameter.getName();
        
        // 确定参数类型
        if (containsAnnotation(annotations, PathVariable.class)) {
            this.parameterType = ParameterType.PATH_VARIABLE;
            PathVariable pathVariable = getAnnotation(annotations, PathVariable.class);
            this.name = pathVariable.value().isEmpty() ? parameter.getName() : pathVariable.value();
        }
        else if (containsAnnotation(annotations, RequestParam.class)) {
            this.parameterType = ParameterType.REQUEST_PARAM;
            RequestParam requestParam = getAnnotation(annotations, RequestParam.class);
            this.name = requestParam.value().isEmpty() ? parameter.getName() : requestParam.value();
        }
        else if (containsAnnotation(annotations, RequestBody.class)) {
            this.parameterType = ParameterType.REQUEST_BODY;
        }
        else if (HttpServletRequest.class.isAssignableFrom(type)) {
            this.parameterType = ParameterType.REQUEST_OBJECT;
        }
        else if (HttpServletResponse.class.isAssignableFrom(type)) {
            this.parameterType = ParameterType.RESPONSE_OBJECT;
        }
        else {
            this.parameterType = ParameterType.REQUEST_PARAM;
        }
    }
    
    // 根据参数类型解析参数值
    public Object resolve(Map<String, String> pathVariables, HttpServletRequest request, HttpServletResponse response) {
        switch (this.parameterType) {
            case PATH_VARIABLE:
                String pathValue = pathVariables.get(this.name);
                return convertValue(pathValue, this.type);
                
            case REQUEST_PARAM:
                String paramValue = request.getParameter(this.name);
                return convertValue(paramValue, this.type);
                
            case REQUEST_BODY:
                try {
                    BufferedReader reader = request.getReader();
                    StringBuilder sb = new StringBuilder();
                    String line;
                    while ((line = reader.readLine()) != null) {
                        sb.append(line);
                    }
                    // 使用JSON转换器将请求体转换为对象
                    return JSON.parseObject(sb.toString(), this.type);
                } catch (IOException e) {
                    throw new RuntimeException(e);
                }
                
            case REQUEST_OBJECT:
                return request;
                
            case RESPONSE_OBJECT:
                return response;
                
            default:
                throw new IllegalStateException("Unsupported parameter type: " + this.parameterType);
        }
    }
    
    // 类型转换方法
    private Object convertValue(String value, Class<?> type) {
        if (value == null) {
            return null;
        }
        
        if (type == String.class) {
            return value;
        }
        else if (type == int.class || type == Integer.class) {
            return Integer.parseInt(value);
        }
        else if (type == long.class || type == Long.class) {
            return Long.parseLong(value);
        }
        // 其他类型的转换...
        
        throw new IllegalArgumentException("Unsupported type conversion: " + type.getName());
    }
}
```

## 请求处理流程

`DispatcherServlet` 处理请求的完整流程：

```java
@Override
protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
    String method = req.getMethod();
    String path = req.getRequestURI().substring(req.getContextPath().length());
    
    // 查找匹配的分发器
    Dispatcher dispatcher = this.registry.findDispatcher(method, path);
    if (dispatcher == null) {
        // 未找到处理方法，返回404
        resp.sendError(404, "Not Found");
        return;
    }
    
    try {
        // 处理请求
        Result result = dispatcher.process(path, req, resp);
        
        // 设置响应内容类型
        String contentType = result.contentType();
        resp.setContentType(contentType);
        
        // 写入响应
        result.write(resp);
    } catch (Exception e) {
        // 异常处理
        // ...
    }
}
```

## 响应处理

控制器方法的返回值会被包装为`Result`对象，用于生成HTTP响应：

```java
public interface Result {
    // 获取响应状态码
    int statusCode();
    
    // 获取内容类型
    String contentType();
    
    // 将结果写入响应
    void write(HttpServletResponse response) throws IOException;
}

// JSON响应实现
public class JsonResult implements Result {
    private final Object data;
    private final int statusCode;
    
    public JsonResult(Object data) {
        this.data = data;
        this.statusCode = 200;
    }
    
    @Override
    public int statusCode() {
        return this.statusCode;
    }
    
    @Override
    public String contentType() {
        return "application/json;charset=utf-8";
    }
    
    @Override
    public void write(HttpServletResponse response) throws IOException {
        response.setStatus(statusCode());
        PrintWriter pw = response.getWriter();
        pw.write(JSON.toJSONString(this.data));
        pw.flush();
    }
}
```

## 完整请求示例

以一个完整的Web请求为例，展示处理流程：

当客户端发送请求 `GET /api/users/123`：

1. `DispatcherServlet` 接收请求，查找匹配的 `Dispatcher`
2. 找到对应 `@GetMapping("/api/users/{id}")` 的控制器方法
3. `Dispatcher` 解析URL路径变量 `id=123`
4. `Param` 对象将 `123` 转换为方法参数类型（如 `Integer`）
5. 调用控制器方法 `getUserById(123)`
6. 控制器方法返回用户对象
7. 返回值被包装为 `JsonResult`
8. `JsonResult` 将用户对象序列化为JSON并写入响应

通过这种设计，MiniSpring框架实现了基于注解的请求映射和参数绑定，提供了类似Spring MVC的开发体验，同时保持了代码的简洁性和可扩展性。

