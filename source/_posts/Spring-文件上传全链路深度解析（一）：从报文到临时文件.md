---
title: "Spring 文件上传全链路深度解析（一）：从报文到临时文件"
date: 2026-10-23 10:50:00
tags:
  - spring
  - tomcat
categories:
  - develop
---

## 引子：3MB 传不上去，200MB 却传上去了

同一个项目里有两个上传接口。一个用 Spring MVC 写的 `POST /upload`，传 3MB 的 PDF 会报错；另一个用 WebFlux 写的接口，用户往上传 200MB 的视频，什么事也没有。

遇到这种差别，第一反应通常是"哪个配置没写对"。先别急着翻配置，换一个问法：**一个文件正在上传的那几秒里，它究竟躺在服务器的什么地方？**

答案和直觉不太一样。用 Spring Boot 的默认配置，上传请求里的每一个文件都会先落到磁盘上，哪怕它只有 1 字节；而这个临时文件什么时候被删，取决于请求什么时候走完，不取决于你什么时候把它存到自己的目录里。

这两件事，撑起了上传链路上绝大多数"玄学问题"：临时目录被慢慢撑爆、异常抛出后临时文件残留、文件已经存进业务目录了却还占着一份空间。

> - `multipart/form-data` 在字节层面长什么样？为什么不能像普通表单那样整体读进内存再解析？
> - 解析这件事是 Spring 干的，还是 Tomcat 干的？什么时候开始干？
> - Spring MVC 默认用哪个组件把上传请求包装成可用的对象？为什么有的资料说"classpath 里有 commons-fileupload，它就会自动切换实现"？
> - `MultipartFile` 的内容什么时候在内存、什么时候在磁盘？
> - 临时文件是谁创建的、谁删除的、什么时机删？
> - 同一个项目里，另一个走 WebFlux 的接口为什么反而敢收 200MB？

这篇按上面的顺序走一遍，看清"字节从网卡进来、到变成你手里的 `MultipartFile`"这段路。限额、故障排查和不落盘直传，留给下一篇。

源码与配置以 Boot 3.2.x 为基线（按 3.2.12 的依赖清单，对应 Spring Framework 6.1.15、Tomcat 10.1.33）。

> 和[Spring 文件下载全方案深度解析](/2026/09/15/Spring-文件下载全方案深度解析/)对照着看会有意思：下载是"我们从磁盘往网卡推字节"，上传是"我们从网卡往磁盘收字节"。方向一反过来，主动权、内存模型、临时文件的生命周期全都不一样。


## 一、先看清报文：上传请求在网线上到底是什么

### 1.1 一个 body 里装着好几样东西，靠 boundary 切开

写上传接口的时候，它看起来和普通表单没什么两样：字段有名字，文件字段也能起个名字。但真正发到网线上的东西，是另一个样子。请求头里只有一行关键信息：

```text
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW
```

`boundary` 是客户端随手生成的一串随机字符。它不属于内容，只是一把"刀"：body 里每个字段（part）靠 `--` 加上它来分界。

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

- 文本字段和二进制文件混在同一个 body 里。每个 part 有自己的头部，头部里的 `Content-Disposition` 描述"这个 part 叫什么名字、带不带文件名"：带 `filename` 的是文件，不带的是普通字段。
- 头部和内容之间空一行。
- 整个 body 以 `--boundary--` 结束，比普通分界多两个连字符。

### 1.2 上传得另起一套格式：普通表单装不下二进制，也存不住文件名

先退一步。表单提交本来就有一种格式，登录、搜索用的都是它：

```text
Content-Type: application/x-www-form-urlencoded

title=%E4%B8%80%E4%BB%BD%E6%8A%A5%E5%91%8A&page=2
```

它的规矩是，所有内容都当文字处理，写不进 URL 的字节一律转义。上面那个"一"字在 body 里占了九个字符，因为它被拆成了 `%E4%B8%80`。

