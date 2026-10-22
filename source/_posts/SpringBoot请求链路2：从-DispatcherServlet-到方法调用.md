---
title: "SpringBoot 请求链路（二）：从 DispatcherServlet 到方法调用"
date: 2026-08-26 13:51:17
tags:
  - spring
  - tomcat
categories:
  - develop
---

## 引子：所有分岔都发生在一个方法里

`DispatcherServlet` 是个 `HttpServlet`，`service()` → `doGet()` 那一串都是规范规定的样板代码。真正开始干活只有一行：`doDispatch(request, response)`。

**你后来遇到的大部分现象——404、405、400、406，以及"拦截器没执行""参数解析器选了别人"——都是这一个方法内部某一步的结果。** 它值得逐行读，而不是背五个步骤的名字。

> 本篇接[第一篇](/2026/08/26/SpringBoot请求链路1：请求是怎么到-Tomcat-的/)（`FilterChain` 已经放行），覆盖系列导读里的第六~九节：`doDispatch()` 的完整流程、`HandlerMapping` 如何把 URL 变成一个 `HandlerMethod`、27 个参数解析器怎么排队、拦截器的顺序与中断语义。

> 本篇要回答的问题：
>
> - `doDispatch()` 按什么顺序做事？哪一步失败是 404，哪一步是 400？
> - 一个 URL 是怎么匹配到方法的？为什么线上生效的路由顺序和配置文件里写的顺序不一样？
> - 27 个内置参数解析器的先后顺序是怎么定的？自己注册的为什么会被"抢走"？
> - `@RequestParam` 和 `@RequestBody` 取的到底是不是同一个"参数"？
> - `preHandle` 返回 `false` 之后，为什么 `afterCompletion` 还会被调用？
> - 为什么 404 和 304 的请求，一个拦截器回调都不会触发？

## 六、DispatcherServlet：把 doDispatch 完整读一遍

请求穿过滤镜链后，落到 `DispatcherServlet`。它是 Spring MVC 的前端控制器，继承自 `HttpServlet`，核心方法是 `doDispatch()`。

这个方法是整个系列的枢纽——**前面所有铺垫是为了走到这里，后面所有细节都是从这里分出去的**。所以它值得逐行读。

### 6.1 6.1.8 的真实源码

```java
// DispatcherServlet.java:1049-1129（Spring Framework 6.1.8，逐字）
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    boolean multipartRequestParsed = false;

    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

    try {
        ModelAndView mv = null;
        Exception dispatchException = null;

        try {
            processedRequest = checkMultipart(request);
            multipartRequestParsed = (processedRequest != request);

            // Determine handler for the current request.
            mappedHandler = getHandler(processedRequest);
            if (mappedHandler == null) {
                noHandlerFound(processedRequest, response);
                return;
            }

            // Determine handler adapter for the current request.
            HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

            // Process last-modified header, if supported by the handler.
            String method = request.getMethod();
            boolean isGet = HttpMethod.GET.matches(method);
            if (isGet || HttpMethod.HEAD.matches(method)) {
                long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
                if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
                    return;
                }
            }

            if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                return;
            }

            // Actually invoke the handler.
            mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

            if (asyncManager.isConcurrentHandlingStarted()) {
                return;
            }

            applyDefaultViewName(processedRequest, mv);
            mappedHandler.applyPostHandle(processedRequest, response, mv);
        }
        catch (Exception ex) {
            dispatchException = ex;
        }
        catch (Throwable err) {
            dispatchException = new ServletException("Handler dispatch failed: " + err, err);
        }
        processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
    }
    catch (Exception ex) {
        triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
    }
    catch (Throwable err) {
        triggerAfterCompletion(processedRequest, response, mappedHandler,
                new ServletException("Handler processing failed: " + err, err));
    }
    finally {
        if (asyncManager.isConcurrentHandlingStarted()) {
            // Instead of postHandle and afterCompletion
            if (mappedHandler != null) {
                mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
            }
        }
        else {
            // Clean up any resources used by a multipart request.
            if (multipartRequestParsed) {
                cleanupMultipart(processedRequest);
            }
        }
    }
}
```

### 6.2 几个容易被看漏的地方

**① 三个方法级变量，是为了 `finally` 里还能用。**

`processedRequest`、`mappedHandler`、`multipartRequestParsed` 声明在最外层 `try` 之外；而 `mv` 和 `dispatchException` 在内层。这不是风格问题——**外层 `catch` 拿不到 `mv`**，它只负责 `triggerAfterCompletion`，不负责渲染。

**② 304 的短路发生在拦截器之前。**

```java
if (isGet || HttpMethod.HEAD.matches(method)) {
    long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
    if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
        return;
    }
}
```

浏览器带 `If-Modified-Since` 且资源未变时，直接返回 304 并 `return`。**此时 `preHandle` 和 `afterCompletion` 都不会执行。** 如果你的拦截器里做了"统计所有请求耗时"，304 的请求是统计不到的——这类差异在压测数据对不齐时很致命。

**③ 404 分支不会触发任何 `afterCompletion`。**

`mappedHandler == null` 时调用 `noHandlerFound()` 然后 `return`。**chain 根本不存在，自然没有拦截器可回调。** 所以"为什么 404 请求没有触发我的 `afterCompletion` 日志"这个问题，答案是：它连拦截器链都没建立。

`noHandlerFound()` 的默认行为是 `response.sendError(404)`，只有把 `throwExceptionIfNoHandlerFound` 打开，才会抛 `NoHandlerFoundException`（从而能被 `@ControllerAdvice` 捕获）。

