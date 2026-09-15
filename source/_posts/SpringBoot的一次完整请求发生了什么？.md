---
title: SpringBoot的一次完整请求发生了什么？
date: 2026-08-26 13:51:15
tags:
  - spring
  - tomcat
categories:
  - develop
---

# 从点击按钮到 JSON 返回：一次 Spring Boot 请求的完整生命周期

## 引言

前端点了一下按钮，后端返回了一个 JSON。这中间发生了什么？

大多数时候，你不需要关心这个问题。但总有一些时刻，你会被逼着去搞清楚。线上突然出现一批 400，前端说参数明明传了；接口偶尔 500，日志里只有一行 `Handler dispatch failed`；Nginx 报 502，但服务明明活着。

这些问题的答案，藏在请求的完整链路里。

这篇文章会从浏览器点击按钮开始，一路走到 JSON 返回，把中间经过的每一个关键环节都拆开来看。我们不追求逐行解读源码，但会保证关键环节的技术准确性，并在每个阶段给出可操作的代码示例和 Debug 线索。

先看一张全局地图：

```text
浏览器
  │  fetch / XHR / axios
  ▼
操作系统协议栈（DNS → TCP → TLS → HTTP）
  ▼
物理网络（网卡 → 交换机 → 路由器 → 公网）
  ▼
服务器入口（负载均衡 / K8s Ingress / Nginx / API 网关）
  ▼
服务器内核（网卡 → 协议栈 → Socket → 监听端口）
  ▼
Tomcat / Netty
  │  Acceptor → Poller → Worker
  │  解析 HTTP → 封装 HttpServletRequest
  ▼
FilterChain
  │  编码、安全、跨域、日志
  ▼
DispatcherServlet
  │  HandlerMapping → HandlerAdapter → Interceptor
  │  ArgumentResolver → Converter / HttpMessageConverter
  ▼
Controller
  │  你的业务代码
  ▼
DispatcherServlet
  │  ReturnValueHandler → HttpMessageConverter
  ▼
Tomcat / Netty
  │  写回 HttpServletResponse → Socket
  ▼
浏览器
```

下面，我们从最外层开始，逐层深入。


## 一、网络层：请求如何到达服务器

### 1.1 封装与解封装

当浏览器发出一个 HTTP 请求时，数据并不是直接“飞”到服务器的。它需要在操作系统的网络协议栈中经历一次逐层封装的过程。

在发送端，HTTP 报文在应用层被构造出来，然后向下传递。传输层加上 TCP 头，包含源端口、目标端口、序号等字段，形成 TCP 段。网络层加上 IP 头，包含源 IP、目标 IP、TTL 等字段，形成 IP 包。数据链路层加以太网头，包含源 MAC、目标 MAC，形成以太网帧。最后在物理层变成电信号或光信号发送出去。

接收端的处理顺序完全相反。网卡收到信号后，通过 DMA 写入内存，内核触发中断，协议栈开始逐层拆解。物理层还原比特流，数据链路层检查 MAC 地址，网络层检查 IP 地址，传输层根据目标端口找到正在监听的 Socket。最终，内核把连接放入对应 Socket 的接收队列，监听该端口的进程被唤醒。

这个过程的核心意义在于：你的 Spring Boot 服务能收到请求，前提是内核已经完成了所有底层的拆包工作。如果请求根本没到达服务，问题一定出在这一层之前的某个环节。

### 1.2 中间基础设施与常见故障

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


## 二、容器层：Tomcat 如何接住请求

### 2.1 Spring Boot 启动内嵌 Tomcat 的过程

当项目引入 `spring-boot-starter-web` 时，Spring Boot 会在启动过程中自动配置一个内嵌 Tomcat。

核心自动配置类是 `ServletWebServerFactoryAutoConfiguration`。它根据 classpath 决定创建哪种容器工厂。存在 `spring-boot-starter-web` 时，创建 `TomcatServletWebServerFactory`。在容器刷新阶段，Spring Boot 调用该工厂的 `getWebServer()` 方法，创建并启动 `TomcatWebServer`。