把这套规矩用在上传上有两个问题。一是膨胀：几乎所有二进制字节都要转义，发出去的数据量会涨到三倍左右，3MB 的 PDF 要占 9MB 上下。二是它没有地方放文件名——这个格式只能表达"某个字段的值是什么"，说不出"这个字段是一个文件，它叫 report.pdf"。而服务端存盘的时候，文件名往往是唯一的线索。

所以上传另起了一套格式，就是上一节那个 `multipart/form-data`：内容不再转义，原样拼进 body，字段之间用 boundary 隔开。

### 1.3 格式这么定，解析就只能边读边做

"原样拼进 body"这句话，把麻烦转交给了写解析器的人。

boundary 是客户端随机生成的，服务端事先不知道，只能先从 `Content-Type` 里读出来。麻烦在于，这串字符理论上可以出现在文件内容当中——概率极低，但必须处理。所以解析器没有"先按分隔符把 body 切开、再一段段看"的余地，它只能从头往后扫：扫到下一个 boundary，才知道上一个字段到这里结束。

这样一来，两件事就没得选。它事先不知道文件有多大，大小不在任何头部里，只有读到下一个 boundary 才能确定；它也不可能把整个 body 先攒进内存，一个 200MB 的视频没有地方放，几百个并发上传更会把服务撑垮。

于是解析器在读第一块字节的时候，就得决定这块内容去哪：留在内存，还是写进磁盘上的临时文件。这个在读取途中做出的决定，是后面所有问题的源头。

知道了 body 长什么样、解析器只能怎么读，接下来该问的是：这段"读"，是谁在干？

## 二、谁在读这些字节：Tomcat 干活，Spring 只包一层

第一章留下的问题是"谁在读"。分清这一层不是为了猎奇，它一次性回答了三个很具体的问题：

> - 你写在 `application.yml` 里的上传上限，凭什么能管住容器的解析？
> - 为什么 Filter 里只是读一下请求参数，文件就会在进 Controller 之前落盘？
> - 上传留下的临时文件，谁创建、谁负责删？

三个问题指向同一个事实：读 body 是容器的活，Spring 只是在外面包了一层。

### 2.1 真正伸进 body 里读字节的，是 Tomcat 的 Request

先给答案：不是 Spring 在读，是 Servlet 容器在读。Spring MVC 跑在 Tomcat 上时，真正逐字节扫 boundary、切 part、决定每段内容去哪的，是 Tomcat 的代码。

在贴代码之前，先说清这些源码在哪。Tomcat 里代表"一个请求"的对象是 `org.apache.catalina.connector.Request`——可以把它想成 Tomcat 内部记这次请求的账本，body、参数、上传的 part 都记在上面；它来自 `tomcat-embed-core` 这个 jar，也就是 Spring Boot 内嵌 Tomcat 时用的那个。下面还会用到 `spring-webmvc`、`spring-web`、`spring-boot-autoconfigure`，它们在 IDEA 里按 `Cmd+O`（Windows 是 `Ctrl+N`）输入类名就能打开，提示下载源码时点一下即可。

上传的解析入口，是 `Request` 上的 `parseParts`。它按顺序做四件事：先看解析结果有没有缓存（有就直接返回），再取这次上传的配置，接着决定临时目录、建好上传处理器，最后才真正开始读 body、边读边切出每个 part。

其中"取配置"这一步最关键：

```java
// org.apache.catalina.connector.Request#parseParts
MultipartConfigElement mce = getWrapper().getMultipartConfigElement();
if (mce == null) {
    if (context.getAllowCasualMultipartParsing()) {
        // 没有单独配置上传规则时，退回用 Connector 的表单参数上限
        mce = new MultipartConfigElement(null, connector.getMaxPostSize(), ...);
    } else {
        if (explicit) { partsParseException = new IllegalStateException(...); } // 应用明确要 part：抛错
        else { parts = Collections.emptyList(); }                              // 只是顺带解析：当作没有 part
        return;
    }
}
```

