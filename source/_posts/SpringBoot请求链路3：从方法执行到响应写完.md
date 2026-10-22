---
title: "SpringBoot 请求链路（三）：从方法执行到响应写完"
date: 2026-08-26 13:51:15
tags:
  - spring
  - tomcat
categories:
  - develop
---

## 引子：参数都对，为什么还是 500 / 406 / 空响应

方法签名对了，参数也齐了，`InvocableHandlerMethod.doInvoke()` 那一行执行——从这一刻起，出错的方式彻底变了：不再是 400，而是 500、415、406，或者状态码 200 但 body 一个字都没有。

**异常、返回值、异步、写回，这四件事常被当成"框架内部细节"，可线上事故恰恰密集地堆在这里。**

> 本篇接[第二篇](/2026/08/26/SpringBoot请求链路2：从-DispatcherServlet-到方法调用/)（参数已就绪、拦截器已放行），覆盖系列导读里的第十~十五节和结语：反射调用与异常解析、返回值处理器与内容协商、异步请求的三根线程、响应从 `OutputBuffer` 到 Socket 的最后一跳，最后收在扩展点速查表和 Debug 指南上。

> 本篇要回答的问题：
>
> - Controller 抛出的异常，最终是谁把它变成 500 或 400 的？优先级怎么排？
> - `@ResponseBody` 是怎么把对象变成 JSON 的？406 为什么出在返回值阶段？
> - 异步请求为什么会有三根线程？为什么 `ThreadLocal`、MDC、`SecurityContext` 会丢？
> - "返回了"为什么不等于"发出去了"？flush 到底什么时候发生？
> - 状态码 200 但 body 为空，最可能错在哪一层？

## 十、Controller 执行与异常解析

### 10.1 反射调用

参数解析完成后，`InvocableHandlerMethod` 通过反射调用目标方法。这一步本身没有魔法，但有两个连带影响：

- **方法体执行之前的所有失败，都发生在方法体之外。** 参数解析、`@RequestBody` 反序列化、校验失败，全都属于这一类。这正是引子里"断点没停"的原因——**不是请求没到，是方法根本没被调用**。
- **方法体抛出的异常会被 `InvocationTargetException` 包一层**，Spring 会拆包后再交给异常解析器，因此 `@ExceptionHandler` 拿到的仍然是原始异常。

### 10.2 异常解析器的优先级

`Controller` 抛出的异常（以及参数解析阶段抛出的异常）都会汇集到 `processDispatchResult()`，再由 `HandlerExceptionResolver` 按顺序尝试处理。默认三个解析器的顺序由 `DispatcherServlet.properties` 给出，`WebMvcConfigurationSupport` 的类级 javadoc 也描述了同样的次序：

| 顺序 | 解析器 | 负责 |
|:---|:---|:---|
| 1 | `ExceptionHandlerExceptionResolver` | `@ExceptionHandler`（本类内 + `@ControllerAdvice`） |
| 2 | `ResponseStatusExceptionResolver` | `@ResponseStatus` / `ResponseStatusException` |
| 3 | `DefaultHandlerExceptionResolver` | Spring MVC 标准异常（`ErrorResponse` 家族） |

`@ExceptionHandler` 排在最前面，所以**它有机会覆盖掉框架的默认行为**。这也是为什么"我在 `@ControllerAdvice` 里处理了 `MethodArgumentTypeMismatchException`，就不再返回 400 了"——因为第 1 个解析器就把它接走了，`DefaultHandlerExceptionResolver` 根本没轮到。

`@ExceptionHandler` 的查找范围有两层：**当前 Controller 类内的 `@ExceptionHandler` 优先于 `@ControllerAdvice`**。这个优先级经常被忽略——一个 Controller 里定义的局部处理器，会悄悄盖掉全局的。

### 10.3 回到引子：那行日志到底是谁打的

现在可以精确回答引子里的问题了。

`MissingServletRequestParameterException` 的继承链是：

```text
MissingServletRequestParameterException
  └─ MissingRequestValueException
      └─ ServletRequestBindingException implements ErrorResponse
          └─ ServletException
```

它实现了 `ErrorResponse`，所以 `DefaultHandlerExceptionResolver.doResolveException()` 会走这条路：

```java
// DefaultHandlerExceptionResolver.java（节选）
if (ex instanceof ErrorResponse errorResponse) {
    ModelAndView mav = null;
    ...
    else if (ex instanceof MissingServletRequestParameterException theEx) {
        mav = handleMissingServletRequestParameter(theEx, request, response, handler);
    }
    ...
    return (mav != null ? mav :
            handleErrorResponse(errorResponse, request, response, handler));
}
```

`handleMissingServletRequestParameter()` 返回 `null`，于是落到 `handleErrorResponse()`：

```java
// DefaultHandlerExceptionResolver.java（节选）
protected ModelAndView handleErrorResponse(ErrorResponse errorResponse,
        HttpServletRequest request, HttpServletResponse response, @Nullable Object handler) throws IOException {

    if (!response.isCommitted()) {
        HttpHeaders headers = errorResponse.getHeaders();
        headers.forEach((name, values) -> values.forEach(value -> response.addHeader(name, value)));

        int status = errorResponse.getStatusCode().value();
        String message = errorResponse.getBody().getDetail();
        if (message != null) {
            response.sendError(status, message);
        }
        else {
            response.sendError(status);
        }
    }
    ...
    return new ModelAndView();
}
```

**注意两个关键点：**

**① 它 `return new ModelAndView()`——一个非 null 的空对象。** 这就是日志被打印的开关。回到基类：

```java
// AbstractHandlerExceptionResolver.java:173-186（节选）
ModelAndView result = doResolveException(request, response, handler, ex);
if (result != null) {
    // Print debug message when warn logger is not enabled.
    if (logger.isDebugEnabled() && (this.warnLogger == null || !this.warnLogger.isWarnEnabled())) {
        logger.debug(buildLogMessage(ex, request) + (result.isEmpty() ? "" : " to " + result));
    }
    // Explicitly configured warn logger in logException method.
    logException(ex, request);
}
return result;
```

```java
// AbstractHandlerExceptionResolver.java:256-259
protected String buildLogMessage(Exception ex, HttpServletRequest request) {
    return "Resolved [" + LogFormatUtils.formatValue(ex, -1, true) + "]";
}
```

**`Resolved [...]` 这个格式就来自这里**，是三个默认解析器共享的基类方法。它被打印出来，意味着"这个异常**被成功解析了**"——而不是"请求失败了"。

**② 400 这个状态码不是"返回"的，是 `response.sendError()` 发出去的。** 这带来一个重要连带效果：`sendError()` 会触发容器的**错误派发（error dispatch）**，容器会用 `DispatcherType.ERROR` 再跑一次请求，最终落到 Spring Boot 的 `/error` 端点（`BasicErrorController`）。

