---
title: Spring MVC 中那些“不走寻常路”的返回类型
date: 2026-09-14 16:44:12
tags:
  - spring
categories:
  - develop
---

# Spring MVC 中那些“不走寻常路”的返回类型：完整指南

## 一、从“普通返回”说起

在 Spring MVC 中，最熟悉的控制器返回值是这样的：

```java
@GetMapping("/user")
public User getUser() {
    return userService.findById(1L);
}
```

Spring 拿到 `User` 对象，用 Jackson 序列化成 JSON，写入响应。响应是**一次性的、同步的、内容完整的**。

这个过程的背后，是 Tomcat 从线程池中分配了一个**工作线程**，这个线程从接收请求、执行 Controller、序列化结果、写入响应，一直到响应写完，才归还给线程池。

**问题在于**：如果 `findById` 需要 30 秒（比如调用一个慢速的 AI 服务），那么这个工作线程就要被占用 30 秒。Tomcat 默认最大线程数是 200，意味着 200 个这样的请求就能把 Tomcat 打满，第 201 个请求只能排队等待。

Spring MVC 提供了一系列特殊返回类型来应对不同的响应模式。它们从不同维度偏离了“普通序列化返回”：

- **时间维度**：结果不是立即产生的，需要异步等待
- **次数维度**：响应不止写入一次，需要流式推送
- **控制维度**：需要精确控制状态码、响应头，或完全手写响应
- **路径维度**：走视图渲染而非序列化
- **转换器维度**：走专用转换器而非 JSON 序列化
- **模型维度**：使用响应式编程模型

下面逐一拆解，并把底层原理自然地嵌入到每一类中。


## 二、异步单值返回类型

### 2.1 核心思想：让 Tomcat 线程先回去

异步单值类型解决的是“结果来得慢”的问题。它的核心思想只有一句话：

> **让 Tomcat 工作线程先回去处理别的请求，等结果好了再派一个新的线程回来写响应。**

打个比方：你去餐厅点了一道需要慢炖两小时的菜。服务员（工作线程）不会站在你桌边干等两小时，他先把订单交给厨房（业务线程），然后去服务其他客人。等菜做好了，再派一个服务员端上来。

### 2.2 DeferredResult：一张“取餐凭证”

```java
@GetMapping("/deferred")
public DeferredResult<String> deferred() {
    DeferredResult<String> result = new DeferredResult<>(5000L);
    executor.submit(() -> {
        String value = slowService.compute();
        result.setResult(value);
    });
    return result;
}
```

控制器返回 `DeferredResult` 后，Tomcat 工作线程**立即释放**。等到 `executor` 线程算完并调用 `setResult()`，Spring 会让请求重新进入处理流程，把结果写成响应。

**底层发生了什么？**

当 `DispatcherServlet` 识别到返回值是 `DeferredResult` 时，Spring MVC 内部会调用 `request.startAsync()`。这个调用会触发 Tomcat 内部一个叫 `AsyncStateMachine` 的状态机，把请求的状态从 `DISPATCHED`（同步处理中）切换为 `STARTING`，表示“这个请求已经进入异步模式”。

随后，`DispatcherServlet` 的 `service()` 方法执行完毕，状态机变为 `STARTED`。此时，**Tomcat 会把承载这个请求的 `Http11Processor` 对象放入一个叫 `waitingProcessors` 的集合**，然后释放当前工作线程。TCP 连接保持打开，由 Tomcat 的 Poller 线程通过 `Selector` 持续监控。

当业务线程调用 `deferredResult.setResult(value)` 时，Spring 通过 `AsyncContext.dispatch()` 触发重新派发。Tomcat 的 Poller 检测到该连接的写事件后，**从工作线程池中取出一个新的线程**，让它重新执行请求处理。这次 `DispatcherServlet` 看到的是 `DispatcherType.ASYNC`，不会重新执行 Controller，而是直接取出 `DeferredResult` 中已设置的值，经 `HttpMessageConverter` 序列化后，以阻塞方式写入响应。

**一句话总结**：`DeferredResult` 是“一次性结果交付”，它的写入发生在**重新派发后的新工作线程**中，写入一次完整结果。

### 2.3 Callable 和 WebAsyncTask