`MultipartConfigElement` 是一个 Servlet 规范定义的配置对象，装着四样东西：临时目录、单文件上限、整份请求上限、内存阈值。代码里的 `getWrapper()` 取到的，是当前这个 servlet 在容器里的包装对象，那份配置就挂在它上面。所以这段代码说明的是：容器不会自己决定上传怎么限制，**限制全部来自挂在 servlet 上的那份配置**；拿不到它，明确要 part 的场景就只能报错（报错信息是 `coyoteRequest.noMultipartConfig`，意思是"这个 servlet 没有配上传规则"）。

`getAllowCasualMultipartParsing()` 是 Tomcat 上的一个开关，默认关闭，允许"没配上传规则也照解析"；`maxPostSize` 是 Connector 限制表单参数总大小的上限（默认 2MB），在这里被拿来兜底。在 Spring Boot 的默认配置下，这两条都走不到——Boot 一定会把配置准备好，先记住这个缺口，2.4 会回来补。

### 2.2 什么时候开始读：谁先开口要，谁就触发解析

Tomcat 不会在请求一进来就把 body 读掉（所有请求都是，不只是对于文件上传，body可能作为字节流存放在TCP 连接/内核缓冲区/Tomcat 连接器中）。它在等一个信号：有人开口要。

要的方式有两种。第一种，要文件：某个组件调用 `Request#getParts()`，明确说"把上传的 part 给我"，这时 `parseParts` 收到的是 `true`。第二种，要参数：任何一处代码调用 `request.getParameter("...")`，比如使用注解@RequstParam读取表单内参数（也就是form-data），Tomcat 也会顺手把整份 body 解析掉——因为它得先读完，才能确认这些参数里有没有来自 form-data 的字段，这时 `parseParts` 收到的是 `false`。后者对应 `Request` 里"取参数"的入口 `parseParameters`：

```java
// org.apache.catalina.connector.Request#parseParameters（Tomcat 10.1.x 较新版本）
String mediaType = MediaType.parseMediaTypeOnly(getContentType());
if ("multipart/form-data".equals(mediaType)) {
    parseParts(false);
    success = true;
    return;
}
```

同一个 `parseParts`，一个被明确调用、一个被顺手调用，差别就在上一节那段代码里：拿不到上传配置时，明确要 part 会直接抛错，顺手解析则安静地返回一个空列表。

这里的坑在于：解析是一次性的，而且一次就是整份 body。所以在 Controller 执行之前，只要有一个 `getParameter` 路过，磁盘上就已经多出那个临时文件了。很多人问"我还没开始处理文件，磁盘怎么就先涨了"，答案常常就在这里。

还有一件事值得记住：body 只能被读一次。`getParameter()` 和 `getInputStream()` 都是 `HttpServletRequest` 上的方法，一个要把 body 当参数解析，一个要直接把原始字节交出来；谁先读走，另一个就拿不到完整内容了。

### 2.3 Spring 只做包装：把 Part 套成 MultipartFile

在知道容器是去做实际解析工作后，Spring 要做的事就简单了：在你拿到 MultipartFile 对象之前，调用容器解析，把原始请求包一层。

**入口**在 `DispatcherServlet`——Spring MVC 的前端控制器，所有请求都先到它这里，此刻tomcat还没有开始解析文件（这对理解整个处理链路十分关键）。它的 `doDispatch` 一开始就做这一步改造：

```java
// org.springframework.web.servlet.DispatcherServlet#doDispatch
processedRequest = checkMultipart(request);
```

`checkMultipart` 只在请求确实是 multipart 时，才让 `MultipartResolver`（Boot 默认给的是 `StandardServletMultipartResolver`）去包装。这个解析器的 `resolveMultipart` 只有一行：`new StandardMultipartHttpServletRequest(request, resolveLazily)`；真正的解析发生在后者的 `parseRequest` 里：

