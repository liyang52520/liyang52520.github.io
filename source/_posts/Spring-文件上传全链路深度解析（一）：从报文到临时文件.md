---
title: "Spring 文件上传全链路深度解析（一）：从报文到临时文件"
date: 2026-10-23 10:50:00
tags:
  - spring
  - tomcat
categories:
  - develop
---

## 引子：同一项目里的两种表现——3MB 传不上去，200MB 却传上去了

同一个项目里有两个上传接口：`POST /upload`（Spring MVC + Tomcat）传一个 3MB 的 PDF 直接报错；另一个接口（Spring WebFlux，流式接收）却敢让人传 200MB 的视频。很多人第一反应是"配置没写对"——但更值得先问一句：**一个 200MB 的文件传到服务器上，它在某一刻到底躺在哪里？**

答案很反直觉：用 Spring Boot 的默认配置，一个上传请求里**每一个文件都会先落到磁盘上**，哪怕它只有 1 字节；而**这个临时文件什么时候被删，取决于请求什么时候走完，不取决于你什么时候把它存到自己的目录里**。

这两件事，决定了上传链路上绝大多数"玄学问题"：磁盘被 `upload_*.tmp` 撑爆、`transferTo` 偶尔变慢、异常抛出后临时文件残留。

> - Spring MVC 默认用的是哪个 `MultipartResolver`？真的是"classpath 里有 commons-fileupload 就自动切换"吗？
> - 从 Boot 2.x 升到 3.x，"上传坏了"到底坏在哪儿？
> - `multipart/form-data` 在字节层面长什么样？为什么不能像表单那样整体读进内存再解析？
> - 解析发生在 Spring 里，还是 Tomcat 里？什么时候触发？
> - `MultipartFile` 的内容什么时候在内存、什么时候在磁盘？
> - 临时文件是谁创建的、谁删除的、什么时机删？
> - WebFlux 下为什么是完全不同的模型？它的限额又在哪几个配置项里？

这篇先把这条链路从头走一遍，看清楚"字节从网卡进来、到变成你手里的 `MultipartFile`"这段路。限额、故障排查和不落盘直传，留给下一篇。

> 和[Spring 文件下载全方案深度解析](/2026/09/15/Spring-文件下载全方案深度解析/)对照着看会有意思：下载是"我们从磁盘往网卡推字节"，上传是"我们从网卡往磁盘收字节"。方向一反过来，主动权、内存模型、临时文件的生命周期全都不一样。


## 一、先看清报文：`multipart/form-data` 在字节层面长什么样

### 1.1 它不是一个"结构化的 JSON"，而是一段用分隔符切开的字节流

请求头里只有一行关键信息：

```text
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW
```

`boundary` 是客户端随机生成的分隔串——它不属于内容，只是一把"刀"。body 里每个字段（part）的边界就是 `--` 加上这个串：

```text
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="title"

一份报告
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="file"; filename="report.pdf"
Content-Type: application/pdf

%PDF-1.7
...二进制字节...
------WebKitFormBoundary7MA4YWxkTrZu0gW--
```

三件事一眼可见：

- **文本字段和二进制文件混在同一个 body 里**，靠 `Content-Disposition` 区分。带 `filename` 参数的是文件，不带的是普通表单字段。
- 每个 part 有**自己的头部**（`Content-Type` 可省），头部和内容之间用一个空行分隔。
- 结束标志是 `--boundary--`，比普通分隔多两个连字符。

### 1.2 为什么不能复用 `application/x-www-form-urlencoded`

`urlencoded` 把表单序列化成 `title=%E6%8A%A5%E5%91%8A&file=...`，一切内容都要能被百分号编码。二进制这么编码会膨胀到约 3 倍——按 RFC 3986，每个非保留字节都要变成 `%XX` 三个字符；即便先 Base64 再百分号编码，也还要叠加约 33%。而且它没法表达"文件名"这种元数据——这是 RFC 7578 另外定义 `multipart/form-data` 的直接原因。

更麻烦的一点：**boundary 是在请求头里声明的，而它有可能碰巧出现在文件内容中**（概率极低但必须处理）。所以解析器不能"全局替换分隔符"，只能顺序扫描、按位置切分。

### 1.3 关键结论：body 必须边读边解析