```java
@GetMapping("/callable")
public Callable<String> callable() {
    return () -> slowService.compute();
}
```

`Callable` 更省事——Spring 会自动把它提交到一个 `AsyncTaskExecutor` 线程池去执行。执行完毕后，同样通过 `dispatch()` 回到 Servlet 容器，一次性写入结果。

`WebAsyncTask` 是 `Callable` 的增强版，额外提供了超时时间和回调：

```java
@GetMapping("/webAsyncTask")
public WebAsyncTask<String> webAsyncTask() {
    WebAsyncTask<String> task = new WebAsyncTask<>(3000L, () -> slowService.compute());
    task.onTimeout(() -> "任务超时");
    task.onError(() -> "任务出错");
    return task;
}
```

**一个容易踩的坑**：`onTimeout` 的返回值是最终返回给客户端的内容，但 `Callable` **不会被中断**，它会继续跑完（结果被丢弃）。

### 2.4 CompletableFuture 和 ListenableFuture

`CompletableFuture<T>`（Spring 4.2+）和 `ListenableFuture<T>`（Spring 4.1+）用法与 `DeferredResult` 类似。Spring 内部会把它们**适配为 `DeferredResult`**，当 Future 完成时，结果被设置到内部的 `DeferredResult` 上，触发相同的重新派发流程。

### 2.5 异步单值的共同特征

| 类型 | 谁负责执行异步任务 | 超时控制 | 最终写入方式 |
| :--- | :--- | :--- | :--- |
| `DeferredResult` | 你自己（任意线程） | 构造时设置 | 重新派发后一次性写入 |
| `Callable` | Spring 的 `AsyncTaskExecutor` | 全局配置或 `WebAsyncTask` | 重新派发后一次性写入 |
| `WebAsyncTask` | Spring 的 `AsyncTaskExecutor` | 构造时设置 + 回调 | 重新派发后一次性写入 |
| `CompletableFuture` | 你创建时指定的线程池 | 无内置，需自行处理 | 适配为 `DeferredResult` |
| `ListenableFuture` | 你创建时指定的线程池 | 无内置 | 适配为 `DeferredResult` |

它们最终都会 `dispatch()` 回 Servlet 容器，**一次性写入响应**。这就是“异步单值”的含义。


## 三、流式返回类型

### 3.1 从“一次性倒水”到“开水龙头”

异步单值类型解决的是“结果来得慢”，但最终**只返回一个结果**。如果你要做一个 AI 聊天，希望 AI 每生成一个字就推给前端，就需要在一个响应里**多次写入**。

| 对比 | 普通响应 | 流式响应 |
| :--- | :--- | :--- |
| 比喻 | 一杯水，一次性倒给顾客 | 一个水龙头，一段一段地放水 |
| HTTP 层面 | 先声明 `Content-Length` 再发 body | 使用分块传输编码，不预先声明长度 |
| 代码层面 | 返回一个对象，序列化一次 | 多次调用 `send()`，每次写入一块 |
| 线程模型 | 同步或异步单值写入 | 保持异步上下文开放，随时可写 |

### 3.2 ResponseBodyEmitter：通用流式发送器

```java
@GetMapping("/stream")
public ResponseBodyEmitter stream() {
    ResponseBodyEmitter emitter = new ResponseBodyEmitter(30_000L);
    executor.submit(() -> {
        try {
            for (int i = 1; i <= 5; i++) {
                emitter.send("第 " + i + " 条消息\n");
                Thread.sleep(1000);
            }
            emitter.complete();
        } catch (Exception e) {
            emitter.completeWithError(e);
        }
    });
    return emitter;
}
```

**关键点**：
- 构造时传入超时时间，`-1` 或 `0` 表示永不超时
- 每次 `send()` 的对象都会经过 `HttpMessageConverter` 序列化
- **必须调用 `complete()` 或 `completeWithError()`**，否则连接挂到超时
- 适合 AI 流式输出、NDJSON 等自定义格式

**底层原理**：与 `DeferredResult` 共享同一个起点——`request.startAsync()`。`AsyncStateMachine` 同样经历 `STARTING → STARTED`，`Http11Processor` 同样被放入 `waitingProcessors`。