**④ 异常被"收敛"而不是抛出。**

```java
catch (Exception ex) {
    dispatchException = ex;
}
```

`Controller` 抛的异常不会直接冲出 `doDispatch`，而是被存进 `dispatchException`，交给统一的 `processDispatchResult()` 处理。**这是 `@ExceptionHandler` 能生效的结构前提**：异常必须在同一个方法里被接住，才有机会转交给 `HandlerExceptionResolver`。

**⑤ `Throwable` 也被接住，但包装方式变了。**

```java
catch (Throwable err) {
    dispatchException = new ServletException("Handler dispatch failed: " + err, err);
}
```

**这是 5.3 → 6.x 的一个真实差异**：Spring 5.3 用的是 `NestedServletException("Handler dispatch failed", err)`，6.x 改成 `ServletException`，消息里还多了 `": " + err`。`NestedServletException` 类本身**仍然存在**，只是标了 `@Deprecated(since = "6.0")`。

这意味着：如果你的监控或告警规则里靠匹配 `NestedServletException` 来识别异常，升级到 6.x 之后这条规则会静默失效。

**⑥ `finally` 里的异步分支——最容易被整段忽略的一段。**

```java
finally {
    if (asyncManager.isConcurrentHandlingStarted()) {
        // Instead of postHandle and afterCompletion
        if (mappedHandler != null) {
            mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
        }
    }
    else {
        if (multipartRequestParsed) {
            cleanupMultipart(processedRequest);
        }
    }
}
```

异步请求启动后，`doDispatch` 会在 `ha.handle()` 之后提前 `return`，**`postHandle` 和 `afterCompletion` 都不会执行**，取而代之的是 `applyAfterConcurrentHandlingStarted()`：

```java
// HandlerExecutionChain.java
void applyAfterConcurrentHandlingStarted(HttpServletRequest request, HttpServletResponse response) {
    for (int i = this.interceptorList.size() - 1; i >= 0; i--) {
        HandlerInterceptor interceptor = this.interceptorList.get(i);
        if (interceptor instanceof AsyncHandlerInterceptor asyncInterceptor) {
            try {
                asyncInterceptor.afterConcurrentHandlingStarted(request, response, this.handler);
            }
            catch (Throwable ex) {
                ...
            }
        }
    }
}
```

它**逆序**调用每个实现了 `AsyncHandlerInterceptor` 的拦截器。官方 javadoc 说得很直白：`Called instead of postHandle and afterCompletion when the handler is being executed concurrently`，典型用途是**清理 `ThreadLocal`**。

> 这就是一个经典线上事故的根因：日志 Filter / 拦截器在 `preHandle` 里 `MDC.put(requestId)`，本来指望 `afterCompletion` 里清掉。一旦某个接口是异步的，`afterCompletion` 不执行，**`requestId` 就留在了那个 Worker 线程上**——下一个复用到这个线程的请求，日志里会带上别人的 trace id。
>
> 修法不是"记得清 MDC"，而是实现 `AsyncHandlerInterceptor`，在 `afterConcurrentHandlingStarted` 里也清一次。

### 6.3 出口：processDispatchResult

```java
// DispatcherServlet.java:1178-1186
if (WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
    // Concurrent handling started during a forward
    return;
}

if (mappedHandler != null) {
    // Exception (if any) is already handled..
    mappedHandler.triggerAfterCompletion(request, response, null);
}
```

所有正常结束的请求，最终都在这里以 `ex = null` 调用 `triggerAfterCompletion`。注意那个 `null`——**正常返回和"被拦截器中断"在 `afterCompletion` 里看起来是一样的**，这一点在第九节还会用到。

## 七、路由：HandlerMapping 找到了什么

### 7.1 getHandler 只是顺序遍历

```java
// DispatcherServlet.java:1280-1290
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    if (this.handlerMappings != null) {
        for (HandlerMapping mapping : this.handlerMappings) {
            HandlerExecutionChain handler = mapping.getHandler(request);
            if (handler != null) {
                return handler;
            }
        }
    }
    return null;
}
```

命中第一个非 null 就返回。所以 **`handlerMappings` 的顺序直接决定了路由结果**——某个宽泛的映射排在前面，后面的映射就永远没机会执行。

### 7.2 真实顺序不是配置文件里的顺序

`DispatcherServlet.properties` 里写着：

```properties
org.springframework.web.servlet.HandlerMapping=\
	org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping,\
	org.springframework.web.servlet.function.support.RouterFunctionMapping
```

但**这只是"容器里一个 `HandlerMapping` Bean 都没有"时的兜底**。真实应用（有 Spring Boot 或 `@EnableWebMvc`）走的是排序：

```java
// DispatcherServlet.java:626-628
this.handlerMappings = new ArrayList<>(matchingBeans.values());
// We keep HandlerMappings in sorted order.
AnnotationAwareOrderComparator.sort(this.handlerMappings);
```

各 `HandlerMapping` 的真实 order 值：

| HandlerMapping | order | 出处 |
|:---|:---|:---|
| `RouterFunctionMapping` | **-1** | `WebMvcConfigurationSupport.java:555`（注释：`go before RequestMappingHandlerMapping`） |
| `RequestMappingHandlerMapping` | **0** | `WebMvcConfigurationSupport.java:309` |
| `viewControllerHandlerMapping` | **1** | `WebMvcConfigurationSupport.java:477` |
| `BeanNameUrlHandlerMapping` | **2** | `WebMvcConfigurationSupport.java:532` |
| `resourceHandlerMapping`（`SimpleUrlHandlerMapping`） | `LOWEST_PRECEDENCE - 1` | `ResourceHandlerRegistry.java:77` |
| `defaultServletHandlerMapping` | `Integer.MAX_VALUE` | `WebMvcConfigurationSupport.java:614` |