所以你在浏览器里看到的那个 JSON 错误体，**往往不是你的代码产生的，也不是 `@ControllerAdvice` 产生的，而是 `BasicErrorController` 产生的**。这也解释了为什么 `@ControllerAdvice` 里没有处理某个异常，响应体却依然是规整的 JSON。

`sendError` 的这个机制还有两个实战后果：

- **响应一旦 `sendError`，之后写的 body 都会被丢弃**（响应已提交）。所以"我想在 `@ControllerAdvice` 里统一包装错误 JSON"，必须确保自己处理了该异常，别让它漏到 `DefaultHandlerExceptionResolver`。
- **`DispatcherType.ERROR` 的这次派发会再经过一遍 Filter** —— 如果你的日志 Filter 不支持 ERROR 派发，错误请求的日志就会缺失。

### 10.4 为什么 Filter 里的异常进不了 @ControllerAdvice

这是被问得最多的问题之一，答案在结构里：**`HandlerExceptionResolver` 是 `DispatcherServlet` 内部的机制。**

```text
Filter.doFilter()
   └─ chain.doFilter()          ← Filter 里抛异常，在这里飞出
        └─ DispatcherServlet.doService()
             └─ doDispatch()
                  └─ processDispatchResult()
                       └─ processHandlerException()   ← 异常解析器在这里，只处理它内部抛的异常
```

Filter 在 `DispatcherServlet` **外面**。它抛出的异常会沿着调用栈向上飞，穿出 `DispatcherServlet`，落到 Tomcat 的 `StandardHostValve` 手里，由容器决定是转发到错误页还是交给 `/error`。

所以：

| 异常抛出位置 | 能被 `@ControllerAdvice` 捕获吗 | 最终表现 |
|:---|:---|:---|
| Controller / 参数解析 / 返回值处理 | ✅ 能 | 由你的 `@ExceptionHandler` 决定 |
| Filter（`chain.doFilter` 之前或之后） | ❌ 不能 | 容器错误页 / `BasicErrorController` |
| Interceptor 的 `preHandle` | ✅ 能 | 它在 `doDispatch` 内部，属第 1 类 |

最后一行值得注意：**Interceptor 的异常能被 `@ControllerAdvice` 捕获，Filter 的不能**——尽管两者都常被叫作"拦截器"。区别只在于它们**在不在 `doDispatch` 内部**。

> 一个与之配套的实战建议：需要统一错误格式时，把"捕获 Filter 异常"这件事交给 Filter 自己（`try/catch` + 自己写响应），或者依赖 `/error` + `ErrorAttributes` 定制，而不要指望 `@ControllerAdvice`。

## 十一、返回值处理与内容协商

### 11.1 15 个内置返回值处理器

`Controller` 方法返回后，返回值交给 `HandlerMethodReturnValueHandler` 处理。清单在同一个 `RequestMappingHandlerAdapter` 里：

```java
// RequestMappingHandlerAdapter.java:738-778（节选）
private List<HandlerMethodReturnValueHandler> getDefaultReturnValueHandlers() {
    List<HandlerMethodReturnValueHandler> handlers = new ArrayList<>(20);

    // Single-purpose return value types
    handlers.add(new ModelAndViewMethodReturnValueHandler());
    handlers.add(new ModelMethodProcessor());
    handlers.add(new ViewMethodReturnValueHandler());
    ...
```

数字怎么来的：方法体里 17 处 `add`，减去 1 处 `addAll(getCustomReturnValueHandlers())`，再减去 1（因为最后是 `if/else` 互斥分支，只会加一个）——

**非 Kotlin、无自定义的情况下，共 15 个内置处理器。**

| # | 处理器 | 对应返回类型 |
|:---|:---|:---|
| 1 | `ModelAndViewMethodReturnValueHandler` | `ModelAndView` |
| 2 | `ModelMethodProcessor` | `Model` |
| 3 | `ViewMethodReturnValueHandler` | `View` |
| 4 | `ResponseBodyEmitterReturnValueHandler` | `ResponseBodyEmitter` / `SseEmitter` |
| 5 | `StreamingResponseBodyReturnValueHandler` | `StreamingResponseBody` |
| 6 | `HttpEntityMethodProcessor` | `HttpEntity` / `ResponseEntity` |
| 7 | `HttpHeadersReturnValueHandler` | `HttpHeaders` |
| 8 | `CallableMethodReturnValueHandler` | `Callable` |
| 9 | `DeferredResultMethodReturnValueHandler` | `DeferredResult` |
| 10 | `AsyncTaskMethodReturnValueHandler` | `WebAsyncTask` |
| 11 | `ServletModelAttributeMethodProcessor(false)` | `@ModelAttribute` |
| 12 | `RequestResponseBodyMethodProcessor` | **`@ResponseBody`** |
| 13 | `ViewNameMethodReturnValueHandler` | `String`（视图名） |
| 14 | `MapMethodProcessor` | `Map` |
| — | **自定义处理器插入点** | — |
| 15 | 兜底：`ModelAndViewResolverMethodReturnValueHandler` 或 `ServletModelAttributeMethodProcessor(true)` | — |

和参数解析器一样，**自定义处理器插在 14 个内置之后、兜底之前**：能覆盖兜底，覆盖不了前 14 个。

这张表还能解释一个常见疑惑：**返回 `String` 到底是"返回 JSON 字符串"还是"返回视图名"？** 关键不在 `String` 本身，而在类上有没有 `@ResponseBody`（或 `@RestController`）。有，则由第 12 个处理器接走，序列化成 JSON；没有，则由第 13 个处理器接走，当成视图名去解析模板。

### 11.2 @ResponseBody 的写出路径

```java
// RequestResponseBodyMethodProcessor.java:174-193（节选）
@Override
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
        ModelAndViewContainer mavContainer, NativeWebRequest webRequest)
        throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

    mavContainer.setRequestHandled(true);
    ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
    ServletServerHttpResponse outputMessage = createOutputMessage(webRequest);
    ...
    writeWithMessageConverters(returnValue, returnType, inputMessage, outputMessage);
}
```

`mavContainer.setRequestHandled(true)` 这一行很关键：**它告诉 `DispatcherServlet` "响应已经写完了，不要再去渲染视图"**。这就是 `@ResponseBody` 能"跳过视图解析"的机制——不是特判，而是打了个标记。

写出的媒体类型选择，先看有没有预设 `Content-Type`：有就直接用；没有才走内容协商求交集。

