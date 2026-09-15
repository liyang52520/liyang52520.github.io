---
title: Spring 文件下载全方案深度解析
date: 2026-09-15 14:11:44
tags:
  - spring
categories:
  - develop
---

## 一、一个下载请求，到底卡住了什么？

当我们说"大文件下载把服务搞挂了"，很多人第一反应是磁盘慢、网络慢。但真正先被拖垮的，通常是**请求处理线程**。

在 Spring MVC + Tomcat 的默认模型中，一个 HTTP 请求从进入 Tomcat 到响应完全写回客户端，全程占用同一个 Tomcat 工作线程。这个线程来自 Tomcat 的线程池，默认最大约 200。如果有一个 1GB 的文件需要下载，这个线程要等到最后一个字节写入 Socket 之后才能归还。这期间它无法处理任何其他请求。200 个这样的并发下载，就能让整个服务不可用。

但问题不止一个。当数据从磁盘流向网卡时，还隐藏着第二个瓶颈：**数据在内核态与用户态之间被反复拷贝**。你的 Java 程序充当了一个低效的中间人，数据被来回搬运，白白消耗 CPU 和内存带宽。

所以，一个文件下载请求，同时面对两个独立的瓶颈：

- **瓶颈一：线程占用。** 谁在写数据，写了多久，线程池就被占用了多久。
- **瓶颈二：数据拷贝。** 数据从磁盘到网卡，经过了多少次无谓的 CPU 搬运。

Spring 提供的所有方案，以及底层 I/O 优化技术，本质上都在回答这两个问题中的一个或两个。理解这一点，是理解所有方案优劣的钥匙。


## 二、数据传输的本质：为什么必须占用一个线程

在讨论具体方案之前，先回答一个更根本的问题：数据传输到底在干什么？为什么它需要一个线程？

假设你要把一个 1GB 的文件从服务器磁盘发到客户端。这个过程拆开看，就是反复执行同一组动作：

1. 从磁盘读一块数据（比如 8KB）到内存。
2. 把这块数据写入网络连接的发送缓冲区。
3. 重复第 1、2 步，直到 1GB 全部发完。

这就是"数据传输"的全部含义。没有什么神秘的东西，就是一个循环：读一块，写一块，再读一块，再写一块。

问题在于，在传统的阻塞式 I/O 中，`read()` 和 `write()` 都是**阻塞操作**。当网卡发送缓冲区满的时候，`write()` 不会返回，调用它的线程会停在那里等待，直到缓冲区有空间。同理，如果磁盘 I/O 慢，`read()` 也会停在那里等待。

所以，一个线程执行文件传输的过程，真实情况是这样的：

```
线程：read() → 等待磁盘返回数据 → 拿到 8KB
线程：write() → 等待网络缓冲区有空间 → 写入成功
线程：read() → 等待磁盘返回数据 → 拿到 8KB
线程：write() → 等待网络缓冲区有空间 → 写入成功
...循环 13 万次，直到 1GB 传完
```

这个线程在大部分时间里不是在"干活"，而是在**等待**。但问题是：它不能去干别的事。因为 `read()` 和 `write()` 还没返回，它被"钉"在这个循环里了。

这就是为什么数据传输必须占用一个线程：**必须有人来发起这些 `read()` 和 `write()` 调用，并且在调用返回之前，这个人哪儿也去不了。**

传统的 Servlet API（`InputStream`、`OutputStream`）就是阻塞式的。它的设计就是"调用 `read()` 就等到数据来，调用 `write()` 就等到写完"。你没法用它来实现"一个线程管多个连接"。所以，在 Spring MVC 的默认模型里，一个下载请求就必须绑定一个线程，从头到尾。

理解了这一点，再看所有方案，就清楚它们在解决什么问题了。


## 三、编程层：Spring MVC 提供了哪些下载方式

### 3.1 直接写入 HttpServletResponse 的 OutputStream

最原始的方式：Controller 返回 `void`，手动获取 `response.getOutputStream()` 并写入文件内容。

