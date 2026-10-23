---
title: "Spring 文件上传全链路深度解析（二）：限额、故障与不落盘直传"
date: 2026-10-23 10:50:01
tags:
  - spring
  - tomcat
categories:
  - develop
---

浏览器控制台一行 `net::ERR_CONNECTION_RESET`，Network 面板里那条 `POST /api/upload` 停在待响应状态，状态码是空的。翻后端日志——`INFO` 翻到 `DEBUG`，一行异常都没有。把同一个文件切成 5 MB 分片再传，一切正常。

> - 一个上传请求，从 Nginx 到对象存储，中间有几道限额？分别由谁拦下？
> - 为什么被拒绝之后，客户端看到的是"连接被重置"而不是一个 413？
> - `maxSwallowSize` 是干什么的？为什么它能让应用连一句日志都留不下？
> - `@ExceptionHandler` 明明写了，为什么没被调用？
> - 上传一定要先落到本地临时文件吗？能不能边收边写进对象存储？
> - 预签名 URL 直传和"应用流式转发"差在哪里，代价是什么？
> - `MultipartFile.getOriginalFilename()` 到底能不能信？

这篇接着 [（一）：从报文到临时文件](/2026/10/23/Spring-文件上传全链路深度解析（一）：从报文到临时文件/) 往下走。第一篇讲报文怎么被解析、`MultipartFile` 从哪里来；这一篇只关心一件事：**这套机制在真实系统里以什么方式失败，以及怎么绕开它。** 口径是 Spring Boot 3.2+ / Spring Framework 6.1 / Tomcat 10.1，2.x 与 3.0/3.1 的差异单独标注。数据路径的架构选择可以和姊妹篇 [Spring 文件下载全方案深度解析](/2026/09/15/Spring-文件下载全方案深度解析/) 的「五、架构层」对着看：下载是把数据踢出应用进程，上传是把数据挡在应用进程之外，方向相反，取舍同源。

## 六、四道限额，四种失败表现

### 6.1 全链路图

```text
浏览器
  │  ① client_max_body_size        默认 1m      拦在 Nginx，返回 413
  ▼
Nginx / 网关
  │  ② maxPostSize                 默认 2 MiB   拦在 Tomcat，只算表单字段
  │     maxParameterCount          默认 10000
  │     maxPartCount               默认 50
  ▼
Tomcat Connector ──► ③ spring.servlet.multipart.max-request-size   默认 10 MB
  │                    spring.servlet.multipart.max-file-size      默认 1 MB
  ▼
DispatcherServlet ──► ④ maxSwallowSize  默认 2 MiB  拦在连接层，没有状态码
```

最反直觉的一点：**越靠外的限制，越容易让客户端拿不到一个像样的响应。**

### 6.2 第一道：Nginx 的 `client_max_body_size`

```nginx
location /api/upload {
    client_max_body_size 20m;   # 默认 1m，不显式放大就只能传 1MB
}
```

