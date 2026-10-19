---
title: 一次下载失败排查：被 Filter 吞掉的异步响应流
date: 2026-10-14 15:45:09
tags:
  - spring
  - tomcat
categories:
  - develop
---

下载接口的响应头一切正常：状态码 `200`、`Content-Disposition: attachment; filename=big.zip`、`Content-Type` 也对。唯独文件是 **0 字节**。

更让人摸不着头脑的是它不稳定——偶尔又能下下来一小段，前几 KB 是完整的，后面戛然而止。日志里干干净净，没有任何异常堆栈。

把项目里那个统一的日志 Filter 注释掉，一切立刻恢复正常。

> - 状态码和响应头都写对了，为什么一个字节都没发出去？
> - `StreamingResponseBody` 拿到的那个 `OutputStream`，到底是谁的？
> - 为什么它时好时坏，而不是稳定失败？
> - Filter 里判断一下 `DispatcherType.ASYNC`，能不能救回来？
> - 正确写法是什么？Spring 自己是怎么处理这个问题的？

这篇文章按排查的顺序走一遍：先定位到嫌疑对象，再从源码把时间线还原出来，最后给出几种修法。如果你对 `StreamingResponseBody` 的工作方式还不太熟，建议先看 [Spring 文件下载全方案深度解析](/2026/09/15/Spring-文件下载全方案深度解析/)的 3.4 节，以及 [Spring MVC 中那些“不走寻常路”的返回类型](/2026/09/14/Spring-MVC-中那些“不走寻常路”的返回类型/)的 3.4 与 8.3 节——这篇是它们的续集。

## 一、先把范围缩到最小

### 1.1 一个能稳定复现的组合

Controller 是标准写法，没有任何特别之处：

```java
@RestController
public class DownloadController {

    @GetMapping("/download")
    public ResponseEntity<StreamingResponseBody> download() {
        StreamingResponseBody body = out -> {
            try (InputStream in = new FileInputStream("/tmp/big.zip")) {
                byte[] buffer = new byte[8192];
                int read;
                while ((read = in.read(buffer)) != -1) {
                    out.write(buffer, 0, read);
                }
            }
        };
        return ResponseEntity.ok()
                .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=big.zip")
                .body(body);
    }
}
```

Filter 是项目里的统一日志 / 链路追踪过滤器，做了三件事：写 MDC、缓存请求体、缓存响应体，并在 `chain.doFilter` 前后各打一次日志：

```java
public class TraceFilter implements Filter {

    private SystemConfig systemConfig;

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse,
                         FilterChain chain) throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest) servletRequest;
        HttpServletResponse response = (HttpServletResponse) servletResponse;
        try {
            String requestIdKey = systemConfig.getRequest().getIdKey();
            String requestIdValue = StrUtil.blankToDefault(request.getHeader(requestIdKey),
                    StrUtils.getRequestUUID());
            MDC.put(requestIdKey, requestIdValue);

            // ① 请求包装：multipart 不包，其他按 payload 上限缓存请求体
            HttpServletRequest requestWrapper = request;
            if (!ServletFileUpload.isMultipartContent(request)) {
                requestWrapper = new ContentCachingRequestWrapper(request,
                        systemConfig.getInterceptor().getMaxPayloadLength());
            }

            // ② 响应包装：缓存整个响应体
            ContentCachingResponseWrapper responseWrapper =
                    new ContentCachingResponseWrapper(response);

            this.print(requestWrapper, responseWrapper);

            chain.doFilter(requestWrapper, responseWrapper);

            this.print(responseWrapper, requestWrapper);

            // ③ 把缓存里的响应体写回真正的响应
            responseWrapper.copyBodyToResponse();
        } finally {
            MDC.clear();
        }
    }
}
```

单独跑这个 Controller，`curl` 下来的文件完整；加上 `TraceFilter`，文件变成 0 字节。二分定位到这一步就够了。

### 1.2 两个包装器，先分清嫌疑人

Filter 里包了两个东西，名字很像，职责完全不同：

| 包装器 | 包的是 | 缓存什么 | 和本次故障的关系 |
|:---|:---|:---|:---|
| `ContentCachingRequestWrapper` | `HttpServletRequest` | 请求体（表单、JSON 等） | 无关 |
| `ContentCachingResponseWrapper` | `HttpServletResponse` | **响应体** | **元凶** |

一开始容易怀疑请求包装器，但它是无罪的：它只拦截 `getInputStream()` / `getReader()`，把读到的字节抄一份到自己的 `ByteArrayOutputStream` 里，和响应怎么发出去没有任何关系。

真正把数据“接走”的是**响应**包装器。下面先把它拆开看。

## 二、`ContentCachingResponseWrapper` 是怎么把数据关起来的

这个类的核心只有一句话：**所有写操作都进内存，只有显式调用 `copyBodyToResponse()` 才会落到客户端**。

