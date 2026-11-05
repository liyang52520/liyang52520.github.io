---
title: "Spring 文件上传全链路深度解析（二）：限额、故障与不落盘直传"
date: 2026-10-23 10:50:01
tags:
  - spring
  - tomcat
categories:
  - develop
---

浏览器控制台一行 `net::ERR_CONNECTION_RESET`，Network 面板里那条 `POST /api/upload` 停在待响应状态，状态码是空的。翻后端日志——应用日志里一行异常都没有。把同一个文件切成 5 MB 分片再传，一切正常。

> - 一个上传请求，从 Nginx 到对象存储，中间有几道限额？分别由谁拦下？
> - 为什么被拒绝之后，客户端看到的是"连接被重置"而不是一个 413？
> - Tomcat 有个 `maxSwallowSize`（"最多替客户端吞掉多少请求体"）——它为什么能让应用连一句日志都留不下？
> - 为什么写在 Controller 里的异常处理方法，明明写了却没被调用？
> - 上传一定要先落到本地临时文件吗？能不能边收边写进对象存储？
> - 预签名 URL 直传（服务端用存储凭证签出一个带时效的地址，客户端直接 PUT 到对象存储、不经过应用）和"应用流式转发"差在哪里，代价是什么？
> - `MultipartFile.getOriginalFilename()` 到底能不能信？

这篇接着 [（一）：从报文到临时文件](/2026/10/23/Spring-文件上传全链路深度解析（一）：从报文到临时文件/) 往下走。第一篇讲报文怎么被解析、`MultipartFile` 从哪里来；这一篇只关心一件事：**这套机制在真实系统里以什么方式失败，以及怎么绕开它。** 口径是 Spring Boot 3.2+ / Spring Framework 6.1 / Tomcat 10.1，2.x 与 3.0/3.1 的差异单独标注。

文中源码来自 `tomcat-embed-core`（Tomcat 10.1.x）、`spring-web` / `spring-webmvc`（6.1.x）、`spring-boot-autoconfigure`（3.2.x），IDEA 里按类名搜索、下载源码即可打开。

## 五、四道限额：同一份文件，四个地方拦你

引子里那次重置，是被这四道里的哪一道拦下的？先看全景，再一层层拆。

### 5.1 四道限额，守在四个不同的位置

```text
浏览器
  │  ① client_max_body_size 1m            超限：Nginx 自己回 413
  ▼
Nginx / 网关
  ▼
Tomcat Connector
  │  ④ maxSwallowSize 2 MiB               拒绝之后吞不完 → 断连，没有状态码
  ▼
DispatcherServlet ──checkMultipart──► Tomcat 的 parseParts
  │  ② maxPostSize 2 MiB / maxPartCount 50 / maxParameterCount 10000
  │  ③ max-request-size 10MB / max-file-size 1MB（来自 Spring 的 MultipartConfigElement）
  ▼
Controller
```

**越靠近连接层，失败越说不清**：最外层至少还能回一个 413，最里面那道连状态码都没有。

### 5.2 第一道：Nginx 直接回 413，请求根本到不了应用

```nginx
location /api/upload {
    client_max_body_size 30m;   # 默认 1m，不显式放大就只能传 1MB
}
```