**这张表的实用价值在于最后两行：静态资源永远排在 `@RequestMapping` 之后。**

于是可以解释一个非常常见的问题："为什么我的静态资源 404 了？"——只要你写了一个能匹配到该路径的 `@Controller`（比如 `@GetMapping("/**")` 之类过度宽泛的映射），`ResourceHttpRequestHandler` 根本没有机会执行。`DefaultServletHttpRequestHandler` 更是排在最后，它基本只在"其他人都没匹配上"时才兜底。

顺带纠正一个流传的说法：**`SimpleUrlHandlerMapping` 并不在 `DispatcherServlet.properties` 里。** 静态资源用的那个实例，是 `ResourceHandlerRegistry` 自己 `new` 出来并注册成 Bean 的。

### 7.3 AbstractHandlerMapping 的模板方法

`getHandler()` 在 `AbstractHandlerMapping` 里是 **`final`** 的，子类只能覆写 `getHandlerInternal()`：

```java
// AbstractHandlerMapping.java:506-549（节选）
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    Object handler = getHandlerInternal(request);
    if (handler == null) {
        handler = getDefaultHandler();
    }
    if (handler == null) {
        return null;
    }
    // Bean name or resolved handler?
    if (handler instanceof String handlerName) {
        handler = obtainApplicationContext().getBean(handlerName);
    }
    ...
    HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);
    ...
}
```

这段骨架固定做了四件事，顺序不能变：

1. 委托子类找 handler；
2. 找不到就用 `defaultHandler`（这正是 `defaultServletHandlerMapping` 的兜底机制）；
3. 如果返回的是 `String`，**当成 Bean 名字去容器里取**——这是 `BeanNameUrlHandlerMapping` 的实现基础；
4. 把拦截器附加成 `HandlerExecutionChain`。

第 4 步值得多看一眼，因为**拦截器不是"每次请求现算"的**：

```java
// AbstractHandlerMapping.java:622-637（节选）
for (HandlerInterceptor interceptor : this.adaptedInterceptors) {
    if (interceptor instanceof MappedInterceptor mappedInterceptor) {
        if (mappedInterceptor.matches(request)) {
            chain.addInterceptor(mappedInterceptor.getInterceptor());
        }
    }
    else {
        chain.addInterceptor(interceptor);
    }
}
```

`adaptedInterceptors` 在启动期就装配好了。**`MappedInterceptor` 会带上路径匹配条件**，其他拦截器则无差别附加到每一个 chain 上——所以不写 `addPathPatterns` 的拦截器会作用到**所有**请求，包括静态资源和错误页。

### 7.4 RequestMappingHandlerMapping：路由表的建立与查找

`@Controller` 的扫描和映射表建立发生在**启动期**：`RequestMappingHandlerMapping` 遍历所有 Bean，把类和方法上的 `@RequestMapping` 合并成 `RequestMappingInfo`，和 `HandlerMethod` 一起存进 `mappingRegistry`。

查找时有一条快路径：

```java
// AbstractHandlerMethodMapping.java:400-408（节选）
List<T> directPathMatches = this.mappingRegistry.getMappingsByDirectPath(lookupPath);
if (directPathMatches != null) {
    addMatchingMappings(directPathMatches, matches, request);
}
if (matches.isEmpty()) {
    addMatchingMappings(this.mappingRegistry.getRegistrations().keySet(), matches, request);
}
```

**先按完整路径查索引，没命中才全量扫描所有注册项。** 这个设计意味着：路径里带变量（`/user/{id}`）的映射无法走直接索引，只能落到全量扫描那一支。**如果路由表特别大（几千个映射），带路径变量的接口会明显比固定路径的接口慢**——这是"路由表膨胀"的隐性代价。

`RequestMappingInfo` 的匹配维度不止 URL：

| 维度 | 来源 |
|:---|:---|
| 路径 | `value` / `path` |
| HTTP 方法 | `method` |
| 请求参数条件 | `params` |
| 请求头条件 | `headers` |
| `Content-Type` | `consumes` |
| `Accept` | `produces` |

**`params` / `headers` 条件也是路由的一部分**，因此"路径明明对得上却 404"的另一种可能是：`params` 条件没满足（比如要求 `?type=json` 而请求没带）。

### 7.5 路径匹配：PathPatternParser 已经成为默认

```java
// AbstractHandlerMapping.java:93
private PathPatternParser patternParser = new PathPatternParser();
```

**这是 6.x 的一个硬性变化。** 在 Spring 5.3 里默认还是 `AntPathMatcher`，从 Spring 6.0 起 `PathPatternParser` 成为默认；只有显式把 `patternParser` 设成 `null` 才会退回 `AntPathMatcher`。

同时，另一处让很多人踩坑的默认值也变了：

```java
// RequestMappingHandlerMapping.java:98
private boolean useTrailingSlashMatch = false;
```

```java
// RequestMappingHandlerMapping.java:161-162
@Deprecated(since = "6.0")
public void setUseTrailingSlashMatch(boolean useTrailingSlashMatch) {
```

**尾斜杠匹配默认关闭。** `/user` 和 `/user/` 在 6.x 下是两个不同的路径。这正是 Spring Boot 2.6 之后大量"升级完突然 404"的隐秘原因——**这不是 bug，是被有意收紧的行为**。