```java
public class ContentCachingResponseWrapper extends HttpServletResponseWrapper {

    private final FastByteArrayOutputStream content = new FastByteArrayOutputStream(1024);

    @Override
    public ServletOutputStream getOutputStream() throws IOException {
        if (this.outputStream == null) {
            this.outputStream = new ResponseServletOutputStream(getResponse().getOutputStream());
        }
        return this.outputStream;
    }

    private class ResponseServletOutputStream extends ServletOutputStream {

        private final ServletOutputStream os;   // 原始响应流，只用来转发 isReady()

        @Override
        public void write(int b) throws IOException {
            content.write(b);                   // ← 写进内存，不是写给客户端
        }

        @Override
        public void write(byte[] b, int off, int len) throws IOException {
            content.write(b, off, len);         // ← 同上
        }

        @Override
        public boolean isReady() {
            return this.os.isReady();
        }

        @Override
        public void setWriteListener(WriteListener writeListener) {
            this.os.setWriteListener(writeListener);
        }
    }
}
```

请注意这个内部类的成员：它虽然持有了原始的 `ServletOutputStream os`，但**只用在 `isReady()` / `setWriteListener()` 上，`write` 一次都没转发过去**。

再看它有没有留下“数据自己跑出去”的口子：

```java
@Override
public void flushBuffer() throws IOException {
    // no-op
}
```

`flushBuffer()` 被写成空方法。而 `ResponseServletOutputStream` 也没有覆写 `flush()` 和 `close()`——`OutputStream` 的这两个方法默认就是空实现。

**结论：`write` 无法逃逸，`flush` 逃逸不了，`flushBuffer` 被堵死。** 唯一的出口是：

```java
protected void copyBodyToResponse(boolean complete) throws IOException {
    if (this.content.size() > 0) {                       // ← 缓存为空时，整个方法体都不进
        HttpServletResponse rawResponse = (HttpServletResponse) getResponse();
        if (!rawResponse.isCommitted()) {
            if (complete || this.contentLength != null) {
                if (rawResponse.getHeader(HttpHeaders.TRANSFER_ENCODING) == null) {
                    rawResponse.setContentLength(complete ? this.content.size() : this.contentLength);
                }
                this.contentLength = null;
            }
        }
        this.content.writeTo(rawResponse.getOutputStream());   // ← 唯一一次真正写客户端
        this.content.reset();                                 // ← 写完清空
        if (complete) {
            super.flushBuffer();
        }
    }
}
```

这里有两个细节后面会用到：

1. **`content.size() > 0` 是一个前置条件。** 如果调用 `copyBodyToResponse()` 时缓存还是空的，这个方法什么都不做——不设置 `Content-Length`，不 flush，不 commit。
2. **`content.reset()` 写在 `writeTo` 之后。** 写回即清空，缓存不会被复用。

对普通的同步接口来说，这套“先攒后发”的模型完全没问题：Controller 在容器线程上跑完，响应体也写完了，随后 `copyBodyToResponse()` 一次性倒出去。

但 `StreamingResponseBody` 恰恰打破了这个前提。

## 三、异步的流，写进了一个没人再看的缓存

### 3.1 谁把流接走了

`ResponseEntity<StreamingResponseBody>` 由 `StreamingResponseBodyReturnValueHandler` 处理。下面是 Spring Framework 5.3.x 的关键逻辑（`Spring Boot 2.7.x` 使用 `javax.servlet`）：

```java
@Override
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
        ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

    if (returnValue == null) {
        mavContainer.setRequestHandled(true);
        return;
    }

    // ① 从当前请求上下文里取 response —— 拿到的是 Filter 塞进来的包装器
    HttpServletResponse response = webRequest.getNativeResponse(HttpServletResponse.class);
    ServerHttpResponse outputMessage = new ServletServerHttpResponse(response);

    if (returnValue instanceof ResponseEntity) {
        ResponseEntity<?> responseEntity = (ResponseEntity<?>) returnValue;
        response.setStatus(responseEntity.getStatusCodeValue());
        outputMessage.getHeaders().putAll(responseEntity.getHeaders());   // ② 响应头写进包装器
        returnValue = responseEntity.getBody();
        // ... 省略 null 判断与 flush
    }

    ServletRequest request = webRequest.getNativeRequest(ServletRequest.class);
    ShallowEtagHeaderFilter.disableContentCaching(request);               // ③ 第五节的重点

    Assert.isInstanceOf(StreamingResponseBody.class, returnValue, "StreamingResponseBody expected");
    StreamingResponseBody streamingBody = (StreamingResponseBody) returnValue;

    // ④ 把流取出来，交给异步任务
    Callable<Void> callable = new StreamingResponseBodyTask(outputMessage.getBody(), streamingBody);
    WebAsyncUtils.getAsyncManager(webRequest).startCallableProcessing(callable, mavContainer);
}
```

第 ④ 行的 `outputMessage.getBody()` 是整个故障的关键。它的实现是：