Tomcat 的解析器是这样一个状态机：读一块字节 → 在缓冲区内找下一个 `--boundary` → 找到后解析 part 头部 → 把这一段内容写入该 part 的目标。**它不能在解析之前先知道"这个文件有多大"**：文件大小只能靠一直读到下一个 boundary 才知道，而这个 boundary 只有读到那儿才出现。

这带来两个后果，后文会反复用到：

1. **内容必须在读取过程中就被决定去向。** 读到第 N 块字节时，解析器必须已经知道这一块是留在内存还是写进临时文件——不能攒到最后统一决定。
2. **"读一点点就能拒绝"是做不到的**，除非对 `Content-Length` 做整体判断。所以按文件大小配置的上限，只能在字节流推进的过程中被触发。

顺便说一句，这也是为什么 `getParameter()` 与 `getInputStream()` 在处理上传请求时不能混用：一旦你把 body 当流读走了，参数解析就拿不到东西了。


## 二、解析发生在哪一层：Tomcat 的 `parseParts` 与 Spring 的 `MultipartResolver`

### 2.1 Tomcat：`Request#parseParts` 是真正的劳动现场

Servlet 3.0 之后，容器负责解析 multipart，应用侧拿到的是 `jakarta.servlet.http.Part`（Boot 3 / Tomcat 10.1；Boot 2 / Tomcat 9 是 `javax.servlet.http.Part`）。Tomcat 的实现入口很直接：

```java
// org.apache.catalina.connector.Request#getParts
@Override
public Collection<Part> getParts() throws IOException, IllegalStateException, ServletException {
    parseParts(true);          // ← explicit = true
    if (partsParseException != null) { /* 按类型重抛 */ }
    return parts;
}
```

`parseParts(boolean explicit)` 做的事，按顺序是：先看解析缓存（`parts != null` 就直接返回）、再取 `MultipartConfigElement`、决定临时目录、建 `DiskFileItemFactory` 与 `FileUpload`、最后 `upload.parseRequest(...)` 真正读 body。

取配置这一行是关键：

```java
// org.apache.catalina.connector.Request#parseParts
MultipartConfigElement mce = getWrapper().getMultipartConfigElement();
if (mce == null) {
    if (context.getAllowCasualMultipartParsing()) {
        // 没有 multipart-config 时，退回用 Connector 的 maxPostSize 当上限
        mce = new MultipartConfigElement(null, connector.getMaxPostSize(),
                connector.getMaxPostSize(), connector.getMaxPostSize());
    } else {
        if (explicit) {
            partsParseException = new IllegalStateException(sm.getString("coyoteRequest.noMultipartConfig"));
        } else {
            parts = Collections.emptyList();
        }
        return;
    }
}
```

也就是说：**容器并不会"自己想"怎么做上传限制，限制全部来自挂在 servlet 上的 `MultipartConfigElement`**。没有它就报 `noMultipartConfig`。

### 2.2 触发时机：懒解析，而且有两个入口

`parseParts` 有两个调用点，时机完全不同：

- `getParts()` / `getPart(name)` → `parseParts(true)`，**显式请求**。Spring MVC 走的正是这条路。
- `parseParameters()` 里，当 `Content-Type` 是 `multipart/form-data` 时 → `parseParts(false)`。

```java
// org.apache.catalina.connector.Request#parseParameters
String mediaType = MediaType.parseMediaTypeOnly(getContentType());
if ("multipart/form-data".equals(mediaType)) {
    parseParts(false);
    success = true;
    return;
}
```

第二处的意义在于：**第一次调用 `request.getParameter(...)` 也会触发整份 body 的解析**。也就是说，一个写在 Filter/Interceptor 里的 `getParameter("x")`，就可能让上传的临时文件在 Controller 之前就落盘。这一点在排查"为什么还没进方法磁盘就涨了"时非常有用。

再看 `parseParts` 内部第一行的短路条件——解析结果和解析异常都会被缓存：

```java
if (parts != null || partsParseException != null) {
    return;   // 已经解析过（成功或失败），不再重复读 body
}
```

### 2.3 Spring 侧：`MultipartResolver` 只是"包装者"，不是"解析者"

`DispatcherServlet#doDispatch` 一进来就做这一步改造：

```java
// org.springframework.web.servlet.DispatcherServlet#doDispatch（6.1.x）
processedRequest = checkMultipart(request);
multipartRequestParsed = (processedRequest != request);
```