与此同时，`DispatcherServletAutoConfiguration` 创建 `DispatcherServlet` 并注册为一个 Servlet，通常映射到 `/`。`WebMvcAutoConfiguration` 创建 Spring MVC 所需的 `HandlerMapping`、`HandlerAdapter`、`HandlerExceptionResolver` 等组件。

需要明确的是：Spring Boot 并没有“取代”Servlet 容器，而是将 Tomcat 作为内嵌容器启动。`DispatcherServlet` 本质上仍然是一个运行在 Tomcat 中的 Servlet。

### 2.2 Tomcat 的 NIO 线程模型

Tomcat 默认使用 NIO 连接器，其线程模型包含三类关键线程。

**Acceptor 线程** 阻塞在 `ServerSocket.accept()` 上，负责接收新的 TCP 连接。收到连接后，将 Socket 包装成 `NioSocketWrapper` 并注册到 Poller。

**Poller 线程** 使用 `Selector` 轮询 IO 事件。当某个 Socket 有数据可读时，Poller 为每个就绪的 `NioSocketWrapper` 生成一个 `SocketProcessor` 任务，扔进 Executor 线程池去处理。

**Worker 线程池** 负责运行 `SocketProcessor` 任务。在 Worker 线程中，会完成从 Socket 中读取 HTTP 请求、解析成 `HttpServletRequest` 对象、分派到相应的 Servlet 并完成逻辑，然后将响应通过 Socket 发回客户端。线程名通常为 `http-nio-8080-exec-*`，默认最大线程数为 200。

一个请求在 Tomcat 内部的核心路径如下：

```text
Acceptor 接收连接
  → Poller 检测到可读
  → Worker 线程通过 Http11Processor 解析 HTTP 报文
  → CoyoteAdapter 将 Tomcat 内部 Request 适配为 HttpServletRequest
  → FilterChain
  → DispatcherServlet
```

其中，`CoyoteAdapter` 是 Tomcat 内部对象与 Servlet 规范对象之间的桥梁。你的 Controller 能拿到 `HttpServletRequest`，正是因为 Tomcat 已经完成了原始字节流到规范对象的转换。


## 三、Filter 层：在 DispatcherServlet 之前

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

多个 Filter 的执行顺序是：请求阶段按注册顺序正序执行，响应阶段倒序执行。Filter A 先于 Filter B 注册时：

```text
FilterA.doFilter() 前半段
  → FilterB.doFilter() 前半段
    → DispatcherServlet
  → FilterB.doFilter() 后半段
→ FilterA.doFilter() 后半段
```

**Filter 与 Interceptor 的区别**：Filter 属于 Servlet 规范，在 `DispatcherServlet` 之前执行，不能直接获取 Spring Bean，也无法获取到即将执行的 Handler 信息。Interceptor 属于 Spring MVC，在 `HandlerMapping` 之后执行，可以获取 Handler 信息。


## 四、DispatcherServlet：整个流程的总调度中心

请求穿过 FilterChain 后，到达 `DispatcherServlet`。它是 Spring MVC 的前端控制器，继承自 `HttpServlet`，核心方法是 `doDispatch()`。

