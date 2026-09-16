---
title: SpringBoot的一次完整请求发生了什么？
date: 2026-08-26 13:51:15
tags:
  - spring
  - tomcat
categories:
  - develop
---

## 引言

线上系统突然出现一批 400 错误，前端坚持说参数已经传了，后端日志里只有一行 `Resolved [org.springframework.web.bind.MissingServletRequestParameterException]`。你打开 IDEA，在 Controller 方法上打了个断点，发现请求根本没进来。于是你开始怀疑：是 Nginx 转发错了？是网关把参数吃了？还是 Tomcat 根本没收到请求？

这类问题，如果对请求的完整链路没有清晰的认知，就只能靠猜。而猜，是最低效的 Debug 方式。

这篇文章试图回答一个看似简单的问题：用户在前端点了一下按钮，前端发出请求，一直到数据返回，中间到底发生了什么？答案会涉及网络协议栈、Tomcat 的线程模型、Servlet 规范、Spring MVC 的调度逻辑，以及一系列扩展点。我们不会死抠每一行源码，但会保证关键环节的技术准确性，并指出每个阶段可能出现的典型错误。


## 一、全局视角：请求的完整路径

在深入细节之前，先建立一张全局地图。后面所有内容，都是在这张地图上填充细节。

```text
浏览器 / 前端
  │  点击按钮，fetch / XHR / axios 发出 HTTP 请求
  ▼
操作系统协议栈
  │  DNS → TCP → TLS → HTTP
  ▼
物理网络
  │  网卡 → 交换机 → 路由器 → 公网 / 专线
  ▼
服务器入口
  │  负载均衡 / K8s Ingress / Nginx / API 网关
  ▼
服务器内核
  │  网卡 → 协议栈 → Socket → 监听端口
  ▼
Tomcat / Netty
  │  Acceptor → Poller → Worker
  │  解析 HTTP，封装 HttpServletRequest / HttpServletResponse
  ▼
FilterChain
  │  编码、安全、跨域、日志
  ▼
DispatcherServlet
  │  HandlerMapping → HandlerAdapter → Interceptor
  │  ArgumentResolver → Converter / HttpMessageConverter
  ▼
Controller
  │  业务代码
  ▼
DispatcherServlet
  │  ReturnValueHandler → HttpMessageConverter
  ▼
Tomcat / Netty
  │  写回 HttpServletResponse → Socket
  ▼
网络
  │  原路返回
  ▼
浏览器 / 前端
```

这张图里的每一个箭头，都对应着一次真实的函数调用或一次数据结构的转换。下面我们从最外层开始，逐层深入。


## 二、网络层：请求如何到达服务器

### 2.1 封装与解封装

当浏览器发出一个 HTTP 请求时，数据并不是直接“飞”到服务器的。它需要在操作系统的网络协议栈中经历一次逐层封装的过程。

在发送端，HTTP 报文在应用层被构造出来，然后向下传递。传输层加上 TCP 头，包含源端口、目标端口、序号等字段，形成 TCP 段。网络层加上 IP 头，包含源 IP、目标 IP、TTL 等字段，形成 IP 包。数据链路层加以太网头，包含源 MAC、目标 MAC，形成以太网帧。最后在物理层变成电信号或光信号发送出去。

接收端的处理顺序完全相反。网卡收到信号后，通过 DMA 写入内存，内核触发中断，协议栈开始逐层拆解。物理层还原比特流，数据链路层检查 MAC 地址，网络层检查 IP 地址，传输层根据目标端口找到正在监听的 Socket。最终，内核把连接放入对应 Socket 的接收队列，监听该端口的进程被唤醒。

这个过程的核心意义在于：你的 Spring Boot 服务能收到请求，前提是内核已经完成了所有底层的拆包工作。如果请求根本没到达服务，问题一定出在这一层之前的某个环节。

### 2.2 中间基础设施与常见故障