```java
@GetMapping("/download")
public void download(HttpServletResponse response) throws IOException {
    response.setContentType("application/octet-stream");
    response.setHeader("Content-Disposition", "attachment; filename=file.zip");
    try (OutputStream os = response.getOutputStream();
         InputStream is = new FileInputStream(file)) {
        byte[] buffer = new byte[8192];
        int bytesRead;
        while ((bytesRead = is.read(buffer)) != -1) {
            os.write(buffer, 0, bytesRead);
        }
    }
}
```

**线程占用**：Tomcat 线程全程占用，执行上面那个 `while` 循环，直到文件传完。**数据拷贝**：传统流复制，数据经过用户态，4 次拷贝。**核心矛盾**：内存效率尚可，但线程占用无缓解。

**适用场景**：简单的内部工具，对响应控制要求极低。

### 3.2 ResponseEntity<byte[]>

将整个文件读入字节数组，封装进 `ResponseEntity` 返回。

**线程占用**：Tomcat 线程全程占用。**核心矛盾**：内存效率最差，整个文件必须驻留在堆内存中，大文件请求可能直接触发 `OutOfMemoryError`。

**适用场景**：仅限小文件，如配置文件、图标、验证码图片。

### 3.3 ResponseEntity<Resource>：三种 Resource 的分野

`Resource` 是一个接口，不同实现类的行为差异很大。

**FileSystemResource** 包装文件系统路径。`ResourceHttpMessageConverter` 通过 `StreamUtils.copy()` 将文件内容从磁盘复制到响应输出流。很多资料声称它支持"零拷贝"，实际情况是：Spring Framework 的 `ResourceHttpMessageConverter` 目前仍然使用流复制，真正的 `sendfile` 零拷贝发生在 Tomcat 的 `DefaultServlet` 处理静态资源时。不过，`FileSystemResource` 在内存效率上确实优秀，同时 `ResourceHttpMessageConverter` 原生支持 HTTP Range 请求（断点续传）。

```java
@GetMapping("/download")
public ResponseEntity<Resource> download() {
    FileSystemResource resource = new FileSystemResource(filePath);
    return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=file.zip")
            .body(resource);
}
```

**InputStreamResource** 包装已有的 `InputStream`，适合从数据库 BLOB、远程服务等非文件系统源获取数据。陷阱：流的关闭时机。如果在 Controller 方法中用 try-with-resources 包装 InputStream，方法返回前流就被关闭了，Spring 写响应时会抛出 `Stream Closed` 异常。此外，`InputStream` 只能被读取一次，无法实现断点续传。

**ByteArrayResource** 与 `ResponseEntity<byte[]>` 本质相同，内存问题也相同。

### 3.4 StreamingResponseBody：把阻塞转移出去

`StreamingResponseBody` 是 Spring 4.2 引入的函数式接口，核心思想是将文件写入操作从 Tomcat 工作线程中抽离。

```java
@GetMapping("/download")
public ResponseEntity<StreamingResponseBody> download() {
    StreamingResponseBody body = outputStream -> {
        try (InputStream is = new FileInputStream(filePath)) {
            byte[] buffer = new byte[8192];
            int bytesRead;
            while ((bytesRead = is.read(buffer)) != -1) {
                outputStream.write(buffer, 0, bytesRead);
            }
        }
    };
    return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=file.zip")
            .body(body);
}
```

要理解它到底做了什么，需要看清一个请求在 Tomcat 中的完整生命周期。

Tomcat NIO 的线程模型由三个核心组件构成：**Acceptor 线程**负责接受 TCP 连接；**Poller 线程**（默认 2 个）通过 Selector 多路复用监听所有已建立连接的 I/O 读写事件；**工作线程池**（Executor，默认最大 200）负责执行应用层业务逻辑。

一个同步下载请求的流程是：Poller 线程监听到读事件后，将任务提交给工作线程池。工作线程执行 Controller 方法，然后由这个工作线程自己循环读取文件并写入 Socket。文件传输的整个过程都绑定在这个工作线程上。

`StreamingResponseBody` 改变了这个流程。Controller 方法返回 `StreamingResponseBody` 对象后，Spring MVC 调用 `request.startAsync()`，将请求切换到异步模式。此时，Tomcat 工作线程执行完 Controller 方法就**立即归还线程池**。异步线程池中的另一个线程接手执行 lambda，完成文件读取和写入。