```java
// AbstractMessageConverterMethodProcessor.java:379-395（节选）
Set<MediaType> mediaTypes =
        (Set<MediaType>) request.getAttribute(HandlerMapping.PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);
if (!CollectionUtils.isEmpty(mediaTypes)) {
    return new ArrayList<>(mediaTypes);
}
Set<MediaType> result = new LinkedHashSet<>();
for (HttpMessageConverter<?> converter : this.messageConverters) {
    if (converter instanceof GenericHttpMessageConverter<?> ghmc && targetType != null) {
        if (ghmc.canWrite(targetType, valueClass, null)) {
            result.addAll(converter.getSupportedMediaTypes(valueClass));
        }
    }
    else if (converter.canWrite(valueClass, null)) {
        result.addAll(converter.getSupportedMediaTypes(valueClass));
    }
}
```

**`@RequestMapping(produces = ...)` 不是元数据，它以最高优先级终结探测。** 它被写进 `PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE`，一眼就被读走；只有这个属性为空时，才回退去问"哪些转换器能写这个返回类型"。

于是写出阶段的完整判定是：

```text
Accept 头（经内容协商）∩ producible（produces / 转换器能力）
    ├─ 交集为空  → 406 Not Acceptable
    └─ 有交集     → 按 specificity 排序，选具体度最高的
                      └─ 找不到能写的转换器 → 500 / 406
```

**读和写的依据是完全不对称的**，这一点必须分清：

| 方向 | 依据 | `produces` 参与吗 |
|:---|:---|:---|
| 读 `@RequestBody` | 请求的 `Content-Type` × `canRead` | ❌ 不参与 |
| 写 `@ResponseBody` | `Accept`（协商）× `produces` × `canWrite` | ✅ 高优先级 |

### 11.3 内容协商：6.x 的默认值必须纠正

关于内容协商，流传最广的一句话是"**后缀 > 请求参数 > Accept 头**"。**这句话在 Spring 6.x 的默认配置下不成立。**

先看事实。`build()` 里的顺序确实是这个骨架：

```java
// ContentNegotiationManagerFactoryBean.java:325-363（节选）
public ContentNegotiationManager build() {
    List<ContentNegotiationStrategy> strategies = new ArrayList<>();

    if (this.strategies != null) {
        strategies.addAll(this.strategies);
    }
    else {
        if (this.favorPathExtension) {          // ← 默认 false
            ...
            strategies.add(strategy);
        }
        if (this.favorParameter) {              // ← 默认 false
            ...
            strategies.add(strategy);
        }
        if (!this.ignoreAcceptHeader) {         // ← 默认 !false = true
            strategies.add(new HeaderContentNegotiationStrategy());
        }
        if (this.defaultNegotiationStrategy != null) {
            strategies.add(this.defaultNegotiationStrategy);
        }
    }
```

再看两个开关的默认值：

```java
// ContentNegotiationManagerFactoryBean.java:107-119（节选）
private boolean favorParameter = false;

private String parameterName = "format";

private boolean favorPathExtension = false;
...
private boolean ignoreAcceptHeader = false;
```

| 策略 | 默认是否生效 | 说明 |
|:---|:---|:---|
| 路径后缀（`/user.json`） | ❌ **不生效** | `favorPathExtension = false`，且该方法已 `@Deprecated (since 5.2.4)` |
| 请求参数（`?format=json`） | ❌ **不生效** | `favorParameter = false`（**注意：它本身没有标记 `@Deprecated`**） |
| `Accept` 头 | ✅ **生效** | `ignoreAcceptHeader = false` |

**结论：6.x 默认下，真正参与内容协商的只有 `Accept` 头一个策略。**

```java
// ContentNegotiationManager.java:126-135
public List<MediaType> resolveMediaTypes(NativeWebRequest request) throws HttpMediaTypeNotAcceptableException {
    for (ContentNegotiationStrategy strategy : this.strategies) {
        List<MediaType> mediaTypes = strategy.resolveMediaTypes(request);
        if (mediaTypes.equals(MEDIA_TYPE_ALL_LIST)) {
            continue;
        }
        return mediaTypes;
    }
    return MEDIA_TYPE_ALL_LIST;
}
```

首个返回有效结果的策略胜出——但**默认情况下列表里只有 Header 策略，前两个压根没进列表**。`WebMvcConfigurationSupport` 也从不调用 `setFavorPathExtension` / `setFavorParameter`，所以"默认开启"这个前提本身就不存在。

因此正确的表述是：

> `build()` 里"后缀 → 参数 → Accept"的书写顺序，**只是条件分支的先后，不代表默认生效顺序**。要让它成立，必须先显式打开 `favorPathExtension` / `favorParameter`。默认配置下，内容协商由 `Accept` 头单独决定。

**这个收紧是有安全原因的。** 路径后缀参与内容协商会打开 RFD（Remote File Download）攻击面，Spring 在 5.2.4 起就开始废弃相关配置，并在 6.0 连同后缀匹配一起收紧（见 7.5 节里那条"后缀匹配与 `PathPatternParser` 不能共存"的断言）。

### 11.4 406 与 500 分别从哪里来

| 错误码 | 触发点 | 常见原因 |
|:---|:---|:---|
| **406** | `writeWithMessageConverters` 里交集为空 | `Accept: application/xml` 但没有 XML 转换器；`produces` 与 `Accept` 冲突 |
| **415** | `readWithMessageConverters` 里 `canRead` 全不匹配 | `Content-Type` 不支持（缺失时会退化成 `application/octet-stream`） |
| **500** | 序列化过程抛异常 | 循环引用、懒加载触发额外查询、缺少 getter、`HttpMessageNotWritableException` |

**406 是最容易被误判的**：它看起来像服务端错误，其实是"客户端要的格式我没有"。一个典型场景是前端多带了一个 `Accept: text/html`，而接口只会返回 JSON——此时交集为空，直接 406。

### 11.5 顺带纠正：转换器清单有两个

还有一个容易混淆的点，涉及"为什么我明明加了 Gson，却还是 Jackson 在干活"：

```java
// WebMvcConfigurationSupport.java:910-922（节选）
if (jackson2Present) {
    Jackson2ObjectMapperBuilder builder = Jackson2ObjectMapperBuilder.json();
    if (this.applicationContext != null) {
        builder.applicationContext(this.applicationContext);
    }
    messageConverters.add(new MappingJackson2HttpMessageConverter(builder.build()));
}
else if (gsonPresent) {
    messageConverters.add(new GsonHttpMessageConverter());
}
else if (jsonbPresent) {
    messageConverters.add(new JsonbHttpMessageConverter());
}
```

**JSON 家族的三个转换器是互斥的 `if / else if` 链。** 只要 Jackson 在 classpath 上（Spring Boot Web 默认就在），`GsonHttpMessageConverter` 和 `JsonbHttpMessageConverter` **都不会被注册**。想换掉 Jackson，得先把它从依赖里排除。

同时要注意，**默认转换器清单其实有两份，用途不同**：