生产环境中的请求路径远比“浏览器直连服务器”复杂。典型的链路包括 DNS 解析、CDN 和负载均衡分发、K8s Ingress 七层转发、Service 通过 kube-proxy 或 IPVS 将请求转发到某个 Pod，以及 API 网关的鉴权、路由和限流。

每多一层，就多一个故障点：

| 现象 | 大概率原因 |
|---|---|
| `ERR_NAME_NOT_RESOLVED` | DNS 解析失败 |
| `Connection refused` | 端口未监听，或防火墙拦截 |
| Nginx 502 | 后端服务未启动，或连接被拒绝 |
| Nginx 504 | 后端处理超时，网关等不到响应 |
| K8s Pod 未就绪 | Service 没有可用 Endpoint |

因此，Debug 的第一步不是打开 IDE，而是确认请求是否真正到达了你的服务。用 `curl` 直接访问 Pod IP 和端口，或者查看 Nginx 的 access log，通常能快速排除或定位问题。


## 三、容器层：Tomcat 如何接住请求

### 3.1 Spring Boot 启动内嵌 Tomcat 的过程

当项目引入 `spring-boot-starter-web` 时，Spring Boot 会在启动过程中自动配置一个内嵌 Tomcat。

核心自动配置类是 `ServletWebServerFactoryAutoConfiguration`。它根据 classpath 决定创建哪种容器工厂。存在 `spring-boot-starter-web` 时，创建 `TomcatServletWebServerFactory`。在容器刷新阶段，Spring Boot 调用该工厂的 `getWebServer()` 方法，创建并启动 `TomcatWebServer`。

与此同时，`DispatcherServletAutoConfiguration` 创建 `DispatcherServlet` 并注册为一个 Servlet，通常映射到 `/`。`WebMvcAutoConfiguration` 创建 Spring MVC 所需的 `HandlerMapping`、`HandlerAdapter`、`HandlerExceptionResolver` 等组件。

需要明确的是：Spring Boot 并没有“取代”Servlet 容器，而是将 Tomcat 作为内嵌容器启动。`DispatcherServlet` 本质上仍然是一个运行在 Tomcat 中的 Servlet。

### 3.2 Tomcat 的 NIO 线程模型

Tomcat 默认使用 NIO 连接器，其线程模型包含三类关键线程。NIO 模式下，Acceptor、Poller 和 Worker 线程的协作方式直接决定了应用能承受多大的并发压力。

**Acceptor 线程**通常只有 1 个，负责监听端口，接受新的 TCP 连接。收到连接后，将 Socket 包装成 `NioSocketWrapper` 对象，注册到 Poller 的事件队列中。

**Poller 线程**数量通常与 CPU 核心数相关，使用 `Selector` 轮询已建立的连接，检查是否有数据可读或可写。当某个 Socket 有数据可读时，Poller 创建 `SocketProcessorBase` 任务，提交到 Worker 线程池执行。

**Worker 线程池**负责实际处理 HTTP 请求，线程名通常为 `http-nio-8080-exec-*`，默认最大线程数为 200。Worker 线程通过 `Http11Processor` 解析 HTTP 报文，经由 `CoyoteAdapter` 将 Tomcat 内部对象适配为 Servlet 规范的 `HttpServletRequest`，然后分派到 Servlet。

需要特别注意的是：Worker 线程池的大小并不等于最大并发数。一个请求在 Worker 线程中处理的时间，可能远大于在 Poller 中等待的时间。如果 Worker 线程全部被占用，新的请求即使已经建立了 TCP 连接，也会在 Poller 的队列中等待。这也是为什么在高并发场景下，慢查询和外部 API 调用会迅速耗尽线程池。


## 四、Filter 层：在 DispatcherServlet 之前

在请求进入 `DispatcherServlet` 之前，会先经过 FilterChain。Filter 属于 Servlet 规范，与 Spring 无关。