这里有一个容易误解的点：**数据传输阶段仍然需要一个线程来做，而且这个线程在做阻塞式的 `outputStream.write()`。** 它没有被"非阻塞化"。改变的只是"谁来做"——从 Tomcat 工作线程变成了异步线程池的线程。

Poller 线程不参与这个写入过程。它依然只负责监听事件和派发任务，不会因为你的文件正在传输而被阻塞。

所以 `StreamingResponseBody` 的本质是：**把阻塞从 Tomcat 工作线程转移到了异步线程池中的线程。** Tomcat 线程池的容量不再被慢速下载占用，但你必须为异步线程池配置足够的线程，否则它同样会成为瓶颈。如果不显式配置 `AsyncTaskExecutor`，Spring 默认使用 `SimpleAsyncTaskExecutor`，它每次调用都新建线程，高并发下线程数会失控。正确做法是通过 `WebMvcConfigurer.configureAsyncSupport()` 配置一个 `ThreadPoolTaskExecutor`。

**适用场景**：动态生成的大文件下载，以及需要低内存占用的高并发下载场景。

### 3.5 DeferredResult 与 WebAsyncTask：异步的"等待"

这两个方案解决的不是"写入慢"，而是"准备慢"。假设报表需要先调用外部系统生成文件，等待数十秒。`DeferredResult` 允许 Controller 先返回占位符，然后在任意线程中设置结果。

```java
@GetMapping("/report")
public DeferredResult<ResponseEntity<Resource>> downloadReport() {
    DeferredResult<ResponseEntity<Resource>> result = new DeferredResult<>(60_000L);
    
    CompletableFuture.runAsync(() -> {
        File file = reportService.generateReport();
        FileSystemResource resource = new FileSystemResource(file);
        result.setResult(ResponseEntity.ok()
                .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=report.xlsx")
                .body(resource));
    });
    
    return result;
}
```

**线程占用**：等待阶段释放 Tomcat 线程，写回阶段仍占用 Tomcat 线程。**核心矛盾**：只解决了"等待阶段"的线程占用。一旦结果被设置并触发 `dispatch()`，写回响应仍发生在 Tomcat 线程上。

### 3.6 编程层小结

| 方案 | Tomcat 线程 | 内存占用 | Range 支持 | 异步线程池 |
|:---|:---|:---|:---|:---|
| 直接写 OutputStream | 全程占用 | 低 | 需手动 | 无 |
| `ResponseEntity<byte[]>` | 全程占用 | 极高 | 需手动 | 无 |
| `ResponseEntity<FileSystemResource>` | 全程占用 | 低 | 原生支持 | 无 |
| `ResponseEntity<InputStreamResource>` | 全程占用 | 低 | 不支持 | 无 |
| `StreamingResponseBody` | 仅请求接收阶段 | 低 | 需手动 | 必需配置 |
| `DeferredResult` | 等待阶段释放，写回阶段占用 | 取决于包装类型 | 取决于包装类型 | 视实现而定 |

编程层解决的是"谁写、怎么写"。`StreamingResponseBody` 已经能把 Tomcat 工作线程从数据传输阶段解放出来，但它没有解决第二个瓶颈：数据仍然要经过用户态，被反复拷贝。


## 四、数据拷贝：传统流复制到底浪费在哪里

### 4.1 从编程层留下的问题说起

上一章结束时，我们有一个结论：`StreamingResponseBody` 能让 Tomcat 工作线程在数据传输阶段脱身，异步线程池接手完成文件发送。但无论谁在写数据，只要用的是 `InputStream.read()` + `OutputStream.write()` 这套传统流复制，数据从磁盘到网卡的过程中就一直在被反复搬运。

这个搬运过程到底浪费了什么？怎么才能省掉？这是本章要回答的问题。

### 4.2 一次传统流复制的完整旅程

假设你要传一个 1GB 的文件，代码就是上一章 `StreamingResponseBody` 里那段 `while` 循环。在内核层面，数据从磁盘到网卡要经历 **4 次拷贝**和 **4 次上下文切换**：