`checkMultipart` 只在 `multipartResolver.isMultipart(request)` 为真时，才调用 `resolveMultipart(request)` 把原请求包一层。默认实现 `StandardServletMultipartResolver#resolveMultipart` 的全部工作是：

```java
return new StandardMultipartHttpServletRequest(request, this.resolveLazily);
```

而 `StandardMultipartHttpServletRequest#parseRequest` 才去调容器的 `getParts()`，把每个 `Part` 按有没有 `filename` 分成"文件"和"普通参数"：

```java
// org.springframework.web.multipart.support.StandardMultipartHttpServletRequest#parseRequest
Collection<Part> parts = request.getParts();          // ← 真正干活的是容器
for (Part part : parts) {
    ContentDisposition disposition = ContentDisposition.parse(part.getHeader(HttpHeaders.CONTENT_DISPOSITION));
    String filename = disposition.getFilename();
    if (filename != null) {
        files.add(part.getName(), new StandardMultipartFile(part, filename));
    } else {
        this.multipartParameterNames.add(part.getName());
    }
}
```

所以整条链路是**两层**：Tomcat 读字节、切 part、决定落盘；Spring 把 `Part` 适配成 `MultipartFile`。`MultipartResolver` 这个名字容易让人以为它在解析，其实它只是决定"要不要包、什么时候包、用哪个容器的解析器"。

`resolveLazily` 决定了包的时候解不解析：默认 `false`，构造函数里立刻调用 `parseRequest`，即**解析错误在 `checkMultipart` 阶段就抛出**；设成 `true` 就推迟到第一次访问文件或参数时。Boot 把它暴露为 `spring.servlet.multipart.resolve-lazily`，默认 `false`。

### 2.4 这些 Bean 是怎么被装配起来的

Boot 的 `MultipartAutoConfiguration` 在类注释里就把答案写清楚了：`ServletWebServerApplicationContext` 会把 `MultipartConfigElement` bean 关联到所有 `Servlet` bean。具体落在这两处：

```java
// ServletContextInitializerBeans#addAdaptableBeans / getMultipartConfig
MultipartConfigElement multipartConfig = getMultipartConfig(beanFactory);   // 从容器里取唯一的 MultipartConfigElement
addAsRegistrationBean(beanFactory, Servlet.class, new ServletRegistrationBeanAdapter(multipartConfig));
```

```java
// ServletRegistrationBean#configure
if (this.multipartConfig != null) {
    registration.setMultipartConfig(this.multipartConfig);   // ← 最终落到 ServletRegistration.Dynamic
}
```

而我们熟悉的 `spring.servlet.multipart.*` 配置，经由 `MultipartProperties#createMultipartConfig()` 变成那个 `MultipartConfigElement`：

```java
// org.springframework.boot.autoconfigure.web.servlet.MultipartProperties
@ConfigurationProperties(prefix = "spring.servlet.multipart", ignoreUnknownFields = false)
public class MultipartProperties {
    private DataSize maxFileSize = DataSize.ofMegabytes(1);       // max-file-size
    private DataSize maxRequestSize = DataSize.ofMegabytes(10);   // max-request-size
    private DataSize fileSizeThreshold = DataSize.ofBytes(0);     // file-size-threshold
    // ...
    public MultipartConfigElement createMultipartConfig() {
        MultipartConfigFactory factory = new MultipartConfigFactory();
        // 逐项 map 到 factory，再 createMultipartConfig()
    }
}
```

链条闭合：**application.yml → MultipartProperties → MultipartConfigElement Bean → ServletRegistration → Servlet 上的 multipart-config → Tomcat `Request#parseParts` 里的 `mce`**。

### 2.5 2.x → 3.x：真正的断点在哪

这里有两条流传很广但站不住脚的说法，值得先拆掉。

**说法一："Boot 2.x 用的是 `CommonsMultipartResolver`，3.x 才换成 `StandardServletMultipartResolver`。"**

对照源码，Boot 2.7.18 与 3.2.x 的 `MultipartAutoConfiguration` 里，这个 Bean 的写法只差 3.2.x 多出的最后一行：

```java
@Bean(name = DispatcherServlet.MULTIPART_RESOLVER_BEAN_NAME)
@ConditionalOnMissingBean(MultipartResolver.class)
public StandardServletMultipartResolver multipartResolver() {
    StandardServletMultipartResolver multipartResolver = new StandardServletMultipartResolver();
    multipartResolver.setResolveLazily(this.multipartProperties.isResolveLazily());
    multipartResolver.setStrictServletCompliance(...);   // ← 3.2 起才有这一行
    return multipartResolver;
}
```