```java
// org.springframework.web.multipart.support.StandardMultipartHttpServletRequest#parseRequest
Collection<Part> parts = request.getParts();   // ← 真正干活的是容器
// 逐个 part 看 Content-Disposition：带 filename 的包成 StandardMultipartFile（文件），
// 不带的记为普通参数
```

所以链路是两层：容器读字节、切 part、决定落盘；Spring 把 `Part` 适配成 `MultipartFile`。

上面那个 `resolveLazily` 对应配置项 `spring.servlet.multipart.resolve-lazily`，默认 `false`，意思是"包的时候立刻解析"——也就是说默认配置下，解析发生在进入 Controller 之前。设成 `true`，解析会推迟到你第一次碰文件或参数时，好处是解析错误晚一点抛。但要记住：它管不了 2.2 里那种由 `getParameter` 引发的解析，容器该读还是会读。

### 2.4 配置是怎么到容器手里的：在 servlet 注册那一刻交出去

回到 2.1 留的缺口：那份 `MultipartConfigElement` 从哪来？

链路并不长。`application.yml` 里的 `spring.servlet.multipart.*` 先被 Boot 的 `MultipartProperties` 读走（就在 `spring-boot-autoconfigure` 里），由它生成一个 `MultipartConfigElement` bean；注册 `DispatcherServlet` 时，这个 bean 被交给 servlet 的 registration，容器再把它存到对应 servlet 上。最后取出来的地方，就是 2.1 里那句 `getWrapper().getMultipartConfigElement()`。

三个默认值可以直接看 `MultipartProperties` 的字段：

```java
// org.springframework.boot.autoconfigure.web.servlet.MultipartProperties
private DataSize maxFileSize = DataSize.ofMegabytes(1);     // max-file-size：单文件 1MB
private DataSize maxRequestSize = DataSize.ofMegabytes(10); // max-request-size：整份请求 10MB
private DataSize fileSizeThreshold = DataSize.ofBytes(0);   // file-size-threshold：0
```

看到第一个数，开头那个 3MB 的 PDF 为什么直接报错就有答案了：限制不来自你的代码，来自这份在 servlet 注册时就交出去的配置。第三个数先记住，第三章会说它为什么重要。

这里的要点是"交出去的时机"：这些值不在 Controller 里生效，也不在 Spring 的解析器里生效，而是在 servlet 注册那一刻就被交给了容器。所以想改上传上限只有两条路：改配置，或者自己提供一个 `MultipartConfigElement` bean。

`StandardServletMultipartResolver` 本身只有两个开关：一个是是否懒解析，另一个是 3.2 才加的 `strict-servlet-compliance`（默认 `false`——此时任何 `multipart/` 开头的 Content-Type 都会被当成上传处理，改成 `true` 就只认 `multipart/form-data`）。里面没有任何"上限"的设置项，上限根本不归它管。

接下来看第三个问题：解析出来的字节，去了哪里。

## 三、读出来的字节去了哪：默认配置下每个文件都会落盘

第二章讲清了"谁在读、什么时候读"，但没讲读完放在哪。这个问题直接决定你的磁盘会不会涨、内存会不会爆，也是"我只传了个几百 KB 的表单，临时目录里怎么多了一堆文件"的答案。

### 3.1 一个默认值为 0 的阈值，决定了每个文件都会落盘

你可能以为"小文件留内存、大文件才落盘"是默认行为。不是。决定这件事的阈值，Boot 默认给的是 0。

Tomcat 把配置里的值原样转交给它自带的一个上传库。这个库是 Commons FileUpload 在 Tomcat 里的重新打包版本，包名是 `org.apache.tomcat.util.http.fileupload`，不是 `org.apache.commons.fileupload`——它和 Spring 自己那个 `CommonsMultipartResolver`（应用内自己解析的那条老路）不是一回事：前者是容器内部真正干活的解析库，后者是 Spring 提供的一条可选实现。