```
① DMA 拷贝：磁盘 → 内核页缓存（Page Cache）
② CPU 拷贝：内核页缓存 → 用户缓冲区（你的 JVM 堆内存）
③ CPU 拷贝：用户缓冲区 → Socket 发送缓冲区
④ DMA 拷贝：Socket 发送缓冲区 → 网卡
```

第 ① 步和第 ④ 步由 DMA 引擎完成，CPU 不参与，这是硬件层面的必要搬运，省不掉。

**第 ② 步和第 ③ 步是纯浪费。** 你的 Java 程序拿到数据后什么都没做，只是把它从内核的读缓冲区搬到了写缓冲区。对于 1GB 的文件，这意味着 1GB 的数据被额外拷贝了两遍，消耗 CPU 周期和内存带宽。

### 4.3 零拷贝的思路：让数据不经过用户态

零拷贝的核心思想非常朴素：**如果应用程序不需要修改数据，为什么要让它经过用户态？**

你的 Java 程序在文件下载场景中，确实不需要修改文件内容。它只是把数据从 A 搬到 B，不加工、不转换、不计算。既然如此，能不能告诉内核："这个文件的数据，你直接从磁盘送到网卡，别经过我的进程"？

这就是 Linux 的 `sendfile()` 系统调用做的事。数据路径变成：

```
① DMA 拷贝：磁盘 → 内核页缓存
② DMA 拷贝（gather）：内核页缓存 → 网卡
```

数据完全不经过用户态。CPU 拷贝从 2 次减少到 0 次，上下文切换从 4 次减少到 2 次。

**"零拷贝"这个名字有误导性。** 它不是说真的没有数据拷贝，而是说**没有 CPU 参与的数据拷贝，数据不经过用户态**。磁盘到内核页缓存、内核页缓存到网卡，这两次 DMA 拷贝始终存在，但由硬件完成，不占 CPU。

Linux 2.4 内核为 `sendfile` 引入了 DMA gather 操作：内核不需要把数据从页缓存拷贝到 Socket 缓冲区，而是把页缓存中对应的数据描述信息（内存地址、偏移量）记录到网络缓冲区，DMA 引擎根据这些描述信息直接将数据从页缓存拷贝到网卡，省去了内核空间中仅剩的 1 次 CPU 拷贝。

### 4.4 Java 里怎么用零拷贝：FileChannel.transferTo

Java NIO 的 `FileChannel.transferTo()` 是对底层 `sendfile` 系统调用的封装。它的签名是：

```java
public abstract long transferTo(long position, long count, WritableByteChannel target)
```

含义：把这个文件从 `position` 开始、长度为 `count` 的数据，直接送到 `target` 这个 Channel。

一个完整的用法示例：

```java
public void transferFile(Path source, WritableByteChannel target) throws IOException {
    try (FileChannel fileChannel = FileChannel.open(source, StandardOpenOption.READ)) {
        long position = 0;
        long fileSize = fileChannel.size();

        while (position < fileSize) {
            // 单次最多传 2GB，必须分片
            long chunk = Math.min(fileSize - position, Integer.MAX_VALUE);
            long transferred = fileChannel.transferTo(position, chunk, target);
            if (transferred == 0) {
                break; // 防止死循环
            }
            position += transferred;
        }
    }
}
```

**关键限制：单次调用最多传输 2GB。** 该限制源于底层 `sendfile` 系统调用对长度参数使用有符号 32 位整型定义，JVM 保守处理为 `Integer.MAX_VALUE`。超过 2GB 的文件必须循环分片调用。

### 4.5 但在 Spring MVC 里，transferTo 大概率不生效

这是最关键、也最容易被误导的地方。

**在 Spring MVC Controller 里手动调用 `transferTo`，你大概率得不到零拷贝。**

原因在于：`transferTo` 的 target 必须是一个**真正的 `SocketChannel`**，才能触发底层 `sendfile`。但在 Servlet 容器中，你拿到的是 `response.getOutputStream()`，它返回的是 `ServletOutputStream`。你用 `Channels.newChannel(outputStream)` 把它包装成 `WritableByteChannel`，**但 Tomcat 并没有为这个包装后的 Channel 实现底层的 `sendfile` 优化**。

下面是一个典型的"看起来用了零拷贝，实际没有"的例子：