2.x 的默认解析器从来就是 `StandardServletMultipartResolver`。那 `CommonsMultipartResolver` 为什么会出现在 2.7 的自动配置里？因为它出现在另一个方法的条件里：

```java
// Boot v2.7.18（3.2.x 已无此行）
@Bean
@ConditionalOnMissingBean({ MultipartConfigElement.class, CommonsMultipartResolver.class })
public MultipartConfigElement multipartConfigElement() { ... }
```

它的作用是：**当用户自己声明了 `CommonsMultipartResolver` 时，让 `MultipartConfigElement` 退避**——因为 Commons 解析器在应用内自己解析，不需要容器再准备一套 multipart-config。它不是"检测到 classpath 里有 commons-fileupload 就自动切换实现"。

**说法二："3.x 把解析器实现换掉了，所以升级后上传报错。"**

真正的断点在别处，有两条，都能直接查证：

**（1）Spring Framework 6.0 删除了 `CommonsMultipartResolver`。**
在 5.3.x 里，`spring-web/src/main/java/org/springframework/web/multipart/commons/CommonsMultipartResolver.java` 存在，类声明为 `public class CommonsMultipartResolver extends CommonsFileUploadSupport implements MultipartResolver, ServletContextAware`；到 6.0.0，同一路径不存在了，而且**5.3.x 全程没给它加过 `@Deprecated`**——没有弃用预警，6.0 直接移除。所以"手动声明 `CommonsMultipartResolver`"这条路在 Framework 6.0 / Boot 3.0 就断了——项目会直接编译或启动失败，而不是"行为悄悄变了"。这也正好解释了为什么 Boot 2.7 的 `multipartConfigElement()` 条件里要额外放一个 `CommonsMultipartResolver.class`：给那条即将消失的路让位。

**（2）Boot 2.0 把配置前缀从 `spring.http.multipart` 改成了 `spring.servlet.multipart`。**
Boot 1.5 的 `MultipartProperties` 上写的是 `@ConfigurationProperties(prefix = "spring.http.multipart")`，Boot 2.0 起改为 `spring.servlet.multipart` 并沿用至今。从 1.x 升上来的老项目，如果配置项还写在旧前缀下，它会被**静默忽略**——不报错，只是限制没生效。这是"升级后上传行为变了"最常见的原因。

顺带一个更晚的差异：3.2.x 的 `MultipartProperties` 里还多了一项 `strict-servlet-compliance`（默认 `false`），2.7.18 里没有。它的语义由 `StandardServletMultipartResolver#isMultipart` 决定——为 `false` 时任何 `multipart/` 开头的 Content-Type 都会被包装，为 `true` 时只认 `multipart/form-data`。

小结一下这一节：**容器解析，Spring 包装**；触发点是 `checkMultipart` 或任意一次 `getParameter`；2.x→3.x 的真实断点是"前缀改名"和"Commons 解析器被删"，而不是"默认实现换了"。

接下来看第三个问题：解析出来的字节，到底去了哪里。


## 三、字节去了哪：阈值、临时文件与生命周期

### 3.1 判断"内存还是磁盘"的那一行代码

Tomcat 把 `MultipartConfigElement` 里的三个值原样转交给 Commons FileUpload 的重新打包版本（注意包名是 `org.apache.tomcat.util.http.fileupload`，不是 `org.apache.commons.fileupload`）：

```java
// org.apache.catalina.connector.Request#parseParts
DiskFileItemFactory factory = new DiskFileItemFactory();
factory.setRepository(location.getCanonicalFile());   // 临时目录
factory.setSizeThreshold(mce.getFileSizeThreshold()); // ← 阈值

FileUpload upload = new FileUpload();
upload.setFileItemFactory(factory);
upload.setFileSizeMax(mce.getMaxFileSize());     // max-file-size
upload.setSizeMax(mce.getMaxRequestSize());      // max-request-size
```

在每个 part 对应的 `DiskFileItem` 里，真正的决策者是 `DeferredFileOutputStream`，而它继承的 `ThresholdingOutputStream` 判断是这样的：

```java
// org.apache.tomcat.util.http.fileupload.ThresholdingOutputStream
public boolean isThresholdExceeded() {
    return written > threshold;                       // 注意是严格大于
}

protected void checkThreshold(final int count) throws IOException {
    if (!thresholdExceeded && written + count > threshold) {
        thresholdExceeded = true;
        thresholdReached();                           // 切到磁盘输出流
    }
}
```