**Filter 的关键特性是“环绕式”执行。** `chain.doFilter()` 调用之前的代码在请求进入时执行，调用之后的代码在响应返回时执行。这意味着 Filter 不仅在请求阶段起作用，也能在响应阶段处理逻辑。

一个典型的日志 Filter：

```java
public class LoggingFilter implements Filter {
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        long start = System.currentTimeMillis();
        chain.doFilter(req, res);  // 请求继续向下传递
        long cost = System.currentTimeMillis() - start;
        log.info("Request [{}] cost: {}ms", ((HttpServletRequest) req).getRequestURI(), cost);
    }
}
```

如果 Filter 在 `chain.doFilter()` 之前直接写回了响应，请求就不会进入 `DispatcherServlet`。Spring Security 在认证失败时直接返回 401，就是这种模式。

多个 Filter 的执行顺序是：请求阶段按注册顺序正序执行，响应阶段倒序执行。

**Filter 与 Interceptor 的区别**：Filter 属于 Servlet 规范，在 `DispatcherServlet` 之前执行，不能直接获取 Spring Bean。Interceptor 属于 Spring MVC，在 `HandlerMapping` 之后执行，可以获取 Handler 信息。


## 五、DispatcherServlet：整个流程的总调度中心

请求穿过 FilterChain 后，到达 `DispatcherServlet`。它是 Spring MVC 的前端控制器，继承自 `HttpServlet`，核心方法是 `doDispatch()`。所有请求首先进入 `DispatcherServlet` 的 `doDispatch()` 方法，这是整个处理流程的总控中心。Spring MVC 的设计精妙地融合了“前端控制器”与“责任链”模式，将 HTTP 请求的处理过程拆解为高度标准化、可插拔的执行链路。