还有一条连带约束，在 `afterPropertiesSet()` 里以断言的形式写死：

```java
// RequestMappingHandlerMapping.java:229-232
if (getPatternParser() != null) {
    this.config.setPatternParser(getPatternParser());
    Assert.isTrue(!this.useSuffixPatternMatch && !this.useRegisteredSuffixPatternMatch,
            "Suffix pattern matching not supported with PathPatternParser.");
}
```

**后缀匹配（`/users` 也能匹配 `/users.json`）与 `PathPatternParser` 不能共存。** 一旦你显式开启后缀匹配，`patternParser` 会被置空、退回 `AntPathMatcher`。这也是为什么第七节看起来只是"路由"，却和第十一节的内容协商紧紧绑在一起——**后缀匹配、内容协商、安全（RFD）在 Spring 6 里被一起收紧了**。

## 八、参数解析：注解决定"从哪里取值"

这一节回到[系列导读](/2026/08/26/SpringBoot的一次完整请求发生了什么？/)里那个 400。

### 8.1 从 HandlerAdapter 到方法调用

拿到 handler 之后，`DispatcherServlet` 需要找能执行它的适配器：

```java
// DispatcherServlet.java:1316-1326
protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
    if (this.handlerAdapters != null) {
        for (HandlerAdapter adapter : this.handlerAdapters) {
            if (adapter.supports(handler)) {
                return adapter;
            }
        }
    }
    throw new ServletException("No adapter for handler [" + handler +
            "]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
}
```

`@RequestMapping` 方法由 `RequestMappingHandlerAdapter` 处理。注意它**判断的是 `HandlerMethod` 这个类型**，而不是"类上有没有 `@Controller`"：

```java
// AbstractHandlerMethodAdapter.java:68-70
public final boolean supports(Object handler) {
    return (handler instanceof HandlerMethod handlerMethod && supportsInternal(handlerMethod));
}
```

### 8.2 getMethodArgumentValues：逐个参数找解析器

真正的参数解析在这里：

```java
// InvocableHandlerMethod.java:204-222（spring-web，注意包的归属）
protected Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
        Object... providedArgs) throws Exception {

    MethodParameter[] parameters = getMethodParameters();
    if (ObjectUtils.isEmpty(parameters)) {
        return EMPTY_ARGS;
    }

    Object[] args = new Object[parameters.length];
    for (int i = 0; i < parameters.length; i++) {
        MethodParameter parameter = parameters[i];
        parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
        args[i] = findProvidedArgument(parameter, providedArgs);
        if (args[i] != null) {
            continue;
        }
        if (!this.resolvers.supportsParameter(parameter)) {
            throw new IllegalStateException(formatArgumentError(parameter, "No suitable resolver"));
        }
        try {
            args[i] = this.resolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
        }
        catch (Exception ex) {
            ...
            throw ex;
        }
    }
    return args;
}
```

三个细节：

- **`initParameterNameDiscovery()` 并不立即反射取参数名**，它只是把 discoverer 挂到参数上；真正的解析延迟到 `getParameterName()` 被调用时。
- **`findProvidedArgument()` 是解析器之前的捷径**：`@ModelAttribute` 之类已经准备好的对象直接命中，不必再走解析器。
- **找不到解析器抛的是 `IllegalStateException("No suitable resolver")`**，不是 `IllegalArgumentException`。这个区别在写全局异常处理器时很实际——它属于"配置错误"，而不属于"客户端错误"，**不应该被翻译成 400**。

注意这个类的位置：`InvocableHandlerMethod` 在 **spring-web** 的 `org.springframework.web.method.support` 包下，不在 spring-webmvc。**参数解析机制本身与 Servlet 无关**，这也是 WebFlux 能复用同一套 `HandlerMethodArgumentResolver` 接口的原因。

### 8.3 27 个内置解析器，顺序就是优先级

```java
// RequestMappingHandlerAdapter.java:653-698（节选）
private List<HandlerMethodArgumentResolver> getDefaultArgumentResolvers() {
    List<HandlerMethodArgumentResolver> resolvers = new ArrayList<>(30);

    // Annotation-based argument resolution
    resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), false));
    resolvers.add(new RequestParamMapMethodArgumentResolver());
    resolvers.add(new PathVariableMethodArgumentResolver());
    ...
```

先说清楚数字怎么来的：这个方法体里有 29 处 `add`，其中 1 处是 `addAll(getCustomArgumentResolvers())`（自定义块）；28 个内置里又有 1 个被 `KotlinDetector.isKotlinPresent()` 守着。所以：

- **非 Kotlin 工程：27 个内置解析器**
- **Kotlin 在 classpath 上：28 个**（多一个 `ContinuationHandlerMethodArgumentResolver`）

（顺带一提，`new ArrayList<>(30)` 里的 30 是**容量预分配**，不是元素个数，别被它带偏。）

按"注解驱动"和"类型驱动"分成两组：