```java
@GetMapping("/download")
public void download(HttpServletResponse response) throws IOException {
    Path path = Paths.get("/data/files/large-file.zip");
    response.setContentType("application/octet-stream");
    response.setHeader("Content-Disposition", "attachment; filename=large-file.zip");

    try (FileChannel fileChannel = FileChannel.open(path, StandardOpenOption.READ);
         WritableByteChannel outChannel = Channels.newChannel(response.getOutputStream())) {

        long position = 0;
        long fileSize = fileChannel.size();
        while (position < fileSize) {
            long chunk = Math.min(fileSize - position, Integer.MAX_VALUE);
            long transferred = fileChannel.transferTo(position, chunk, outChannel);
            if (transferred == 0) break;
            position += transferred;
        }
    }
}
```

这段代码看起来用了 `transferTo`，但在 Tomcat 环境下，`outChannel` 指向的是 `ServletOutputStream` 的包装，JVM 检测到目标不是真正的 `SocketChannel`，**自动退化成普通的读写循环**。数据照样经过用户态，零拷贝的优势完全没有。

Servlet API 不允许访问源 Channel；当 servlet 看到数据时，它们已经在用户空间中了。

### 4.6 那 Spring MVC 里怎么才能用上零拷贝？让 Tomcat 来做

**不要手动调用 `transferTo`。让 Tomcat 的 `DefaultServlet` 来做。**

Tomcat 的 `DefaultServlet` 在传输静态文件时，会自动使用 `sendfile`。你只需要在 Controller 里设置几个请求属性，告诉 Tomcat"这个文件你直接用 sendfile 发"：

```java
@GetMapping("/download")
public void download(HttpServletRequest request, HttpServletResponse response) throws IOException {
    File file = new File("/data/files/large-file.zip");

    response.setHeader("Content-Disposition", "attachment; filename=large-file.zip");
    response.setContentLengthLong(file.length());

    // 告诉 Tomcat 用 sendfile 发送文件
    request.setAttribute("org.apache.tomcat.sendfile.filename", file.getCanonicalPath());
    request.setAttribute("org.apache.tomcat.sendfile.start", 0L);
    request.setAttribute("org.apache.tomcat.sendfile.end", file.length());
}
```

Controller 方法返回后，Tomcat 的 `Poller` 线程会检测到 `sendfile` 操作，由连接器本身完成文件传输，**数据完全不经过你的应用代码，也不经过用户态**。`NioEndpoint.Poller` 中有一个专门的 `processSendfile` 方法处理 `sendfile` 操作，初始的非阻塞 `sendfile` 调用通常立即返回，由内核异步完成传输。Tomcat 的 `sendfile` 支持需要请求属性 `org.apache.tomcat.sendfile.support` 为 `Boolean.TRUE`，并且需要正确设置 `Content-Length`。Servlet 设置好三个请求属性后，**不应再向响应写入任何数据**，因为响应体将由连接器自己发送。

**注意**：使用 `sendfile` 会禁用 Tomcat 可能执行的响应压缩。大于 48KB 的静态文件将以非压缩方式发送。可以通过连接器的 `useSendfile` 属性关闭此功能。

### 4.7 那 transferTo 对什么场景真正有效？Netty

Servlet 容器里拿不到真正的 SocketChannel，那什么场景下能拿到？

**答案是 Netty。** Netty 直接操作 SocketChannel，其 `FileRegion` 封装了文件通道的 `transferTo` 方法。在 Netty 的 native transport（EPOLL）中，`writeDefaultFileRegion()` 调用 `socket.sendFile()`，直接映射到 Linux 的 `sendfile()` 系统调用，数据从文件 Page Cache 到 Socket 缓冲区完全在内核态完成，无用户态拷贝。

一个完整的 Netty 零拷贝文件下载示例：