默认 `1m` 来自 [Nginx 官方文档](https://nginx.org/en/docs/http/ngx_http_core_module.html#client_max_body_size)。它可写在 `http` / `server` / `location` 三层，**但只在请求真正经过的那一层生效**——放在 `http` 里被内层 `location` 覆盖是很常见的翻车点。

超限时 Nginx 自己返回 `413 Request Entity Too Large`，**请求根本不会到后端**。文档还提醒：浏览器没法正确显示这个错误——这正是后面要把它改写成 JSON 的原因。

### 6.3 第二道：Tomcat Connector 的几个数

| 属性 | 默认值 | 限制的是什么 |
|:---|:---|:---|
| `maxPostSize` | `2097152`（2 MiB） | 会被转换成请求参数的请求体字节数 |
| `maxParameterCount` | `10000` | 参数 + 上传文件的总个数，超出部分被忽略 |
| `maxPartCount` | `50` | `multipart/form-data` 的 part 个数上限 |
| `maxPartHeaderSize` | `512` | 每个 part 的头部字节上限，超出直接拒绝 |
| `maxSwallowSize` | `2097152`（2 MiB） | 连接层"吞掉"未读请求体的上限 |

`maxParameterCount` 有个必须交代的来源：`Connector` 源码的字段注释写着 **"10000 by default. The default Tomcat server.xml configures a lower default of 1000."** Tomcat 自带的 `server.xml` 会把它调成 1000，而 **Spring Boot 内嵌容器不读 `server.xml`**，所以内嵌场景用 10000，外置 Tomcat 场景（沿用官方 `server.xml`）是 1000。照着外置经验对日志会对不上，原因就在这里。

`maxPostSize` 是最大的坑：Tomcat 文档明确说它**不是请求体总大小的通用上限**，只在 Tomcat 为解析参数而处理请求体时生效。落到 `multipart/form-data` 上，它只约束"用来生成 `getParameter()` 那批数据"，也就是普通表单字段：

```java
// org.apache.catalina.connector.Request#parseParts（Tomcat 10.1.x）
if (part.getSubmittedFileName() == null) {          // ← 没有文件名 = 普通表单字段
    if (maxPostSize >= 0) {
        // 原码用 Math.addExact 逐项累加：name 字节数 + 1 + value 字节数 + 1
        postSize = name.getBytes(charset).length + 1 + part.getSize() + 1;
        if (postSize > maxPostSize) {
            parameters.setParseFailedReason(FailReason.POST_TOO_LARGE);
            throw new IllegalStateException(sm.getString("coyoteRequest.maxPostSizeExceeded"));
        }
    }
} else {
    maxParameterCount--;                            // ← 文件 part 只扣参数计数，不进 postSize
}
```

**文件部分不算进 `maxPostSize` 的账。** 把 `server.tomcat.max-http-form-post-size` 从 2 MB 调到 50 MB，对"用户传了一个 30 MB 的文件"毫无帮助。

Spring Boot 只暴露了其中两个（默认值来自 Boot 3.2 的 `ServerProperties`，映射在 `TomcatWebServerFactoryCustomizer`）：

```yaml
server:
  tomcat:
    max-swallow-size: 2MB          # → AbstractHttp11Protocol#setMaxSwallowSize
    max-http-form-post-size: 2MB   # → Connector#setMaxPostSize
```

**`maxParameterCount` 和 `maxPartCount` 没有对应的 Boot 属性**，要自己加 `TomcatConnectorCustomizer`：

```java
factory.addConnectorCustomizers(connector -> {
    connector.setMaxPartCount(100);          // 默认 50
    connector.setMaxParameterCount(5000);    // 内嵌默认 10000
});
```

调大前先算账。Tomcat 文档的内存估算公式是 `maxPartHeaderSize × maxPartCount × maxConnections × 2`——按默认值：`512 × 50 × 8192 × 2`，约 **400 MB**。这是文档明确提示的 DoS 面。

### 6.4 第三道：Spring 的 multipart 配置

```yaml
spring:
  servlet:
    multipart:
      max-file-size: 10MB        # 默认 1MB
      max-request-size: 50MB     # 默认 10MB
      file-size-threshold: 0B     # 默认 0B：几乎所有文件都直接落盘
      location: /data/upload-tmp
      resolve-lazily: false      # 默认 false：进入 DispatcherServlet 就解析
```

默认值来自 `MultipartProperties`（`maxFileSize = DataSize.ofMegabytes(1)`、`maxRequestSize = DataSize.ofMegabytes(10)`、`fileSizeThreshold = DataSize.ofBytes(0)`、`resolveLazily = false`）。它们被写进 `MultipartConfigElement`，再交给 Tomcat 内嵌的 fileupload：

```java
// Request#parseParts
upload.setFileSizeMax(mce.getMaxFileSize());   // → max-file-size
upload.setSizeMax(mce.getMaxRequestSize());    // → max-request-size
```

`StandardServletMultipartResolver` 的类注释里有一句关键的话：**这些设置在 Servlet 容器层面被应用，容器不允许从 `MultipartResolver` 层面去改。** 所以别指望用 `MultipartResolver` bean 设上限——Boot 已经把这些值落到 servlet 注册的 `MultipartConfigElement` 上了。

### 6.5 第四道：`maxSwallowSize`——没有状态码的那一道

前三道至少发生在 HTTP 语义之内。第四道不是。

HTTP/1.1 的连接可以复用。Tomcat 处理完一个请求，若 `keep-alive` 还开着，就必须保证 socket 里下一个字节是下一个请求的起始行。如果当前请求体还没读完（比如应用已经拒绝并返回 413），这些字节会污染下一个请求——所以 Tomcat 必须把它们读掉丢弃，这个动作叫 **swallow（吞掉）**。`maxSwallowSize` 就是"最多吞多少"，默认 2 MiB，源码里是写死的常量：

```java
// org.apache.coyote.http11.AbstractHttp11Protocol
private int maxSwallowSize = 2 * 1024 * 1024;   // ← 默认 2 MiB
```

超了会怎样？准备响应头时先预判一次：

```java
// org.apache.coyote.http11.Http11Processor#checkMaxSwallowSize
if (contentLength > 0 && protocol.getMaxSwallowSize() > -1 &&
        (contentLength - request.getBytesRead() > protocol.getMaxSwallowSize())) {
    keepAlive = false;   // 要吞的比允许的多，连接注定要关，补上 Connection: close
}
```

顺带一提，413 本身就会关掉 keep-alive：

```java
// Http11Processor#statusDropsConnection
return status == 400 || status == 408 || status == 411 || status == 413 ||
       status == 414 || status == 500 || status == 503 || status == 501;
```

真正下手的是输入侧的 filter：

```java
// org.apache.coyote.http11.filters.IdentityInputFilter#end
final boolean maxSwallowSizeExceeded = (maxSwallowSize > -1 && remaining > maxSwallowSize);
while (remaining > 0) {
    int nread = buffer.doRead(this);
    if (nread > 0) {
        swallowed += nread;
        remaining -= nread;
        if (maxSwallowSizeExceeded && swallowed > maxSwallowSize) {
            // 故意不提前失败，好让客户端有机会先把响应读走
            throw new IOException(sm.getString("inputFilter.maxSwallow"));
        }
    } else {
        remaining = 0;
    }
}
```

那句注释后面挂着 Apache 的一篇文档，讲的正是"服务端先关连接、客户端还没读完响应"这个经典问题。**这就是"连接被重置"的来源。**

这个 `IOException` 的归宿，解释了为什么日志里什么都没有：

```java
// Http11Processor#endRequest
try {
    inputBuffer.endRequest();
} catch (IOException ioe) {
    setErrorState(ErrorState.CLOSE_CONNECTION_NOW, ioe);   // ← 就地吞掉，不打印
} catch (Throwable t) {
    log.error(sm.getString("http11processor.request.finish"), t);
}
```

它只把错误状态改成 `CLOSE_CONNECTION_NOW`（`isError = true`、`isIoAllowed = false`），一声不吭——**只有非 `IOException` 的 `Throwable` 才会走到下面那个 ERROR 分支**，所以"这条路径永不打印"只对 `IOException` 成立。要退回 socket 层才有 `catch`：

```java
// org.apache.coyote.AbstractProtocol.ConnectionHandler#process
} catch (IOException ioe) {
    // IOExceptions are normal
    if (getLog().isDebugEnabled()) {
        getLog().debug(sm.getString("abstractConnectionHandler.ioexception.debug"), ioe);
    }
}
```

**DEBUG 级别才可见。** 默认 INFO 的生产日志里，一次连接被吞爆就是这么消失的。

串起来，引子里的故障有了完整解释：

1. 客户端发起 100 MB 上传，`Content-Length: 104857600`；
2. `checkMultipart` 触发 `getParts()`，Tomcat 发现 `Content-Length > max-request-size`，抛 `SizeLimitExceededException`（还没开始读请求体）；
3. Spring 把它翻译成 `MaxUploadSizeExceededException`，写出 413；
4. 请求体还剩约 100 MB，超过 `maxSwallowSize`：`checkMaxSwallowSize()` 关掉 keep-alive；
5. `IdentityInputFilter.end()` 吞到第 2 MiB 时抛 `IOException`；
6. `endRequest()` 就地捕获，状态变 `CLOSE_CONNECTION_NOW`，`service()` 返回 `SocketState.CLOSED`——**Tomcat 直接关连接，客户端还在写，于是收到 RST**；
7. 已经写进 socket 缓冲的那份 413，随着连接一起没了。浏览器报 `ERR_CONNECTION_RESET`。

还有一个容易误读的细节：`isError` 变 true 之后，`service()` 里会执行 `response.setStatus(500)`。但响应早就 commit 了，改不动报文，只能让**访问日志和统计里出现一个 500**。于是三方各说各话：前端说连接重置，访问日志说 500，应用日志什么都没有。

### 6.6 四种失败表现对照

| 拦在哪一层 | 触发条件 | 状态码 | 客户端看到什么 | 后端日志 |
|:---|:---|:---|:---|:---|
| Nginx / 网关 | 请求体 > `client_max_body_size`（默认 1m） | `413` | Nginx 错误页；浏览器可能显示不了 | 无，请求没到应用 |
| Tomcat `maxPostSize` | 表单字段累计超 2 MiB（文件不计） | 无固定值 | 取决于应用 | 默认无 |
| Tomcat `maxPartCount` / `maxParameterCount` | part 或参数个数超限 | 无固定值（`FailedRequestFilter` → 400） | 取决于应用 | 默认无 |
| Spring `max-file-size` / `max-request-size` | 单文件或请求体超限 | Boot 3.2+ 是 `413`；Boot 3.0/3.1 与 2.x 是 `500` | 可能拿到 413，也可能被重置 | 默认几乎无 |
| Tomcat `maxSwallowSize` | 拒绝之后客户端还在发，剩余 > 2 MiB | **无** | `ERR_CONNECTION_RESET` | 默认无（DEBUG 才有） |

一句话总结：**只要失败发生在"应用已经拒绝，但请求体还在路上"这个时间窗里，你能拿到的诊断信息几乎为零。**

## 七、为什么 `@ExceptionHandler` 接不住

先给结论：它大多数时候能接住——接不住的是另外几种情况。要分清，得先知道异常在哪里、以什么类型、在哪个时机抛出来。

### 7.1 原始形态：不是 Spring 的异常

解析发生在 Tomcat 内嵌的 fileupload 里，size 超限至少有这么几处抛点：

```java
// org.apache.tomcat.util.http.fileupload.impl.FileItemIteratorImpl#init
if (requestSize != -1 && requestSize > sizeMax) {
    throw new SizeLimitExceededException("the request was rejected because its size ...");  // max-request-size
}

// impl/FileItemStreamImpl（构造时；max-file-size）
// 注意：异常被 FileUploadIOException 包了一层，7.2 会拆开
if (fileSizeMax != -1 && pContentLength != -1 && pContentLength > fileSizeMax) {
    e.setFileName(pName);
    e.setFieldName(pFieldName);
    throw new FileUploadIOException(new FileSizeLimitExceededException(...));
}
```

注意第一个抛点：**它拿 `Content-Length` 先判，body 一个字节都没读。** 这是 Tomcat 自带的"快速拒绝"，不需要你写。此外还有两处兜底：`Content-Length` 缺席（chunked）时，`FileItemStreamImpl$LimitedInputStream#raiseError` 会在读到超限时抛出；part 头部超 `maxPartHeaderSize` 则由 `MultipartStream#readHeaders` 抛出。异常层级是：

```text
FileUploadException
└── SizeException（抽象）
    ├── SizeLimitExceededException      请求体超过 maxRequestSize
    └── FileSizeLimitExceededException  单个文件超过 maxFileSize
```

`FileCountLimitExceededException`（part 个数超限）直接继承 `FileUploadException`，不在 `SizeException` 之下——这个差别后面会用上。

### 7.2 一路被包成 `IllegalStateException`，再被 Spring"猜"出来

7.1 里那层 `FileUploadIOException` 会在这里先被拆掉：

```java
// org.apache.tomcat.util.http.fileupload.FileUploadBase#getItemIterator
try {
    return new FileItemIteratorImpl(this, ctx);
} catch (final FileUploadIOException e) {
    // unwrap encapsulated SizeException
    throw (FileUploadException) e.getCause();
}
```

所以落到 `parseParts` 手上时，它已经是干净的 `SizeException`。`parseParts` 再把它们统一收口：

```java
} catch (SizeException | FileCountLimitExceededException e) {
    parameters.setParseFailedReason(FailReason.POST_TOO_LARGE);
    checkSwallowInput();
    partsParseException = new IllegalStateException(e);   // ← 类型信息在这里被抹掉
}
```

Spring 侧接住它的是：

```java
// org.springframework.web.multipart.support.StandardMultipartHttpServletRequest
protected void handleParseFailure(Throwable ex) {
    Throwable cause = ex;
    do {
        String msg = cause.getMessage();
        if (msg != null) {
            msg = msg.toLowerCase(Locale.ROOT);
            if ((msg.contains("exceed") && (msg.contains("size") || msg.contains("length"))) ||
                    (msg.contains("request") && (msg.contains("big") || msg.contains("large")))) {
                throw new MaxUploadSizeExceededException(-1, ex);
            }
        }
        cause = cause.getCause();
    }
    while (cause != null);
    throw new MultipartException("Failed to parse multipart servlet request", ex);
}
```

这是本篇最值得盯住的代码。**Spring 认不出异常类型，它是在遍历 cause 链、匹配异常消息文本。** 两个后果：

- 命中条件是"消息里同时有 exceed 和 size/length"（或 request + big/large）。Tomcat 的英文消息恰好满足；**换个容器、换个语言环境、或换一个自定义包装异常，这条匹配就可能失手**，退化成 `MultipartException`。
- 即便命中，上限也是 **-1**。`getMaxUploadSize()` 返回 -1 只表示"上限未知"——真实的 `max-file-size` / `max-request-size` 已经在这一路包装里丢掉了，**不要拿它拼错误文案**。

所以 advice 至少要留两条：一条 `MaxUploadSizeExceededException`，一条兜底的 `MultipartException`。

### 7.3 时机：`checkMultipart` 跑在 `getHandler` 之前

```java
// DispatcherServlet#doDispatch
try {
    processedRequest = checkMultipart(request);       // ← 第一件事
    multipartRequestParsed = (processedRequest != request);
    mappedHandler = getHandler(processedRequest);     // ← 然后才找 handler
    ...
}
catch (Exception ex) {
    dispatchException = ex;
}
processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
```

抛异常时 `mappedHandler` 还是 `null`，这个 `null` 一路传到异常解析器：

```java
// ExceptionHandlerExceptionResolver#getExceptionHandlerMethod
Class<?> handlerType = null;
if (handlerMethod != null) {
    // 局部 @ExceptionHandler：只有 handler 已解析出来时才会走到这里
    handlerType = handlerMethod.getBeanType();
    ...
}
// 全局 @ControllerAdvice
for (Map.Entry<ControllerAdviceBean, ExceptionHandlerMethodResolver> entry : this.exceptionHandlerAdviceCache.entrySet()) {
```

结论很硬：**写在 Controller 里的 `@ExceptionHandler(MaxUploadSizeExceededException.class)` 不会被执行，只有全局 `@ControllerAdvice` 的会。** 因为异常在"知道该由哪个 Controller 处理"之前就抛出来了。

顺带一个能解释"行为不一致"的开关：把 `resolve-lazily` 设成 `true`，解析推迟到参数解析阶段，那时 `handlerMethod` 已经有值，Controller 里的局部处理器就能生效了。同一个异常，换个配置换条路径。

### 7.4 没人接也不会是 500（Spring 6.1 / Boot 3.2 起）

`MaxUploadSizeExceededException` **从 Spring 6.1 起**实现了 `ErrorResponse`（6.0.x 及更早没有）：

```java
public class MaxUploadSizeExceededException extends MultipartException implements ErrorResponse {
    private final ProblemDetail body =
            ProblemDetail.forStatusAndDetail(HttpStatus.PAYLOAD_TOO_LARGE, "Maximum upload size exceeded");
```

所以即便你什么都不写，优先级最低的 `DefaultHandlerExceptionResolver` 也会兜底：

```java
if (ex instanceof ErrorResponse errorResponse) {
    return (mav != null ? mav : handleErrorResponse(errorResponse, request, response, handler));
}

// handleErrorResponse
if (!response.isCommitted()) {
    response.sendError(errorResponse.getStatusCode().value(),
            errorResponse.getBody().getDetail());     // → 413
} else if (logger.isWarnEnabled()) {
    logger.warn("Ignoring exception, response committed already: " + errorResponse);
}
```

客户端拿到的 413 很可能不是你写的，是它送的。

**这处分界容易被画错：分界点是 Spring 6.1（Boot 3.2），不是 6.0。** 逐版本核对下来，`v5.3.39`、`v6.0.0`、`v6.0.14` 的 `MaxUploadSizeExceededException` 都还是 `extends MultipartException`，没有 `ErrorResponse`；直到 `v6.1.0` 才变成 `implements ErrorResponse`（引入提交 `3b80f2c4`，提交信息自述把状态码从 500 改成 413）。机制上的分工也要分清：`DefaultHandlerExceptionResolver` 的 `ErrorResponse` 分支 **6.0.0 就已存在**，只是 6.0 的这个异常没实现 `ErrorResponse`，所以 6.0 依然走 500。

所以正确读法是：**Boot 2.x（5.3.x）和 Boot 3.0 / 3.1（6.0.x）→ 500；Boot 3.2+（6.1+）→ 413。** 跨版本对比错误响应时先确认这一点。

补一个保守限定：上面说的是默认配置。如果显式打开了 `spring.mvc.problemdetails.enabled`（Boot 里该属性默认 `false`），兜底者会变成 `ProblemDetailsExceptionHandler`，响应体形态不同——**但状态码仍是 413**。

### 7.5 真正接不住的四类

| 情况 | 为什么接不住 |
|:---|:---|
| 请求被 Nginx / 网关拦下 | 应用根本没被调用，`DispatcherServlet` 没参与 |
| 请求在 `DispatcherServlet` 之前被拒 | `MultipartFilter`、Servlet Filter、Tomcat 参数解析都在 handler 链之外 |
| 响应已经 commit | `handleErrorResponse` 只在 `!response.isCommitted()` 时改状态码，否则只打一条 warn |
| 连接已经被关 | 处理器可能跑了，但写出去的字节没人读——6.5 那条路 |

第一类里有个容易踩的组合：`MultipartFilter` 在 DispatcherServlet **之前**解析 multipart，异常发生在 filter 里，`@ControllerAdvice` 完全看不到。

## 八、把「失败」变成可诊断的响应

### 8.1 全局 advice：让每次拒绝都留痕

```java
@RestControllerAdvice
public class UploadExceptionAdvice {

    private static final Logger log = LoggerFactory.getLogger(UploadExceptionAdvice.class);

    @ExceptionHandler(MaxUploadSizeExceededException.class)
    public ResponseEntity<Map<String, Object>> onTooLarge(MaxUploadSizeExceededException ex) {
        // getMaxUploadSize() 在 Servlet 容器解析失败时恒为 -1，不能当真实上限用
        log.warn("upload rejected: maxUploadSize={}, rootCause={}",
                ex.getMaxUploadSize(), rootCause(ex).toString());
        return ResponseEntity.status(HttpStatus.PAYLOAD_TOO_LARGE)
                .body(Map.of("code", "UPLOAD_TOO_LARGE", "message", "文件超过大小限制"));
    }

    @ExceptionHandler(MultipartException.class)   // ← 消息匹配失手时的兜底
    public ResponseEntity<Map<String, Object>> onMultipart(MultipartException ex) {
        log.warn("multipart parse failed", ex);
        return ResponseEntity.badRequest()
                .body(Map.of("code", "MULTIPART_INVALID", "message", "上传报文无法解析"));
    }

    private Throwable rootCause(Throwable ex) {
        Throwable cause = ex;
        while (cause.getCause() != null) {
            cause = cause.getCause();
        }
        return cause;
    }
}
```

三个要点都是前面推理的直接后果：**必须放全局**（7.3）；**必须留 `MultipartException` 兜底**（7.2 的消息匹配不可靠）；**不要把 `-1` 当前上限**（7.2）。

### 8.2 解析之前先看一眼 `Content-Length`

Tomcat 自己已有 `Content-Length` 预检（7.1），所以这一层的价值不在"更快拒绝"，而在两点：**网关没拦住时给出一致的 JSON**，以及**完全绕开 multipart 解析器的初始化与 part 缓冲**。

```java
@Override
protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                FilterChain chain) throws ServletException, IOException {
    if (request.getContentLengthLong() > maxBodySize) {          // maxBodySize 与 max-request-size 一致
        response.setStatus(HttpServletResponse.SC_REQUEST_ENTITY_TOO_LARGE);
        response.setContentType("application/json;charset=UTF-8");
        response.getWriter().write("{\"code\":\"UPLOAD_TOO_LARGE\"}");
        return;
    }
    chain.doFilter(request, response);
}
```

边界必须说清：`Content-Length` 是**声明值**，可以伪造，也可以缺席——chunked 编码的请求就没有它，`getContentLengthLong()` 返回 -1，这个 guard 直接放行。所以它是"快速失败"的优化，**不是限额**；真正的限额仍是 6.5 那几道。

### 8.3 网关层也要给 JSON

Nginx 产生的 413 和后端产生的 413 形态不一致，前端就得写两套解析。统一它：

```nginx
error_page 413 = @too_large;

location @too_large {
    default_type application/json;
    return 413 '{"code":"UPLOAD_TOO_LARGE","message":"请求体超过网关限制"}';
}

location /api/upload {
    client_max_body_size 20m;    # 必须 ≥ 后端 spring.servlet.multipart.max-request-size
}
```

一条容易被忽略的一致性约束：**网关的上限要大于后端的上限。** 反过来的话，后端还没机会回 413，Nginx 已经把连接掐了——也就是引子里那种现象。

### 8.4 让 Tomcat 的静默 `IOException` 说话

把 `org.apache.coyote` 的日志级别临时开到 DEBUG，能看到 `abstractConnectionHandler.ioexception.debug` 那条消息。四种策略的代价：

| 做法 | 怎么配 | 代价 |
|:---|:---|:---|
| 默认 | `max-swallow-size` 2MB | 超限就断连；日志默认静默 |
| 放大或关闭上限 | 调大 `server.tomcat.max-swallow-size` | 处理线程会被"吞请求体"占住（Tomcat 侧小于 0 表示不限制） |
| 干脆不吞 | Context 属性 `swallowAbortedUploads = false`（`StandardContext` 默认 `true`） | 连接立即关闭，响应可能到不了客户端；线程立刻释放 |
| 网关先拦 | `client_max_body_size` | 上限要和后端对齐 |

第三行值得展开一句，因为它走的是另一条代码路径：`checkSwallowInput()` 只在 `swallowAbortedUploads` 为 false 时才真的下发 `DISABLE_SWALLOW_INPUT`；`AbstractProcessor` 收到后调 `disableSwallowRequest()` 并把状态置为 `CLOSE_CLEAN`（可写完响应、但连接必须关）。**这个开关默认是 `true`，所以默认路径不在这里**——默认走的是 6.5 那条"吞到上限再抛 `IOException`"的路。

生产上更稳的组合是**网关先拦 + 后端保留默认 `maxSwallowSize` + 拒绝路径写完整日志**：把"连接被重置"变成"客户端收到了 413 JSON"。

## 九、不落盘：流式直传对象存储

### 9.1 "先落盘再上传"浪费在哪

`file-size-threshold` 默认 `0B`，意味着 Tomcat 几乎会把每个 part 都写进磁盘——写到哪由 `MultipartConfigElement` 的 `location` 决定，没配就是容器临时目录。于是最常见的写法是这样：

```java
Path tmp = Paths.get("/data/upload-tmp", UUID.randomUUID().toString());
file.transferTo(tmp);                       // ① 又一次落盘
try (InputStream in = Files.newInputStream(tmp)) {
    ossClient.putObject(bucket, key, in);   // ② 读回来，再发出去
}
Files.deleteIfExists(tmp);                  // ③ 还得记得删
```

一次上传，数据落两遍盘、读两遍。**这段代码里唯一必要的动作是第 ② 步。**

### 9.2 边收边写：把 part 的流直接交给 SDK

```java
@Service
public class DirectUploadService {

    private final S3Client s3Client;
    private final String bucket;

    public String upload(MultipartFile file, String key) throws IOException {
        PutObjectRequest request = PutObjectRequest.builder()
                .bucket(bucket)
                .key(key)
                .contentType(file.getContentType())
                .contentLength(file.getSize())    // 流式 body 没有长度，SDK 无从推断
                .build();
        try (InputStream in = file.getInputStream()) {
            s3Client.putObject(request, RequestBody.fromInputStream(in, file.getSize()));
        }
        return key;
    }
}
```

`RequestBody.fromInputStream(InputStream, long)` 的第二个参数是必需的，它和 `contentLength(...)` 声明的是同一个长度，**两处必须一致**。写错的后果不是"被拒"，而是**静默丢数据**：SDK 的 javadoc 写明，**声明长度小于实际内容时，客户端会把流截断到声明的字节数、只发这么多，且不报错**；反过来声明长度大于实际内容时，服务端等不到足够的字节，请求才会失败。前者最危险——业务侧会以为文件传完了。`S3Client.putObject(PutObjectRequest, RequestBody)` 是 SDK v2 的标准签名。

### 9.3 流式直传的四条边界

- **一次性的流不可重放，所以不可重试。** `InputStream` 只能读一遍，SDK 的重试在流式 body 上无法重新读取。要重试就得放在可重放的载体上（本地文件、字节数组），或者接受"失败即让客户端重传"——后者正是分片上传要解决的问题。
- **超时要覆盖两端。** 一次请求里串了"客户端传到应用"和"应用传到对象存储"两段，按**两段之和**估。
- **"不落盘"不等于零内存。** 容器读 socket 有自己的缓冲，SDK 写 HTTP 也有缓冲。省掉的是**应用自己那一次额外的落盘与读取**。
- **失败语义变粗。** 对象存储写失败时请求体可能已被消费一部分，没法回滚，只能让客户端整份重来。

### 9.4 `transferTo` 的四个坑

它的 javadoc 写得很直白：目标已存在会先被删除；**如果文件在文件系统里是被"移动"过去的，这个操作之后就不能再调用了，所以只能调一次。**

**坑一：相对路径的基准目录不是你的当前目录。** javadoc 里有一句"相对目标路径的基准目录由容器决定"。落到 Tomcat 上：

```java
// org.apache.catalina.core.ApplicationPart#write
if (!file.isAbsolute()) {
    file = new File(location, fileName);   // ← location 即 MultipartConfigElement 的 location
}
```

`file.transferTo(new File("uploads/a.png"))` 会写进临时目录下的 `uploads/a.png`，然后被容器清理掉。**传绝对路径。**

**坑二：只能调一次。** 落盘的 part 走的是 `renameTo`（失败才退化成复制），源临时文件随之消失：

```java
// DiskFileItem#write
if (file.exists() && !file.delete()) {
    throw new FileUploadException("Cannot write uploaded file to disk!");
}
if (!outputFile.renameTo(file)) {
    // 退化成流复制
}
```

`DiskFileItem.write` 的 javadoc 同样写明"只保证第一次调用有效"。所以 `transferTo` 之后再 `getInputStream()`，读到的不是原来的内容——顺序错了就是静默的空文件。

**坑三：目标文件已存在会被先删掉。** 就是上面那行 `if (file.exists() && !file.delete())`。`transferTo(new File(dir + file.getOriginalFilename()))` 等于让客户端决定覆盖谁。

**坑四：绝对路径是 Spring 帮你兜的底。** Servlet 规范没保证 `Part.write` 支持绝对路径：

```java
// StandardMultipartFile#transferTo(File)
this.part.write(dest.getPath());
if (dest.isAbsolute() && !dest.exists()) {
    // 有些容器会把绝对路径"翻译"到临时目录下，这里手动补一次复制兜底
    FileCopyUtils.copy(this.part.getInputStream(), Files.newOutputStream(dest.toPath()));
}
```

这段兜底是**尽力而为**：只在"目标不存在"时才触发。

### 9.5 分片上传、断点续传、秒传的骨架

大文件不要走单请求。最小骨架三步：

```java
// 1) 初始化，拿到 uploadId
String uploadId = s3Client.createMultipartUpload(
        CreateMultipartUploadRequest.builder().bucket(bucket).key(key).build()).uploadId();

// 2) 逐片上传：每一片都要把 ETag 记下来
String etag = s3Client.uploadPart(UploadPartRequest.builder()
                .bucket(bucket).key(key).uploadId(uploadId)
                .partNumber(partNumber)          // 1 ~ 10000
                .build(),
        RequestBody.fromInputStream(chunkStream, chunkLength)).eTag();

// 3) 合并：把 (partNumber, ETag) 按序交回去
s3Client.completeMultipartUpload(CompleteMultipartUploadRequest.builder()
        .bucket(bucket).key(key).uploadId(uploadId)
        .multipartUpload(CompletedMultipartUpload.builder()
                .parts(CompletedPart.builder().partNumber(partNumber).eTag(etag).build())
                .build())
        .build());
```

AWS 文档给的硬数字要先拿来框住方案：**单次 PUT 最大 5 GB**；分片上传最多 **10000 片**、片号 1~10000；**每片 5 MiB ~ 5 GiB**，最后一片没有最小限制；对象最大 48.8 TiB。

几个设计点：

- **分片大小取多少。** 5 MiB 是下限不是建议值。片太小 → 请求数与元数据暴涨、合并变慢；片太大 → 单次失败的重传代价高。按"目标大小 ÷ 1000 片以内"反推，常见落在 8~64 MiB。
- **并发与顺序。** 片之间独立，可以并发，但要限并发数，否则一条大上传能把出口带宽吃满。合并按 `partNumber` 排序，**所以完成顺序不影响结果**——这正是分片能并发的根本原因。
- **断点续传。** 要持久化的只有两样：`uploadId` 和"哪些片已成功"。续传前用 `ListParts` 问一次服务端，比本地记录更可靠。
- **秒传。** 上传前先算整文件摘要，命中"摘要 → 对象"映射就直接建引用。四条取舍：客户端要能算摘要；摘要必须能**流式**算；服务端要防止"拿别人的摘要引用别人的文件"（至少校验对象大小）；分片上传时整文件摘要客户端不方便算，常见替代是用各片 ETag 拼出对象 ETag 对账。
- **别忘回收。** 没有 complete 的 multipart upload 会一直占着存储和计费，要配生命周期规则定时中止。

## 十、绕过应用进程：预签名 URL 直传

### 10.1 三种数据路径

下载篇用「五、架构层」把传输从应用进程里剥离出去，上传这边是同一个问题的镜像：

| | 经应用转发（传统） | 应用流式转发 | 预签名直传 |
|:---|:---|:---|:---|
| 数据路径 | 客户端 → 应用 → 存储 | 客户端 → 应用 → 存储 | 客户端 → 存储 |
| 应用带宽 | 双向（进 + 出） | 双向 | **0** |
| 应用内存 | 取决于缓冲策略 | 常数级 | **0** |
| 占用线程 | 整个上传期间 | 整个上传期间 | 只有签名那一瞬间 |
| 鉴权时机 | 上传前 | 上传前 | 签名时；之后由签名有效期约束 |
| 写入前能否看到内容 | 能 | 能 | **不能** |
| 进度回调 | 应用能拿到字节数 | 应用能拿到字节数 | 只有客户端知道 |
| 失败重试 | 应用内部可重试 | 可重试（body 必须可重放） | 客户端重试同一 URL（有效期内） |
| 本地落盘 | 至少一次 | 无 | 无 |
| 适用 | 小文件、需要落地前强校验 | 需要边收边转（加密 / 转码 / 查毒） | 大文件、高并发、内容无关的落地 |

最关键的一行是"写入前能否看到内容"：**预签名直传把数据面交出去的同时，也交出了"在写入之前检查内容"的能力。** 这是它唯一且无法绕过的代价。

### 10.2 服务端：只签一个 PUT

```java
@PostMapping("/presign")
public Map<String, Object> presign(@RequestBody PresignRequest req) {
    // key 由服务端生成：不要把客户端文件名拼进路径
    String key = "upload/" + LocalDate.now() + "/" + UUID.randomUUID();

    PresignedPutObjectRequest signed = presigner.presignPutObject(
            PutObjectPresignRequest.builder()
                    .signatureDuration(Duration.ofMinutes(15))
                    .putObjectRequest(b -> b.bucket(bucket).key(key)
                            .contentType(req.contentType())
                            .contentLength(req.size()))
                    .build());

    return Map.of("key", key,
            "url", signed.url().toString(),
            "expiresAt", signed.expiration().toString());
}
```

用到的都是 SDK v2 里手写的部分，可以放心查证：`S3Presigner#presignPutObject(PutObjectPresignRequest)`、`PutObjectPresignRequest.Builder#signatureDuration(Duration)` 与 `#putObjectRequest(...)`、`PresignedRequest#url()` / `#expiration()`。

把 `contentType` 和 `contentLength` 一起签进去是有意为之：**它们进入签名，客户端就改不动了。** 不签的话，一个"只允许传 2 MB 图片"的 URL 可以被拿去传别的东西。

### 10.3 客户端：浏览器直接 PUT

```javascript
const meta = await fetch('/api/uploads/presign', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ contentType: file.type, size: file.size }),
}).then(r => r.json());

await fetch(meta.url, {                       // 不经过后端，直接发给对象存储
  method: 'PUT',
  headers: { 'Content-Type': file.type },
  body: file,
});

await fetch('/api/uploads/commit', {          // 让后端确认并落库
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ key: meta.key }),
});
```

### 10.4 直传的边界条件

- **单次 PUT 有大小上限。** S3 单次 PUT 最大 5 GB，超过必须走分片；而分片直传要**逐片签名**——`uploadId` 和 `partNumber` 都要进入签名，不能一个签名复用所有片。
- **签名只管"能不能写"，不管"写了什么"。** 所以 key 必须由服务端决定，前缀按租户隔离。
- **进度回调得自己接。** 应用看不到字节流，想知道进度只能让客户端上报；也无法在传输中途取消，只能依赖签名过期或后置校验。
- **完成后必须校验一次。** 前端说"传完了"不算数——后端要用 `headObject` 确认对象确实存在、大小符合预期，再写业务记录。
- **有效期是安全边界，而且有硬上限。** 有效期内同一 URL 可反复 PUT（覆盖语义），TTL 越长泄漏后可利用时间越久。`S3Presigner` 的类注释写明预签名有效期**不能超过 7 天**（生成时就报错），所以 15 分钟这类短 TTL 是常规起点而不是硬性规定。
- **内容校验只能后置。** 需要查毒 / 转码时，唯一可行的做法是"上传完成后触发异步任务，不通过就删除对象"，并把状态告诉业务方。**这一步漏掉，直传就是一个任意文件落地入口。**

## 十一、安全与工程清单

### 11.1 `getOriginalFilename()` 不可信

这是 Spring 自己在 javadoc 里写明的：文件名可能包含目录部分，可能包含 `..` 这类恶意字符，**建议不要直接使用，而是自己生成唯一文件名**。而它的实现就是把 `Content-Disposition` 里解析出的字符串原样返回：

```java
// StandardMultipartHttpServletRequest.StandardMultipartFile
public String getOriginalFilename() {
    return this.filename;
}
```

于是下面这行是典型的目录穿越：

```java
// ❌ 客户端可以传 ../../../../etc/cron.d/x，也可以传 /etc/passwd
Path target = Paths.get("/data/uploads").resolve(file.getOriginalFilename());
```

正确姿势分两种。**默认方案：key 由服务端生成**（UUID / 雪花 ID），原始文件名只作为元数据入库、展示时转义。**必须用原名时**，至少归一化再校验前缀：

```java
Path root = Paths.get("/data/uploads").toAbsolutePath().normalize();
Path target = root.resolve(name).normalize();
if (!target.startsWith(root)) {
    throw new IllegalArgumentException("illegal file name");
}
```

### 11.2 空字节：Tomcat 会拦，但拦的方式影响错误处理

文件名里带 `\u0000` 是经典的"骗过后缀校验、截断真实路径"手法。Tomcat 内嵌的 fileupload 直接拒绝：

```java
// org.apache.tomcat.util.http.fileupload.util.Streams#checkFileName
// 源码里是局部 StringBuilder sb，把 \u0000 转成字面量 \0 后拼进消息
if (fileName != null && fileName.indexOf('\u0000') != -1) {
    throw new InvalidFileNameException(fileName, "Invalid file name: " + sb);
}
```

它在 `parseParts` 里走的是另一个分支：

```java
} catch (InvalidContentTypeException | InvalidFileNameException e) {
    parameters.setParseFailedReason(FailReason.INVALID_CONTENT_TYPE);
    partsParseException = new ServletException(e);      // ← 不是 IllegalStateException
}
```

两个后果：**不会被包成 `MaxUploadSizeExceededException`**，所以"文件过大"的 advice 接不到它；消息里也没有 exceed/size 关键字，Spring 只会抛通用的 `MultipartException`。**兜底处理器必须存在，否则客户端等到的是一个 500。**

### 11.3 数量与头部：`maxPartCount` / `maxPartHeaderSize`

- 单个 part 的头部默认上限 512 字节，超了直接拒绝——这是防止超长 `Content-Disposition` 撑爆内存的第一道。
- part 个数默认上限 50，参数总数默认上限 10000（外置 Tomcat 见 6.3 的 `server.xml` 说明）。这两个 Boot 没有属性。
- 调大之前复习那条公式：`maxPartHeaderSize × maxPartCount × maxConnections × 2`，默认约 400 MB。

### 11.4 扩展名 vs 魔数

扩展名和 `Content-Type` 都是客户端给的，`Content-Disposition` 里的 `filename` 也是。只按它们做白名单，等于让上传者自己声明"我是安全的"。真正的校验要读文件头：

```java
// 魔数校验必须发生在写入最终存储之前
try (InputStream in = file.getInputStream()) {
    byte[] head = in.readNBytes(8);
    if (!isPng(head) && !isJpeg(head)) {
        throw new IllegalArgumentException("unsupported file type");
    }
}
```

这条约束和预签名直传是冲突的——直传时应用在写入前读不到字节。所以**"是否需要写入前强校验"本身就是选择架构的决定性条件**，而不只是安全清单里的一项。

### 11.5 存储位置、临时目录、并发、审计

- **绝不落在 webroot。** 上传目录不能位于 Tomcat 的 `webapps/`、`src/main/resources/static` 或任何被静态资源处理器覆盖的路径下。放进去等于给上传者开了一个可访问、可执行的入口。
- **临时目录会先被打满。** `file-size-threshold` 默认 `0B`，part 直接落盘，并发上传大文件时先耗尽的往往是磁盘而不是内存。要么用 `multipart.location` 指到独立分区并配配额与监控，要么走预签名直传，本地根本不落。
- **并发与配额。** 上传是少数能同时把带宽、磁盘、线程吃满的接口。按用户/租户限流、限制单请求体积、限制同一账号并行的分片上传数。
- **日志与审计。** 记全"谁、何时、传了什么、多大、结果如何"：对象 key、size、contentType、摘要、IP、租户。**记录前先过滤控制字符**（原始文件名可能带 `\u0000`）。最后一条：大文件上传失败时前后端各留一份记录——后端记拒绝原因，前端记 `ERR_CONNECTION_RESET`，否则就是引子里那种"三方各说各话"。

## 结语：四道限额，三条数据路径

开头那个故障不是 bug，是四个独立机制叠在一起的结果：Tomcat 发现超限就拒绝（6.4）、拒绝后要吞掉剩余体（6.5）、吞不下的部分直接断连（6.5）、断连的异常默认不记日志（6.5）。缺任何一环，这个现象都不会这么"干净"。

压成一套可执行的决策顺序：

1. **先把四道限额对齐。** 从外到内逐层收窄：`client_max_body_size` ≥ 网关上限 ≥ `max-request-size` ≥ `max-file-size`。并且记住 `maxPostSize` 只算表单字段，别拿它当文件上限。
2. **再把拒绝路径做成可读的响应。** 全局 `@ControllerAdvice` + 网关 413 JSON + 留意 `maxSwallowSize` 那条无状态码的路径。这一步投入产出比最高——它把"连接被重置"变成"客户端收到 413 JSON 且后端有日志"。
3. **然后决定数据路径。** 小文件、需要写入前强校验 → 应用内处理；需要边收边转（加密 / 转码 / 查毒）→ 应用流式转发；大文件、高并发、内容无关 → 预签名直传；超大文件 → 无论走哪条路都要分片 + 断点续传。
4. **最后才谈优化。** 流式直传省掉的是应用侧那一次额外的落盘与读取；分片带来并发能力，也带来 ETag 记账、未完成分片回收这些新的运维面。

**三个最值得记住的结论：**

1. `maxSwallowSize` 是"故障看起来像网络问题、日志里却什么都没有"的根因，它的异常默认只在 DEBUG 级别可见。
2. `@ExceptionHandler` 失效多半不是写错了，而是异常抛出得太早（`checkMultipart` 早于 `getHandler`）、太晚（响应已 commit、连接已关），或者干脆不经过 `DispatcherServlet`。
3. 预签名直传最大的代价不是复杂度，是**你放弃了"写入前看到内容"的能力**——需要强内容校验的场景不要用它。

### 延伸阅读

- [（一）：从报文到临时文件](/2026/10/23/Spring-文件上传全链路深度解析（一）：从报文到临时文件/)——multipart 报文格式、`parseParts` 内部实现、`MultipartFile` 的内存/磁盘阈值原理，以及 2.x→3.x 的解析器选型与配置前缀变化，本篇只做引用。
- [Spring 文件下载全方案深度解析](/2026/09/15/Spring-文件下载全方案深度解析/)——「五、架构层」的三种模式与本篇「十、绕过应用进程」互为镜像。
- [一次下载失败排查：被 Filter 吞掉的异步响应流](/2026/10/14/一次下载失败排查：被 Filter 吞掉的异步响应流/)——同样是"响应写了但客户端拿不到"，那边的元凶是缓存包装器，这边是连接被关闭。
- [深入 WebFlux 与 Spring Cloud Gateway](/2026/09/21/深入-WebFlux-与-Spring-Cloud-Gateway：为什么网关没有-Controller，却是一个-WebFlux-应用？/)——如果网关是 Spring Cloud Gateway，上传体在 Reactor Netty 上的处理又是另一套机制。