简化后的 `doDispatch()` 结构如下：

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response)
        throws Exception {
    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    ModelAndView mv = null;
    Exception dispatchException = null;

    try {
        // 1. 检查是否是文件上传请求
        processedRequest = checkMultipart(request);

        // 2. 根据请求找到 Handler（Controller 方法 + 拦截器链）
        mappedHandler = getHandler(processedRequest);
        if (mappedHandler == null) {
            noHandlerFound(processedRequest, response);  // 404
            return;
        }

        // 3. 找到能执行 Handler 的 HandlerAdapter
        HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

        // 4. 执行拦截器 preHandle
        if (!mappedHandler.applyPreHandle(processedRequest, response)) {
            return;  // 某个拦截器返回 false，请求中断
        }

        // 5. 执行 Controller 方法
        mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

        // 6. 异步请求直接返回
        if (asyncManager.isConcurrentHandlingStarted()) {
            return;
        }

        applyDefaultViewName(processedRequest, mv);

        // 7. 执行拦截器 postHandle
        mappedHandler.applyPostHandle(processedRequest, response, mv);
    } catch (Exception ex) {
        dispatchException = ex;
    }

    // 8. 处理结果：渲染视图或处理异常
    processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
}
```


## 六、HandlerMapping：路由匹配

`getHandler(request)` 遍历所有 `HandlerMapping`，找到匹配的 Handler。最常用的是 `RequestMappingHandlerMapping`，它在启动时扫描所有 `@Controller`，将类和方法上的 `@RequestMapping` 注解值合并，封装成 `RequestMappingInfo`，与 `HandlerMethod` 建立映射关系。请求到来时，根据 URL 路径、HTTP 方法、请求参数、请求头等条件匹配到唯一的 `HandlerMethod`。

**路径匹配策略的演变值得关注。** Spring 5.3 引入了 `PathPatternParser`，从 Spring 6.0 开始默认启用，取代了传统的 `AntPathMatcher`。这一变化导致某些宽松的 Ant 风格路径模式不再被支持，也是 Spring Boot 2.6 之后某些 404 的隐秘原因。

匹配成功后返回 `HandlerExecutionChain`，包含目标 `HandlerMethod` 和 `HandlerInterceptor` 列表。

**常见错误**：

- **404 Not Found**：没有匹配到 Handler。检查 URL 是否正确、`context-path` 是否配置、HTTP 方法是否匹配、网关是否转发正确。
- **405 Method Not Allowed**：路径匹配了，但 HTTP 方法不匹配。例如 `@GetMapping` 收到了 POST 请求。


## 七、参数解析：Spring MVC 的策略模式

`HandlerMapping` 返回的是 Handler，但 `DispatcherServlet` 并不知道如何执行它。`HandlerAdapter` 负责屏蔽不同类型 Handler 的执行差异。对于 `@RequestMapping` 方法，使用 `RequestMappingHandlerAdapter`。

真正的参数解析发生在 `InvocableHandlerMethod.getMethodArgumentValues()` 中。Spring MVC 默认注册了 27 个参数解析器。对 Controller 方法的每个参数，Spring 会按顺序遍历这些解析器，找到第一个 `supportsParameter()` 返回 `true` 的解析器，然后调用它的 `resolveArgument()` 方法。

```java
private Object[] getMethodArgumentValues(NativeWebRequest request, ...) {
    MethodParameter[] parameters = getMethodParameters();
    Object[] args = new Object[parameters.length];

    for (int i = 0; i < parameters.length; i++) {
        MethodParameter parameter = parameters[i];
        HandlerMethodArgumentResolver resolver = getArgumentResolver(parameter);
        args[i] = resolver.resolveArgument(parameter, mavContainer, request, binderFactory);
    }
    return args;
}
```

两个关键点：**顺序敏感**——如果多个解析器都返回 `true`，排在列表前面的优先；**结果被缓存**——第一次解析后，参数与解析器的对应关系被缓存，后续请求不再重复遍历。

常见的解析器与注解对应关系如下：

| 注解 / 类型 | 解析器 |
|---|---|
| `@RequestParam` | `RequestParamMethodArgumentResolver` |
| `@PathVariable` | `PathVariableMethodArgumentResolver` |
| `@RequestHeader` | `RequestHeaderMethodArgumentResolver` |
| `@RequestBody` | `RequestResponseBodyMethodProcessor` |
| `@ModelAttribute` | `ServletModelAttributeMethodProcessor` |
| `HttpServletRequest` | `ServletRequestMethodArgumentResolver` |

### 7.1 没有注解的参数会怎样

规则取决于参数类型，核心判断标准是：**简单类型** vs **复杂类型**。

**简单类型**（String、int、Long、boolean 等 `ConversionService` 能直接转换的类型）由 `RequestParamMethodArgumentResolver` 处理。它把参数名当作请求参数名，从查询参数或表单参数中取值。**复杂类型**（自定义 POJO）由 `ServletModelAttributeMethodProcessor` 处理，等价于给参数加了 `@ModelAttribute`。

**路径变量是例外。** 即使参数名与 `@GetMapping("/user/{id}")` 中的 `{id}` 一致，没有 `@PathVariable` 注解也无法绑定。

这里有一个容易踩的坑：**编译时必须保留参数名。** Java 8+ 需要编译时加上 `-parameters` 参数，Spring Boot 的 Maven 插件默认已开启。如果参数名丢失，Spring 会报 `IllegalArgumentException`。

### 7.2 Converter 与 HttpMessageConverter 的分工

**Converter** 用于参数绑定阶段的类型转换。当 `@RequestParam` 或 `@PathVariable` 从请求中提取出字符串后，需要将其转换为方法参数声明的 Java 类型。例如请求 `?id=123` 中的 `"123"` 是字符串，方法参数是 `Long`，Spring 通过 `ConversionService` 完成转换。Converter 是通用的类型转换接口，可以在任意 Object 之间转换；Formatter 专门用于字符串与目标类型之间的双向转换，支持 locale 感知。

**HttpMessageConverter** 用于请求体和响应体的序列化与反序列化。当使用 `@RequestBody` 或 `@ResponseBody` 时，Spring 不会使用普通的 Converter，而是选择合适的 `HttpMessageConverter`。最常见的 `MappingJackson2HttpMessageConverter` 负责 JSON 与 Java 对象之间的转换。

两者的分工可以总结为：Converter 处理“单个参数值”的类型转换，HttpMessageConverter 处理“整个请求体或响应体”的序列化与反序列化。

### 7.3 自定义参数解析器

理解了策略模式之后，自定义参数解析器就变得非常自然。以 `@CurrentUser` 为例，它让 Controller 直接拿到当前登录用户：

```java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface CurrentUser {
}
```

实现 `HandlerMethodArgumentResolver`：

```java
@Component
public class CurrentUserArgumentResolver implements HandlerMethodArgumentResolver {