```java
// ServletServerHttpResponse
@Override
public OutputStream getBody() throws IOException {
    this.bodyUsed = true;
    writeHeaders();
    return this.servletResponse.getOutputStream();   // ← 立刻向当前 response 要流
}
```

链路对上了：

```text
webRequest.getNativeResponse()          →  ContentCachingResponseWrapper（Filter 塞进来的）
  └─ new ServletServerHttpResponse(它)
       └─ getBody()
            └─ ContentCachingResponseWrapper.getOutputStream()
                 └─ ResponseServletOutputStream     ← 一个 write 只会写进内存的流
```

这个流对象被塞进 `StreamingResponseBodyTask`：

```java
private static class StreamingResponseBodyTask implements Callable<Void> {

    private final OutputStream outputStream;   // ← 就是上面那个缓存流，被"接走"了

    private final StreamingResponseBody streamingBody;

    @Override
    public Void call() throws Exception {
        this.streamingBody.writeTo(this.outputStream);   // 异步线程上执行，全部写进缓存
        this.outputStream.flush();                       // 空操作
        return null;
    }
}
```

也就是说：**Controller 返回的那一刻，Spring 就把“往缓存包装器里写”这件事确定下来了。** 之后不管谁来写、在哪个线程写，字节都只会进 `content` 这个内存数组。

### 3.2 时间线：`startAsync` 与 `copyBodyToResponse` 的赛跑

`WebAsyncManager#startCallableProcessing` 的实际顺序是这样的：

```java
interceptorChain.applyBeforeConcurrentHandling(this.asyncWebRequest, callable);
startAsyncProcessing(processingContext);          // ← 先 request.startAsync()
try {
    Future<?> future = this.taskExecutor.submit(() -> {
        Object result = null;
        try {
            interceptorChain.applyPreProcess(this.asyncWebRequest, callable);
            result = callable.call();             // ← 再提交异步任务
        }
        // ... 省略异常与拦截器处理
        setConcurrentResultAndDispatch(result);   // ← 完成后触发 ASYNC 派发
        return null;
    });
    // ...
}
```

`startAsync()` 在 `handleReturnValue` 内部就执行完了，异步任务也在同一时刻提交。这意味着：**`chain.doFilter` 返回时，异步线程可能已经在往那个缓存里写了。**

于是 Filter 收尾的两行代码，和异步任务里的 `writeTo`，是并发执行的：

```java
chain.doFilter(requestWrapper, responseWrapper);   // ← 这里返回时，异步线程可能已经开跑
this.print(responseWrapper, requestWrapper);       // 读缓存
responseWrapper.copyBodyToResponse();              // 写回缓存（唯一一次）
```

谁先谁后，取决于线程调度。这就是“时好时坏”的来源。

**时间线 A：Filter 抢在前面（最常见）**

```text
Tomcat 工作线程                                    异步线程池
──────────────────────────────────────────────────────────────────────────
handleReturnValue: startAsync() + submit ────────────► 任务入队
chain.doFilter 返回
print(responseWrapper)      content.size() == 0
copyBodyToResponse()        size > 0 不成立 → 什么都不做
Filter 返回，容器线程交棒                            callable.call()
                                                     streamingBody.writeTo(流)
                                                       → content.write(...)  ← 写进了没人看的缓存
                                                     outputStream.flush()   ← 空操作
请求结束：客户端拿到 0 字节
```

注意这里连 `flushBuffer()` 都不会被调用——因为 `copyBodyToResponse` 整体被 `if` 挡在了外面。

**时间线 B：异步任务抢在前面**

```text
chain.doFilter 返回
                                                    streamingBody.writeTo(流)
                                                      → content 里攒了前 N KB
copyBodyToResponse()        size > 0 → 写回 N KB
                            setContentLength(N)
                            content.reset()          ← 缓存被清空
                                                    writeTo 继续 → 又写进同一个缓存
                                                      → 依旧无人问津
请求结束：客户端拿到 N KB 的残缺文件，而且 Content-Length 就是 N
```

时间线 B 比 A 更阴险：**`Content-Length` 被设置成了 N，实际写入的字节数也确实是 N，HTTP 层面完全自洽**。客户端拿到的文件长度正确、下载“成功”，只是内容是残的。如果这个 zip 没有校验，损坏会一路传到用户手上。

> 顺带一提，时间线 B 还有一层隐患：`FastByteArrayOutputStream` 本身不是线程安全的，而此刻容器线程在 `writeTo` / `reset`，异步线程在 `write`——两个线程同时操作同一个缓冲区。

### 3.3 异步派发回来了，但没人再管那份缓存

响应最终还是要结束的，容器会做一次 `ASYNC` 派发。那这一趟能不能顺手把 `wrapper1` 的缓存倒出去？

先看 Filter 会不会被再次调用，这里要分两种情况：