`doDispatch()` 实现了将请求分发到具体 Handler、执行拦截器的 `preHandle` 方法、调用 Controller 处理具体逻辑、执行拦截器的 `postHandle` 方法、处理返回的 `ModelAndView` 或异常、执行拦截器的 `afterCompletion` 方法。

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
    } catch (Throwable err) {
        dispatchException = new NestedServletException("Handler dispatch failed", err);
    }

    // 8. 处理结果：渲染视图或处理异常
    processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
}
```

下面按阶段拆解。


## 五、HandlerMapping：路由匹配

`getHandler(request)` 遍历所有 `HandlerMapping`，找到匹配的 Handler。

最常用的是 `RequestMappingHandlerMapping`。它在启动时扫描所有 `@Controller`，对标注了 `@Controller` 或 `@RequestMapping` 注解的类中方法进行遍历，将类和方法上的 `@RequestMapping` 注解值进行合并，封装成 `RequestMappingInfo`，与 `HandlerMethod` 建立映射关系。请求到来时，根据 URL 路径、HTTP 方法、请求参数、请求头等条件匹配到唯一的 `HandlerMethod`。

匹配成功后返回 `HandlerExecutionChain`，包含目标 `HandlerMethod` 和 `HandlerInterceptor` 列表。

**常见错误**：

- **404 Not Found**：没有匹配到 Handler。检查 URL 是否正确、`context-path` 是否配置、HTTP 方法是否匹配、网关是否转发正确。
- **405 Method Not Allowed**：路径匹配了，但 HTTP 方法不匹配。例如 `@GetMapping` 收到了 POST 请求。


## 六、HandlerAdapter 与参数解析

### 6.1 HandlerAdapter 的作用

`HandlerMapping` 返回的是 Handler，但 `DispatcherServlet` 并不知道如何执行它。`HandlerAdapter` 负责屏蔽不同类型 Handler 的执行差异。

对于 `@RequestMapping` 方法，使用 `RequestMappingHandlerAdapter`。它在初始化时会准备 `HandlerMethodArgumentResolver`（解析方法参数）和 `HandlerMethodReturnValueHandler`（处理返回值），然后调用 `invokeAndHandle()`。

### 6.2 参数解析：HandlerMethodArgumentResolver

`InvocableHandlerMethod.getMethodArgumentValues()` 遍历方法参数，对每个参数找到对应的 `HandlerMethodArgumentResolver`。Spring MVC 会按顺序遍历所有已注册的解析器，调用 `supportsParameter()` 判断是否支持当前参数，找到第一个支持的解析器后调用 `resolveArgument()`。

常见的解析器如下：

| 注解 / 类型 | 解析器 |
|---|---|
| `@RequestParam` | `RequestParamMethodArgumentResolver` |
| `@PathVariable` | `PathVariableMethodArgumentResolver` |
| `@RequestHeader` | `RequestHeaderMethodArgumentResolver` |
| `@RequestBody` | `RequestResponseBodyMethodProcessor` |
| `@ModelAttribute` | `ServletModelAttributeMethodProcessor` |
| `HttpServletRequest` | `ServletRequestMethodArgumentResolver` |

### 6.3 Converter 与 HttpMessageConverter 的分工

这是两个容易混淆的概念，必须严格区分。

**Converter** 用于参数绑定阶段的类型转换。当 `@RequestParam` 或 `@PathVariable` 从请求中提取出字符串后，需要将其转换为方法参数声明的 Java 类型。例如：

```java
@GetMapping("/user")
public User getUser(@RequestParam("id") Long id) { ... }
```

请求 `?id=123` 中的 `"123"` 是字符串，方法参数是 `Long`。Spring 通过 `ConversionService` 完成转换。`ConversionService` 内部维护了一系列 `Converter`，如 `StringToNumberConverterFactory`、`StringToBooleanConverter` 等。

Converter 是通用的类型转换接口，可以在任意 Object 之间转换。Formatter 专门用于字符串与目标类型之间的双向转换，通常用于 Web 场景。两者的区别在于：Converter 不依赖 Locale，适合结构固定的场景；Formatter 支持 locale 感知，适合日期、货币等格式化类型。

自定义 Converter 示例——将 `"name,age"` 格式的字符串转为 `UserQuery` 对象：

```java
public class StringToUserQueryConverter implements Converter<String, UserQuery> {
    @Override
    public UserQuery convert(String source) {
        if (source == null || source.trim().isEmpty()) {
            return null;
        }
        String[] parts = source.split(",");
        return new UserQuery(parts[0].trim(), Integer.parseInt(parts[1].trim()));
    }
}
```

注册方式——通过 `WebMvcConfigurer.addFormatters()` 注册：

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addFormatters(FormatterRegistry registry) {
        registry.addConverter(new StringToUserQueryConverter());
    }
}
```

这样，`?q=Tom,18` 就能自动转成 `UserQuery` 对象：