| # | 解析器 | 认的注解 / 类型 |
|:---|:---|:---|
| 1 | `RequestParamMethodArgumentResolver(false)` | `@RequestParam` |
| 2 | `RequestParamMapMethodArgumentResolver` | `@RequestParam Map` |
| 3 | `PathVariableMethodArgumentResolver` | `@PathVariable` |
| 4 | `PathVariableMapMethodArgumentResolver` | `@PathVariable Map` |
| 5 | `MatrixVariableMethodArgumentResolver` | `@MatrixVariable` |
| 6 | `MatrixVariableMapMethodArgumentResolver` | `@MatrixVariable Map` |
| 7 | `ServletModelAttributeMethodProcessor(false)` | `@ModelAttribute` |
| 8 | `RequestResponseBodyMethodProcessor` | **`@RequestBody`** |
| 9 | `RequestPartMethodArgumentResolver` | `@RequestPart` |
| 10 | `RequestHeaderMethodArgumentResolver` | `@RequestHeader` |
| 11 | `RequestHeaderMapMethodArgumentResolver` | `@RequestHeader Map` |
| 12 | `ServletCookieValueMethodArgumentResolver` | `@CookieValue` |
| 13 | `ExpressionValueMethodArgumentResolver` | `@Value` |
| 14 | `SessionAttributeMethodArgumentResolver` | `@SessionAttribute` |
| 15 | `RequestAttributeMethodArgumentResolver` | `@RequestAttribute` |
| 16 | `ServletRequestMethodArgumentResolver` | `WebRequest` / `ServletRequest` / `InputStream` / `Reader` … |
| 17 | `ServletResponseMethodArgumentResolver` | `ServletResponse` / `OutputStream` / `Writer` |
| 18 | `HttpEntityMethodProcessor` | `HttpEntity` |
| 19 | `RedirectAttributesMethodArgumentResolver` | `RedirectAttributes` |
| 20 | `ModelMethodProcessor` | `Model` |
| 21 | `MapMethodProcessor` | `Map` |
| 22 | `ErrorsMethodArgumentResolver` | `Errors` / `BindingResult` |
| 23 | `SessionStatusMethodArgumentResolver` | `SessionStatus` |
| 24 | `UriComponentsBuilderMethodArgumentResolver` | `UriComponentsBuilder` |
| — | **自定义解析器插入点** | — |
| 25 | `PrincipalMethodArgumentResolver` | `Principal` |
| 26 | `RequestParamMethodArgumentResolver(true)` | 兜底：简单类型 |
| 27 | `ServletModelAttributeMethodProcessor(true)` | 兜底：复杂类型 |

### 8.4 缓存：为什么第二次请求不再遍历

匹配结果会被缓存，但**缓存不在你以为的地方**——它在 composite 上：

```java
// HandlerMethodArgumentResolverComposite.java:43
private final Map<MethodParameter, HandlerMethodArgumentResolver> argumentResolverCache =
```

```java
// HandlerMethodArgumentResolverComposite.java:130-142
@Nullable
public HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
    HandlerMethodArgumentResolver result = this.argumentResolverCache.get(parameter);
    if (result == null) {
        for (HandlerMethodArgumentResolver resolver : this.argumentResolvers) {
            if (resolver.supportsParameter(parameter)) {
                result = resolver;
                this.argumentResolverCache.put(parameter, result);
                break;
            }
        }
    }
    return result;
}
```

**首次匹配即锁定**：顺序遍历，第一个 `supportsParameter()` 返回 `true` 的被 `break` 选中，然后按 `MethodParameter` 缓存。后续请求直接命中缓存，不再重复遍历。

### 8.5 自定义解析器排在哪里（一个广为流传的错误说法）

很多资料（包括网上很多中文资料）说"自定义解析器排在所有内置解析器之后，所以会被内置的抢走"。**这个说法不准确。**

真实插入点是**中间**：

```java
// RequestMappingHandlerAdapter.java:687-695
// Custom arguments
if (getCustomArgumentResolvers() != null) {
    resolvers.addAll(getCustomArgumentResolvers());
}

// Catch-all
resolvers.add(new PrincipalMethodArgumentResolver());
resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), true));
resolvers.add(new ServletModelAttributeMethodProcessor(true));
```

结论应该这样表述：

> **自定义解析器插在 24 个内置解析器之后、3 个兜底解析器之前。**

由此得到两条精确的推论：

- ✅ 自定义解析器**能**抢在兜底逻辑之前——比如一个无注解的 `User user` 参数，你可以用自己的解析器接管，而不必担心被 `ServletModelAttributeMethodProcessor(true)` 抢先。
- ❌ 自定义解析器**覆盖不了前面那 24 个内置解析器**。只要参数上写了 `@PathVariable`，它就会被 `PathVariableMethodArgumentResolver` 接走，轮不到你。

同样的规律也适用于返回值处理器：15 个内置处理器里，自定义插在 14 个之后、兜底之前，**只能覆盖兜底**。

### 8.6 没有注解的参数会怎样

规则由两个兜底解析器的**先后顺序**决定。`RequestParamMethodArgumentResolver(true)` 排在前面，于是形成一条二分法则：

```java
// RequestParamMethodArgumentResolver.java:126-152（节选）
@Override
public boolean supportsParameter(MethodParameter parameter) {
    if (parameter.hasParameterAnnotation(RequestParam.class)) {
        ...
    }
    else {
        if (parameter.hasParameterAnnotation(RequestPart.class)) {
            return false;
        }
        parameter = parameter.nestedIfOptional();
        if (MultipartResolutionDelegate.isMultipartArgument(parameter)) {
            return true;
        }
        else if (this.useDefaultResolution) {
            return BeanUtils.isSimpleProperty(parameter.getNestedParameterType());
        }
        else {
            return false;
        }
    }
}
```

```java
// ModelAttributeMethodProcessor.java:90-94
@Override
public boolean supportsParameter(MethodParameter parameter) {
    return (parameter.hasParameterAnnotation(ModelAttribute.class) ||
            (this.annotationNotRequired && !BeanUtils.isSimpleProperty(parameter.getParameterType())));
}
```