    private final TokenService tokenService;

    public CurrentUserArgumentResolver(TokenService tokenService) {
        this.tokenService = tokenService;
    }

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(CurrentUser.class)
                && User.class.isAssignableFrom(parameter.getParameterType());
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
                                  NativeWebRequest webRequest, WebDataBinderFactory binderFactory) {
        HttpServletRequest request = webRequest.getNativeRequest(HttpServletRequest.class);
        String token = request.getHeader("Authorization");
        User user = tokenService.parseUser(token);
        if (user == null) {
            throw new UnauthorizedException("未登录或 Token 无效");
        }
        return user;
    }
}
```

注册到 Spring MVC：

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        resolvers.add(new CurrentUserArgumentResolver(tokenService));
    }
}
```

**为什么 `@CurrentUser` 不会和 `@RequestBody` 冲突？** 因为 `supportsParameter()` 检查的是注解。`RequestResponseBodyMethodProcessor` 只认 `@RequestBody`，`CurrentUserArgumentResolver` 只认 `@CurrentUser`。参数上有什么注解，就由对应的解析器处理，两者互不干扰。如果同一个参数上同时加两个注解，排在前面的解析器会赢。

**优先级陷阱**：自定义解析器默认排在所有内置解析器之后。如果某个内置解析器的 `supportsParameter()` 也对你的参数返回 `true`，内置的会优先。需要让自定义解析器优先时，可以用 `resolvers.add(0, ...)` 插入到列表最前面。

### 7.4 参数解析阶段的常见错误

- **400 Bad Request**：缺少必填参数（`MissingServletRequestParameterException`）、类型转换失败（`MethodArgumentTypeMismatchException`）、JSON 格式错误（`HttpMessageNotReadableException`）。
- **415 Unsupported Media Type**：`Content-Type` 不支持。

调试 400 时，可以开启 `spring.mvc.log-request-details=true`。生产环境应关闭此选项，避免敏感信息泄露。


## 八、Interceptor：拦截器的执行顺序与中断逻辑

在 Controller 方法执行前后，`HandlerInterceptor` 提供了三个切入时机：`preHandle()` 在 Controller 执行前调用，`postHandle()` 在 Controller 执行后、视图渲染前调用，`afterCompletion()` 在整个请求完成后调用，用于资源清理。

**执行顺序**：当多个拦截器同时存在时，`preHandle` 按注册顺序正序执行，`postHandle` 和 `afterCompletion` 按注册顺序倒序执行。这种设计符合栈的语义：先进入的拦截器后退出。

**中断逻辑**：如果某个拦截器的 `preHandle` 返回 `false`，请求立即中断，Controller 不会执行。返回 `false` 的拦截器之前的拦截器的 `afterCompletion` 会被执行，但 `postHandle` 不会执行。

**Filter 与 Interceptor 的对比**：

| 维度 | Filter | Interceptor |
|---|---|---|
| 规范 | Servlet 规范 | Spring MVC |
| 位置 | DispatcherServlet 之前 | HandlerMapping 之后 |
| 能否拿到 Handler | 不能 | 能 |
| 环绕式执行 | 是（doFilter 前后） | 是（preHandle / postHandle） |
| 典型用途 | 编码、安全、跨域 | 鉴权、日志、业务拦截 |