```java
public class FileDownloadHandler extends ChannelInboundHandlerAdapter {

    private final String filePath;

    public FileDownloadHandler(String filePath) {
        this.filePath = filePath;
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        if (msg instanceof HttpRequest) {
            File file = new File(filePath);
            RandomAccessFile raf = new RandomAccessFile(file, "r");
            long fileLength = raf.length();

            HttpResponse response = new DefaultHttpResponse(
                    HttpVersion.HTTP_1_1, HttpResponseStatus.OK);
            response.headers().set(HttpHeaderNames.CONTENT_LENGTH, fileLength);
            response.headers().set(HttpHeaderNames.CONTENT_TYPE, "application/octet-stream");
            response.headers().set(HttpHeaderNames.CONTENT_DISPOSITION,
                    "attachment; filename=\"" + file.getName() + "\"");

            ctx.write(response);

            // 关键：使用 FileRegion 实现零拷贝
            // FileRegion 的 transferTo 底层调用 sendfile
            FileRegion region = new DefaultFileRegion(raf.getChannel(), 0, fileLength);
            ctx.write(region);

            ctx.writeAndFlush(LastHttpContent.EMPTY_LAST_CONTENT)
                    .addListener(ChannelFutureListener.CLOSE);
        }
    }
}
```

Netty 的 `DefaultFileRegion` 在 Linux 上会调用 `FileChannel.transferTo()`，由于目标是一个真正的 `SocketChannel`（Netty 管理的），底层的 `sendfile` 能够正常触发，零拷贝真实生效。

### 4.8 实测数据：零拷贝到底快多少

有一组实测数据（1GB 文件复制，NVMe SSD，OpenJDK 17）：

| 文件大小 | transferTo 耗时 | 传统 IO 耗时 | 性能提升 |
|:---|:---|:---|:---|
| 100MB | 32ms | 156ms | ~4.9x |
| 500MB | 147ms | 892ms | ~6.1x |
| 1GB | 310ms | 1.8s | ~5.8x |

随着文件体积增大，`transferTo()` 的优势越发明显，整体性能比传统 IO 快 4~6 倍以上。

零拷贝对上下文切换的减少同样显著。对于 1.4MB 的数据，传统方式在 32KB 用户缓冲区下经历了约 176 次上下文切换，而使用零拷贝仅需 2 次。

### 4.9 本章小结

| 问题 | 答案 |
|:---|:---|
| 传统流复制浪费在哪 | 数据从内核页缓存到用户缓冲区、再从用户缓冲区到 Socket 缓冲区，两次 CPU 拷贝是纯浪费 |
| 零拷贝省掉了什么 | 省掉了这两次 CPU 拷贝，数据不经过用户态 |
| Java 里怎么用 | `FileChannel.transferTo()`，底层是 `sendfile` 系统调用 |
| 在 Spring MVC 里能直接用吗 | 大概率不能，因为拿不到真正的 SocketChannel |
| 那 Spring MVC 里怎么用 | 设置 Tomcat 的 sendfile 请求属性，让 `DefaultServlet` 来传 |
| 什么场景下 transferTo 真正有效 | Netty 的 `FileRegion`，因为 Netty 直接操作 SocketChannel |
| 它解决线程占用问题吗 | 不解决。调用它的线程仍然会阻塞到传输完成 |


## 五、架构层：把文件传输从应用进程中剥离

传输层的分析已经说明了一个关键事实：在应用进程内部，你很难同时解决线程占用和数据拷贝两个瓶颈。`StreamingResponseBody` 能释放 Tomcat 工作线程，但数据仍然要经过应用进程；`transferTo` 在 Servlet 环境里大概率退化。要同时解决两个瓶颈，必须把文件传输从应用进程中剥离出去。

核心思路是：**让 Spring 当调度员，让 Nginx、对象存储、CDN 当搬运工。**

### 5.1 模式 A：预生成 + 对象存储 + 预签名 URL

**思路**：文件不放在你的服务器上，放在对象存储（比如阿里云 OSS、腾讯云 COS、AWS S3）。Spring 只负责生成一个临时的、带签名的下载链接，客户端拿着这个链接直接去对象存储下载。

**流程**：

1. 客户端请求下载。
2. Spring 校验权限，检查文件是否已经生成。
3. 如果没生成，提交一个异步任务去生成，告诉客户端"处理中，请稍后查询"。
4. 文件生成后上传到对象存储。
5. 客户端再次请求，Spring 返回一个预签名 URL。
6. 客户端直接访问这个 URL，从对象存储下载文件。**流量完全不经过 Spring。**