| 参数类型 | 交给谁 | 等价于 |
|:---|:---|:---|
| 简单类型（`String`、`int`、`Long`、`boolean`…） | `RequestParamMethodArgumentResolver(true)` | `@RequestParam` |
| 复杂类型（自定义 POJO） | `ServletModelAttributeMethodProcessor(true)` | `@ModelAttribute` |

**这里有一个很容易踩的不对称**：`RequestParamMethodArgumentResolver` 用的是 `parameter.nestedIfOptional().getNestedParameterType()`（**会解包 `Optional<T>`**），而 `ModelAttributeMethodProcessor` 用的是 `parameter.getParameterType()`（**不解包**）。

后果是：**一个无注解的 `Optional<Foo>` 参数不会被兜底当成模型属性**，它会落到"都不认领"的境地，最终抛出 `IllegalStateException("No suitable resolver")`。这不是理论推演，而是那两条 `supportsParameter()` 直接决定的。

**路径变量则是彻底的例外。** 它的守卫在第一行，没有任何默认兜底：

```java
// PathVariableMethodArgumentResolver.java:72-75
public boolean supportsParameter(MethodParameter parameter) {
    if (!parameter.hasParameterAnnotation(PathVariable.class)) {
        return false;
    }
```

原因很自然：路径变量只存在于 URI 模板里，**无法像请求参数那样靠"参数名"反推**。所以即使参数名和 `@GetMapping("/user/{id}")` 里的 `{id}` 完全一致，缺了 `@PathVariable` 也不会绑定——它会一路落到兜底，被当成**查询参数** `?id=` 处理，然后在引子里的那个报错上撞墙。

### 8.7 编译期就必须确定的事：-parameters

无注解参数和未指定 `name` 的注解，都依赖"从字节码里读出参数名"。这个能力由 `DefaultParameterNameDiscoverer` 提供：

```java
// DefaultParameterNameDiscoverer.java:38-45
public DefaultParameterNameDiscoverer() {
    if (KotlinDetector.isKotlinReflectPresent()) {
        addDiscoverer(new KotlinReflectionParameterNameDiscoverer());
    }

    // Recommended approach on Java 8+: compilation with -parameters.
    addDiscoverer(new StandardReflectionParameterNameDiscoverer());
}
```

**在 6.1.8 里，`LocalVariableTableParameterNameDiscoverer` 已经彻底移除**——整个源码树里搜不到这个类，也没有保留任何 `@Deprecated` 占位。现在只剩两个：条件性的 Kotlin 反射，和依赖 `Parameter.isNamePresent()` 的标准反射。后者正是 `-parameters` 编译参数在起作用。

没有 `-parameters` 时，报错长这样：

```java
// AbstractNamedValueMethodArgumentResolver.java:179-189（节选）
String name = info.name;
if (info.name.isEmpty()) {
    name = parameter.getParameterName();
    if (name == null) {
        throw new IllegalArgumentException("""
                Name for argument of type [%s] not specified, and parameter name information not \
                available via reflection. Ensure that the compiler uses the '-parameters' flag."""
                    .formatted(parameter.getNestedParameterType().getName()));
    }
}
```

**这次的异常类型是 `IllegalArgumentException`**（和 8.2 里的 `IllegalStateException` 不是一回事，排查时注意区分）。Spring Boot 的 `spring-boot-starter-parent` 默认已开启 `-parameters`，所以这个坑通常只在自定义构建配置（或某些 Kotlin/Gradle 组合）里出现。

### 8.8 Converter、Formatter、HttpMessageConverter 的分工

这三者经常被混为一谈，其实分工很清晰。

| 接口 | 处理对象 | 何时生效 | 典型实现 |
|:---|:---|:---|:---|
| `Converter<S,T>` | 单个值的类型转换 | 参数绑定阶段 | `StringToIntegerConverter` |
| `Formatter<T>` | 字符串 ↔ 对象的**双向**转换（支持 locale） | 参数绑定阶段 | `DateFormatter` |
| `HttpMessageConverter<T>` | **整个请求体 / 响应体** | `@RequestBody` / `@ResponseBody` | `MappingJackson2HttpMessageConverter` |

一句话区分：**Converter / Formatter 处理"一个参数值"，HttpMessageConverter 处理"整个请求体或响应体"。**

`?id=123` 里的 `"123"` 是字符串，方法参数是 `Long`——这是 Converter 的活。而一个 JSON 请求体变成一个对象，是 HttpMessageConverter 的活。

### 8.9 @RequestBody 的读取路径：只看 Content-Type

`@RequestBody` 的解析由 `RequestResponseBodyMethodProcessor` 承担，它同时实现了入参解析和返回值处理两个接口（这是它能同时管 `@RequestBody` 和 `@ResponseBody` 的原因）。

```java
// RequestResponseBodyMethodProcessor.java:155-167
@Override
@Nullable
protected <T> Object readWithMessageConverters(NativeWebRequest webRequest, MethodParameter parameter,
        Type paramType) throws IOException, HttpMediaTypeNotSupportedException, HttpMessageNotReadableException {

    ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
    Object arg = readWithMessageConverters(inputMessage, parameter, paramType);
    if (arg == null && checkRequired(parameter)) {
        throw new HttpMessageNotReadableException("Required request body is missing: " +
                parameter.getExecutable().toGenericString(), inputMessage);
    }
    return arg;
}
```