现在代入 Boot 的默认值 `file-size-threshold: 0`：只要解析器写出**第一个字节**（`written + count > 0` 必真），`thresholdReached()` 就会被调用，`DeferredFileOutputStream` 随即创建临时文件并把后续内容全部写盘。

**结论：默认配置下，上传的每一个文件都会落到磁盘的临时文件里，与文件大小无关。** 这不是 bug，是阈值 0 的直接后果。`file-size-threshold` 配成一个大于 0 的值，才会出现"小文件留在内存、大文件才落盘"的分层。

### 3.2 临时文件叫什么、在哪

临时目录的选取也在 `parseParts` 里：

```java
String locationStr = mce.getLocation();
if (locationStr == null || locationStr.length() == 0) {
    location = (File) context.getServletContext().getAttribute(ServletContext.TEMPDIR);
} else {
    location = new File(locationStr);                 // 相对路径 → 相对 TEMPDIR
    if (!location.isAbsolute()) {
        location = new File((File) context.getServletContext().getAttribute(ServletContext.TEMPDIR),
                locationStr).getAbsoluteFile();
    }
}
if (!location.exists() && context.getCreateUploadTargets()) { location.mkdirs(); }
if (!location.isDirectory()) {
    partsParseException = new IOException(sm.getString("coyoteRequest.uploadLocationInvalid", location));
    return;
}
```

要点：`spring.servlet.multipart.location` 为空时用容器的 `TEMPDIR`；配相对路径时也是相对 `TEMPDIR` 解析；目录不存在且允许创建时会 `mkdirs()`。**`location` 配错（比如指向一个不存在的、又不允许自动创建的目录）的失败点就在这里**，表现是 `IOException`，不是"上传不能用"。

文件名由 `DiskFileItem#getTempFile` 生成：

```java
final String tempFileName = String.format("upload_%s_%s.tmp", UID, getUniqueId());
tempFile = new File(tempDir, tempFileName);
```

`UID` 是类初始化时的 `UUID.randomUUID().toString().replace('-', '_')`，`getUniqueId()` 是一个进程内自增计数器。所以你在临时目录里看到的一堆 `upload_*.tmp`，就是它们。

### 3.3 `MultipartFile` 是什么：`Part` 的一层适配

Spring 侧的 `StandardMultipartFile` 是 `Part` 的薄包装：`getSize()` → `part.getSize()`、`getBytes()` → `FileCopyUtils.copyToByteArray(part.getInputStream())`、`transferTo(File)` → `part.write(dest.getPath())`。

注意 `MultipartFile` **本身不暴露"内容在内存还是磁盘"**。它只有 `getInputStream()` 这类流式入口，以及 `getBytes()` / `transferTo()` 这两类"我要一份完整内容"的入口。

### 3.4 阈值配置 ↔ 实际落点：速查表

| `file-size-threshold` | 传一个 1KB 的文件 | 传一个 100MB 的文件 | 堆内存代价 |
|---|---|---|---|
| `0`（Boot 默认） | 解析时就写进 `location` 下的 `upload_*.tmp` | 同上，边读边写 | 几乎不占堆，但每个文件都留下一个临时文件 |
| `2MB` | 全部留在堆内存（`DeferredFileOutputStream` 的字节数组） | 越过 2MB 后切到磁盘，已缓冲的 2MB 也一并写入该文件 | 阈值以下的内容全在堆上；并发上传时是主要风险 |
| `10KB` | 同样留在内存（1KB 没到 10KB） | 越过 10KB 后切到磁盘 | 堆上最多只驻留阈值大小的内容 |

判断"越过"的那一行在 `ThresholdingOutputStream#checkThreshold`：`written + count > threshold`。阈值越小，落盘越早。

`max-file-size` / `max-request-size` 与落点无关，它们决定的是"读到多少字节时抛异常"——这是下一篇的主题。

### 3.5 `transferTo` 与 `getBytes`：两种内存代价