| 出处 | 内容 | 何时生效 |
|:---|:---|:---|
| `WebMvcConfigurationSupport.addDefaultHttpMessageConverters()` | 完整清单（Jackson、XML、Form、String…） | Spring Boot / `@EnableWebMvc` 的常规路径 |
| `RequestMappingHandlerAdapter.initMessageConverters()` | 只有 3 个（`ByteArray`、`String`、`Form`） | 独立使用该 adapter 时的兜底 |

而且第一份有个开关语义：**只要在 `configureMessageConverters()` 里手动加了任意一个转换器，整份默认清单就全部不生效。**

```java
// WebMvcConfigurationSupport.java:838-848（节选）
protected final List<HttpMessageConverter<?>> getMessageConverters() {
    if (this.messageConverters == null) {
        this.messageConverters = new ArrayList<>();
        configureMessageConverters(this.messageConverters);
        if (this.messageConverters.isEmpty()) {
            addDefaultHttpMessageConverters(this.messageConverters);
        }
        extendMessageConverters(this.messageConverters);
    }
    return this.messageConverters;
}
```

这条 `isEmpty()` 判断是"我加了个自定义转换器，结果 PDF 下载、字符串返回全坏了"的根因。**要追加而不是替换，用 `extendMessageConverters()`。**

## 十二、异步：一次请求，两次进入 DispatcherServlet

### 12.1 先说结论：一次请求，三个线程

同步请求全程只有一个 Worker 线程。**异步请求会拆成三段，落在三个不同的线程上：**

| 阶段 | 线程 | 名字 | 做什么 |
|:---|:---|:---|:---|
| **T1** | 连接器 Worker | `http-nio-8080-exec-N` | Filter → `DispatcherServlet` → 调用 Controller → 启动异步 → **退出** |
| **T2** | 业务执行线程 | `MvcAsyncN`（Spring MVC 默认）/ `task-N`（Spring Boot） | 执行 `Callable` / 业务逻辑 |
| **T3** | 连接器 Worker（**另一个**） | `http-nio-8080-exec-M` | 第二次进 `DispatcherServlet`，处理异步结果、写响应 |

**注意 T3 不是 T1。** T1 早就退出、归还线程池了；T3 是异步分派时从同一个池子里**重新取**的一根线程。所以"同一个请求"在日志里会出现两段毫不相干的线程名——这正是调试异步接口时最容易被绕晕的地方。

### 12.2 T1：异步是怎么启动的

Controller 返回 `Callable` 后，由 `CallableMethodReturnValueHandler` 接手：

```java
// CallableMethodReturnValueHandler.java:42-52
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
        ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

    if (returnValue == null) {
        mavContainer.setRequestHandled(true);
        return;
    }

    Callable<?> callable = (Callable<?>) returnValue;
    WebAsyncUtils.getAsyncManager(webRequest).startCallableProcessing(callable, mavContainer);
}
```

注意传进去的 `mavContainer`——它会被当成 `processingContext` 存起来，**第二遍要拿它恢复现场**。

`startCallableProcessing` 做三件事：置状态、提交任务、注册回调。其中真正调用 Servlet `startAsync()` 的是这一步：

```java
// WebAsyncManager.java:479-491（节选）
private void startAsyncProcessing(Object[] processingContext) {
    synchronized (WebAsyncManager.this) {
        this.concurrentResult = RESULT_NONE;
        this.concurrentResultContext = processingContext;
    }
    ...
    this.asyncWebRequest.startAsync();
}
```

往下追一层，才是 Servlet API 的真实调用：

```java
// StandardServletAsyncWebRequest.java:158-159
this.asyncContext = getRequest().startAsync(getRequest(), getResponse());
this.asyncContext.addListener(this);
```

同时，业务任务被提交到 executor：

```java
// WebAsyncManager.java:344-357（节选）
Future<?> future = this.taskExecutor.submit(() -> {
    Object result = null;
    try {
        interceptorChain.applyPreProcess(this.asyncWebRequest, callable);
        result = callable.call();
    }
    catch (Throwable ex) {
        result = ex;
    }
    finally {
        result = interceptorChain.applyPostProcess(this.asyncWebRequest, callable, result);
    }
    setConcurrentResultAndDispatch(result);
});
```

**T1 在这里就结束了。** `doDispatch` 在 `ha.handle()` 之后立刻返回：

```java
// DispatcherServlet.java:1089-1093
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

if (asyncManager.isConcurrentHandlingStarted()) {
    return;
}
```

这个 `return` 意味着：**第一遍不执行 `applyPostHandle`、也不执行 `processDispatchResult`**，而是走 `finally` 里的异步分支，调用 `applyAfterConcurrentHandlingStarted()`（见 6.2 节⑥）。

再往下，Tomcat 侧把 `SocketState.LONG` 返回给 `AbstractProtocol`，processor 被放进 `waitingProcessors` 等待，**T1 归还线程池**。Spring 在这里留了一条很直白的 debug 日志：

```java
// FrameworkServlet.java:1115
logger.debug("Exiting but response remains open for further handling");
```

> **"Exiting but response remains open"——这句话就是异步请求的全部要点。** 线程走了，但那个 HTTP 连接还开着，响应还没写。

### 12.3 T2 → T3：结果是怎么被"派回来"的

T2 执行完 `callable.call()`，拿到结果后调用：

```java
// WebAsyncManager.java:366-395（节选）
private void setConcurrentResultAndDispatch(@Nullable Object result) {
    Assert.state(this.asyncWebRequest != null, "AsyncWebRequest must not be null");
    synchronized (WebAsyncManager.this) {
        if (!this.state.compareAndSet(State.ASYNC_PROCESSING, State.RESULT_SET)) {
            ...
            return;
        }

        this.concurrentResult = result;
        ...
        if (this.asyncWebRequest.isAsyncComplete()) {
            ...
            return;
        }
        ...
        this.asyncWebRequest.dispatch();
    }
}
```

这里有两个设计值得注意：

- **状态用 CAS 迁移**（`NOT_STARTED → ASYNC_PROCESSING → RESULT_SET`）。这是幂等保证——超时回调和正常结果同时到达时，只有一个能赢。
- **最后一行 `dispatch()` 才是"派回来"的动作**，不是在 T1 提前安排的。

往下就是标准 Servlet 异步协议：

```text
StandardServletAsyncWebRequest.dispatch()
  └─ asyncContext.dispatch()                       StandardServletAsyncWebRequest.java:169
      └─ Tomcat AsyncContextImpl.dispatch()
          └─ coyoteRequest.action(ActionCode.ASYNC_DISPATCH, null)
              └─ AsyncStateMachine.asyncDispatch()
                  └─ processSocketEvent(SocketEvent.OPEN_READ, true)
                      └─ AbstractEndpoint.processSocket(...)
                          └─ executor.execute(sc)   ← 在 T2 上提交，在 T3 上执行
                              └─ 容器 Pipeline → DispatcherServlet.service() 第二遍
```