真正的选择逻辑在父类：**读的判定依据是请求的 `Content-Type`，乘以转换器的 `canRead()`**。

```java
// AbstractMessageConverterMethodArgumentResolver.java:155-180（节选）
try {
    contentType = inputMessage.getHeaders().getContentType();
}
catch (InvalidMediaTypeException ex) {
    throw new HttpMediaTypeNotSupportedException(
            ex.getMessage(), getSupportedMediaTypes(targetClass != null ? targetClass : Object.class));
}
if (contentType == null) {
    noContentType = true;
    contentType = MediaType.APPLICATION_OCTET_STREAM;
}
...
for (HttpMessageConverter<?> converter : this.messageConverters) {
    ...
    if (genericConverter != null ? genericConverter.canRead(targetType, contextClass, contentType) :
            (targetClass != null && converter.canRead(targetClass, contentType))) {
```

两个要点：

- **`produces` 完全不参与读路径。** 它只影响写。这是很常见的误解。
- **`Content-Type` 缺失时会被当成 `application/octet-stream`**（不是"跳过校验"），于是通常匹配不到 JSON 转换器，直接 415。

**`Content-Type` 不支持时抛 `HttpMediaTypeNotSupportedException` → 415；JSON 语法错误或反序列化失败时抛 `HttpMessageNotReadableException` → 400。** 这两个错误码的边界就在这里。

### 8.10 回到引子：为什么 @RequestParam 取不到 JSON 里的字段

现在可以把引子里的报错完整解释了。

| | `@RequestParam` | `@RequestBody` |
|:---|:---|:---|
| 解析器 | `RequestParamMethodArgumentResolver` | `RequestResponseBodyMethodProcessor` |
| **取值位置** | query string、表单参数（`application/x-www-form-urlencoded`） | **请求体**（经 `HttpMessageConverter` 反序列化） |
| 依据 | 参数名 / `name` 属性 | `Content-Type` |
| 取值失败 | 400 `MissingServletRequestParameterException` | 400 `HttpMessageNotReadableException` |

**它们取的根本不是同一个"参数"。** `@RequestParam` 是"从 URL 和表单里按名字捞一个值"，`@RequestBody` 是"把整个请求体反序列化成一个对象"。参数确实传了——它传在请求体里；但解析 `@RequestParam` 的那个解析器，只会在 query string 和表单里找。

修法有三种，选哪种取决于接口语义：

```java
// ① 前端改成 query 参数：GET /order?userId=123
@GetMapping("/order")
public Order getOrder(@RequestParam String userId) { ... }

// ② 保持 JSON 请求体，改用 @RequestBody + DTO（POST/PUT 场景的正确姿势）
@PostMapping("/order")
public Order getOrder(@RequestBody OrderQuery query) { ... }

// ③ 两者兼有：URI 里传资源标识，body 里传变更内容
@PutMapping("/order/{id}")
public Order update(@PathVariable Long id, @RequestBody OrderUpdate update) { ... }
```

> **不要用"给 `@RequestParam` 加 `required = false`"来糊过去。** 那只是把 400 换成了 `null`，然后在下游某处变成 NPE 或者一条脏数据——把"客户端明确报错"改成了"数据静默出错"，是更差的交易。

### 8.11 自定义参数解析器（修正版）

理解了策略链之后，`@CurrentUser` 这种注解就顺手了。

```java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface CurrentUser {
}
```

```java
// 注册成 Bean，让依赖注入正常工作
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

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    private final CurrentUserArgumentResolver currentUserArgumentResolver;

    public WebConfig(CurrentUserArgumentResolver currentUserArgumentResolver) {
        this.currentUserArgumentResolver = currentUserArgumentResolver;
    }

    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        resolvers.add(currentUserArgumentResolver);   // 从容器里取，不要 new
    }
}
```

> 之前的写法在 `addArgumentResolvers` 里直接 `new CurrentUserArgumentResolver(tokenService)`，那样这个解析器就不是 Spring Bean 了——它内部再用 `@Autowired`、`@Transactional`、AOP 代理，全部都不会生效。**注册进 MVC 的组件，尽量从容器里取。**

**`@CurrentUser` 会和 `@RequestBody` 冲突吗？不会。** `RequestResponseBodyMethodProcessor.supportsParameter()` 只认 `@RequestBody`，自定义解析器只认 `@CurrentUser`，注解不同，认领者就不同。但如果同一个参数上同时写了两个注解，就会变成"谁排在前面谁赢"——而这正是 8.5 节那张顺序表在做的事。

## 九、Interceptor：顺序与中断语义

### 9.1 三个方法的真实实现

`HandlerExecutionChain` 的三个字段决定了全部行为：

```java
// HandlerExecutionChain.java:44-48
private final Object handler;

private final List<HandlerInterceptor> interceptorList = new ArrayList<>();

private int interceptorIndex = -1;
```

**`interceptorIndex` 是整个中断语义的核心。** 看 `applyPreHandle`：

```java
// HandlerExecutionChain.java:143-153
boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response) throws Exception {
    for (int i = 0; i < this.interceptorList.size(); i++) {
        HandlerInterceptor interceptor = this.interceptorList.get(i);
        if (!interceptor.preHandle(request, response, this.handler)) {
            triggerAfterCompletion(request, response, null);
            return false;
        }
        this.interceptorIndex = i;
    }
    return true;
}
```

注意赋值那一行的位置：`this.interceptorIndex = i;` 在 `if` 判断**之后**。也就是说，**它只在 `preHandle` 成功之后才被推进**。