但接下来的路径不同。当业务线程调用 `emitter.send(data)` 时，数据被序列化后写入一个中间缓冲区。Spring 检测到 Tomcat 支持非阻塞 I/O 时，会调用 `ServletOutputStream.setWriteListener()` 注册一个回调。

Tomcat 的 **Poller 线程**检测到 Socket 可写时，回调 `WriteListener.onWritePossible()`。在这个回调中，缓冲区中的数据通过 `ServletOutputStream.write()` 写入 Socket。如果一次写不完，`onWritePossible()` 会在下次可写时再次被调用。**整个过程由 Poller 线程驱动，不需要工作线程参与**。

调用 `emitter.complete()` 时，`AsyncStateMachine` 进入 `COMPLETING` 状态，最终回到 `DISPATCHED`。

**分块传输编码（Chunked Transfer Encoding）** 是这一切的协议基础。普通响应必须先告诉客户端“我要发 1000 字节”，然后一次性发完。分块编码则允许服务器说“我将分块发送，每块自带大小”，最后一个大小为 0 的块表示结束。这使得服务器可以在不知道整体长度的情况下开始传输。

### 3.3 SseEmitter：SSE 协议专用

```java
@GetMapping("/sse")
public SseEmitter sse() {
    SseEmitter emitter = new SseEmitter(30_000L);
    executor.submit(() -> {
        try {
            for (int i = 1; i <= 5; i++) {
                emitter.send(SseEmitter.event()
                        .id(String.valueOf(i))
                        .name("message")
                        .data("第 " + i + " 条推送"));
                Thread.sleep(1000);
            }
            emitter.complete();
        } catch (Exception e) {
            emitter.completeWithError(e);
        }
    });
    return emitter;
}
```

`SseEmitter` 继承自 `ResponseBodyEmitter`，底层写入机制完全相同。唯一的区别是：它自动设置 `Content-Type: text/event-stream`，并在 `send()` 时将数据包装为 SSE 格式（`id:`、`event:`、`data:`）。浏览器 `EventSource` API 原生支持，自动断线重连。

### 3.4 StreamingResponseBody：直接写字节

```java
@GetMapping("/download")
public ResponseEntity<StreamingResponseBody> download() {
    StreamingResponseBody body = out -> {
        for (int i = 0; i < 1_000_000; i++) {
            out.write(("line " + i + "\n").getBytes(StandardCharsets.UTF_8));
        }
    };
    return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=data.txt")
            .body(body);
}
```

`StreamingResponseBody` 是函数式接口，**完全绕过 `HttpMessageConverter`**，直接把 `OutputStream` 递给你写原始字节。适合大文件下载、CSV 导出等场景。

**底层原理**：它的处理器同样调用 `startAsync()`，但**不会将请求放入 `waitingProcessors` 等待 dispatch**。它把你的 `writeTo(OutputStream)` 方法提交到一个 `TaskExecutor` 中异步执行。在这个异步线程中，你拿到 `ServletOutputStream`，以**阻塞方式**直接写入数据。没有 `WriteListener`，没有 Poller 回调。

写入完成后，调用 `AsyncContext.complete()`，`AsyncStateMachine` 从 `STARTED` 进入 `COMPLETING`，回到 `DISPATCHED`。

**与 `ResponseBodyEmitter` 的关键区别**：`ResponseBodyEmitter` 可以多次 `send()`，且写入可以由 Poller 线程以非阻塞方式驱动；`StreamingResponseBody` 是一次性连续写入，且写入发生在你的业务线程中（阻塞方式）。

### 3.5 流式类型的共同特征

| 类型 | 写入方式 | 写入线程 | I/O 模型 | 典型场景 |
| :--- | :--- | :--- | :--- | :--- |
| `ResponseBodyEmitter` | 多次 `send()` | Poller 线程（非阻塞）或工作线程（退化） | 非阻塞优先 | AI 流式输出 |
| `SseEmitter` | 多次 `send()`，SSE 格式 | 同上 | 非阻塞优先 | 服务器推送 |
| `StreamingResponseBody` | 直接写 `OutputStream` | 业务线程 | 阻塞 | 大文件下载 |


## 四、直接控制响应的返回类型

### 4.1 ResponseEntity