`DispatcherType.ASYNC` 这个标记，是在容器准备第二次派发时设置的：

```java
// ApplicationDispatcher.java:546（Tomcat 10.1.24）
wrequest.setAttribute(Globals.DISPATCHER_TYPE_ATTR, DispatcherType.ASYNC);
```

**有了这个标记，容器的行为会和第一次派发不同**——最直接的一点：Filter 会**按 `dispatcherTypes` 重新筛选**（见 5.1 节），只有声明支持 `ASYNC` 的 Filter 才会再跑一遍。这就是"为什么我的日志 Filter 在异步接口上打了两条日志"和"为什么有的 Filter 只打了一条"的根源。

### 12.4 第二遍：Controller 不会被再次调用

第二遍进入 `DispatcherServlet` 时，有一个反直觉的事实：

**`isConcurrentHandlingStarted()` 变成 `false` 了。**

原因是 `AsyncWebRequest` **每一遍都会新建一个**（并把状态从上一遍继承过来），新对象的 `asyncContext` 字段是 `null`，所以 `isConcurrentHandlingStarted()` 返回 `false`。

这意味着**第二遍走的是完整的同步流程**：`applyPostHandle` 会执行、`processDispatchResult` 会执行。第一遍被跳过的部分，在第二遍补上了。

那结果是怎么被复用的？关键在 `RequestMappingHandlerAdapter`：

```java
// RequestMappingHandlerAdapter.java:913-926（节选）
if (asyncManager.hasConcurrentResult()) {
    Object result = asyncManager.getConcurrentResult();
    Object[] resultContext = asyncManager.getConcurrentResultContext();
    Assert.state(resultContext != null && resultContext.length > 0, "Missing result context");
    mavContainer = (ModelAndViewContainer) resultContext[0];
    asyncManager.clearConcurrentResult();
    LogFormatUtils.traceDebug(logger, traceOn -> {
        String formatted = LogFormatUtils.formatValue(result, !traceOn);
        return "Resume with async result [" + formatted + "]";
    });
    invocableMethod = invocableMethod.wrapConcurrentResult(result);
}

invocableMethod.invokeAndHandle(webRequest, mavContainer);
```

**注意 `mavContainer` 是从第一遍存下来的 `processingContext` 里取回来的**——这就是 12.2 节里那个 `mavContainer` 参数的用途。

而 `wrapConcurrentResult()` 做的事非常巧妙：

```java
// ServletInvocableHandlerMethod.java:203-205
ServletInvocableHandlerMethod wrapConcurrentResult(@Nullable Object result) {
    return new ConcurrentResultHandlerMethod(result, new ConcurrentResultMethodParameter(result));
}
```

```java
// ServletInvocableHandlerMethod.java:214-227（节选）
private class ConcurrentResultHandlerMethod extends ServletInvocableHandlerMethod {

    public ConcurrentResultHandlerMethod(@Nullable Object result, ConcurrentResultMethodParameter returnType) {
        super((Callable<Object>) () -> {
            if (result instanceof Exception exception) {
                throw exception;
            }
            else if (result instanceof Throwable throwable) {
                throw new ServletException("Async processing failed: " + result, throwable);
            }
            return result;
        }, CALLABLE_METHOD);
        ...
    }
}
```

**它用"返回固定结果的假方法"替换了真正的 Controller 方法。** 于是第二遍"调用"的其实是一个直接 `return result` 的 `Callable`——Controller 方法体**不会被再次执行**。

但这一步的意义在于：**结果被送进了正常的返回值处理管线**。也就是说，异步返回的对象会像同步返回一样走 `HandlerMethodReturnValueHandler` → `HttpMessageConverter` → JSON。

而且如果 T2 里抛了异常，这个假方法会**原样抛出来**——于是 `@ExceptionHandler` 照常生效。**异步请求的异常处理能力，就是这么接上去的。**

### 12.5 DeferredResult 与 StreamingResponseBody 的差异

`DeferredResult` 的启动方式和 `Callable` 完全不同：

```java
// WebAsyncManager.java（startDeferredResultProcessing 节选）
this.asyncWebRequest.addTimeoutHandler(() -> {
    ...
    Object result = interceptorChain.triggerAfterTimeout(this.asyncWebRequest, callable);
    ...
});
```

它**不做任何线程提交**，只是把回调注册到 `DeferredResult` 上（`setResultHandler`）。因此：

- **启动时没有线程切换**——`DeferredResult` 由 Controller 在 T1 上创建并返回。
- **结果由哪个线程产生，就在哪个线程上触发分派。** 如果用 MQ 消费者调 `deferredResult.setResult(...)`，那触发 `dispatch()` 的就是 MQ 的业务线程，而不是任何容器线程。

这是一条重要的边界：**`DeferredResult` 不提供线程池，它只提供"回调 + 分派"。** 想控制并发，得自己管产生结果的那个线程池。

`StreamingResponseBody` 则是第三种形态——它**边算边写**：

```java
// StreamingResponseBodyReturnValueHandler.java:87-93（节选）
ShallowEtagHeaderFilter.disableContentCaching(request);
...
new StreamingResponseBodyTask(outputMessage.getBody(), streamingBody),
...
startCallableProcessing(callable, mavContainer);
```

关键在 `outputMessage.getBody()`：**它在 T1 上就被调用**，拿到的是 `response.getOutputStream()` 包装后的对象；而 `writeTo` 在 T2 上执行。于是——

> **`StreamingResponseBody` 的 socket 写入发生在 T2 这个业务线程上，而不是容器线程上。**

这也解释了为什么它能"边生成边下发"：数据直接从业务线程写向连接，不必等回到容器线程。代价是**业务线程会一直阻塞在写上**——客户端下载慢，就会把一个业务线程占住不放。

### 12.6 异步场景的线程名与那个默认 executor

Spring MVC 默认的异步 executor 是这个：

```java
// RequestMappingHandlerAdapter.java:172
private AsyncTaskExecutor taskExecutor = new MvcSimpleAsyncTaskExecutor();
```

```java
// RequestMappingHandlerAdapter.java:1057-1063（节选）
private class MvcSimpleAsyncTaskExecutor extends SimpleAsyncTaskExecutor {

    private static boolean taskExecutorWarning = true;

    MvcSimpleAsyncTaskExecutor() {
        super("MvcAsync");
    }
```

所以默认线程名前缀是 `MvcAsync`，也就是 `MvcAsync1`、`MvcAsync2`……**但 Spring 自己对这个默认值非常不满意**，它甚至内置了一条警告日志：

```java
// RequestMappingHandlerAdapter.java:1067-1076（节选）
logger.warn("""
        !!!
        Performing asynchronous handling through the default Spring MVC SimpleAsyncTaskExecutor.
        This executor is not suitable for production use under load.
        Please, configure an AsyncTaskExecutor through the WebMvc config.
        -------------------------------
        !!!""");
```