- **Filter 直接实现 `Filter` 接口**：Spring Boot 的 `AbstractFilterRegistrationBean` 在没有显式 `setDispatcherTypes` 时，只会注册 `DispatcherType.REQUEST`。异步派发根本进不来。
- **Filter 继承 `OncePerRequestFilter`**：Spring Boot 会把派发类型设为**全部**（包括 `ASYNC`），但 `OncePerRequestFilter#skipDispatch` 里有一道判断——`shouldNotFilterAsyncDispatch()` 默认返回 `true`，异步派发会被主动跳过。

```java
private boolean skipDispatch(HttpServletRequest request) {
    if (isAsyncDispatch(request) && shouldNotFilterAsyncDispatch()) {
        return true;
    }
    // ...
}
```

两条路都通向同一个结果：**Filter 的逻辑只会执行一次，`copyBodyToResponse()` 也只会被调用一次。**

退一步说，就算真的在异步派发里又进了 Filter，也不会有新的字节被写出来。因为 `Async` 派发回到 `DispatcherServlet` 后，Controller 并不会被真正重新执行——`RequestMappingHandlerAdapter#invokeHandlerMethod` 会复用第一次的 `ModelAndViewContainer`：

```java
if (asyncManager.hasConcurrentResult()) {
    Object result = asyncManager.getConcurrentResult();
    Object[] resultContext = asyncManager.getConcurrentResultContext();
    Assert.state(resultContext != null && resultContext.length > 0, "Missing result context");
    mavContainer = (ModelAndViewContainer) resultContext[0];        // ← 复用第一次的容器
    asyncManager.clearConcurrentResult();
    invocableMethod = invocableMethod.wrapConcurrentResult(result); // ← 不再调用 Controller
}
invocableMethod.invokeAndHandle(webRequest, mavContainer);
```

`StreamingResponseBodyTask` 是 `Callable<Void>`，返回 `null`。所以这次 `invokeAndHandle` 拿到的返回值是 `null`，走到 `StreamingResponseBodyReturnValueHandler#handleReturnValue` 的第一行就干净地返回了：

```java
if (returnValue == null) {
    mavContainer.setRequestHandled(true);
    return;
}
```

**异步派发这一趟，不会再有任何一次写响应体的动作。**

## 四、等看到 ASYNC，已经晚了

很自然的想法是：既然是异步惹的祸，那就在 Filter 里补一个派发类型判断，异步场景单独处理。

```java
if (request.getDispatcherType() == DispatcherType.ASYNC) {
    ContentCachingResponseWrapper wrapper = new ContentCachingResponseWrapper(response);
    chain.doFilter(request, wrapper);
    wrapper.copyBodyToResponse();
    return;
}
```

先说最直接的问题：**这段代码根本不会被执行。** 3.3 节已经说过，直接实现 `Filter` 接口的 Bean，Spring Boot 默认只注册 `REQUEST` 派发，ASYNC 那一趟压根进不来。所以它给人的感觉是“明明加了判断，还是没用”——实际上是一行都没跑。

那退一步，把它注册上 `DispatcherType.ASYNC`，就能救回来吗？仍然不能。看这一趟请求的派发类型就够了：

```text
第 1 趟   DispatcherType = REQUEST   ← 包装器在这里创建，startAsync() 也在这里发生；
                                       之后异步线程把数据写进这个包装器
第 2 趟   DispatcherType = ASYNC     ← 容器才回来收尾，Filter 才有机会再跑一次
```

**你只有在第 2 趟才看得到 `ASYNC`，而字节早在第 1 趟之后就被异步线程写完了；写进去的还是第 1 趟那个包装器，你在这一趟 new 出来的新包装器是空的。** 想“在 ASYNC 分支里补救”，补救的对象已经不在手上了。

所以正确做法不是新建一个包装器，而是**把第 1 趟那个包装器拿回来**。

好消息是它本来就会被传回来。`StandardServletAsyncWebRequest#startAsync` 用的是 `startAsync` 的**两参数版本**，把当前的请求和响应对象（也就是 Filter 包装过的那些）显式交给了容器：

```java
this.asyncContext = getRequest().startAsync(getRequest(), getResponse());
```

Tomcat 会把这两个对象存进 `AsyncContext`，异步派发时原样交回来，于是收尾趟拿到的 `response`，就是第一次那个包装器。

要把“两趟”分开，`OncePerRequestFilter` 已经备好了两个工具方法，用的时候理解成一句话就行：

- `isAsyncDispatch(request)`：这一趟是不是 ASYNC 派发——等价于 `DispatcherType.ASYNC.equals(request.getDispatcherType())`。
- `isAsyncStarted(request)`：异步是不是已经启动、还没结束。它返回 `true` 的含义是**“后面还有一趟”**，所以这一刻不该收尾。

（这两者不是一回事：在 ASYNC 收尾趟里，`isAsyncStarted()` 反而是 `false`——异步已经结束，这一趟就是最后一趟。）

按这套“两趟分工”改写，就是标准的异步安全写法：