```java
// org.apache.tomcat.util.http.fileupload.disk.DiskFileItem#write
public void write(final File file) throws Exception {
    if (isInMemory()) {
        try (OutputStream fout = Files.newOutputStream(file.toPath())) {
            fout.write(get());                    // 内存 → 目标文件：拷一次
        }
    } else {
        final File outputFile = getStoreLocation();
        size = outputFile.length();
        if (file.exists() && !file.delete()) { throw new FileUploadException(...); }
        if (!outputFile.renameTo(file)) {          // ← 同文件系统：改名即成，O(1)
            // 跨文件系统：退化成 BufferedInputStream → BufferedOutputStream 的流复制
        }
    }
}
```

- **`transferTo(dest)`**：内容在磁盘上且 `dest` 与临时目录**在同一个文件系统**时走 `renameTo`，几乎零成本；跨文件系统（比如临时目录在容器可写层、`dest` 挂在另一个卷上）就退化成整份拷贝。内容在内存时必然是一次完整写出。
- **`getBytes()`**：把整份内容读进**堆内存**，返回一个 `byte[]`。`StandardMultipartFile#getBytes` 的实现是 `copyToByteArray(part.getInputStream())`，无论内容原本在哪，**这一次一定占堆**。并发上传时，这是最容易被忽略的内存放大点。

再补两个容易踩的边界：`DiskFileItem#write` 只在第一次调用时可靠——如果它走了 `renameTo`，临时文件已经不在原处了；`part.write()` 也不保证支持绝对路径（Spring 在 `transferTo` 里对"传了绝对路径但目标没出现"的情况做了兜底复制）。

### 3.6 谁创建、谁删除、什么时机删

**创建**：解析过程中，`DeferredFileOutputStream` 越过阈值时由 `DiskFileItem#getTempFile` 生成。

**删除**有三条独立路径，看清哪条生效很重要：

**（1）解析失败时——`parseParts` 自己在 finally 里清。** Tomcat 10.1.x 的实现里，解析过程中若 `success` 仍为 `false`，会遍历已产出的 `FileItem` 逐个 `delete()`。代码给的理由是：临时副本本可由 GC 兜底，但既然已经知道上传失败，就在这里显式清掉。

**（2）请求结束时——`Request#recycle` 兜底。** Tomcat 回收请求对象时逐个删：

```java
// org.apache.catalina.connector.Request#recycle
if (parts != null) {
    for (Part part : parts) {
        try { part.delete(); } catch (Throwable t) { log.warn(...); }
    }
    parts = null;
}
```

**（3）响应写完后——Spring 主动清。** `DispatcherServlet#doDispatch` 的 `finally`：

```java
if (multipartRequestParsed || asyncManager.isMultipartRequestParsed()) {
    cleanupMultipart(processedRequest);
}
```

最终落到 `StandardServletMultipartResolver#cleanupMultipart`：

```java
for (Part part : request.getParts()) {
    if (request.getFile(part.getName()) != null) {   // 只删文件 part
        part.delete();
    }
}
```

所以要记住的判断是：

- **临时文件的默认寿命 = 一次请求**，不是"你 `transferTo` 之后就删"。你把文件搬到业务目录，是"内容被复制/改名走了"，临时文件仍然活到请求结束（`renameTo` 的情况除外）。
- **异常路径也有清理**：解析失败走（1），请求正常收尾走（2）（3）。真正会残留的，是进程被强杀、或者 GC 迟迟不回收时的兜底文件。
- 想让临时文件更早消失，只有一条正路：在放盘之前就用足够大的 `file-size-threshold` 把内容留在内存（前提是它真的小）。

`MultipartFile` 的内容在哪、怎么删，到此都清楚了。但同一份上传请求，如果跑在 WebFlux 上，上面这套结论会整体失效。


## 四、WebFlux 下是完全不同的模型

### 4.1 没有 `Part`，只有 `FilePart` 和流

WebFlux 不经过 Servlet 的 `Part` 体系。它的 `HttpMessageReader` 把 body 解析成 `Flux<Part>`，其中文件部分是 `FilePart`，普通字段是 `FormFieldPart`。如果你要边收边处理（比如边写对象存储），还有 6.0 引入的 `PartEvent`（`PartEventHttpMessageReader`），它把整个 part 拆成"头部事件 + 若干内容事件 + 结束事件"，内容以 `DataBuffer` 形式逐块下发。

**关键差别是背压（backpressure）**：MVC 侧 `Part` 一旦解析完，内容就已经完整躺在内存或磁盘上，你什么时候读是你的事；WebFlux 侧 `FilePart#content()` 是一个 `Flux<DataBuffer>`，你不 `subscribe`，数据就不会被继续读进来。这是两种模型的根本分歧。