**Spring 源码原话：这个 executor 不适合生产环境负载。** 原因在 `SimpleAsyncTaskExecutor` 的实现里：

```java
// SimpleAsyncTaskExecutor.java:311-313
protected void doExecute(Runnable task) {
    newThread(task).start();
}
```

**每个任务新建一根线程，没有任何上限。** 高并发下异步接口一多，线程数就会失控——这是"配置了异步反而更容易 OOM"的经典原因。

所以正确的做法是显式配置 executor：

```java
@Configuration
public class AsyncConfig implements WebMvcConfigurer {

    @Override
    public void configureAsyncSupport(AsyncSupportConfigurer configurer) {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(16);
        executor.setMaxPoolSize(64);
        executor.setQueueCapacity(200);
        executor.setThreadNamePrefix("mvc-async-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        configurer.setTaskExecutor(executor);
        configurer.setDefaultTimeout(30_000);
    }
}
```

**Spring Boot 会自动帮你做这件事**：

```java
// WebMvcAutoConfiguration.java:229-241（节选）
public void configureAsyncSupport(AsyncSupportConfigurer configurer) {
    if (this.beanFactory.containsBean(TaskExecutionAutoConfiguration.APPLICATION_TASK_EXECUTOR_BEAN_NAME)) {
        Object taskExecutor = this.beanFactory
            .getBean(TaskExecutionAutoConfiguration.APPLICATION_TASK_EXECUTOR_BEAN_NAME);
        if (taskExecutor instanceof AsyncTaskExecutor asyncTaskExecutor) {
            configurer.setTaskExecutor(asyncTaskExecutor);
        }
    }
    ...
}
```

`applicationTaskExecutor` 这个 Bean 的线程名前缀默认是 `task-`：

```java
// TaskExecutionProperties.java:43
private String threadNamePrefix = "task-";
```

**所以日志里看到什么线程名，可以直接反推环境：**

| 线程名 | 含义 |
|:---|:---|
| `MvcAsync1` | Spring MVC 默认 executor，**生产环境应视为告警**（会打上面那条 warn） |
| `task-1` | Spring Boot 的 `applicationTaskExecutor`（较正常） |
| `mvc-async-1` | 自己配了 executor |
| `http-nio-8080-exec-N` | 容器 Worker（T1 或 T3） |

### 12.7 异步场景最容易出问题的四件事

**① `ThreadLocal` 不会跟着跨线程。** `MDC`、`RequestContextHolder`、自定义的 `ThreadLocal` 全都在 T1 上，T2 里读不到。要么在提交前手动传递，要么用 `TaskDecorator`。这是"异步接口里日志没有 traceId"的根因。

**② 线程池会把这个污染传给下一个请求。** 6.2 节⑥ 讲的那个 `MDC` 泄漏，真正的危害不在"这次请求没清掉"，而在**它是留在容器线程上的**。T3 是从同一个 Worker 池里取的线程，下一个被分配到这根线程的请求，就会读到上一个人的 `requestId`。**泄漏点在第一遍，受害者在下一个请求**——这种 bug 之所以难查，就是因为它跨了请求边界。

**③ Filter 是否支持 `ASYNC` 决定了它跑几遍。** 不支持 `ASYNC` 派发的 Filter 只跑一次（T1），支持的两遍都跑。这会让日志、埋点、鉴权出现"有时两条有时一条"的诡异现象。

**④ 默认超时是容器的。** `DeferredResult` / `Callable` 的超时最终交给容器处理，超时后同样走一次派发，结果是 `AsyncRequestTimeoutException`。必要时用 `configurer.setDefaultTimeout(...)` 收紧，别让请求悬在那里。

## 十三、响应写回：从 OutputBuffer 到 Socket

### 13.1 响应不是"返回时就发出去的"

一个符合直觉但错误的想象是：Controller 返回对象、序列化成 JSON、然后立刻写进 socket。真实情况是——**写进的是内存缓冲，socket 写入要等到请求收尾。**

三层缓冲，恰好都是 8KB 默认值，但**来源完全不同**：

| 层 | 类与字段 | 默认大小 | 出处 |
|:---|:---|:---|:---|
| 响应头序列化缓冲 | `Http11OutputBuffer.headerBuffer` | **8KB** | 大小取自 `maxHttpHeaderSize`，`AbstractHttp11Protocol.java:263` |
| 响应体缓冲 | `catalina.connector.OutputBuffer.bb` | **8KB** | `OutputBuffer.java:53` `DEFAULT_BUFFER_SIZE = 8 * 1024` |
| socket 写缓冲 | `SocketProperties.appWriteBufSize` | **8192** | `SocketProperties.java:96` |

写操作先落到 `OutputBuffer` 的 `ByteBuffer` 里，写满了才会真正往下走：

```text
OutputBuffer.append(...)                    写满 → flushByteBuffer()      OutputBuffer.java:667-674
  └─ flushByteBuffer()
      └─ realWriteBytes(...)                                             OutputBuffer.java:319-345
          └─ coyoteResponse.doWrite(...)                                 Response.java:621-625
              └─ Http11OutputBuffer.doWrite(...)                         未 commit 则先 COMMIT
                  └─ SocketOutputBuffer.doWrite(...)  socketWrapper.write(isBlocking(), chunk)
                      └─ SocketWrapperBase.flush(true)  → 真正写进 channel
```

**所以一个几百字节的 JSON 响应，在 Controller 返回之后仍然停在内存里。** 直到请求收尾：

```java
// CoyoteAdapter.service 末尾（节选）
if (!request.isAsync()) {
    request.finishRequest();
    response.finishResponse();
}
```

`finishResponse()` → `OutputBuffer.close()` → `doFlush(false)`，把剩余数据刷出去并提交响应头；随后 `endRequest()` → `outputBuffer.end()` → `socketWrapper.flush(true)`，才把连接级写缓冲真正写进 channel。

**整条链都在同一个容器 Worker 线程上完成**（同步场景下就是 T1）。

### 13.2 flush 的触发条件

`OutputBuffer` 里有一个 `doFlush` 标记，它就是"hint 机制"的本体：

| 触发 | 效果 |
|:---|:---|
| 缓冲写满 | 自动 flush |
| `flush()` 被调用 | `doFlush(true)` → 立即 flush |
| `doFlush` 标记为 `true` 期间 | 任何一次 `writeBytes` 都立即 flush（不再攒） |
| 请求收尾 | `close()` → `doFlush(false)` |

这就是为什么**大响应的第一块字节和最后一块字节的时机完全不同**：小响应一次性在收尾时发出；超过 8KB 的响应会在写满时提前发出若干次。压测时看到的"首字节延迟"差异，很多时候是这个缓冲在起作用。