```java
public class TraceFilter extends OncePerRequestFilter {

    /** 收尾趟也要进来 */
    @Override
    protected boolean shouldNotFilterAsyncDispatch() {
        return false;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                    FilterChain chain) throws IOException, ServletException {
        if (isAsyncDispatch(request)) {
            // 收尾趟：response 就是第一次那个包装器，把缓存里的字节倒出去
            ContentCachingResponseWrapper wrapper =
                    WebUtils.getNativeResponse(response, ContentCachingResponseWrapper.class);
            chain.doFilter(request, response);
            if (wrapper != null) {
                this.print(wrapper, request);
                wrapper.copyBodyToResponse();
            }
            return;
        }

        // 请求趟：只包装，不收尾
        ContentCachingResponseWrapper responseWrapper =
                new ContentCachingResponseWrapper(response);
        // ... 省略 MDC 与请求包装
        chain.doFilter(request, responseWrapper);

        if (!isAsyncStarted(request)) {     // 异步已启动 → 响应还没写完，别收尾
            this.print(responseWrapper, request);
            responseWrapper.copyBodyToResponse();
        }
    }
}
```

这里用 `WebUtils.getNativeResponse` 而不是直接强转，是因为容器在异步派发时可能在外面再包一层 `ApplicationHttpResponse`；这个方法会顺着包装链找到真正的 `ContentCachingResponseWrapper`。

但**仍然不建议把它当首选修法**：它要求响应体完整躺在内存里，还依赖“异步任务在 ASYNC 派发之前写完”这个隐含时序（对本例成立，但它不是任何接口承诺的契约）。而且它只是“把 `ContentCachingResponseWrapper` 用对了”，并没有消除“先攒后发”这个模型本身的代价。

更根本的问题是：**这一整类麻烦，其实都可以通过“换个包装策略”绕开。**

## 五、Spring 自己的答案：留一个可以反悔的取流入口

回到第三节那段源码，第 ③ 行有一句很容易被当成噪音划过去的调用：

```java
ServletRequest request = webRequest.getNativeRequest(ServletRequest.class);
ShallowEtagHeaderFilter.disableContentCaching(request);          // ← 在 getBody() 之前

Assert.isInstanceOf(StreamingResponseBody.class, returnValue, "StreamingResponseBody expected");
StreamingResponseBody streamingBody = (StreamingResponseBody) returnValue;

Callable<Void> callable = new StreamingResponseBodyTask(outputMessage.getBody(), streamingBody);
```

为什么 `StreamingResponseBody` 的处理器要去动 `ShallowEtagHeaderFilter` 的开关？

因为 `ShallowEtagHeaderFilter` 用的**也是** `ContentCachingResponseWrapper`——它需要拦截响应体来计算 ETag。如果它老老实实继承 `ContentCachingResponseWrapper`，那么遇到 `StreamingResponseBody` 时同样会吞掉流。Spring 的做法不是“禁止包装”，而是给包装器留了一个后门：

```java
private static class ConditionalContentCachingResponseWrapper extends ContentCachingResponseWrapper {

    private final HttpServletRequest request;

    @Override
    public ServletOutputStream getOutputStream() throws IOException {
        return (isContentCachingDisabled(this.request) || hasETag() ?
                getResponse().getOutputStream() :   // ← 透写：直接给原始响应流
                super.getOutputStream());           // ← 缓存：给写内存的流
    }

    @Override
    public PrintWriter getWriter() throws IOException {
        return (isContentCachingDisabled(this.request) || hasETag() ?
                getResponse().getWriter() :
                super.getWriter());
    }
    // ...
}

private static boolean isContentCachingDisabled(HttpServletRequest request) {
    return (request.getAttribute(STREAMING_ATTRIBUTE) != null);
}
```

差别就在这一处，值得对照着看两遍：

- **裸的 `ContentCachingResponseWrapper`**：`getOutputStream()` 永远返回缓存流。业务代码拿到的流是“死”的，写进去就出不来——这正是本次故障。
- **`ShallowEtagHeaderFilter` 的子类**：`getOutputStream()` 在**被调用的那一刻**问一句“现在还需要缓存吗”。业务方（这里是 `StreamingResponseBodyReturnValueHandler`）已经通过 request attribute 说了“不要缓存”，它就把原始响应流交出去，数据直接透写。

**包装本身没有问题，“取流”这个动作必须留一个可以反悔的入口。** 又因为 `disableContentCaching(request)` 写在 `outputMessage.getBody()` 之前，这个入口在正确的时刻被命中。

在上一节分清 `isAsyncDispatch` 和 `isAsyncStarted` 之后，再读 `ShallowEtagHeaderFilter#doFilterInternal` 就清楚了：

```java
@Override
protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
        FilterChain filterChain) throws ServletException, IOException {

    HttpServletResponse responseToUse = response;
    if (!isAsyncDispatch(request) && !(response instanceof ConditionalContentCachingResponseWrapper)) {
        responseToUse = new ConditionalContentCachingResponseWrapper(response, request);  // ①
    }

    filterChain.doFilter(request, responseToUse);

    if (!isAsyncStarted(request) && !isContentCachingDisabled(request)) {
        updateResponse(request, responseToUse);   // ②
    }
}
```