默认 `1m` 来自 [Nginx 官方文档](https://nginx.org/en/docs/http/ngx_http_core_module.html#client_max_body_size)。它可写在 `http` / `server` / `location` 三层，**只在请求真正经过的那一层生效**——放在 `http` 里被内层 `location` 覆盖，很常见。

超限时 Nginx 自己返回 `413 Request Entity Too Large`，**请求根本不会到后端**。文档还提醒浏览器显示不了它——这正是 7.2 要改写成 JSON 的原因。

### 5.3 第二道：`maxPostSize` 管的不是文件，是表单字段

Connector 上有五个数，最容易配错的是 `maxPostSize`。"上传上限调大了却没用"的故事，大多出在它身上。

| 属性 | 默认值 | 限制的是什么 |
|:---|:---|:---|
| `maxPostSize` | `2097152`（2 MiB） | 会被转换成请求参数的请求体字节数 |
| `maxParameterCount` | `10000`（外置 `server.xml` 配成 1000） | 参数 + 上传文件的总个数，超出部分被忽略 |
| `maxPartCount` | `50` | `multipart/form-data` 的 part 个数上限 |
| `maxPartHeaderSize` | `512` | 每个 part 的头部字节上限，超出直接拒绝 |
| `maxSwallowSize` | `2097152`（2 MiB） | 连接层"吞掉"未读请求体的上限 |

`maxPostSize` 是最大的坑：Tomcat 文档明确说它**不是请求体总大小的通用上限**，只在 Tomcat 为解析参数而处理请求体时生效。落到 `multipart/form-data` 上，它只约束"用来生成 `getParameter()` 那批数据"，也就是普通表单字段：

```java
// org.apache.catalina.connector.Request#parseParts（Tomcat 10.1.x）
if (part.getSubmittedFileName() == null) {          // ← 没有文件名 = 普通表单字段
    if (maxPostSize >= 0) {
        // 源码用 Math.addExact 逐项累加
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

Boot 只暴露了其中两个：`server.tomcat.max-swallow-size`（2MB）对应 `AbstractHttp11Protocol#setMaxSwallowSize`，`server.tomcat.max-http-form-post-size`（2MB）对应 `Connector#setMaxPostSize`。

**`maxParameterCount` 和 `maxPartCount` 没有对应的 Boot 属性**，要自己加 `TomcatConnectorCustomizer`（Boot 定制内嵌 Tomcat 的接口）；`factory` 是 `TomcatServletWebServerFactory`：`factory.addConnectorCustomizers(connector -> connector.setMaxPartCount(100))`，`connector` 就是那个 Connector。

调大之前，先看代价。part 头部要先进内存。假设攻击者占满 8192 连接（`maxConnections` 的默认值），每条连接发一个塞满 50 个 part（`maxPartCount`）、每个 part 头部写满 512 字节（`maxPartHeaderSize`）的请求——按文档估算，光头部就要占 512 × 50 × 8192 × 2 ≈ **400 MB** 堆内存。不用传文件内容，靠连接数和 header 就能顶上去，这就是文档把 `maxPartCount` 列为 DoS 面的原因。调大它，上界按比例涨。

### 5.4 第三道：上限是 Spring 定的，Tomcat 只负责抛

Tomcat 抛异常时并不知道上限是多少——这个数是 Spring 在注册 Servlet 时交给它的。送进解析器的就这几项：

| 配置项（`spring.servlet.multipart.*`） | 默认值 | 作用 |
|:---|:---|:---|
| `max-file-size` | 1MB | 单个文件的大小上限 |
| `max-request-size` | 10MB | 整个请求体的大小上限 |
| `file-size-threshold` | 0B | 超过这个值才落盘；0 表示第一个字节就落盘 |
| `location` | 容器临时目录 | 落盘的目录 |
| `resolve-lazily` | false | 是否推迟到真正用到时再解析 |

这些值来自 `MultipartProperties`，写进 `MultipartConfigElement`，再交给 Tomcat 内嵌的 fileupload——`Request#parseParts` 里就两行：`upload.setFileSizeMax(mce.getMaxFileSize())` 与 `upload.setSizeMax(mce.getMaxRequestSize())`。

`StandardServletMultipartResolver` 的类注释里有一句关键的话：**这些设置在 Servlet 容器层面被应用，容器不允许从 `MultipartResolver` 层面去改。** 所以别指望用 `MultipartResolver` bean 设上限。

### 5.5 第四道：拒绝之后，连接怎么收场

前三道在 HTTP 语义之内：拒绝就是给个状态码。第四道不在语义里——它决定的是"拒绝之后，剩下的请求体怎么处理"。引子里那个"没有状态码"的故障，就是它造成的。

HTTP/1.1 的连接可以复用。Tomcat 处理完请求后，若 `keep-alive` 还开着，就必须保证 socket 里下一个字节是下一个请求的起始行。请求体没读完（比如应用已经拒绝并返回 413），这些字节会污染下一个请求——所以 Tomcat 必须把它们读掉丢弃，这个动作叫 **swallow（吞掉）**。`maxSwallowSize` 就是"最多吞多少"，默认 2 MiB，是 `AbstractHttp11Protocol` 里写死的 `2 * 1024 * 1024`。

超了会怎样？准备响应头时，`Http11Processor#checkMaxSwallowSize` 拿"剩余待吞字节"和上限比，要吞的比允许的多，就把 `keepAlive` 置为 false，补一个 `Connection: close`。另外 413 本身也在 `statusDropsConnection` 的名单里（和 400、408、411、414、500、503、501 并列），一旦返回它就放弃复用连接。

真正下手的是 Tomcat 的 InputFilter：

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

那句注释背后的文档讲的是"服务端先关连接、客户端还没读完响应"这个经典问题。**这就是"连接被重置"的来源。**

这个 `IOException` 的归宿，解释了为什么日志里什么都没有：`Http11Processor#endRequest` 捕获它之后，只把错误状态改成 `CLOSE_CONNECTION_NOW`，一声不吭——**只有非 `IOException` 的 `Throwable` 才会走 `log.error`**。要退回 socket 层才有一个 `catch`，而那里只在 DEBUG 打印。默认 INFO 的生产日志里，一次连接被吞爆就是这么消失的。

串起来，引子里的故障有了完整解释：

1. 客户端发起 100 MB 上传，`Content-Length: 104857600`；
2. `checkMultipart` 触发 `getParts()`，Tomcat 发现 `Content-Length > max-request-size`，抛出 `SizeLimitExceededException`（还没开始读请求体）；
3. Spring 把它翻译成 `MaxUploadSizeExceededException`，写出 413；此时请求体还剩约 100 MB，超过 `maxSwallowSize`，`checkMaxSwallowSize()` 关掉 keep-alive；
4. `IdentityInputFilter.end()` 吞到第 2 MiB 时抛 `IOException`，连接随即被关——客户端还在写，收到 RST；已经写进 socket 缓冲的那份 413 也随连接消失，浏览器报 `ERR_CONNECTION_RESET`。

引子里"切成 5 MB 分片就正常"也有了答案：每一片都远小于四道限额，根本没走到"拒绝—吞噬"这条路径。

`isError` 变 true 后，`AbstractProcessor#service` 收尾会 `response.setStatus(500)`，但响应已 commit，改不动报文，只让访问日志里出现一个 500：前端说连接重置、访问日志说 500、应用日志什么都没有。

### 5.6 五种失败表现对照

| 拦在哪一层 | 触发条件 | 状态码 | 客户端看到什么 | 后端日志 |
|:---|:---|:---|:---|:---|
| Nginx / 网关 | 请求体 > `client_max_body_size`（默认 1m） | `413` | Nginx 错误页；浏览器可能显示不了 | 无，请求没到应用 |
| Tomcat `maxPostSize` | 表单字段累计超 2 MiB（文件不计） | 无固定值 | 取决于应用 | 默认无 |
| Tomcat `maxPartCount` / `maxParameterCount` | part 或参数个数超限 | 无固定值（`FailedRequestFilter` 默认关；打开后 part 超限 413、参数超限 400） | 取决于应用 | 默认无 |
| Spring `max-file-size` / `max-request-size` | 单文件或请求体超限 | Boot 3.2+ 是 `413`；Boot 3.0/3.1 与 2.x 是 `500` | 可能拿到 413，也可能被重置 | 默认几乎无 |
| Tomcat `maxSwallowSize` | 拒绝之后客户端还在发，剩余 > 2 MiB | **无** | `ERR_CONNECTION_RESET` | 默认无（DEBUG 才有） |

一句话：**只要失败发生在"应用拒绝、请求体还在路上"这个时间窗里，诊断信息几乎为零。**

## 六、异常不是 Spring 抛的，也不在它以为的时机抛

`@ExceptionHandler` 是 Spring MVC 里"在一个 handler 类（Controller 或全局 advice）上声明「我来处理这个异常」"的那套机制。先给结论：它大多数时候能接住——接不住的是另外几种情况。要分清，得先知道异常在哪里、以什么类型、在哪个时机抛出来。

### 6.1 抛出来的不是 Spring 的异常

超限时抛出的都是 Tomcat 内嵌 fileupload 自己的异常类型：

```java
// org.apache.tomcat.util.http.fileupload.impl.FileItemIteratorImpl#init
if (requestSize != -1 && requestSize > sizeMax) {
    throw new SizeLimitExceededException("the request was rejected because its size ...");  // max-request-size
}
```

**它拿 `Content-Length` 先判，body 一个字节都没读。** 单个文件超 `max-file-size` 时，解析器有两种发现时机：有 `Content-Length` 就在构造 part 时预判，chunked 则读到超限才炸——两处抛的都是包了一层 `FileUploadIOException` 的 `FileSizeLimitExceededException`。异常层级是：`SizeException` 下分 `SizeLimitExceededException`（请求体超限）和 `FileSizeLimitExceededException`（单文件超限）。

`FileCountLimitExceededException`（part 个数超限）直接继承 `FileUploadException`，不在 `SizeException` 之下——这个差别后面会用上。

### 6.2 Spring 认不出类型，它在猜消息

`FileUploadBase#getItemIterator` 会先拆掉 `FileUploadIOException` 这层包装，把真正的 `SizeException` 重新抛出；所以落到 `parseParts` 手上时，它已经是干净的类型，再被统一收口：

```java
// org.apache.catalina.connector.Request#parseParts（catch 段，try 里省略）
try {
    ...
} catch (SizeException | FileCountLimitExceededException e) {   // ← 6.1 说的继承差别，就体现在这里
    parameters.setParseFailedReason(FailReason.POST_TOO_LARGE);
    checkSwallowInput();
    partsParseException = new IllegalStateException(e);         // ← 类型信息在这里被抹掉
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

这是本篇最值得盯的代码。**Spring 认不出异常类型，它是在遍历 cause 链、匹配异常消息文本。** 两个后果：

- 命中条件是"消息里同时有 exceed 和 size/length"（或 request + big/large）。Tomcat 的英文消息恰好满足；**换个容器、换个语言环境、或换一层包装，这条匹配就可能失手**，退化成 `MultipartException`。
- 即便命中，上限也是 **-1**。`getMaxUploadSize()` 返回 -1 只表示"上限未知"——真实的 `max-file-size` / `max-request-size` 已经在这一路包装里丢掉了，**不要拿它拼错误文案**。

### 6.3 抛得太早：解析发生在知道该由哪个 Controller 处理之前

```java
// DispatcherServlet#doDispatch：这里只看两行的先后顺序
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

抛异常时 `mappedHandler` 还是 `null`，一路传到异常解析器。`ExceptionHandlerExceptionResolver` 的第一道判断是 `if (handlerMethod != null)`——局部 `@ExceptionHandler` 只有 handler 已解析出来时才会被考虑；handler 为 null 时，只剩全局 `@ControllerAdvice`。

结论很硬：**写在 Controller 里的 `@ExceptionHandler(MaxUploadSizeExceededException.class)` 不会被执行，只有全局 `@ControllerAdvice` 的会。** 因为异常在"知道该由哪个 Controller 处理"之前就抛出来了。

最小例子：

```java
// Controller 里：不会执行
@ExceptionHandler(MaxUploadSizeExceededException.class)
ResponseEntity<String> local() { return ResponseEntity.status(413).body("local"); }

// @RestControllerAdvice 里：会执行
@ExceptionHandler(MaxUploadSizeExceededException.class)
ResponseEntity<String> global() { return ResponseEntity.status(413).body("global"); }
```

发一个 100 MB 请求，客户端拿到的是 `global`——`local` 一次都不会执行。局部处理器要先有所属的 Controller，而请求在 `getHandler` 之前就被拒。

还有个开关：把 `resolve-lazily` 设成 `true`，解析推迟到参数解析阶段，那时 `handlerMethod` 已经有值，Controller 里的局部处理器就能生效了。

### 6.4 没人接，也未必是 500

`MaxUploadSizeExceededException` **从 Spring 6.1 起**实现了 `ErrorResponse`（6.0.x 及更早没有）——在 Spring 6 里实现这个接口，就等于声明"这个异常自带 HTTP 状态码和问题详情"：

```java
// Spring Framework 6.1.x 节选（省略 maxUploadSize 与 getBody()）
public class MaxUploadSizeExceededException extends MultipartException implements ErrorResponse {
    private final ProblemDetail body =
            ProblemDetail.forStatusAndDetail(HttpStatus.PAYLOAD_TOO_LARGE, "Maximum upload size exceeded");
    // ...
}
```

即便你什么都不写，优先级最低的 `DefaultHandlerExceptionResolver` 也会兜底：响应未提交时执行 `sendError(413)`；已提交则只打一条 warn。**客户端拿到的 413 很可能不是你写的，是它送的。**

**分界点是 Spring 6.1（Boot 3.2），不是 6.0：** 6.0.14 还是 `extends MultipartException`，6.1.0 才 `implements ErrorResponse`；6.0 的兜底解析器虽然早有 `ErrorResponse` 分支，但异常没实现它，所以仍走 500。

所以正确读法是：**Boot 2.x（5.3.x）和 Boot 3.0 / 3.1（6.0.x）→ 500；Boot 3.2+（6.1+）→ 413。**

### 6.5 真正接不住的四类

| 情况 | 为什么接不住 |
|:---|:---|
| 请求被 Nginx / 网关拦下 | 应用根本没被调用，`DispatcherServlet` 没参与 |
| 请求在 `DispatcherServlet` 之前被拒 | `MultipartFilter`、Servlet Filter、Tomcat 参数解析都在 handler 链之外 |
| 响应已经 commit | `handleErrorResponse` 只在响应未提交时改状态码，已提交就只打一条 warn |
| 连接已经被关 | 处理器可能跑了，但写出去的字节没人读——5.5 那条路 |

有个容易踩的组合：`MultipartFilter`（Spring 提供的一个 Servlet Filter，让不经过 DispatcherServlet 的请求也能拿到解析后的 multipart）在 DispatcherServlet **之前**解析，异常发生在 filter 里，`@ControllerAdvice` 完全看不到。

## 七、把「失败」变成可诊断的响应

接不住是一回事，接住了却只写一句"上传失败"是另一回事。这一章按"离客户端由近到远"补三件事：全局 advice、网关层的 413 JSON、以及把 Tomcat 的静默异常逼进日志。

### 7.1 全局 advice：让每次拒绝都留痕

这份 advice 只有两个方法，补的正是 6.2、6.3 的缺口。

```java
@RestControllerAdvice
public class UploadExceptionAdvice {

    private static final Logger log = LoggerFactory.getLogger(UploadExceptionAdvice.class);

    @ExceptionHandler(MaxUploadSizeExceededException.class)
    public ResponseEntity<Map<String, Object>> onTooLarge(MaxUploadSizeExceededException ex) {
        // getMaxUploadSize() 在 Servlet 容器解析失败时恒为 -1，不能当真实上限用
        log.warn("upload rejected: maxUploadSize={}", ex.getMaxUploadSize(), ex);
        return ResponseEntity.status(HttpStatus.PAYLOAD_TOO_LARGE)
                .body(Map.of("code", "UPLOAD_TOO_LARGE", "message", "文件超过大小限制"));
    }

    @ExceptionHandler(MultipartException.class)   // ← 消息匹配失手时的兜底
    public ResponseEntity<Map<String, Object>> onMultipart(MultipartException ex) {
        log.warn("multipart parse failed", ex);
        return ResponseEntity.badRequest()
                .body(Map.of("code", "MULTIPART_INVALID", "message", "上传报文无法解析"));
    }
}
```

三个要点都是前面的直接后果：**必须放全局**（6.3）；**必须留 `MultipartException` 兜底**（6.2 的消息匹配不可靠）；**不要把 `-1` 当前上限**（6.2）。

### 7.2 网关层也要给 JSON

Nginx 产生的 413 和后端产生的 413 形态不一致，前端就得写两套解析。统一它：

```nginx
# 写在 server 块里
error_page 413 = @too_large;

location @too_large {
    default_type application/json;
    return 413 '{"code":"UPLOAD_TOO_LARGE","message":"请求体超过网关限制"}';
}

location /api/upload {
    client_max_body_size 30m;    # 必须 > 后端 spring.servlet.multipart.max-request-size
}
```

一条容易忽略的约束：**网关的上限要大于后端的上限。** 反过来的话，后端那句 JSON 永远用不上——Nginx 先拒，客户端拿到的是一张错误页。这和引子里那次重置不是一回事：那次是请求已经被后端拒绝、连接又在吞请求体时被关掉。

### 7.3 让 Tomcat 的静默 `IOException` 说话

把 `org.apache.coyote` 的日志级别临时开到 DEBUG（Boot 里一行 `logging.level.org.apache.coyote=DEBUG`），能看到 `abstractConnectionHandler.ioexception.debug` 那条消息。四种策略的代价：

| 做法 | 怎么配 | 代价 |
|:---|:---|:---|
| 默认 | `max-swallow-size` 2MB | 超限就断连；日志默认静默 |
| 放大或关闭上限 | 调大 `server.tomcat.max-swallow-size` | 处理线程会被"吞请求体"占住（Tomcat 侧小于 0 表示不限制） |
| 干脆不吞 | Context 属性 `swallowAbortedUploads = false`（`StandardContext` 默认 `true`）；内嵌用 `TomcatContextCustomizer` 设 | 连接立即关闭，响应可能到不了客户端 |
| 网关先拦 | `client_max_body_size` | 上限要和后端对齐 |

第三行是另一条路：`checkSwallowInput()` 只在 `swallowAbortedUploads` 为 false 时才下发 `DISABLE_SWALLOW_INPUT`，`AbstractProcessor` 收到后调 `disableSwallowRequest()` 并把状态置为 `CLOSE_CLEAN`。**默认是 `true`，所以默认走的是 5.5 那条"吞到上限再抛 `IOException`"的路。**

生产上更稳的组合：网关先拦 + 后端保留默认 `maxSwallowSize` + 拒绝路径写完整日志。

## 八、少落一次盘：把 part 的流直接交给对象存储

前面三章都在回答"为什么失败、怎么诊断"。还有一个更根本的选择：让应用进程少碰这份数据。

动手之前先纠正一个容易含糊的前提：**当 `MultipartFile` 被注入到方法参数时，这个文件已经是一个"成品"了。** Tomcat 会在进入你的方法体之前把整个请求体读完——默认是 `checkMultipart` 那一步，开了 `resolve-lazily` 也只是推迟到参数解析——内容要么完整躺在临时文件里，要么完整待在内存中（阈值见 5.4），你拿到的只是一个句柄。所以这一章的"不落盘"，准确说是**应用不再多存一份**；容器解析时的那次落盘（或内存驻留）仍然存在。真要做到字节不落地，得换掉 `MultipartFile` 这套模型——第九章的预签名直传，或者 WebFlux 的流式 multipart（`PartEvent`）。

### 8.1 "先落盘再上传"浪费在哪

`file-size-threshold` 默认 `0B`，意味着 Tomcat 几乎会把每个 part 都写进磁盘——写到哪由 `MultipartConfigElement` 的 `location` 决定，没配就是容器临时目录。最常见的写法是这样（示意：`file` 是 `MultipartFile`，`ossClient` 是存储 SDK）：

```java
Path tmp = Paths.get("/data/upload-tmp", UUID.randomUUID().toString());
file.transferTo(tmp);                       // ① 又一次落盘
try (InputStream in = Files.newInputStream(tmp)) {
    ossClient.putObject(bucket, key, in);   // ② 读回来，再发出去
}
Files.deleteIfExists(tmp);                  // ③ 还得记得删
```

一次上传，数据落两遍盘、读两遍。**这段代码里唯一必要的动作是第 ② 步。**

`transferTo` 还有两个（一）没解释的坑：落盘的 part 走 `renameTo`，源临时文件被搬走，**只能调一次**，第二次再读原路径会抛 `NoSuchFileException`；目标文件已存在会被**先删掉**——所以路径里别放客户端给的文件名。相对路径的基准在（一）3.2 讲过。

### 8.2 只做第 ② 步：把 part 的流直接交给 SDK

这里读的是容器已存好的那份，省掉的是应用的第 ① 步；单独写出来就是一个完整实现（下面用 AWS SDK v2 举例，依赖是 `software.amazon.awssdk:s3`）：

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

`RequestBody.fromInputStream(InputStream, long)` 的第二个参数是必需的，它和 `contentLength(...)` 声明的是同一个长度，**两处必须一致**。写错的后果不是"被拒"，而是**静默丢数据**：声明长度小于实际内容时，客户端只会发声明的字节数、多出来的部分被截断，而且不会报"长度不符"；反过来声明长度大于实际内容时，服务端等不到足够的字节，请求才会失败。前者最危险——业务侧会以为文件传完了。

### 8.3 边读边发的三条边界

- **`MultipartFile` 能反复开流，交给 SDK 的那个 `InputStream` 不能。** 每次 `file.getInputStream()` 都新开一个；但 `RequestBody.fromInputStream` 只持有一个流，SDK 重试靠 `reset()`，落盘的 part 返回文件流、不支持 mark/reset，第二次读就抛 `IllegalStateException`。要让 SDK 自己重试，得用每次重试都重新开流的 `ContentStreamProvider`（`RequestBody.fromContentProvider`）。
- **超时要覆盖两端，失败语义也变粗。** 一次请求里串了"客户端传到应用"和"应用传到对象存储"两段，按**两段之和**估；对象存储写失败时请求体可能已被消费一部分，只能让客户端重来。
- **"不落盘"不等于零内存。** 容器读 socket 有自己的缓冲，SDK 写 HTTP 也有缓冲。省掉的是**应用自己那一次额外的落盘与读取**。

## 九、绕过应用进程：预签名 URL 直传

流式直传省掉了落盘，但数据仍要穿过应用进程；带宽、线程、超时都还是你的。再往前一步，是把数据面整个交出去。

### 9.1 三种数据路径

下载篇的「五、架构层」是同一问题的镜像：

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
| 适用 | 小文件、需要落地前强校验 | 需要边收边处理（加密 / 压缩 / 查毒） | 大文件、高并发、内容无关的落地 |

表里的"应用流式转发"指真流式（WebFlux 的 `PartEvent`，或绕开 servlet multipart）：客户端传一块，应用就处理一块（加密、压缩、查毒）并转给对象存储，整个文件不在应用里落脚。`MultipartFile.getInputStream()` 读的是临时文件，做不到这一点。

最关键的一行是"写入前能否看到内容"：**预签名直传把数据面交出去的同时，也交出了"在写入之前检查内容"的能力。** 这是它唯一且无法绕过的代价。

### 9.2 服务端：只签一个 PUT

服务端只生成一个带签名的 PUT 地址：

```java
@PostMapping("/api/uploads/presign")
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

把 `contentType` 和 `contentLength` 一起传进签名请求，是为了把"允许传什么类型、多大"和这次授权绑在一起。到底签了哪些头，以 `PresignedRequest#signedHeaders()` 为准，客户端回传时必须一致。不签的话，一个"只允许传 2 MB 图片"的 URL 可以被拿去传别的东西。代价是客户端必须先声明精确大小，发出去的字节数一个都不能差。

### 9.3 客户端：浏览器直接 PUT

客户端拿到的就是一个能直接 PUT 的 URL：先 POST `/api/uploads/presign` 拿到 `url` 和 `key`，再把这个 URL 当 PUT 目标、把文件原样发出去，最后 POST `/api/uploads/commit` 把 `key` 交给后端落库；后端再按 9.4 做一次校验，才算数。

### 9.4 直传的边界条件

- **单次 PUT 有大小上限。** S3 单次 PUT 最大 5 GB，超过必须走分片：最多 10000 片、每片 5 MiB~5 GiB，合并按片号排序；分片直传还要**逐片签名**——`uploadId` 和 `partNumber` 都要进入签名。
- **有效期是安全边界，而且有硬上限。** 有效期内同一 URL 可反复 PUT（覆盖语义），TTL 越长泄漏后可利用时间越久；S3 预签名有效期**不能超过 7 天**，所以 15 分钟这类短 TTL 是常规起点而不是硬性规定。
- **完成不等于结束，内容校验只能后置。** 后端要用 `headObject` 确认对象存在、大小符合预期；需要查毒 / 转码时，上传完成后触发异步任务，不通过就删除对象并告知业务方。**这一步漏掉，直传就是一个任意文件落地入口。**

## 十、数据路径换了，安全责任也跟着换

选完路，还有三件事必须由服务端自己兜住。注意它们不是一张通用清单：**换一条数据路径，这三个检查点的位置就会跟着移动。**

### 10.1 文件名不能信，key 必须由服务端决定

`getOriginalFilename()` 不可信，这是 Spring 在 javadoc 里写明的：文件名可能包含目录部分或 `..` 这类恶意字符，**建议自己生成唯一文件名**。它的实现就是把 `Content-Disposition` 里解析出的字符串原样返回（`return this.filename;`）。

于是下面这行是典型的目录穿越：

```java
// ❌ 客户端可以传 ../../../../etc/cron.d/x，也可以传 /etc/passwd
Path target = Paths.get("/data/uploads").resolve(file.getOriginalFilename());
```

正确姿势分两种。**默认方案：key 由服务端生成**（UUID / 雪花 ID），原始文件名只作为元数据入库、展示时转义。**必须用原名时**，至少归一化再校验前缀：

```java
Path root = Paths.get("/data/uploads").toAbsolutePath().normalize();
Path target = root.resolve(name).normalize();
if (!target.startsWith(root)) throw new IllegalArgumentException("illegal file name");
```

更隐蔽的输入是文件名里的空字节（`\u0000`），能骗过后缀校验、截断真实路径：Tomcat 的 `Streams#checkFileName` 会直接抛 `InvalidFileNameException`，`parseParts` 随后把它包成 `ServletException`（`partsParseException = new ServletException(e)`），而不是 `IllegalStateException`——所以它通常不会被包成 `MaxUploadSizeExceededException`，只会变成通用的 `MultipartException`。**兜底处理器必须存在，否则客户端等到的是一个 500。**

### 10.2 内容校验只能后置——这条决定你选哪条路

扩展名、`Content-Type`、`filename` 都是客户端给的。只按它们做白名单，等于让上传者自己声明"我是安全的"。真正的校验要读文件头：

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

### 10.3 落点与审计：别留可执行入口

上传目录绝不能落在 webroot 或静态资源目录下——那等于给上传者开一个可访问、可执行的入口。审计日志记全 key、size、contentType、IP、租户，先过滤控制字符；失败时前后端各留记录。

## 结语：四道限额，三条数据路径

开头那个故障不是 bug，是四个独立机制叠在一起的结果：Tomcat 发现超限就拒绝（5.4、6.1）、拒绝后要吞掉剩余体（5.5）、吞不下的部分直接断连（5.5）、断连的异常默认不记日志（5.5）。缺任何一环，这个现象都不会这么"干净"。

压成一套可执行的决策顺序：

1. **先把四道限额对齐。** 从外到内逐层收窄：最外层网关（Nginx）的 `client_max_body_size` > 后端 `max-request-size` > `max-file-size`。并且记住 `maxPostSize` 只算表单字段，别拿它当文件上限。
2. **再把拒绝路径做成可读的响应。** 全局 `@ControllerAdvice` + 网关 413 JSON + 留意 `maxSwallowSize` 那条无状态码的路径。这一步投入产出比最高：把"连接被重置"变成"收到 413 JSON 且后端有日志"。
3. **然后决定数据路径。** 小文件、要写入前强校验 → 应用内处理；要边收边处理（加密 / 压缩 / 查毒）→ 真流式转发；大文件、高并发、内容无关 → 预签名直传；超大文件 → 无论哪条路都要分片。
4. **最后才谈优化。** 流式直传省掉的是应用侧那一次额外的落盘与读取；分片带来并发，也带来逐片签名、完成后校验。

**三个最值得记住的结论：**

1. `maxSwallowSize` 是"像网络问题、日志里却没有"的根因，它的异常默认只在 DEBUG 可见。
2. `@ExceptionHandler` 失效多半不是写错，而是异常抛得太早、太晚，或者根本不经过 `DispatcherServlet`。
3. 预签名直传最大的代价不是复杂度，是**你放弃了"写入前看到内容"的能力**。

### 延伸阅读

- [（一）：从报文到临时文件](/2026/10/23/Spring-文件上传全链路深度解析（一）：从报文到临时文件/)——multipart 报文格式、`parseParts` 内部实现、`MultipartFile` 的内存/磁盘阈值原理，以及 2.x→3.x 的解析器选型与配置前缀变化，本篇只做引用。
- [Spring 文件下载全方案深度解析](/2026/09/15/Spring-文件下载全方案深度解析/)——「五、架构层」的三种模式与本篇「九」互为镜像。
- [一次下载失败排查：被 Filter 吞掉的异步响应流](/2026/10/14/一次下载失败排查：被%20Filter%20吞掉的异步响应流/)——同样是"响应写了但客户端拿不到"，那边的元凶是缓存包装器，这边是连接被关闭。
- [深入 WebFlux 与 Spring Cloud Gateway](/2026/09/21/深入-WebFlux-与-Spring-Cloud-Gateway：为什么网关没有-Controller，却是一个-WebFlux-应用？/)——如果网关是 Spring Cloud Gateway，上传体在 Reactor Netty 上的处理又是另一套机制。