## 九、Controller 执行与异常处理

参数解析完成后，Spring 通过反射调用 Controller 方法。如果 Controller 抛出异常，`DispatcherServlet` 在 `processDispatchResult()` 中会调用 `processHandlerException()`，将异常交给 `HandlerExceptionResolver` 处理。

异常处理的优先级从高到低：`@Controller` + `@ExceptionHandler`（本类内）、`@ControllerAdvice` + `@ExceptionHandler`（全局）、`@ResponseStatus` / `ResponseStatusException`（基于注解或异常类型）、`DefaultHandlerExceptionResolver`（Spring MVC 标准异常）。

全局异常处理器示例：

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResult> handleBusinessException(BusinessException ex) {
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
                .body(new ErrorResult(ex.getCode(), ex.getMessage()));
    }

    @ExceptionHandler(MethodArgumentTypeMismatchException.class)
    public ResponseEntity<ErrorResult> handleTypeMismatch(MethodArgumentTypeMismatchException ex) {
        String message = String.format("参数 '%s' 类型错误，期望 %s",
                ex.getName(), ex.getRequiredType().getSimpleName());
        return ResponseEntity.badRequest()
                .body(new ErrorResult("TYPE_MISMATCH", message));
    }
}
```

如果没有任何异常处理器能处理，最终返回 500。需要注意的是，Filter 中抛出的异常不会被 `@ControllerAdvice` 捕获，因为 Filter 执行在 `DispatcherServlet` 之前。


## 十、返回值处理：HandlerMethodReturnValueHandler

Controller 方法执行完毕后，返回值会交给 `HandlerMethodReturnValueHandler` 处理。这是 Spring MVC 中负责处理 Controller 返回值的策略接口，默认有 15 个实现，Spring 会从头遍历，找到第一个 `supportsReturnType()` 返回 `true` 的处理器。

```java
public interface HandlerMethodReturnValueHandler {

    boolean supportsReturnType(MethodParameter returnType);