`@ResponseBody` 返回一个巨大列表时，序列化是由 Jackson **流式写入** `OutputStream` 的——所以一个 100MB 的响应不会被整体缓存在 8KB 缓冲里，而是被切成多次写出去。**内存不会因为响应体本身爆掉**，但慢客户端会让 Worker 线程长期阻塞在写上（这一点和 `StreamingResponseBody` 是同一个问题）。

### 13.3 keep-alive：连接复用了，线程不保证复用

响应写完后，如果连接还要复用：

- `maxKeepAliveRequests` 默认 **100**（`AbstractEndpoint.java:859`），每处理一个请求自减一次；
- processor 被释放，socket 重新注册读事件，连接**交回 Poller** 等下一个请求。

**关键推论：同一个 TCP 连接上的两个请求，很可能由不同的 Worker 线程处理。**

Poller 只负责"等数据可读"，然后从 Worker 池里取一根当前空闲的线程。所以日志里出现"同一个连接的不同请求线程名不一样"是完全正常的——**不能用线程名来判断是不是同一个客户端**。

### 13.4 响应阶段的 Filter 反向执行

响应写回过程中，会**倒序**穿过 Filter 链：

```text
请求进入：F1 → F2 → F3 → DispatcherServlet
响应返回：         F3 → F2 → F1
```

`chain.doFilter()` 之后的代码此时才执行。这也是为什么 Filter 是最适合做"耗时统计"的位置——它天然包住了整个请求过程。

> 但这里有一个必须避开的坑：**如果在 Filter 里把响应包装成了 `ContentCachingResponseWrapper`，一定要记得把内容写回去。** 忘了调用 `copyBodyToResponse()`，就是"状态码 200、响应头齐全、body 却是 0 字节"。这个案例的完整排查过程见 [一次下载失败排查：被 Filter 吞掉的异步响应流](/2026/10/14/一次下载失败排查：被 Filter 吞掉的异步响应流/)。

### 13.5 对照：WebFlux 为什么完全不是这套东西

如果这一篇讲的是 WebFlux，第十三节会是空白的——因为 **Netty 没有 `OutputBuffer`，也没有 `DispatcherServlet`**。

| | Spring MVC（Tomcat） | Spring WebFlux（Netty） |
|:---|:---|:---|
| 入口 | `DispatcherServlet` | `DispatcherHandler` |
| 请求对象 | `HttpServletRequest` | `ServerWebExchange` |
| 返回值处理 | `HandlerMethodReturnValueHandler` | `HandlerResultHandler` |
| 线程模型 | 一请求一线程（Worker 池） | 事件循环，少量线程复用 |
| 响应写回 | `OutputBuffer` + 收尾 flush | `Channel.writeAndFlush`，非阻塞 |

`DispatcherHandler` 同样遵循"前端控制器"模式（`HandlerMapping` → `HandlerAdapter` → 结果处理），但**没有 `HttpServletRequest`，也没有 `HandlerMethodReturnValueHandler`**。这套差异值得单独讲，见 [深入理解 Spring WebFlux 与 Reactor Netty](/2026/09/15/深入理解-Spring-WebFlux-与-Reactor-Netty/) 和 [深入 WebFlux 与 Spring Cloud Gateway](/2026/09/21/深入-WebFlux-与-Spring-Cloud-Gateway：为什么网关没有-Controller，却是一个-WebFlux-应用？/)。

## 十四、扩展点速查表

这张表按"执行位置"排列，并补上了前文反复强调的**执行线程**——这一列往往比接口名更能决定你的代码能不能正常工作。

| 扩展点 | 接口 / 注解 | 注册方式 | 执行时机 | 执行线程 |
|:---|:---|:---|:---|:---|
| 过滤器 | `Filter` | `FilterRegistrationBean` | `DispatcherServlet` 之前/之后 | T1（异步时 T3 可能再来一次） |
| 拦截器 | `HandlerInterceptor` | `WebMvcConfigurer.addInterceptors` | preHandle → Controller → postHandle → afterCompletion | T1；异步时 `afterConcurrentHandlingStarted` |
| 类型转换 | `Converter` / `Formatter` | `WebMvcConfigurer.addFormatters` | 参数解析阶段 | T1（或 T3） |
| 请求体转换 | `HttpMessageConverter` | `extendMessageConverters`（**追加用这个**） | `@RequestBody` / `@ResponseBody` | T1（或 T3） |
| 参数解析 | `HandlerMethodArgumentResolver` | `addArgumentResolvers` | Controller 调用前 | T1（或 T3） |
| 返回值处理 | `HandlerMethodReturnValueHandler` | `addReturnValueHandlers` | Controller 调用后 | T1（异步时 T3 上处理结果） |
| 异常处理 | `@ExceptionHandler` / `HandlerExceptionResolver` | `@ControllerAdvice` | 异常抛出后 | 抛出异常的线程 |
| 视图解析 | `ViewResolver` | `@Bean` | 返回值处理后 | T1（或 T3） |
| 异步支持 | `WebMvcConfigurer.configureAsyncSupport` | `@Configuration` | 异步执行 | 决定 T2 用哪个线程池 |

两个高频踩坑点，都在表里：

- **`HttpMessageConverter` 要用 `extendMessageConverters` 追加，别用 `configureMessageConverters` 覆盖**——后者的 `isEmpty()` 判断会让整份默认清单失效（见 11.5 节）。
- **`addArgumentResolvers` 里传入的组件要来自容器**，不要 `new`——否则它内部的 `@Autowired`、`@Transactional`、AOP 全部失效（见第二篇 8.11 节）。

## 十五、Debug 指南

这一节的表是跨三篇的：`2.x`、`3.x` 在第一篇，`7.x`、`8.x` 在第二篇，`10`~`13.x` 在本篇。节号连续编号，直接按数字翻就行。

### 15.1 错误码 → 哪一层

| 错误码 | 所在层 | 常见原因 | 对应节 |
|:---|:---|:---|:---|
| 404 | `HandlerMapping` | URL 不匹配、尾斜杠（`/user` vs `/user/`）、HTTP 方法不符、`params` / `headers` 条件未满足、被更靠前的映射吃掉 | 7.3 / 7.5 |
| 405 | `HandlerMapping` | 路径匹配了但 HTTP 方法不对 | 7.4 |
| 400 | 参数解析 | 缺少必填参数、类型转换失败、JSON 格式错误、`-parameters` 未开启 | 8 |
| 415 | `HttpMessageConverter` | 请求 `Content-Type` 不支持（缺失时退化为 `application/octet-stream`） | 8.9 |
| 406 | 内容协商 / 返回值处理 | `Accept` 与 `produces` 交集为空 | 11.2 / 11.4 |
| 500 | Controller / 序列化 | 业务异常未处理、循环引用、懒加载、缺少 getter | 10 / 11.4 |
| 502 | 网关 / Nginx | 后端未启动或拒绝连接 | 2.2 |
| 504 | 网关 / Nginx | 后端处理超时 | 3.2 |
| 200 但 body 为空 | Filter 响应包装 | 忘了写回缓存内容 | 13.4 |