```java
@GetMapping("/user/{id}")
public ResponseEntity<User> getUser(@PathVariable Long id) {
    return userRepository.findById(id)
            .map(user -> ResponseEntity.ok()
                    .header("X-Custom-Header", "value")
                    .body(user))
            .orElse(ResponseEntity.notFound().build());
}
```

`ResponseEntity` 允许你精确控制**状态码、响应头、响应体**三样东西。响应体部分仍然经过 `HttpMessageConverter` 序列化。

### 4.2 ResponseEntity<Void>

```java
@DeleteMapping("/user/{id}")
public ResponseEntity<Void> delete(@PathVariable Long id) {
    userRepository.deleteById(id);
    return ResponseEntity.noContent().build(); // 204
}
```

返回 `void` 时 Spring 默认给 200，但 DELETE 操作你想返回 204，就用 `ResponseEntity<Void>`。

### 4.3 HttpHeaders 和 void + HttpServletResponse

```java
@GetMapping("/headers-only")
public HttpHeaders headersOnly() {
    HttpHeaders headers = new HttpHeaders();
    headers.set("X-Custom-Header", "value");
    return headers;
}

@GetMapping("/raw")
public void raw(HttpServletResponse response) throws IOException {
    response.setStatus(418);
    response.getWriter().write("{\"message\":\"I'm a teapot\"}");
}
```

### 4.4 底层原理：HttpEntityMethodProcessor

`ResponseEntity` 的处理器是 `HttpEntityMethodProcessor`，它继承自 `AbstractMessageConverterMethodProcessor`。它的处理逻辑是：

1. 从 `ResponseEntity` 中提取状态码和响应头，设置到 `HttpServletResponse` 上
2. 提取响应体
3. 如果 body 不为 `null`，遍历注册的 `HttpMessageConverter`，找到能处理该类型和 `MediaType` 的转换器，序列化后写入响应
4. 如果 body 是 `Void`，跳过第 3 步，只设置状态码和响应头

**一个重要细节**：`HttpEntityMethodProcessor` 在处理器列表中的位置**很靠前**。这是为了防止 `@ResponseBody` 的处理器抢先处理，导致状态码和响应头设置失效。

**void + HttpServletResponse 的处理**：Spring MVC 的返回值处理链会**提前终止**。`ModelAndViewContainer.setRequestHandled(true)` 被设置，表示“响应已经被直接处理了，不需要后续的视图渲染或序列化”。


## 五、视图与模型返回类型

### 5.1 示例

```java
@GetMapping("/home")
public String home(Model model) {
    model.addAttribute("user", userService.currentUser());
    return "home";
}

@GetMapping("/home")
public ModelAndView home() {
    ModelAndView mav = new ModelAndView("home");
    mav.addObject("user", userService.currentUser());
    return mav;
}

@GetMapping("/home")
public Map<String, Object> home() {
    return Map.of("user", userService.currentUser());
}
```

### 5.2 底层原理：MapMethodProcessor 的“只存不设”

以 `MapMethodProcessor` 为例，它的 `handleReturnValue` 做的事情很简单：**把 Map 里的键值对放进 `ModelAndViewContainer` 的 Model 中**，但**不设置视图名**，也**不调用 `setRequestHandled(true)`**。

这意味着处理还会继续——后续的处理器或默认逻辑会根据请求路径推导出视图名，最终由 `ViewResolver` 解析成具体的模板（JSP、Thymeleaf 等），渲染成 HTML。

这类返回类型的共同点是：**响应内容不由序列化产生，而由模板引擎渲染产生**。


## 六、资源与原始数据类型

### 6.1 Resource

```java
@GetMapping("/image")
public ResponseEntity<Resource> image() throws IOException {
    Resource resource = new ClassPathResource("images/logo.png");
    return ResponseEntity.ok()
            .contentType(MediaType.IMAGE_PNG)
            .contentLength(resource.contentLength())
            .body(resource);
}
```

常见实现：`ByteArrayResource`、`InputStreamResource`、`FileSystemResource`、`ClassPathResource`、`UrlResource`。

### 6.2 底层原理

`Resource` 由 `ResourceHttpMessageConverter` 处理。它的逻辑是：**从 `Resource` 的 `InputStream` 读字节，写到响应的 `OutputStream`**，每次读写 4096 字节。