### 4.2 "默认不落盘"是个需要修正的说法

一个流传的说法是"WebFlux 的 multipart 默认不落盘"。看默认值，这个说法不准确：

```java
// org.springframework.http.codec.multipart.DefaultPartHttpMessageReader（6.1.x）
private int maxInMemorySize = 256 * 1024;    // 每个 part 的内存上限
private int maxHeadersSize = 10 * 1024;
private long maxDiskUsagePerPart = -1;
private int maxParts = -1;
private FileStorage fileStorage = FileStorage.tempDirectory(this::getBlockingOperationScheduler);
```

这个 reader 的类注释把行为写得很清楚：**超过 `maxInMemorySize` 的 part 会被写进 `fileStorageDirectory` 下的临时文件**。而 `FileStorage.TempFileStorage#tempDirectory` 的实现是 `Files.createTempDirectory("spring-multipart-")`——也就是系统临时目录下一个以 `spring-multipart-` 为前缀的随机目录。

所以准确的表述是：**WebFlux 默认也有"内存阈值 + 落盘"的分层，阈值 256KB；只有走 `PartEvent` 流式处理时，文件内容才不会被缓存到临时文件**（Boot 的 `spring.webflux.multipart.file-storage-directory` 在 javadoc 里注明了这一点）。

### 4.3 限额对应哪些配置项

Boot 把 `DefaultPartHttpMessageReader` 和 `PartEventHttpMessageReader` 的参数都接到了同一个前缀上。`ReactiveMultipartProperties` 的注解是 `@ConfigurationProperties(prefix = "spring.webflux.multipart")`，字段与默认值如下：

| 配置项 | `ReactiveMultipartProperties` 默认值 | 落到 reader 的哪个 setter |
|---|---|---|
| `spring.webflux.multipart.max-in-memory-size` | `256KB` | `setMaxInMemorySize` |
| `spring.webflux.multipart.max-headers-size` | `10KB` | `setMaxHeadersSize` |
| `spring.webflux.multipart.max-disk-usage-per-part` | `-1`（不限） | `DefaultPartHttpMessageReader#setMaxDiskUsagePerPart` / `PartEventHttpMessageReader#setMaxPartSize` |
| `spring.webflux.multipart.max-parts` | `-1`（不限） | `setMaxParts` |
| `spring.webflux.multipart.file-storage-directory` | 空 → 用 `Files.createTempDirectory("spring-multipart-")` | `setFileStorageDirectory`（`PartEventHttpMessageReader` 无此项） |
| `spring.webflux.multipart.headers-charset` | `UTF-8` | `setHeadersCharset` |

注意最后两列：`ReactiveMultipartAutoConfiguration` 通过 `CodecCustomizer` 把这些值按 reader 类型分流映射，所以**同一个 `max-disk-usage-per-part` 在两个 reader 上落到不同的 setter**。

另外，`spring.codec.max-in-memory-size` 是**另一个东西**：它管的是"输入流需要聚合时可缓冲的字节上限"，作用于自动配置的编解码器整体，默认未设置（多数 codec 自己的默认值是 256K），与"part 落不落盘"无关。把这两个键混为一谈，是 WebFlux 上传调参时最常见的误判。（顺带一提，Boot 3.5 起该前缀已弃用，改用 `spring.http.codecs.*`，计划 4.0.0 移除。）

### 4.4 与 MVC 的对比

| 维度 | Spring MVC（Boot 3 / Tomcat 10.1） | Spring WebFlux |
|---|---|---|
| 解析者 | 容器（Tomcat `Request#parseParts`） | 框架（`DefaultPartHttpMessageReader`） |
| 编程模型 | `MultipartFile`（同步、阻塞） | `FilePart` / `Flux<Part>` / `PartEvent` |
| 内存阈值配置 | `spring.servlet.multipart.file-size-threshold`（默认 0） | `spring.webflux.multipart.max-in-memory-size`（默认 256KB） |
| 阈值以下的内容在 | 内存（但默认阈值为 0，实际总是落盘） | 内存 |
| 阈值以上的内容 | 磁盘临时文件（`location`） | 临时文件（`spring-multipart-*` 目录，可配） |
| 临时文件何时删 | 请求结束时由 Spring / Tomcat 清理 | 由 `Part#delete()` 或上层释放触发；part 尚未交出时随订阅 dispose 删除 |
| 背压 | 无（`Part` 已是完整结果） | 有（`Flux<DataBuffer>`） |
| 适合的场景 | 常规表单上传、文件落业务目录 | 大文件直传、边收边转、高并发低内存 |