① `isAsyncDispatch(request)`：这一趟是异步派发的收尾趟，不要再新包一层缓存——`response` 此时已经是上一趟建好的那个包装器了。

② `isAsyncStarted(request)`：`chain.doFilter` 返回时异步已经启动，说明响应还没写完，此刻读缓存、写回都是错的。

还有第三块拼图容易被漏掉：`ShallowEtagHeaderFilter` 把 `shouldNotFilterAsyncDispatch()` 重写成了 `false`：

```java
/**
 * The default value is {@code false} so that the filter may delay the generation
 * of an ETag until the last asynchronously dispatched thread.
 */
@Override
protected boolean shouldNotFilterAsyncDispatch() {
    return false;
}
```

`OncePerRequestFilter` 默认返回 `true`（跳过异步派发），这里改成 `false`，注释把目的写得很清楚：**把 ETag 的计算推迟到最后那趟异步派发线程上去做**。于是它形成“两趟分工”——请求趟只包装、不收尾，异步派发趟才收尾。

这三块拼在一起，才是 Spring 对“包装器 + 异步”的完整答案：请求趟不提前收尾，收尾趟回来把缓存倒出去，同时给取流留一个逃生口——本故障的 Filter 三块都没有。但要说清楚：**② 只能阻止“做错事”，不能让数据自动流出去。** 只加 `isAsyncStarted` 判断、继续用裸的 `ContentCachingResponseWrapper`，结果只是把“偶尔 0 字节”变成“稳定的 0 字节”。真正的解法仍是那个取流的入口。

## 六、四种修法

### 6.1 先问一句：响应体真的需要缓存吗

大多数“统一日志 Filter”对响应侧的诉求，其实是这些：状态码、耗时、字节数、`Content-Type`、业务返回的 code/message。

这些**都不需要缓存响应体**：

```java
long start = System.currentTimeMillis();
chain.doFilter(requestWrapper, response);
log.info("uri={} status={} cost={}ms", request.getRequestURI(),
        response.getStatus(), System.currentTimeMillis() - start);
```

如果只有这些诉求，直接删掉 `ContentCachingResponseWrapper` —— 零成本，也没了后面所有麻烦。

只有一种情况必须缓存：要读**响应体内容**做审计、签名、脱敏或断言。那就往下看。

### 6.2 按路径放行（临时止血）

改动最小，但靠的是白名单：

```java
private static final List<String> STREAMING_PATHS = List.of("/download", "/export");

boolean needCacheResponse = STREAMING_PATHS.stream()
        .noneMatch(prefix -> request.getRequestURI().startsWith(prefix));

if (needCacheResponse) {
    ContentCachingResponseWrapper responseWrapper = new ContentCachingResponseWrapper(response);
    try {
        chain.doFilter(requestWrapper, responseWrapper);
        // ... 日志
        responseWrapper.copyBodyToResponse();
    } catch (Exception e) {
        // 异常路径同样要写回，否则错误信息也被吞掉
        responseWrapper.copyBodyToResponse();
        throw e;
    }
} else {
    chain.doFilter(requestWrapper, response);
}
```

两个明显的短板：新增一个流式接口忘记加进白名单就会复现；`StreamingResponseBody` 可以出现在任何接口上，路径不总是可枚举的。**只适合出事之后的紧急回滚，不适合长期方案。**

> 顺便说，`copyBodyToResponse()` 一定要放在 `finally` 里或异常分支里。否则一旦下游抛异常，错误响应也会被关在缓存里，客户端只能看到一个空白的 500。

### 6.3 换成“透写 + 旁路缓存”的包装器（推荐）

既然 `ContentCachingResponseWrapper` 的模型是“先攒后发”，那就换一个“边写边复制”的模型：数据立刻透传给客户端，同时**顺带**抄一份到内存，只留前若干 KB。