它还支持 **HTTP Range 请求**（断点续传）——客户端说“我要第 1000 到 2000 字节”，它就只发这一段。

它本质上仍然是一个 `HttpMessageConverter`，只是在“普通序列化”和“原始字节流”之间找到了一个专用通道。


## 七、响应式类型：在 MVC 中的“伪装”

### 7.1 示例

```java
@GetMapping("/mono")
public Mono<String> mono() {
    return Mono.fromCallable(() -> slowService.compute());
}

@GetMapping(value = "/flux", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> flux() {
    return Flux.interval(Duration.ofSeconds(1)).map(i -> "第 " + i + " 条");
}
```

### 7.2 关键警告

在 Spring MVC（基于 Servlet 栈）中，返回 `Mono` 或 `Flux` **并不会启用真正的非阻塞 I/O 和背压机制**。Spring 只是把它们当异步包装用，底层仍然是阻塞式 Servlet 模型。

`Mono` 会被适配为 `DeferredResult`，`Flux` 会被适配为 `ResponseBodyEmitter` 或类似机制。Spring 的 `ResponseBodyEmitterReturnValueHandler` 内部有一个 `ReactiveTypeHandler`，专门用于处理响应式类型的适配。

**如果你需要真正的非阻塞响应式，用 Spring WebFlux，别在 MVC 里硬凑。**


## 八、统一原理：三个核心机制

所有这些返回类型能工作，靠的是三个机制：

### 8.1 分拣中心：HandlerMethodReturnValueHandler 策略链

Spring MVC 处理返回值靠的是 `HandlerMethodReturnValueHandler` 策略接口。Spring MVC 启动时会注册一个**有序列表**，每种返回类型都有对应的处理器实现。`HandlerMethodReturnValueHandlerComposite` 会遍历这个列表，找到第一个 `supportsReturnType` 返回 `true` 的处理器，把处理权交给它。这是**责任链设计模式**。

**顺序至关重要**。比如 `HttpEntityMethodProcessor` 被明确配置在 `@ResponseBody` 的处理器之前，否则 `ResponseEntity` 就会被后者抢先处理。

### 8.2 电话转接：Servlet 3.0 AsyncContext

异步和流式类型都调用 `request.startAsync()`，让 Tomcat 工作线程释放。

- **异步单值类型**：等结果好了，调用 `AsyncContext.dispatch()` 把请求“转接”回来，**一次性写响应**，然后结束。
- **流式类型**：保持 `AsyncContext` 开放，随时可以调用 `send()` 往响应里写数据，直到调用 `complete()`。

两者共享同一个根基，区别只在于**是否在 dispatch 回来之前写入了数据**。

在 Tomcat 内部，这一切由 `AsyncStateMachine` 状态机跟踪。每个请求对应的 `Http11Processor` 都持有一个状态机实例。当请求进入异步等待时，`Http11Processor` 被放入 `waitingProcessors` 集合，等待后续的 dispatch 或 complete 事件。

### 8.3 分块快递：HTTP Chunked Encoding

普通响应必须先声明 `Content-Length`，然后一次性发送 body。分块编码则允许服务器在不知道整体内容长度的情况下开始传输响应，每个数据块都包含自身的大小。这就是 `ResponseBodyEmitter` 能多次 `send()` 的协议基础。

`StreamingResponseBody` 虽然也基于分块传输（或直接使用 OutputStream），但它是**一次性连续写入**，不涉及多次 `send()` 的调度。


## 九、完整对比表