一句话收口：**MVC 把复杂度交给了容器和临时文件，WebFlux 把复杂度交给了数据流和背压。**


## 五、收口：一个上传请求的完整旅程

把这一篇的结论串成一条线。下文涉及的两条基线是：**Boot 3.2.x（Spring Framework 6.1.15、Tomcat 10.1.33）**与对照用的 **Boot 2.7.18（Spring Framework 5.3.31、Tomcat 9.0.83）**。默认配置下一次 `multipart/form-data` 上传的路径是：

```text
客户端
  │  POST /upload  Content-Type: multipart/form-data; boundary=...（body 里混排文本字段与二进制文件）
  ▼
Nginx / 网关
  │  可先用 client_max_body_size 拒绝，或把 body 转发给后端（缓冲细节见第二篇）
  ▼
Tomcat（工作线程，来自线程池）
  │  CoyoteAdapter → ApplicationFilterChain → DispatcherServlet
  ▼
DispatcherServlet#doDispatch
  ├─ checkMultipart(request)
  │    └─ StandardServletMultipartResolver#resolveMultipart
  │         └─ new StandardMultipartHttpServletRequest(request, false)
  │              └─ parseRequest → request.getParts()
  │                   └─ Tomcat Request#parseParts(true)   ← 真正的字节解析在这里
  │                        ├─ 取 servlet 上的 MultipartConfigElement（来自 spring.servlet.multipart.*）
  │                        ├─ 临时目录 = location ?: ServletContext.TEMPDIR
  │                        ├─ DiskFileItemFactory.setSizeThreshold(0)  → 第一个字节就落盘
  │                        ├─ FileUpload.parseRequest(...) 边读边切 part
  │                        └─ 每个文件 part → ApplicationPart(包装 DiskFileItem) → upload_*.tmp
  ├─ getHandler / HandlerAdapter / 参数解析
  │    └─ MultipartFile 参数拿到的是那个 Part 的薄包装（内容在临时文件里）
  ├─ Controller 方法执行
  │    └─ transferTo / getBytes / 读 InputStream
  └─ finally → cleanupMultipart → StandardServletMultipartResolver#cleanupMultipart
       └─ part.delete()   临时文件在这里消失
```

回过头看开头那两个接口，答案已经落地，而且关键在于**它们根本不是同一条链路**。3MB 的 PDF 走 MVC：限制不来自你的代码，而来自 `spring.servlet.multipart.*` 映射出的 `MultipartConfigElement`，再被 Tomcat 在解析途中触发——`max-file-size` 默认 1MB，3MB 必然过不去。200MB 的视频走 WebFlux：那条链路上没有 `MultipartConfigElement`，也没有 MVC 的 `max-file-size`；`DefaultPartHttpMessageReader` 只按 `spring.webflux.multipart.max-in-memory-size`（默认 256KB）决定"留在内存还是写临时文件"，而磁盘上限 `max-disk-usage-per-part` 默认 `-1`，即不设限。**同一份 `application.yml`，两个前缀管两套完全不同的机制**——这才是"同项目、两种表现"的真相。

下一篇接着往下走：`max-file-size` 与 `max-request-size` 到底在第几个字节被触发、抛出的是哪个异常、为什么有的场景客户端看到的是连接被断开；以及当下最常用的做法——**不落盘直传**（`PartEvent` 边收边转、预签名 URL 直传对象存储）该怎么选、各自的边界在哪。


## 延伸阅读

- [（二）：限额、故障与不落盘直传](/2026/10/23/Spring-文件上传全链路深度解析（二）：限额、故障与不落盘直传/)
- [Spring 文件下载全方案深度解析](/2026/09/15/Spring-文件下载全方案深度解析/)——同一个问题的反方向：线程、拷贝与零拷贝
- [SpringBoot 请求链路（二）：从 DispatcherServlet 到方法调用](/2026/08/26/SpringBoot请求链路2：从-DispatcherServlet-到方法调用/)——`doDispatch` 与参数解析器的完整视角
- Tomcat 10.1 Connector 配置文档：<https://tomcat.apache.org/tomcat-10.1-doc/config/http.html>