### 15.2 按现象定位到线程

先看线程名，这一步几乎不花时间，却能把范围砍掉一半：

| 现象 | 先怀疑 | 怎么确认 |
|:---|:---|:---|
| 断点没停，但有错误日志 | **方法没被执行**（参数解析失败、Filter 短路） | 看日志有没有 `Resolved [...]`：有 → 已进 MVC；没有 → 更外层 |
| 日志线程名是 `-Poller` | 连接层异常 | 检查端口、连接数与内核队列 |
| `-exec-*` 大量堆积且 CPU 不高 | 业务阻塞 | `jstack` 数一下 exec 线程数量和状态 |
| 日志里出现 `MvcAsync` | 默认异步 executor 生效 | 去日志里找那条 `!!!` warn |
| 同一请求出现两个不同的 `-exec-N` | 异步请求 | 对照 12.1 节的三线程表 |
| 日志里 traceId 串了 | MDC 污染 | 检查异步接口是否漏了 `afterConcurrentHandlingStarted` |

### 15.3 关键断点

按"从外到内"的顺序，命中率最高的一批：

| 位置 | 看什么 |
|:---|:---|
| `DispatcherServlet.doDispatch` | 请求是否真的进了 MVC |
| `DispatcherServlet.getHandler` | 路由到了哪个 `HandlerMapping` |
| `AbstractHandlerMethodMapping.lookupHandlerMethod` | 路由表里有没有匹配项 |
| `RequestMappingHandlerAdapter.invokeHandlerMethod` | 第二遍是否复用了并发结果 |
| `InvocableHandlerMethod.getMethodArgumentValues` | 参数解析的真实取值过程 |
| `HandlerMethodArgumentResolverComposite.getArgumentResolver` | **哪个解析器被选中**（排查 400 的最佳位置） |
| `AbstractMessageConverterMethodArgumentResolver.readWithMessageConverters` | `Content-Type` 与转换器匹配 |
| `AbstractMessageConverterMethodProcessor.writeWithMessageConverters` | 内容协商与写出协商（排查 406） |
| `HandlerExecutionChain.applyPreHandle` | 拦截器顺序与中断点 |
| `ExceptionHandlerExceptionResolver.doResolveHandlerMethodException` | 异常最终被谁处理 |

**排查 400 建议直接上 `getArgumentResolver`**：它能一眼告诉你这个参数被哪个解析器认领了——往往在打断点之前，看参数上的注解就能得出同样的结论。

高流量服务记得用**条件断点**（限定 URI 或参数值），否则调试请求本身就会把服务拖慢。

### 15.4 有用的日志配置

```properties
# 请求处理全过程
logging.level.org.springframework.web.servlet.DispatcherServlet=DEBUG

# 异常解析器（Resolved [...] 就在这里）
logging.level.org.springframework.web.servlet.mvc=DEBUG

# 请求参数细节（含敏感信息，仅本地/预发开启）
spring.mvc.log-request-details=true

# 关键：异步 executor 的告警必须能看到
logging.level.org.springframework.web.servlet.mvc.method.annotation=WARN
```

`spring.mvc.log-request-details` 会打印请求体，**生产环境务必关闭**。

## 结语

回到[系列开头那个引子](/2026/08/26/SpringBoot的一次完整请求发生了什么？/)。那行 `Resolved [MissingServletRequestParameterException]` 之所以把人骗到 Nginx 和网关上去，是因为它看起来像"请求出了问题"，而实际上它在说"**请求已经成功走到第二篇的第八节了**"。

现在，系列导读里那张问题清单可以逐条回答了：

- **一次请求换了几个线程？** 同步场景一个（Worker），全程不变；异步场景三个——T1 启动、T2 执行、T3 收尾，而且 T3 不是 T1。
- **Acceptor / Poller / Worker 各管什么？** Acceptor 收连接、Poller 等数据、Worker 干活；在现代 Tomcat 里前两者**各只有一个**，`pollerThreadCount` 这类配置已经不存在了。
- **这些组件什么时候装好的？** 启动期：`ServletWebServerFactoryConfiguration` 选容器，`onRefresh()` 启动 Tomcat 并监听端口，`DispatcherServlet` 注册为映射到 `/` 的 Servlet，MVC 组件在 `initStrategies()` 里被收集。
- **Filter 和 Interceptor 差在哪？** 差在"在不在 `doDispatch` 内部"。Filter 在外，拿不到 Handler，异常也进不了 `@ControllerAdvice`；Interceptor 在内，两样都有。
- **为什么参数解析失败时方法体一行都不执行？** 因为参数解析发生在方法调用之前，失败就没走到反射调用那一步。
- **`@RequestParam` 和 `@RequestBody` 是同一个"参数"吗？** 不是。前者从 query string / 表单按名字取值，后者把整个请求体反序列化。注解不同，取值位置就不同。
- **异步为什么两次进 `DispatcherServlet`？** 第一次启动异步并退出（响应保持打开），结果就绪后 `dispatch()` 把请求派回容器，第二次用 `wrapConcurrentResult` 把结果送进正常的返回值管线——Controller 不会被重复调用。
- **各错误码指向哪一层？** 那就是第十五节那张表。

这条链路上真正的难点，从来不是记住有多少个类，而是知道**每一个环节分别在哪一层、跑在哪根线程上、失败时表现为哪种错误**。把这三件事对应起来，排查就不再靠猜——你会先看日志的措辞和线程名，而不是先打开 IDE。

> **延伸阅读**
>
> - [Spring 接入点：跟着容器过一生](/2026/09/11/Spring-接入点：跟着容器过一生/)——第一篇第四节讲的是"装好了什么"，那篇讲的是"容器在哪个阶段把它们装好"。
> - [一次下载失败排查：被 Filter 吞掉的异步响应流](/2026/10/14/一次下载失败排查：被 Filter 吞掉的异步响应流/)——第十三节那个坑的完整排查过程。
> - [Spring MVC 中那些"不走寻常路"的返回类型](/2026/09/14/Spring-MVC-中那些“不走寻常路”的返回类型/)——第十一节那张返回值处理器表的展开版。
> - [深入理解 Spring WebFlux 与 Reactor Netty](/2026/09/15/深入理解-Spring-WebFlux-与-Reactor-Netty/)——把第一篇的 Tomcat 模型和 EventLoop 模型对照着看。

> 这一篇是系列的最后一篇。三篇的入口、以及那张全链路地图，都在[系列导读](/2026/08/26/SpringBoot的一次完整请求发生了什么？/)。