| 返回类型 | 处理方式 | 走 HttpMessageConverter | 写入次数 | 底层机制 | 典型场景 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `DeferredResult<T>` | 异步，任意线程 setResult | 是（对 T） | 1 次 | startAsync + dispatch | 长轮询、回调 |
| `Callable<T>` | 异步，AsyncTaskExecutor | 是（对 T） | 1 次 | startAsync + dispatch | 简单异步 |
| `WebAsyncTask<T>` | 异步 + 超时/回调 | 是（对 T） | 1 次 | startAsync + dispatch | 异步 + 控制 |
| `CompletableFuture<T>` | 异步，Future 完成 | 是（对 T） | 1 次 | 适配为 DeferredResult | Java 8 异步 |
| `ListenableFuture<T>` | 异步，适配为 DeferredResult | 是（对 T） | 1 次 | 适配为 DeferredResult | Spring 异步 |
| `ResponseBodyEmitter` | 流式，多次 send | 是（每次 send） | N 次 | startAsync + Chunked | 流式输出 |
| `SseEmitter` | 流式，SSE 协议 | 是（包装为 SSE） | N 次 | startAsync + Chunked + SSE | 服务器推送 |
| `StreamingResponseBody` | 流式，直接写 OutputStream | **否** | 1 次（连续写） | startAsync + 直接写流 | 文件下载/导出 |
| `ResponseEntity<T>` | 控制状态码/头/体 | 是（对 body） | 1 次 | HttpEntityMethodProcessor | REST API |
| `ResponseEntity<Void>` | 控制状态码 | 否 | 0 次 | HttpEntityMethodProcessor | DELETE |
| `HttpHeaders` | 仅响应头 | 否 | 0 次 | HttpHeadersReturnValueHandler | 仅返回头 |
| `void` + HttpServletResponse | 手动处理 | 否 | 手动 | setRequestHandled(true) | 特殊场景 |
| `String` / `ModelAndView` / `View` | 视图渲染 | 否 | 1 次（视图渲染） | ViewResolver | 页面渲染 |
| `Map` / `Model` | 模型数据 | 否 | 1 次（视图渲染） | MapMethodProcessor | 页面渲染 |
| `@ModelAttribute` | 模型属性 | 否 | 1 次（视图渲染） | 模型属性绑定 | 页面渲染 |
| `Resource` | ResourceHttpMessageConverter | 是（专用） | 1 次 | 专用转换器（支持 Range） | 文件下载 |
| `Mono<T>` / `Flux<T>` | 响应式包装 | 是 | 1 次或 N 次 | 适配为 DeferredResult/ResponseBodyEmitter | 响应式（MVC 中非真非阻塞） |

**选择指南**：

| 你遇到的问题 | 该用的类型 |
| :--- | :--- |
| 结果由其他线程/回调产生 | `DeferredResult` |
| 想异步，不想管线程池 | `Callable` |
| 异步 + 超时 + 回调 | `WebAsyncTask` |
| 多次推送数据，自定义格式 | `ResponseBodyEmitter` |
| 多次推送数据，浏览器 EventSource 接收 | `SseEmitter` |
| 一次性输出大量原始字节 | `StreamingResponseBody` |
| 控制状态码/响应头，有响应体 | `ResponseEntity<T>` |
| 控制状态码，无响应体 | `ResponseEntity<Void>` |
| 传统页面渲染 | `ModelAndView` / `String` |
| 真正的非阻塞响应式 | 换 Spring WebFlux |


## 十、结语

这些“非普通序列化返回”的类型，本质上都是 Spring MVC 在不同维度上对 HTTP 响应模型的扩展：

- **时间维度**：`DeferredResult`、`Callable`、`WebAsyncTask`、`CompletableFuture`、`ListenableFuture`
- **次数维度**：`ResponseBodyEmitter`、`SseEmitter`、`StreamingResponseBody`
- **控制维度**：`ResponseEntity`、`ResponseEntity<Void>`、`HttpHeaders`、`void`
- **路径维度**：`ModelAndView`、`View`、`String`、`Map`、`@ModelAttribute`
- **转换器维度**：`Resource` 及其子类
- **模型维度**：`Mono`、`Flux`

理解它们的关键，不是记住每个类的用法，而是理解它们各自解决的是什么维度的响应控制问题，以及背后的三个核心机制：**策略链**（分拣中心）、**AsyncContext**（电话转接）、**Chunked Encoding**（分块快递）。

而在 Tomcat 内部，这一切由 `AsyncStateMachine` 状态机跟踪，由 `waitingProcessors` 集合管理等待中的异步请求，由 `Http11Processor` 承载每个请求的完整生命周期，最终通过 `ServletOutputStream` 完成字节的写出。区别只在于：**谁在什么线程上、以什么方式调用 `ServletOutputStream`**。

一旦掌握了这个框架，选择就变成了一件自然而然的事。