```java
@GetMapping("/search")
public List<User> search(@RequestParam("q") UserQuery query) {
    // query.getName() = "Tom", query.getAge() = 18
    return userService.search(query);
}
```

**HttpMessageConverter** 用于请求体和响应体的序列化与反序列化。当使用 `@RequestBody` 或 `@ResponseBody` 时，Spring 不会使用普通的 Converter，而是选择合适的 `HttpMessageConverter`。最常见的 `MappingJackson2HttpMessageConverter` 内部持有 `ObjectMapper`，负责 JSON 与 Java 对象之间的转换。

两者的分工可以总结为：Converter 处理“单个参数值”的类型转换，HttpMessageConverter 处理“整个请求体或响应体”的序列化与反序列化。

### 6.4 自定义参数解析器

当内置的解析器无法满足需求时，可以实现自定义的 `HandlerMethodArgumentResolver`。一个常见的场景是获取客户端 IP。

首先定义注解：

```java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface ClientIp {
}
```

然后实现解析器：

```java
public class ClientIpArgumentResolver implements HandlerMethodArgumentResolver {

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(ClientIp.class);
    }

    @Override
    public Object resolveArgument(MethodParameter parameter,
                                  ModelAndViewContainer mavContainer,
                                  NativeWebRequest webRequest,
                                  WebDataBinderFactory binderFactory) {
        HttpServletRequest request = webRequest.getNativeRequest(HttpServletRequest.class);
        if (request == null) {
            return null;
        }
        String ip = request.getHeader("X-Forwarded-For");
        if (ip == null || ip.isEmpty()) {
            ip = request.getRemoteAddr();
        }
        return ip;
    }
}
```

注册到 Spring MVC：

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        resolvers.add(new ClientIpArgumentResolver());
    }
}
```

在 Controller 中使用：

```java
@GetMapping("/hello")
public String hello(@ClientIp String clientIp) {
    return "Your IP: " + clientIp;
}
```

### 6.5 参数解析阶段的常见错误

- **400 Bad Request**：缺少必填参数（`MissingServletRequestParameterException`）、类型转换失败（`MethodArgumentTypeMismatchException`）、JSON 格式错误（`HttpMessageNotReadableException`）。
- **415 Unsupported Media Type**：`Content-Type` 不支持。例如 `@RequestBody` 期望 `application/json`，但请求发送的是 `text/plain`。

调试 400 时，可以开启以下配置，Spring 会打印请求参数、请求头等详细信息：

```properties
spring.mvc.log-request-details=true
```

生产环境应关闭此选项，避免敏感信息泄露。


## 七、Interceptor：拦截器的执行顺序与中断逻辑

在 Controller 方法执行前后，`HandlerInterceptor` 提供了三个切入时机：

- `preHandle()`：Controller 执行前调用。
- `postHandle()`：Controller 执行后、视图渲染前调用。
- `afterCompletion()`：整个请求完成后调用，用于资源清理。

**执行顺序**：当多个拦截器同时存在时，`preHandle` 按注册顺序正序执行，`postHandle` 和 `afterCompletion` 按注册顺序倒序执行。假设拦截器 A 先于 B 注册：

```text
preHandle(A) → preHandle(B) → Controller
→ postHandle(B) → postHandle(A)
→ afterCompletion(B) → afterCompletion(A)
```

这种设计符合栈的语义：先进入的拦截器后退出。

**中断逻辑**：如果某个拦截器的 `preHandle` 返回 `false`，请求立即中断，Controller 不会执行。返回 `false` 的拦截器之前的拦截器的 `afterCompletion` 会被执行，但 `postHandle` 不会执行。

**Filter 与 Interceptor 的对比**：

| 维度 | Filter | Interceptor |
|---|---|---|
| 规范 | Servlet 规范 | Spring MVC |
| 位置 | DispatcherServlet 之前 | HandlerMapping 之后 |
| 能否拿到 Handler | 不能 | 能 |
| 环绕式执行 | 是（doFilter 前后） | 是（preHandle / postHandle） |
| 典型用途 | 编码、安全、跨域 | 鉴权、日志、业务拦截 |

需要注意的是，Filter 中抛出的异常不会被 `@ControllerAdvice` 捕获。因为 Filter 执行在 `DispatcherServlet` 之前，属于 Servlet 规范范畴。


## 八、Controller 执行与异常处理

参数解析完成后，Spring 通过反射调用 Controller 方法。此时请求正式进入业务代码。

如果 Controller 抛出异常，`DispatcherServlet` 在 `processDispatchResult()` 中会调用 `processHandlerException()`，将异常交给 `HandlerExceptionResolver` 处理。

异常处理的优先级从高到低：

1. **`@Controller` + `@ExceptionHandler`**：本类内的异常处理器，优先级最高。
2. **`@ControllerAdvice` + `@ExceptionHandler`**：全局异常处理器。
3. **`@ResponseStatus` / `ResponseStatusException`**：基于注解或异常类型的状态码。
4. **`DefaultHandlerExceptionResolver`**：处理 Spring MVC 标准异常。

全局异常处理器示例：

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResult> handleBusinessException(BusinessException ex) {
        ErrorResult result = new ErrorResult(ex.getCode(), ex.getMessage());
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(result);
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

如果没有任何异常处理器能处理，最终返回 500。


## 九、返回值处理：HandlerMethodReturnValueHandler

Controller 方法执行完毕后，返回值会交给 `HandlerMethodReturnValueHandler` 处理。

`HandlerMethodReturnValueHandler` 是 Spring MVC 中负责处理 Controller 返回值的策略接口。Spring MVC 支持非常多的返回值类型，比如 Map、ViewName、Callable、异步的 StreamingResponseBody 等，每一种都有其对应的处理器。

接口定义如下：

```java
public interface HandlerMethodReturnValueHandler {