```java
/**
 * 透写响应包装器：所有字节立即写往客户端，同时最多旁路记录前 {@code MAX_CAPTURE} 字节。
 */
public class TeeResponseWrapper extends HttpServletResponseWrapper {

    private static final int MAX_CAPTURE = 64 * 1024;   // 日志只留前 64KB，避免大文件撑爆堆

    private final ByteArrayOutputStream captured = new ByteArrayOutputStream();
    private TeeServletOutputStream outputStream;

    public TeeResponseWrapper(HttpServletResponse response) {
        super(response);
    }

    /** 已经记录下来的响应体片段（最多 MAX_CAPTURE 字节） */
    public byte[] getCapturedBody() {
        return captured.toByteArray();
    }

    @Override
    public ServletOutputStream getOutputStream() throws IOException {
        if (outputStream == null) {
            outputStream = new TeeServletOutputStream(
                    getResponse().getOutputStream(), captured, MAX_CAPTURE);
        }
        return outputStream;
    }

    @Override
    public PrintWriter getWriter() throws IOException {
        // 需要记录 writer 输出时同理包一层；这里直接透传
        return getResponse().getWriter();
    }

    private static class TeeServletOutputStream extends ServletOutputStream {

        private final ServletOutputStream delegate;
        private final ByteArrayOutputStream captured;
        private final int limit;

        TeeServletOutputStream(ServletOutputStream delegate, ByteArrayOutputStream captured, int limit) {
            this.delegate = delegate;
            this.captured = captured;
            this.limit = limit;
        }

        @Override
        public void write(int b) throws IOException {
            delegate.write(b);                          // ← 先保证数据出去
            if (captured.size() < limit) {
                captured.write(b);
            }
        }

        @Override
        public void write(byte[] b, int off, int len) throws IOException {
            delegate.write(b, off, len);                // ← 与 ContentCachingResponseWrapper 的关键差别
            int remain = limit - captured.size();
            if (remain > 0) {
                captured.write(b, off, Math.min(remain, len));
            }
        }

        @Override
        public void flush() throws IOException {
            delegate.flush();                           // 必须转发，否则流式下载的 TTFB 会变差
        }

        @Override
        public boolean isReady() {
            return delegate.isReady();                  // 必须转发，否则 Servlet 3.1 非阻塞写会退化
        }

        @Override
        public void setWriteListener(WriteListener writeListener) {
            delegate.setWriteListener(writeListener);   // 同上
        }
    }
}
```

Filter 侧变得非常简单，不需要 `copyBodyToResponse`：

```java
TeeResponseWrapper responseWrapper = new TeeResponseWrapper(response);
chain.doFilter(requestWrapper, responseWrapper);

String bodyPreview = new String(responseWrapper.getCapturedBody(),
        responseWrapper.getCharacterEncoding());
log.info("uri={} status={} bodyPreview={}", request.getRequestURI(),
        responseWrapper.getStatus(), StringUtils.abbreviate(bodyPreview, 2000));
```

为什么它对异步和流式都安全：

- 数据是透写的，不存在“忘记 `copyBodyToResponse` 就丢数据”的可能；
- 也不存在“Filter 返回后还有东西往缓存里追加”的问题——缓存只是一份旁路副本，写多写少都不影响主链路；
- 内存由 `MAX_CAPTURE` 兜底，大文件下载不会把堆撑爆。

代价也要说清楚：`isReady()` / `setWriteListener()` / `flush()` 必须老老实实委托给原始流，漏掉任何一个都会让异步非阻塞写出问题；`getCharacterEncoding()` 拿到的编码可能和实际响应编码不一致，用 `getCapturedBody()` 做字符串转换时要留意；另外你失去了 `getContentSize()` 这类现成方法，要记录“实际写了多少字节”得自己在 `write` 里累加。

### 6.4 异步感知 + 两趟分工（保留原语义）

也就是第四节的写法：继承 `OncePerRequestFilter`，用 `isAsyncDispatch` / `isAsyncStarted` 把请求趟和收尾趟分开，收尾趟再从包装链里取出那个 `ContentCachingResponseWrapper` 写回。只有当你已经深度依赖它的语义——比如需要拿到**完整**响应体做校验或签名——才值得考虑。代价是响应体必须能完整放进内存，而且它只是把 `ContentCachingResponseWrapper` 用对了，“先攒后发”这个模型本身的代价并没有消失。

### 6.5 简单对照

| 方案 | 解决流式 | 解决异步 | 响应体日志 | 内存风险 | 改动量 |
|:---|:---:|:---:|:---|:---|:---|
| 6.1 不包装 response | ✅ | ✅ | 元信息，无响应体 | 无 | 极小 |
| 6.2 路径白名单 | ✅ | ✅ | 完整（仅非流式接口） | 大响应仍占内存 | 小 |
| 6.3 透写 + 旁路缓存 | ✅ | ✅ | 前 64KB | 可控 | 中 |
| 6.4 两趟分工 + 原包装器 | ✅ | ✅ | 完整 | 必须放得下内存 | 中 |
| 只加 `isAsyncStarted` 判断 | ❌ | ❌ | 无 | 大响应仍占内存 | 极小 |

## 七、这类 bug 的一般形态

把这次故障抽象一下，它的触发条件是三个条件同时成立：

```text
环绕式 Filter                 响应包装器                    异步 / 流式返回
（chain.doFilter 之后           （延迟写出：write 只进        （写响应的线程和时间
  还有逻辑）                      缓存，靠一个显式动作落盘）      都不再受 Filter 控制）
```

三个缺一不可：

- 只有 Filter 没有包装器 → 无事发生；
- 只有包装器没有异步 → `copyBodyToResponse()` 在正确的时机执行，无事发生；
- 三个凑齐 → 写出响应体的那个线程，和负责“把缓存倒出去”的那个线程，是两个线程，而后者只跑一次。