**预签名 URL 代码示例**（以腾讯云 COS 为例）：

```java
@Service
public class CosDownloadService {
    @Autowired
    private COSClient cosClient;

    @Value("${tencent.cos.bucket-name}")
    private String bucketName;

    public String getDownloadUrl(String key, String downloadFileName) {
        GeneratePresignedUrlRequest req =
            new GeneratePresignedUrlRequest(bucketName, key, HttpMethodName.GET);

        Date expirationDate = new Date(System.currentTimeMillis() + 30 * 60 * 1000);
        req.setExpiration(expirationDate);

        ResponseHeaderOverrides responseHeaders = new ResponseHeaderOverrides();
        try {
            String encodedFileName = URLEncoder.encode(downloadFileName, "UTF-8")
                .replace("+", "%20");
            responseHeaders.setContentDisposition(
                "attachment; filename=\"" + encodedFileName + "\"");
        } catch (UnsupportedEncodingException e) {
            responseHeaders.setContentDisposition("attachment; filename=\"file\"");
        }
        req.setResponseHeaders(responseHeaders);

        URL url = cosClient.generatePresignedUrl(req);
        return url.toString();
    }
}
```

**优点**：Spring 线程几乎瞬间释放；带宽由对象存储/CDN 承担；天然支持大文件、断点续传、CDN 加速。

**缺点**：需要引入对象存储；文件生成是异步的，客户端需要轮询。

**适用场景**：动态生成大报表、大数据导出、高并发下载。

### 5.2 模式 B：Nginx X-Accel-Redirect

**思路**：文件放在你的服务器本地磁盘上，但不由 Spring 来传。Spring 只做鉴权，然后告诉 Nginx"把那个文件发出去"。

**流程**：

1. 客户端请求下载。
2. Nginx 把请求转发给 Spring。
3. Spring 校验权限，返回一个特殊的响应头：`X-Accel-Redirect: /internal/files/report.zip`。
4. Nginx 看到这个头，知道"这不是给客户端的响应，是给我的指令"，于是内部重定向到实际文件，用 sendfile 零拷贝发给客户端。

**Nginx 配置示例**：

```nginx
location /internal/files/ {
    internal;
    alias /data/files/;
}
```

`internal` 表示这个路径只能由 Nginx 内部访问，外部不能直接请求。

**Spring 端代码**：

```java
@GetMapping("/download")
public void download(HttpServletResponse response) {
    // ... 鉴权逻辑 ...
    response.setHeader("X-Accel-Redirect", "/internal/files/report.zip");
    response.setHeader("Content-Disposition", "attachment; filename=report.zip");
}
```

这里的零拷贝是**真正生效的**，因为传输发生在 Nginx 进程内部，Nginx 直接操作 Socket，`sendfile` 的 DMA gather 优化可以正常使用。Spring 只做鉴权，线程瞬间释放。

**优点**：不需要对象存储，文件放本地磁盘即可；Nginx 传文件效率极高，零拷贝真实生效；Spring 线程快速释放。

**缺点**：文件必须在 Nginx 能访问的路径；需要配置 Nginx。

**适用场景**：文件已存在本地磁盘、并发大但不想引入对象存储、传统企业应用。

### 5.3 模式 C：独立文件服务

**思路**：把文件下载单独拆成一个服务，可以用 Nginx、Go、Rust 或专门的静态文件服务器。Spring 只负责签发一个短期 Token，客户端拿着 Token 去文件服务下载。

**流程**：

1. 客户端请求下载。
2. Spring 校验权限，签发一个短期 Token（比如 5 分钟有效）。
3. Spring 返回文件服务的地址和 Token。
4. 客户端带着 Token 去文件服务请求下载。
5. 文件服务验证 Token 有效后，从存储读取文件，传给客户端。

**优点**：Spring 和文件服务完全解耦，可以独立扩容；文件服务可以用最适合传文件的技术栈。

**缺点**：多一个服务要部署、监控、维护。

**适用场景**：大型系统，文件下载是核心能力，需要独立扩容。

### 5.4 架构层方案对比