    void handleReturnValue(Object returnValue, MethodParameter returnType,
                          ModelAndViewContainer mavContainer,
                          NativeWebRequest webRequest) throws Exception;
}
```

常见的返回值处理器：

| 返回值类型 | 处理器 |
|---|---|
| `@ResponseBody` | `RequestResponseBodyMethodProcessor` |
| `ResponseEntity` | `HttpEntityMethodProcessor` |
| `String` 视图名 | `ViewNameMethodReturnValueHandler` |
| `ModelAndView` | `ModelAndViewMethodReturnValueHandler` |
| `Callable` | `CallableMethodReturnValueHandler` |
| `DeferredResult` | `DeferredResultMethodReturnValueHandler` |

### 10.1 @ResponseBody 的处理流程

`RequestResponseBodyMethodProcessor` 同时实现了 `HandlerMethodArgumentResolver` 和 `HandlerMethodReturnValueHandler`，既负责请求体反序列化，也负责响应体序列化。对于 `@ResponseBody` 标注的方法，它调用 `writeWithMessageConverters()`，根据 `Accept` 头、返回值类型、`produces` 条件，选择合适的 `HttpMessageConverter`，将对象序列化为 JSON，写入 `HttpServletResponse`。

**常见错误**：

- **406 Not Acceptable**：`Accept` 头要求的类型没有对应的 `HttpMessageConverter`。
- **500 Internal Server Error**：Jackson 序列化失败。常见原因包括循环引用、懒加载对象在序列化时触发额外查询、缺少 getter 方法等。

### 10.2 内容协商：HttpMessageConverter 的选择逻辑

当一个请求同时支持多种响应格式时，Spring 如何决定用哪个 `HttpMessageConverter`？答案是 `ContentNegotiationManager`。

内容协商策略的优先级是：**后缀 > 请求参数 > HTTP 首部 Accept**。常见策略包括请求头策略（`Accept: application/json`）、固定参数策略（`/getUser?format=json`）、扩展名策略（`/getUser.json`）、固定媒体类型策略（`produces` 属性）。

如果 `produces` 指定的类型和后缀、请求参数、`Accept` 冲突，将无法完成内容协商，返回 406。


## 十一、异步请求：Callable 与 DeferredResult

当 Controller 返回 `Callable` 或 `DeferredResult` 时，请求处理流程会发生变化。

**Callable 的处理流程**：Spring MVC 调用 `request.startAsync()`，将 `Callable` 提交到 `TaskExecutor` 在独立线程中处理。同时，`DispatcherServlet` 和所有 Filter 退出请求处理线程，但响应保持打开状态。`Callable` 产生结果后，Spring MVC 将请求重新分派回 Servlet 容器，再次调用 `DispatcherServlet`，通过从 `Callable` 获取的返回值恢复请求处理。

**DeferredResult 的处理流程**：Controller 返回一个 `DeferredResult` 并保存在内存队列中。Spring MVC 调用 `request.startAsync()`，`DispatcherServlet` 和 Filter 退出请求处理线程。应用从某个线程设置 `DeferredResult` 的结果后，Spring MVC 将请求重新分派回 Servlet 容器，`DispatcherServlet` 再次被调用，使用异步产生的返回值继续处理。

关键点在于：异步请求意味着同一个请求会两次进入 `DispatcherServlet`。第一次启动异步处理，第二次处理异步结果。调试时需要注意区分。


## 十二、响应写回与 Filter 的反向执行

无论同步还是异步，最终响应都会写入 `HttpServletResponse`。

对于 Tomcat：`DispatcherServlet` 完成处理后，Tomcat 的 `Response` 对象将响应头和响应体写入内部缓冲区。Worker 线程将缓冲区数据写回 Socket，线程归还线程池。数据经过内核协议栈逐层封装，发回客户端。

对于 WebFlux / Netty：请求处理的核心是 `DispatcherHandler`，它围绕前端控制器模式设计，提供了请求处理的共享算法。`DispatcherHandler` 的处理流程为：`HandlerMapping` 找到匹配的 Handler → 合适的 `HandlerAdapter` 执行 Handler 并返回 `HandlerResult` → `HandlerResultHandler` 处理结果，直接生成响应或渲染视图。WebFlux 中没有 Servlet 的 `HttpServletRequest`，也没有 `HandlerMethodReturnValueHandler`，而是使用 `ServerWebExchange` 和 `HandlerResultHandler`。

请求离开 `DispatcherServlet` 后，会反向穿过 FilterChain。Filter 的 `chain.doFilter()` 之后的代码此时执行。因此，Filter 是环绕式的，既能在请求进入时处理，也能在响应返回时处理。


## 十三、扩展点速查表

Spring MVC 提供了丰富的扩展点。理解它们的位置、注册方式和执行时机，比记住源码更重要。

| 扩展点 | 接口 / 注解 | 注册方式 | 作用 | 执行时机 |
|---|---|---|---|---|
| 过滤器 | `Filter` | `FilterRegistrationBean` | Servlet 级别处理 | DispatcherServlet 之前，响应后再次经过 |
| 拦截器 | `HandlerInterceptor` | `WebMvcConfigurer.addInterceptors` | Spring MVC 级别拦截 | preHandle → Controller → postHandle → afterCompletion |
| 类型转换 | `Converter` / `Formatter` | `WebMvcConfigurer.addFormatters` | 字符串与 Java 类型互转 | 参数解析阶段 |
| 请求体转换 | `HttpMessageConverter` | `WebMvcConfigurer.extendMessageConverters` | HTTP Body 与对象互转 | @RequestBody / @ResponseBody |
| 参数解析 | `HandlerMethodArgumentResolver` | `WebMvcConfigurer.addArgumentResolvers` | 自定义入参解析 | Controller 调用前 |
| 返回值处理 | `HandlerMethodReturnValueHandler` | `WebMvcConfigurer.addReturnValueHandlers` | 自定义返回值处理 | Controller 调用后 |
| 异常处理 | `@ExceptionHandler` / `HandlerExceptionResolver` | `@ControllerAdvice` | 统一异常处理 | 异常抛出后 |
| 视图解析 | `ViewResolver` | `@Bean` | 视图名解析为 View | 返回值处理后 |


## 十四、Debug 指南：错误码与断点位置

### 14.1 常见错误码定位

| 错误码 | 可能阶段 | 常见原因 |
|---|---|---|
| 404 | HandlerMapping | URL 不匹配、HTTP 方法不匹配、context-path 错误、路径匹配策略变更 |
| 400 | 参数解析 | 缺少参数、类型转换失败、JSON 格式错误、日期格式错误 |
| 405 | HandlerMapping | HTTP 方法不支持 |
| 415 | HttpMessageConverter | Content-Type 不支持 |
| 406 | ContentNegotiationManager | Accept 与 produces 不匹配 |
| 500 | Controller / 返回值处理 | 业务异常、序列化失败、无合适返回值处理器 |
| 502 | 网关 / Nginx | 后端服务未启动或连接被拒绝 |
| 504 | 网关 / Nginx | 后端处理超时 |

### 14.2 关键断点

在 IDEA 中调试时，可以在以下位置设置断点：

- `DispatcherServlet.doDispatch`
- `AbstractHandlerMapping.getHandler`
- `RequestMappingHandlerAdapter.invokeHandlerMethod`
- `InvocableHandlerMethod.getMethodArgumentValues`
- `HandlerMethodArgumentResolverComposite.getArgumentResolver`（观察解析器选择过程）
- `RequestResponseBodyMethodProcessor.readWithMessageConverters`
- `RequestResponseBodyMethodProcessor.handleReturnValue`
- `AbstractMessageConverterMethodProcessor.writeWithMessageConverters`
- `ExceptionHandlerExceptionResolver.doResolveHandlerMethodException`

对于高流量服务，建议使用条件断点，避免无关请求频繁中断调试。

### 14.3 有用日志配置

```properties
logging.level.org.springframework.web=DEBUG
logging.level.org.springframework.web.servlet.DispatcherServlet=DEBUG
spring.mvc.log-request-details=true
```


## 结语

回到最初的问题：用户点击按钮，前端发出请求，到数据返回，中间发生了什么？

浏览器发出 HTTP 请求，经过 DNS、TCP、TLS、网络设备、Ingress、Nginx，到达服务器网卡。内核协议栈将数据交给监听端口的 Tomcat。Tomcat 的 Acceptor 接收连接，Poller 检测可读事件，Worker 线程解析 HTTP 报文，封装成 `HttpServletRequest`。请求经过 FilterChain 到达 `DispatcherServlet`。

`DispatcherServlet.doDispatch()` 启动调度流程：`HandlerMapping` 找到 Controller 方法，`HandlerAdapter` 负责执行。执行前，`HandlerMethodArgumentResolver` 按顺序遍历解析器列表，找到合适的解析器，用 `Converter` 和 `HttpMessageConverter` 完成数据转换。拦截器的 `preHandle` 已经执行，`postHandle` 等待返回。Controller 执行完毕后，`HandlerMethodReturnValueHandler` 处理返回值，`HttpMessageConverter` 将对象序列化为 JSON。响应写回 `HttpServletResponse`，Tomcat 将数据写回 Socket，Filter 反向执行，数据经过网络原路返回浏览器。

这条链路上的每一个环节都可能出错。理解链路的意义不在于记住所有源码，而在于当错误发生时，能够迅速判断问题出在哪一层，从而将排查范围从“整个系统”缩小到“某个组件的某个阶段”。这才是掌握请求生命周期的真正价值。