    boolean supportsReturnType(MethodParameter returnType);

    void handleReturnValue(Object returnValue, MethodParameter returnType,
                          ModelAndViewContainer mavContainer,
                          NativeWebRequest webRequest) throws Exception;
}
```

`RequestMappingHandlerAdapter` 内部维护了一组该接口的实现。处理返回值时，它会按顺序遍历这些实现，找到第一个 `supportsReturnType()` 返回 `true` 的处理器，然后调用其 `handleReturnValue()` 方法。

常见的返回值处理器：

| 返回值类型 | 处理器 |
|---|---|
| `@ResponseBody` | `RequestResponseBodyMethodProcessor` |
| `ResponseEntity` | `HttpEntityMethodProcessor` |
| `String` 视图名 | `ViewNameMethodReturnValueHandler` |
| `ModelAndView` | `ModelAndViewMethodReturnValueHandler` |
| `Map` | `MapMethodProcessor` |
| `Callable` | `CallableMethodReturnValueHandler` |
| `DeferredResult` | `DeferredResultMethodReturnValueHandler` |

### 9.1 @ResponseBody 的处理流程

对于 `@ResponseBody` 或 `@RestController` 标注的方法，`RequestResponseBodyMethodProcessor` 会接管返回值处理。其内部调用 `writeWithMessageConverters()`，根据 `Accept` 头、返回值类型、`produces` 条件，选择合适的 `HttpMessageConverter`，通常为 `MappingJackson2HttpMessageConverter`。随后调用 `write()` 将对象序列化为 JSON，写入 `HttpServletResponse` 的 `OutputStream`，并设置 `mavContainer.setRequestHandled(true)` 表示请求已处理完毕。

整个序列化链路的完整路径是：

```text
DispatcherServlet → HandlerAdapter → RequestResponseBodyMethodProcessor
→ MappingJackson2HttpMessageConverter → ObjectMapper → JsonGenerator
→ 写入 HttpOutputMessage 响应流
```

**常见错误**：

- **406 Not Acceptable**：`Accept` 头要求的类型没有对应的 `HttpMessageConverter`。
- **500 Internal Server Error**：Jackson 序列化失败。常见原因包括循环引用、懒加载对象在序列化时触发额外查询、缺少 getter 方法等。
- **415 Unsupported Media Type**：请求体反序列化时 `Content-Type` 不支持。

### 9.2 自定义 HandlerMethodReturnValueHandler

当需要统一处理某种返回值类型时，可以实现自定义的 `HandlerMethodReturnValueHandler`。

假设你定义了一个统一返回结构 `ApiResult`：

```java
public class ApiResult<T> {
    private int code;
    private String message;
    private T data;
    // getter / setter / 构造方法省略
}
```

自定义返回值处理器：

```java
public class ApiResultReturnValueHandler implements HandlerMethodReturnValueHandler {

    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public boolean supportsReturnType(MethodParameter returnType) {
        return ApiResult.class.isAssignableFrom(returnType.getParameterType());
    }