```java
// org.apache.catalina.connector.Request#parseParts
DiskFileItemFactory factory = new DiskFileItemFactory();
factory.setRepository(location.getCanonicalFile());   // 临时目录
factory.setSizeThreshold(mce.getFileSizeThreshold()); // ← 阈值，Boot 默认 0
```

每个 part 在解析时都会变成一份"文件项"（`DiskFileItem`），它背后是一个"先写内存、超过阈值就改写磁盘"的输出流。做这个切换判断的，是这个输出流的父类：

```java
// org.apache.tomcat.util.http.fileupload.ThresholdingOutputStream
protected void checkThreshold(final int count) throws IOException {
    if (!thresholdExceeded && written + count > threshold) {
        thresholdExceeded = true;
        thresholdReached();     // 从这里开始，后续内容写进临时文件
    }
}
```

代入默认值 0：写出第一个字节时 `written + count > 0` 就成立，于是立刻切到磁盘。**默认配置下，每个 part 都会先落成临时文件——文件字段和普通文本字段都一样，与大小无关。**这不是 bug，是阈值 0 的直接后果；把 `file-size-threshold` 配成大于 0 的值，才会出现"小的留内存、大的才落盘"的分层。

### 3.2 临时文件落在 work 目录下，名字是 upload_*.tmp

要清磁盘、要排查"文件残留在哪"，第一个问题就是路径。临时目录的选择也在 `parseParts` 里：

```java
// org.apache.catalina.connector.Request#parseParts
String locationStr = mce.getLocation();
if (locationStr == null || locationStr.length() == 0) {
    location = (File) context.getServletContext().getAttribute(ServletContext.TEMPDIR);
} else {
    location = new File(locationStr);
    if (!location.isAbsolute()) {
        location = new File(TEMPDIR, locationStr).getAbsoluteFile();  // 相对路径 → 相对 TEMPDIR
    }
}
```

`spring.servlet.multipart.location` 没配时，用的是容器的临时目录 `ServletContext.TEMPDIR`。这个目录在 Tomcat 里就是它的 `work` 目录；用内嵌 Tomcat 时，Boot 默认把它建在系统临时目录（`java.io.tmpdir`）下的一个 `tomcat.*` 目录里，所以你在 `/tmp/tomcat.xxxx/work/...` 下面看到上传文件，是正常的。配了相对路径，也是相对这个临时目录解析，不是相对进程的工作目录。目录不存在且允许创建时会自动建，建不出来则是 `IOException`。

文件名由 `DiskFileItem` 生成，格式是 `upload_<一串随机字符>_<自增序号>.tmp`。所以临时目录里那一排排 `upload_*.tmp`，就是它们。

### 3.3 拿到 MultipartFile 时，内容默认已经在磁盘上

这个问题在代码里看不出来，但它决定了你后面每一步的内存代价。

Spring 的 `StandardMultipartFile`（它是 `StandardMultipartHttpServletRequest` 里的内部类）只是 `Part` 的一层薄包装：`getSize()` 转给 `part.getSize()`，`getBytes()` 转给 `copyToByteArray(part.getInputStream())`，`transferTo(File)` 转给 `part.write(...)`。它不暴露"内容是在内存还是磁盘"，只给你两类入口：一类是流（`getInputStream()`），一类是"我要一份完整内容"（`getBytes()` / `transferTo()`）。

按 3.1 的结论，默认配置下你拿到它的时候，内容已经在磁盘上了。

### 3.4 把阈值调大：小的留内存，大的才落盘

| `file-size-threshold` | 1KB 的文件 | 100MB 的文件 | 堆内存代价 |
|---|---|---|---|
| `0`（Boot 默认） | 解析时就写进 `upload_*.tmp` | 同上，边读边写 | 几乎不占堆，但每个文件都留一个临时文件 |
| `2MB` | 留在堆内存 | 越过 2MB 后切到磁盘，已缓冲的 2MB 一起写进去 | 阈值以下的内容全在堆上，并发上传时是主要风险 |
| `10KB` | 留在内存 | 越过 10KB 后切到磁盘 | 堆上最多驻留阈值大小的内容 |