由此推出一条可以直接用在 code review 里的规则：

> **只要你在 Filter 里包装了 `HttpServletResponse`，就必须假定 `chain.doFilter` 返回时响应可能还没写完。**

判断方法就两个调用：

```java
request.isAsyncStarted()          // true 说明容器线程已经把写响应的责任交出去了
request.getDispatcherType()       // REQUEST / ASYNC / ERROR / FORWARD / INCLUDE
```

能触发这个问题的返回类型不只 `StreamingResponseBody`：`ResponseBodyEmitter`、`SseEmitter`、`Callable<ResponseEntity<...>>`、`DeferredResult<...>`、以及返回 `CompletableFuture` 的响应式适配，凡是走 Servlet 异步的，都在射程内。

## 八、排查这类问题的工具箱

### 8.1 断点位置

| 想看什么 | 断在哪里 |
|:---|:---|
| `StreamingResponseBody` 拿到的流是谁 | `StreamingResponseBodyReturnValueHandler#handleReturnValue` 里的 `outputMessage.getBody()` |
| 数据到底写去了哪 | `ContentCachingResponseWrapper.ResponseServletOutputStream#write` |
| 谁先谁后 | `WebAsyncManager#startAsyncProcessing` 与 `StreamingResponseBodyTask#call` |
| 缓存写回了几次 | `ContentCachingResponseWrapper#copyBodyToResponse` |

### 8.2 命令行快速判定

```bash
curl -v -o /tmp/out.bin "http://localhost:8080/download"
ls -l /tmp/out.bin /tmp/big.zip
```

看两件事：

- `Content-Length` 等于 0，或者响应里压根没有 `Content-Length` / `Transfer-Encoding` → 时间线 A，缓存里一个字节都没有；
- `Content-Length` 是个远小于文件大小的正数，且落盘字节数正好等于它 → 时间线 B，缓存被提前写回了。

第二种情况最值得警惕：HTTP 层面完全自洽，只有对比源文件大小才能发现。

### 8.3 两行日志

在 Filter 里补上：

```java
log.info("uri={} dispatcherType={} asyncStarted={} cachedSize={}",
        request.getRequestURI(), request.getDispatcherType(), request.isAsyncStarted(),
        responseWrapper.getContentSize());
```

出问题的下载请求会打出这样的组合：`dispatcherType=REQUEST`（异步派发根本没再进 Filter）、`asyncStarted=true`（容器线程已经交棒）、`cachedSize=0`（缓存还是空的，但数据马上就要写进来）。

三个值放一起看，基本一眼定位。

## 九、回到最初的问题

- **状态码和头都对，为什么没有字节？** 因为头是写在 `ContentCachingResponseWrapper` 上的，它会透传给原始响应；而响应体被 `write` 关进了 `FastByteArrayOutputStream`，唯一的出口 `copyBodyToResponse()` 执行时缓存还是空的。
- **流被谁接走了？** 被 `StreamingResponseBodyReturnValueHandler` 取走的 `outputMessage.getBody()`——它在 Controller 返回的那一刻，就从当前 response（也就是包装器）上要走了缓存流，然后交给异步线程。
- **为什么时好时坏？** `startAsync()` 在 `chain.doFilter` 返回之前就执行了，Filter 的收尾代码和异步任务并发，谁先谁后看调度。
- **加 `DispatcherType.ASYNC` 判断能救吗？** 光加判断不够——ASYNC 派发时数据早就写在第一次那个包装器里了，新建一个包装器只会拿到空的。得按“两趟分工”来：请求趟只包装、不收尾，收尾趟取出第一次那个包装器写回。
- **正确写法是什么？** 要么不缓存响应体，要么用“透写 + 旁路缓存”的包装器；Spring 自己在 `ShallowEtagHeaderFilter` 里给出的范式是——包装可以，但取流入口必须能反悔。

最后留一句给所有写统一 Filter 的人：**Filter 是环绕式的，但“环绕”的终点不等于“响应写完了”。** 在 Servlet 异步面前，容器线程的 `chain.doFilter` 返回只是一个中转站，不是终点站。

## 延伸阅读

- [Spring 文件下载全方案深度解析](/2026/09/15/Spring-文件下载全方案深度解析/)——3.4 节讲 `StreamingResponseBody` 把阻塞转移到哪里，3.5 节讲 `DeferredResult` 与它的叠加用法，正好是本文的前置知识。
- [Spring MVC 中那些“不走寻常路”的返回类型](/2026/09/14/Spring-MVC-中那些“不走寻常路”的返回类型/)——8.3 节把 Servlet 3.0 `AsyncContext` 的两次派发讲透了，理解“ASYNC 派发只是回来收尾”要靠它。
- [SpringBoot 的一次完整请求发生了什么？](/2026/08/26/SpringBoot的一次完整请求发生了什么？/)——第五节是 Filter 层，第十二节是异步请求，可以对着看 Filter 在请求生命周期里的位置。