    @Override
    public void handleReturnValue(Object returnValue, MethodParameter returnType,
                                  ModelAndViewContainer mavContainer,
                                  NativeWebRequest webRequest) throws Exception {
        mavContainer.setRequestHandled(true);

        HttpServletResponse response = webRequest.getNativeResponse(HttpServletResponse.class);
        response.setContentType("application/json;charset=UTF-8");
        response.setStatus(HttpServletResponse.SC_OK);
        response.getWriter().write(objectMapper.writeValueAsString(returnValue));
    }
}
```

注册：

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addReturnValueHandlers(List<HandlerMethodReturnValueHandler> handlers) {
        handlers.add(0, new ApiResultReturnValueHandler());
    }
}
```

注意注册时使用 `handlers.add(0, ...)` 将自定义处理器放在列表首位，确保它在 `RequestResponseBodyMethodProcessor` 之前被检查到。如果放在列表末尾，对于带有 `@ResponseBody` 的返回值可能永远不会被选中。


## 十、异步请求：Callable 与 DeferredResult

当 Controller 返回 `Callable` 或 `DeferredResult` 时，请求处理流程会发生变化。

**Callable 的处理流程**：Spring MVC 调用 `request.startAsync()`，将 `Callable` 提交到 `TaskExecutor` 在独立线程中处理。同时，`DispatcherServlet` 和所有 Filter 退出请求处理线程，但响应保持打开状态。`Callable` 产生结果后，Spring MVC 将请求重新分派回 Servlet 容器，再次调用 `DispatcherServlet`，通过从 `Callable` 获取的返回值恢复请求处理。

```java
@GetMapping("/async")
public Callable<String> asyncHello() {
    return () -> {
        Thread.sleep(2000);  // 模拟耗时操作
        return "Hello from async thread: " + Thread.currentThread().getName();
    };
}
```

**DeferredResult 的处理流程**：Controller 返回一个 `DeferredResult` 并保存在内存队列中。Spring MVC 调用 `request.startAsync()`，`DispatcherServlet` 和 Filter 退出请求处理线程。应用从某个线程设置 `DeferredResult` 的结果后，Spring MVC 将请求重新分派回 Servlet 容器，`DispatcherServlet` 再次被调用，使用异步产生的返回值继续处理。

```java
@GetMapping("/deferred")
public DeferredResult<String> deferredHello() {
    DeferredResult<String> result = new DeferredResult<>(5000L);

    // 模拟从其他线程设置结果
    new Thread(() -> {
        try {
            Thread.sleep(1000);
            result.setResult("Hello from DeferredResult");
        } catch (InterruptedException e) {
            result.setErrorResult(e);
        }
    }).start();

    return result;
}
```

关键点在于：异步请求意味着同一个请求会两次进入 `DispatcherServlet`。第一次启动异步处理，第二次处理异步结果。调试时需要注意区分。


## 十一、响应写回与 Filter 的反向执行

无论同步还是异步，最终响应都会写入 `HttpServletResponse`。

对于 Tomcat：`DispatcherServlet` 完成处理后，Tomcat 的 `Response` 对象将响应头和响应体写入内部缓冲区。Worker 线程将缓冲区数据写回 Socket，线程归还线程池。数据经过内核协议栈逐层封装，发回客户端。