判断"越过"的那一行就是 `written + count > threshold`：阈值越小，落盘越早。另外两个上限 `max-file-size` / `max-request-size` 与落点无关，它们决定的是"读到多少字节时抛异常"，那是下一篇的主题。

### 3.5 transferTo 和 getBytes：两种完全不同的内存代价

先看那段决定代价的代码，它只有十几行：

```java
// org.apache.tomcat.util.http.fileupload.disk.DiskFileItem#write（节选）
public void write(final File file) throws Exception {
    if (isInMemory()) {
        try (OutputStream fout = Files.newOutputStream(file.toPath())) {
            fout.write(get());                   // 内存 → 目标文件：整份拷一次
        }
    } else {
        final File outputFile = getStoreLocation();
        size = outputFile.length();
        if (file.exists() && !file.delete()) { throw new FileUploadException(...); }
        if (!outputFile.renameTo(file)) {        // ← 同文件系统：改名即成，几乎零成本
            // 跨文件系统：renameTo 失败，退化成输入流到输出流的复制
        }
    }
}
```

于是两条路的代价完全不同。`transferTo(dest)` 在内容已落盘、且 `dest` 与临时目录在同一个文件系统时走 `renameTo`，几乎零成本；跨文件系统（比如临时目录在容器可写层、`dest` 挂在另一个卷上）就退化成整份拷贝。`getBytes()` 则是把整份内容读进堆内存，无论它原本在哪，这一次一定占堆——并发上传时，这是最容易被忽略的内存放大点。

同一个 `transferTo`，接收 `File` 和接收 `Path` 走的也不是同一条路：`File` 版本交给容器的 `Part#write`（同卷时就是上面这个 `renameTo`），`Path` 版本在 Spring 里就是一次流复制，临时文件不会因此消失。

### 3.6 谁创建、谁删除：临时文件的寿命是一次请求

创建发生在解析过程中，越过阈值时由 `DiskFileItem` 生成。删除则有三条独立的路径，分清哪条生效很重要——它决定了"我把文件转移到业务目录了，临时文件还在不在"。

第一条，解析失败时，真正读字节的那一层（Tomcat 打包进来的 `FileUploadBase#parseRequest`）会在自己的 `finally` 里清理：既然已经知道这次上传失败了，就把已经产出的临时文件逐个删掉。

第二条，请求结束时，Tomcat 回收请求对象时兜底。请求处理完，`Request` 会走到 `recycle`：

```java
// org.apache.catalina.connector.Request#recycle（每次 delete 都 try/catch，失败只记 warn 日志）
if (parts != null) {
    for (Part part : parts) { part.delete(); }
    parts = null;
}
```

第三条，Spring 在响应写完后的收尾里再删一次。`DispatcherServlet#doDispatch` 的 `finally` 会在这次请求是 multipart 时调用 `cleanupMultipart`，最终落到 `StandardServletMultipartResolver#cleanupMultipart`，逐个 `part.delete()`（基线 6.1.15 里删的是全部 part；6.1.14 及以前只删带文件名的 part，那是给 Resin 留的兼容写法）。

于是可以记住一句话：**临时文件的默认寿命是一次请求，不是"你 `transferTo` 之后就删"。**你把文件搬到业务目录，内容是被复制或改名走了，临时文件本身仍然活到请求结束（同卷 `renameTo` 的情况除外）。异常路径也有清理，真正会残留的是进程被强杀、或者外层没有正常收尾的情况。

想让磁盘上少出现临时文件，唯一的手段是在落盘之前就用足够大的 `file-size-threshold` 把内容留在内存（前提是内容真的小）；至于临时文件的删除时机，你控制不了。

