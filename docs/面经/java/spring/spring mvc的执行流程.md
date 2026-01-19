
### 总体执行流程

1.  **用户请求**：用户的请求首先被发送到前端控制器 `DispatcherServlet`。
2.  **寻找处理器**：`DispatcherServlet` 收到请求后，会调用 **`HandlerMapping`** (处理器映射器) 来查找能够处理该请求的 `Handler` (通常是 Controller 里的一个方法)。
3.  **返回执行链**：`HandlerMapping` 找到对应的 `Handler` 后，会将其与配置的拦截器 (Interceptors) 一起封装成一个 **`HandlerExecutionChain`** (处理器执行链) 并返回给 `DispatcherServlet`。
4.  **获取适配器**：`DispatcherServlet` 根据 `Handler` 的类型，选择一个合适的 **`HandlerAdapter`** (处理器适配器)。适配器的作用是**以一种统一的方式去执行不同类型的 Handler**。
5.  **执行处理器**：`HandlerAdapter` 调用并执行 `HandlerExecutionChain` 中的 `Handler` (即 Controller 方法)。Controller 方法执行完毕后，会返回一个 **`ModelAndView`** 对象，其中包含了逻辑视图名和模型数据。
6.  **调用视图解析器**：`DispatcherServlet` 接收到 `ModelAndView` 后，会将它的逻辑视图名交给 **`ViewResolver`** (视图解析器) 处理。
7.  **解析视图**：`ViewResolver` 根据逻辑视图名，解析并返回一个具体的 **`View`** (视图) 对象。
8.  **视图渲染**：`DispatcherServlet` 使用 `View` 对象和 `ModelAndView` 中的模型数据来进行**视图渲染**，即将数据填充到视图模板中。
9.  **返回响应**：渲染完成后，`DispatcherServlet` 将最终生成的 HTML 页面等响应内容返回给用户。

### 详细流程解析

#### **1. DispatcherServlet 收到请求并开始处理**

*   所有请求首先进入 `DispatcherServlet` 的 `service` 方法，该方法最终会调用 `doService` 方法。
*   `doService` 方法做了一些准备工作后，调用了核心的处理方法 **`doDispatch`**。这个方法是整个流程的“总指挥”。

在 `doDispatch` 方法中，主要执行了以下关键操作：
*   **`mappedHandler = getHandler(processedRequest);`**：调用 `getHandler` 方法获取处理器执行链。
*   **`HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());`**：调用 `getHandlerAdapter` 获取能执行该处理器的适配器。
*   **`mappedHandler.applyPreHandle(...)`**：在执行 Controller 方法前，先执行执行链中所有拦截器的 `preHandle` 方法。
*   **`mv = ha.handle(...)`**：调用 `HandlerAdapter` 的 `handle` 方法，这会**真正地执行我们的 Controller 方法**，并返回 `ModelAndView`。
*   **`mappedHandler.applyPostHandle(...)`**：在 Controller 方法执行后、视图渲染前，执行所有拦截器的 `postHandle` 方法。
*   **`processDispatchResult(...)`**：处理最终的结果，包括视图渲染和异常处理。

#### **2. 查找对应的 Handler 对象 (getHandler)**

*   `DispatcherServlet` 的 `getHandler` 方法会遍历所有已注册的 `HandlerMapping`。
*   它会调用 `HandlerMapping` 的 `getHandler` 方法，该方法内部会调用 `getHandlerInternal`。
*   `getHandlerInternal` 方法会获取请求的 URL (如 `/testSpringMvc`)，然后通过 `lookupHandlerMethod` 在一个注册表 (`mappingRegistry`) 中查找与该 URL 匹配的 `@RequestMapping` 注解的方法。
*   找到匹配的 `HandlerMethod` (包含了 Controller 实例和具体方法) 后，会调用 `getHandlerExecutionChain` 方法。
*   `getHandlerExecutionChain` 方法会创建一个 `HandlerExecutionChain` 对象，将找到的 `HandlerMethod` 放入其中，并遍历所有已配置的拦截器，将与当前 URL 匹配的拦截器也添加到这个执行链中。
*   最终，一个包含处理器和拦截器的执行链被成功创建并返回。

#### **3. HandlerAdapter 执行当前的 Handler (handle)**

*   `DispatcherServlet` 拿到执行链后，通过 `getHandlerAdapter` 方法遍历所有已注册的 `HandlerAdapter`，调用其 `supports` 方法来判断哪个适配器支持当前类型的 `Handler`。对于 `@RequestMapping` 方法，匹配到的会是 **`RequestMappingHandlerAdapter`**。
*   接着，调用 `RequestMappingHandlerAdapter` 的 `handle` 方法。
*   该 `handle` 方法内部会调用 `handleInternal`，最终调用 **`invokeHandlerMethod`**。这个方法通过反射来执行我们编写的 Controller 方法 (`testSpringMvc`)。
*   Controller 方法执行后，返回一个字符串 "success"，Spring MVC 会将这个字符串和方法中 `map` 里的数据 (`note="在看转发二连"`) 封装成一个 `ModelAndView` 对象并返回。

#### **4. 处理最终结果以及渲染 (processDispatchResult & render)**

*   `doDispatch` 方法的最后一步是调用 `processDispatchResult`。
*   该方法负责处理异常，如果没有异常，则调用 **`render(mv, request, response)`** 方法进行视图渲染。
*   `render` 方法的核心逻辑是：
    1.  从 `ModelAndView` 对象中获取逻辑视图名 (即 "success")。
    2.  调用 `resolveViewName` 方法，该方法会遍历所有配置的 `ViewResolver` (视图解析器)。
    3.  在本例中，`ThymeleafViewResolver` 会匹配成功，并根据前缀和后缀规则，将逻辑视图名 "success" 解析成一个具体的 `ThymeleafView` 对象。
    4.  最后，调用 `ThymeleafView` 对象的 **`render`** 方法，将 `ModelAndView` 中的模型数据 (`map`里的数据) 填充到 Thymeleaf 模板文件中，生成最终的 HTML。
*   渲染完成后，`DispatcherServlet` 将生成的响应返回给浏览器，用户最终看到 "在看转发二连" 这段文字。
*   在整个过程结束后（无论成功还是异常），`triggerAfterCompletion` 会被调用，以执行拦截器的 `afterCompletion` 方法，用于资源清理等工作。