```java
// HandlerExecutionChain.java:158-165
void applyPostHandle(HttpServletRequest request, HttpServletResponse response, @Nullable ModelAndView mv)
        throws Exception {

    for (int i = this.interceptorList.size() - 1; i >= 0; i--) {
        HandlerInterceptor interceptor = this.interceptorList.get(i);
        interceptor.postHandle(request, response, this.handler, mv);
    }
}
```

```java
// HandlerExecutionChain.java:172-182
void triggerAfterCompletion(HttpServletRequest request, HttpServletResponse response, @Nullable Exception ex) {
    for (int i = this.interceptorIndex; i >= 0; i--) {
        HandlerInterceptor interceptor = this.interceptorList.get(i);
        try {
            interceptor.afterCompletion(request, response, this.handler, ex);
        }
        catch (Throwable ex2) {
            logger.error("HandlerInterceptor.afterCompletion threw exception", ex2);
        }
    }
}
```

`postHandle` 从尾到头；`afterCompletion` 从 `interceptorIndex` 倒着走到 0。**两者都是逆序。**

### 9.2 中断时到底谁收到了 afterCompletion

假设有三个拦截器 `[I0, I1, I2]`，`I1` 的 `preHandle` 返回 `false`：

| 拦截器 | `preHandle` | `postHandle` | `afterCompletion`（收到什么异常） |
|:---|:---|:---|:---|
| `I0` | ✅ 执行 | ❌ 不执行 | ✅ 执行，`ex = null` |
| `I1` | ❌ 返回 false | ❌ 不执行 | ❌ **不执行** |
| `I2` | ❌ 未执行 | ❌ 不执行 | ❌ 不执行 |

推导过程：`I1` 的 `preHandle` 返回 false 时，`interceptorIndex` 还停在 `0`（`I0` 成功时赋的值），于是 `triggerAfterCompletion` 从 `i = 0` 开始倒序执行，**只碰到 `I0`**。

**这里有两个极易写错的点：**

**① 返回 `false` 的那个拦截器，自己的 `afterCompletion` 不会被调用。** 很多人以为"我拦截了，那我总能收到自己的 `afterCompletion` 做清理"——不会。要清理资源，就得在那个 `preHandle` 返回 `false` 之前自己处理掉。

**② 中断时传下去的 `ex` 是 `null`。**

```java
triggerAfterCompletion(request, response, null);
```

这是最反直觉的地方：**"被拦截器主动拦下"和"请求正常结束"，在 `afterCompletion` 里长得一模一样（`ex` 都是 `null`）。**

而 `processDispatchResult()` 正常结束时的调用也是 `null`：

```java
// DispatcherServlet.java:1183-1186
if (mappedHandler != null) {
    // Exception (if any) is already handled..
    mappedHandler.triggerAfterCompletion(request, response, null);
}
```

所以如果你在 `afterCompletion` 里写"`ex == null` 就记成功日志"，**被拦截器拒绝的请求会被记成成功**。真实的失败信息只能从 HTTP 状态码或你自己放进 request attribute 的标记里拿。

**③ `afterCompletion` 里抛异常不会影响其他拦截器。** `catch (Throwable ex2)` 把它吞掉并记 error，循环继续。这是有意的：清理逻辑不该阻断链上其他清理。

### 9.3 三种"不执行"的情况

| 场景 | preHandle | postHandle | afterCompletion |
|:---|:---|:---|:---|
| 正常结束 | ✅ | ✅ 逆序 | ✅ 逆序，`ex = null` |
| 某个 `preHandle` 返回 false | 到它为止 | ❌ | 只有它**之前**的，`ex = null` |
| 抛异常 | ✅ | ❌（异常在 `ha.handle` 抛出） | ✅ 全部，`ex` 非 null |
| 404（无 chain） | ❌ | ❌ | ❌ |
| 304 短路 | ❌ | ❌ | ❌ |
| 异步启动 | ✅ | ❌ → `afterConcurrentHandlingStarted` | ❌ → `afterConcurrentHandlingStarted` |

最后两行值得再强调一次，因为它们是"埋点数据对不上"的常见来源：

- **404 的请求不会触发任何拦截器回调。** 因为 `mappedHandler == null`，`HandlerExecutionChain` 从未被创建（见 6.2 节③）。
- **304 的请求同样不触发。** 因为 `checkNotModified` 的短路发生在 `applyPreHandle` 之前（见 6.2 节②）。

如果你的访问统计依赖拦截器，那么"静态资源 + 未修改的 GET 请求"这一大批流量是统计不到的——**它们会显得比实际少**。

## 小结

这一篇把请求从 `DispatcherServlet` 门口送到方法调用之前：`doDispatch()` 的主干、`HandlerMapping` 的匹配顺序、27 个参数解析器谁先被问、拦截器的三段语义。四件事都在回答同一个问题——**到底哪个方法、带什么参数被执行**。

拦截器全部放行，`InvocableHandlerMethod` 反射调用的那一行已经在眼前。**到这里，"请求能不能被执行"已经答完；再往后是另一个问题——执行的结果怎么变成 HTTP 响应，执行时抛的异常又变成哪个状态码。** 那是第三篇的内容。

> **下一篇**
>
> [SpringBoot 请求链路（三）：方法执行、异步与响应写回](/2026/08/26/SpringBoot请求链路3：从方法执行到响应写完/)——异常解析器的优先级、15 个返回值处理器与内容协商、异步请求的三根线程，一路写到 `OutputBuffer` 和 Socket。