| | 对象存储 + 预签名 URL | Nginx X-Accel-Redirect | 独立文件服务 |
|:---|:---|:---|:---|
| 文件放哪 | 对象存储 | 本地/共享磁盘 | 任意 |
| 谁传文件 | 对象存储/CDN | Nginx | 独立文件服务 |
| Spring 做什么 | 生成预签名 URL | 返回特殊响应头 | 签发 Token |
| 并发能力 | 极强 | 强 | 极强 |
| 适合动态生成文件 | 适合 | 不适合未落盘文件 | 适合 |
| 运维复杂度 | 中 | 低 | 高 |


## 六、决策框架：两个瓶颈，三层解法

| 层次 | 解决什么 | 代表方案 | 关键限制 |
|:---|:---|:---|:---|
| 编程层 | 线程占用 | `StreamingResponseBody` | 数据仍经过用户态 |
| 传输层 | 数据拷贝 | Tomcat `sendfile` 请求属性 / Nginx `sendfile` / Netty `FileRegion` | 在 Spring MVC 里手动调 `transferTo` 大概率退化 |
| 架构层 | **同时解决两个** | Nginx X-Accel-Redirect / 对象存储 | 引入额外组件 |

**决策路径：**

**第一层判断：文件大小。** 小于 1MB，`ResponseEntity<byte[]>` 最简单。GB 级别，必须考虑 `StreamingResponseBody` 或架构层方案。

**第二层判断：是否需要断点续传。** 如果需要，`ResponseEntity<FileSystemResource>` 是唯一开箱即用的选择。

**第三层判断：并发量。** 并发不高，`FileSystemResource` 可接受。并发数百以上，`StreamingResponseBody` 释放 Tomcat 工作线程，但必须配置异步线程池。

**第四层判断：是否达到架构瓶颈。** "准备耗时 + 并发大 + 文件大"三者叠加时，不要试图在 Spring MVC 里用 `transferTo` 硬扛。文件在本地且已有 Nginx → 模式 B；文件动态生成且愿意上云 → 模式 A；系统大且需要独立扩容 → 模式 C。

| 场景 | 推荐方案 | 核心理由 |
|:---|:---|:---|
| 小型文件下载 | `ResponseEntity<byte[]>` | 实现最快 |
| 大型本地文件，需断点续传 | `ResponseEntity<FileSystemResource>` | 内存友好，原生 Range |
| 大型动态文件，高并发 | `StreamingResponseBody` + 异步线程池 | 释放 Tomcat 工作线程 |
| 从数据库/网络流下载 | `ResponseEntity<InputStreamResource>` | 适合非文件系统源 |
| 文件准备过程耗时 | `DeferredResult<ResponseEntity<Resource>>` | 异步等待文件就绪 |
| 文件在本地，已有 Nginx | **模式 B**：Nginx X-Accel-Redirect | 零拷贝真实生效，线程释放 |
| 文件动态生成，并发高 | **模式 A**：对象存储 + 预签名 URL | 带宽由对象存储承担 |
| 大型系统，需独立扩容 | **模式 C**：独立文件服务 | 控制面与数据面解耦 |


## 七、结语

Spring 文件下载的复杂性，源于它同时面对两个独立的瓶颈：线程占用和数据拷贝。编程层的方案在"谁写"上做取舍，传输层的零拷贝在"数据怎么搬"上做优化，架构层则在两个瓶颈都扛不住时，把传输从应用进程中剥离。

**三个最重要的实践结论**：

1. **不要用 `byte[]` 传大文件**，会 OOM。
2. **不要自己写 `FileChannel.transferTo`**，在 Spring MVC 里大概率不生效。要零拷贝就用 Tomcat 的 sendfile 请求属性、Nginx 的 `X-Accel-Redirect`，或者用 Netty 的 `FileRegion`。
3. **用 `StreamingResponseBody` 一定要配置异步线程池**，否则默认的 `SimpleAsyncTaskExecutor` 会每次新建线程。

真正重要的不是记住"哪个方案最好"，而是建立一套清晰的分析框架：先判断文件大小和并发量级，再判断是否需要断点续传，最后判断文件准备过程的耗时特征。当两个瓶颈都成为问题时，最优雅的架构往往不是"在 Spring 里硬扛"，而是将文件存储交给对象存储或 CDN，Spring 只负责签发临时访问凭证。