到这里，`MultipartFile` 的内容在哪、怎么删、什么时候消失，都说清了。接下来把这条链串成一次完整的旅程，再回答开头那两个接口。

## 四、收口：一个上传请求的完整旅程

把这一篇的结论串成一条线。基线是 Boot 3.2.x（Spring Framework 6.1.15、Tomcat 10.1.33），默认配置下一次 `multipart/form-data` 上传走的是：

```text
客户端
  │  POST /upload  Content-Type: multipart/form-data; boundary=...（body 里混排文本字段与二进制文件）
  ▼
Tomcat（工作线程，来自线程池）
  │  CoyoteAdapter（把连接器收到的请求送进容器的处理管线）→ Filter 链 → DispatcherServlet
  ▼
DispatcherServlet#doDispatch
  ├─ checkMultipart(request)
  │    └─ StandardServletMultipartResolver#resolveMultipart
  │         └─ new StandardMultipartHttpServletRequest(request, false)
  │              └─ parseRequest → request.getParts()
  │                   └─ Tomcat Request#parseParts(true)   ← 真正读字节的地方
  │                        ├─ 取 servlet 上的 MultipartConfigElement（来自 spring.servlet.multipart.*）
  │                        ├─ 临时目录 = location ?: ServletContext.TEMPDIR
  │                        ├─ DiskFileItemFactory.setSizeThreshold(0) → 第一个字节就落盘
  │                        ├─ 边读 body 边切 part
  │                        └─ 每个文件 part → ApplicationPart（Tomcat 对文件项的 Part 包装）→ upload_*.tmp
  ├─ 找 handler、执行参数解析
  ├─ Controller 方法执行
  │    └─ transferTo / getBytes / 读 InputStream
  └─ finally → cleanupMultipart → StandardServletMultipartResolver#cleanupMultipart
       └─ part.delete()   临时文件在这里消失
```

回过头看开头那两个接口，答案并不在同一条链路上。

3MB 的 PDF 走 MVC：限制不来自你的代码，而来自 `spring.servlet.multipart.*` 映射出的 `MultipartConfigElement`，再被 Tomcat 在解析途中触发——`max-file-size` 默认 1MB，3MB 必然过不去。

200MB 的视频走的是另一条链路：WebFlux。那里没有 `Part`、没有 `MultipartConfigElement`，也就没有这个 1MB 上限；限额换到 `spring.webflux.multipart` 前缀下，内存阈值默认 256KB、磁盘默认不限，收字节靠的是流和背压，而不是容器的临时文件。同一份 `application.yml`，两个前缀管两套完全不同的机制——这才是"同项目、两种表现"的真相。这条链路的模型（`Flux`、背压、订阅）在[深入理解 Spring WebFlux 与 Reactor Netty](/2026/09/15/深入理解-Spring-WebFlux-与-Reactor-Netty/) 里讲过，这里不重复。

下一篇接着往下走：`max-file-size` 与 `max-request-size` 到底在第几个字节被触发、抛出的是哪个异常、为什么有的场景客户端看到的是连接被断开；以及当下最常用的做法——不落盘直传（把 part 的流直接交给对象存储、预签名 URL 直传）该怎么选、各自的边界在哪。


## 延伸阅读

- [（二）：限额、故障与不落盘直传](/2026/10/23/Spring-文件上传全链路深度解析（二）：限额、故障与不落盘直传/)
- [Spring 文件下载全方案深度解析](/2026/09/15/Spring-文件下载全方案深度解析/)——同一个问题的反方向：线程、拷贝与零拷贝
- [SpringBoot 请求链路（二）：从 DispatcherServlet 到方法调用](/2026/08/26/SpringBoot请求链路2：从-DispatcherServlet-到方法调用/)——`doDispatch` 与参数解析器的完整视角
- Tomcat 10.1 Connector 配置文档：<https://tomcat.apache.org/tomcat-10.1-doc/config/http.html>