对于 WebFlux / Netty：请求处理的核心是 `DispatcherHandler`，它围绕前端控制器模式设计，提供了请求处理的共享算法，实际工作由可配置的委托组件执行。`DispatcherHandler` 的处理流程为：`HandlerMapping` 找到匹配的 Handler → 合适的 `HandlerAdapter` 执行 Handler 并返回 `HandlerResult` → `HandlerResultHandler` 处理结果，直接生成响应或渲染视图。WebFlux 中没有 Servlet 的 `HttpServletRequest`，也没有 `HandlerMethodReturnValueHandler`，而是使用 `ServerWebExchange` 和 `HandlerResultHandler`。

请求离开 `DispatcherServlet` 后，会反向穿过 FilterChain。Filter 的 `chain.doFilter()` 之后的代码此时执行。因此，Filter 是环绕式的，既能在请求进入时处理，也能在响应返回时处理。


## 十二、扩展点速查表

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


## 十三、Debug 指南：错误码与断点位置

### 13.1 常见错误码定位

| 错误码 | 可能阶段 | 常见原因 |
|---|---|---|
| 404 | HandlerMapping | URL 不匹配、HTTP 方法不匹配、context-path 错误、网关转发错误 |
| 400 | 参数解析 | 缺少参数、类型转换失败、JSON 格式错误、日期格式错误 |
| 405 | HandlerMapping | HTTP 方法不支持 |
| 415 | HttpMessageConverter | Content-Type 不支持 |
| 406 | HttpMessageConverter | Accept 与 produces 不匹配 |
| 500 | Controller / 返回值处理 | 业务异常、序列化失败、无合适返回值处理器 |
| 502 | 网关 / Nginx | 后端服务未启动或连接被拒绝 |
| 504 | 网关 / Nginx | 后端处理超时 |

### 13.2 关键断点

在 IDEA 中调试时，可以在以下位置设置断点：

- `DispatcherServlet.doDispatch`
- `AbstractHandlerMapping.getHandler`
- `RequestMappingHandlerAdapter.invokeHandlerMethod`
- `InvocableHandlerMethod.getMethodArgumentValues`
- `RequestResponseBodyMethodProcessor.readWithMessageConverters`
- `RequestResponseBodyMethodProcessor.handleReturnValue`
- `AbstractMessageConverterMethodProcessor.writeWithMessageConverters`
- `ExceptionHandlerExceptionResolver.doResolveHandlerMethodException`

对于高流量服务，建议使用条件断点，避免无关请求频繁中断调试。

### 13.3 有用日志配置

```properties
logging.level.org.springframework.web=DEBUG
logging.level.org.springframework.web.servlet.DispatcherServlet=DEBUG
spring.mvc.log-request-details=true
```


## 结语

回到最初的问题：用户点击按钮，前端发出请求，到数据返回，中间发生了什么？

浏览器发出 HTTP 请求，经过 DNS、TCP、TLS、网络设备、Ingress、Nginx，到达服务器网卡。内核协议栈将数据交给监听端口的 Tomcat。Tomcat 的 Acceptor 接收连接，Poller 检测可读事件，Worker 线程解析 HTTP 报文，封装成 `HttpServletRequest`。请求经过 FilterChain 到达 `DispatcherServlet`。

`DispatcherServlet.doDispatch()` 启动调度流程：`HandlerMapping` 找到 Controller 方法，`HandlerAdapter` 负责执行。执行前，`HandlerMethodArgumentResolver` 解析参数，`Converter` 和 `HttpMessageConverter` 完成数据转换。拦截器的 `preHandle` 已经执行，`postHandle` 等待返回。Controller 执行完毕后，`HandlerMethodReturnValueHandler` 处理返回值，`HttpMessageConverter` 将对象序列化为 JSON。响应写回 `HttpServletResponse`，Tomcat 将数据写回 Socket，Filter 反向执行，数据经过网络原路返回浏览器。

这条链路上的每一个环节都可能出错。理解链路的意义不在于记住所有源码，而在于当错误发生时，能够迅速判断问题出在哪一层，从而将排查范围从“整个系统”缩小到“某个组件的某个阶段”。这才是掌握请求生命周期的真正价值。