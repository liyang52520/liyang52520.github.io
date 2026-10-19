---
title: SpringBoot 的一次完整请求发生了什么？
date: 2026-08-26 13:51:15
tags:
  - spring
  - tomcat
categories:
  - develop
---

## 引子：断点为什么没有停

线上报警，一批 `400`。

前端很确定参数传了，抓包一看，确实传了。后端日志翻来覆去只有一行：

```text
Resolved [org.springframework.web.bind.MissingServletRequestParameterException:
Required request parameter 'userId' for method parameter type String is not present]
```

你打开 IDEA，在 Controller 方法上打了个断点，刷新页面——断点没停。于是怀疑对象开始扩散：Nginx 配错了？网关把参数吃了？Tomcat 根本没收到请求？

先停一下。**那行日志本身，已经把答案写在脸上了。**

`Resolved [...]` 这个格式来自 Spring MVC 的 `DefaultHandlerExceptionResolver`，它是在 `DispatcherServlet.processHandlerException()` 里被调用的。换句话说：请求不仅到了 Tomcat，还完整穿过了 Filter 链、路由匹配，一路走到了**参数解析**阶段才失败。

断点没停，也不是因为请求没进来，而是因为 **`MissingServletRequestParameterException` 是参数解析时抛出的，Controller 方法体一行都还没开始执行**。断点打在第一行代码上，自然永远等不到。

> 一个把问题定位到"参数解析阶段"的日志，被当成了"请求没进来"的证据。方向一错，剩下的时间就全花在排查 Nginx 和网关上。

真正的原因往往朴素得让人难受：参数确实传了，但它躺在 **JSON 请求体**里，而方法签名用的是 `@RequestParam`——`@RequestParam` 只会去 query string 和表单里找 `userId`，请求体里的 JSON 它看都不看。

这个坑之所以常见，是因为它踩在了整条链路最不直观的一环上：**"参数"不是从请求里"读"出来的一个东西，而是由某个解析器、按某个注解、从某个特定位置取出来的。** 注解换了，取值位置就换了，报的错也完全不同。

> 这篇文章要回答的问题：
>
> - 一次请求从网卡到 Controller，中间换了多少个线程？分别是谁？
> - Tomcat 的 Acceptor、Poller、Worker 各管什么？为什么 Poller 只有一个？
> - Spring Boot 是什么时候把 Tomcat、`DispatcherServlet`、`HandlerMapping` 装好的？
> - Filter 和 Interceptor 都在"拦截"，为什么一个拿不到 Handler，另一个能拿到？
> - 为什么参数解析失败时，Controller 方法体一行都不会执行？
> - `@RequestParam` 和 `@RequestBody` 取的到底是不是同一个"参数"？
> - 异步请求为什么会让同一个请求**两次**进入 `DispatcherServlet`？
> - 400 / 404 / 405 / 406 / 415 / 500 分别指向链路上的哪一层？

### 版本基线

文中源码以 **Spring Framework 6.1.8 + Spring Boot 3.3.x + Tomcat 10.1.x + Java 17** 为基准，全部对照本地 sources jar 逐行核对。与 Spring 5.3 / Tomcat 9 行为不同的地方会单独标出——**这条链路上有好几处"广为流传但已经过时"的说法，本文会逐个纠正。**

如果你还没读过 [Spring 接入点：跟着容器过一生](/2026/09/11/Spring-接入点：跟着容器过一生/)，建议先扫一眼：那里讲了容器启动期各类 `PostProcessor` 的执行位置，而本文第四节的"启动期装了什么"，正是那些机制的直接产物。

## 一、先画一张全链路地图

在钻进任何一段源码之前，先把整条路铺开。后面每一节，都是在这张图上补细节。

```text
① 浏览器
   fetch / XHR / axios
        │
② 操作系统协议栈（内核态）
   DNS → TCP 三次握手 → TLS → HTTP 报文封装
        │
③ 物理网络 / 中间设施
   网卡 → 交换机 → 路由器 → CDN → 负载均衡 → K8s Ingress / Nginx / 网关
        │
④ 服务器内核（内核态）
   网卡 DMA → 软中断 → 协议栈逐层解封装 → 按端口找到监听 Socket
   ┌─────────────────── 第一次跨界：内核 → 用户态 ───────────────────┐
        │
⑤ Tomcat 连接器（用户态）
   Acceptor 线程 ── accept() ──▶ Poller 线程 ── 可读 ──▶ Worker 线程池
                                                          http-nio-8080-exec-N
   Http11Processor 解析 HTTP 报文
   CoyoteAdapter：Tomcat 对象 → Servlet 规范对象
   ┌────────────── 第二次跨界：连接器 → Servlet 容器 ──────────────┐
        │
⑥ Servlet 容器层
   ApplicationFilterChain（Filter 正序进入）
        │
⑦ DispatcherServlet.doDispatch()
   HandlerMapping → HandlerAdapter
   HandlerMethodArgumentResolver → Converter / HttpMessageConverter
   HandlerInterceptor.preHandle
   ┌────────────── 第三次跨界：Servlet 容器 → Spring MVC ──────────┐
        │
⑧ Controller 方法体
        │
⑨ 返回值处理
   HandlerMethodReturnValueHandler → HttpMessageConverter
        │
⑩ 异常出口（如果中间任何一步抛了异常）
   HandlerExceptionResolver
        │
⑪ 响应写回
   DispatcherServlet → Filter 倒序退出 → Tomcat OutputBuffer → Socket
        │
⑫ 网络原路返回
```

### 1.1 全文的主线：三次跨界

这张图里藏着一条主线，比"一层一层往下走"更有解释力：**请求每翻过一道边界，交出去的就不只是数据，还有执行它的线程。**

| 边界 | 从 | 到 | 谁在执行 |
|:---|:---|:---|:---|
| 第一次 | 内核协议栈 | Tomcat 用户态 | 内核软中断 → Acceptor 线程 |
| 第二次 | Tomcat 连接器 | Servlet 容器 | Poller 线程 → **Worker 线程** |
| 第三次 | Servlet 容器 | Spring MVC | 同一个 Worker 线程（同步场景） |

前两次跨界，线程都换了人。第三次没有——**同步请求下，从 Filter 到 Controller 到响应写回，全都跑在同一个 Worker 线程上**。这一点极其重要，因为它同时解释了两件事：

- 为什么 Controller 里能随手用 `ThreadLocal`（比如 `RequestContextHolder`、MDC 日志上下文）——因为整条链没换过线程；
- 为什么 Controller 里一旦阻塞（慢查询、外部 HTTP 调用），被卡住的是**整个 Worker 线程**，而不是某段代码。

> 异步请求会打破这条规律：Worker 线程会在 Controller 返回前就退出，业务逻辑跑到另一个线程上，之后再被"派回来"。这是第十二节的内容，也是 Spring MVC 里最容易被调试器骗到的地方。

### 1.2 每一层分别对应哪些错误

先给一张"反向索引"表，后面每一节讲完，都可以回到这张表上对一次。

| 现象 | 大概率所在层 | 本文对应节 |
|:---|:---|:---|
| `ERR_NAME_NOT_RESOLVED` | DNS | 二 |
| `Connection refused` | 端口未监听 / 防火墙 | 二 |
| 请求有连接但无响应、排队 | Acceptor 队列满 / Worker 打满 | 三 |
| 401 / 403 且断点未进 | Filter（如 Spring Security 直接写回响应） | 五 |
| 404 | `HandlerMapping` | 七 |
| 405 | `HandlerMapping`（路径匹配了但 HTTP 方法不对） | 七 |
| 400 | 参数解析 | 八 |
| 406 | 内容协商 / 返回值处理 | 十一 |
| 415 | `HttpMessageConverter` | 八 |
| 500 | Controller 内部 / 返回值序列化 | 十、十一 |
| 响应头正确但body 空 | Filter 包装了响应却忘了写回 | 十三 |

## 二、内核边界：连接是怎么交到 Tomcat 手里的

### 2.1 三次握手完成之后，内核做了什么

TCP 连接不是在 Java 代码里"建立"的，它由内核完成。整个过程里有两个队列，它们是理解"服务在线但请求进不来"的关键：

- **半连接队列（SYN queue）**：收到 `SYN`、回了 `SYN+ACK`，但还没收到客户端 `ACK` 的连接。队列满则丢弃新 `SYN`，客户端会重试——表现为**连接慢**。
- **全连接队列（Accept queue）**：三次握手已完成，等待应用 `accept()` 取走的连接。队列满时，内核默认**丢弃 `ACK`**（`tcp_abort_on_overflow=0`），客户端会重传——表现为**连接慢或偶发失败**。

全连接队列的长度，由应用侧 `listen()` 时传入的 backlog 决定，而这个值在 Tomcat 里叫 `acceptCount`：

```java
// AbstractEndpoint.java（Tomcat 10.1.24）
private int acceptCount = 100;
```

这是第二个参数容易被忽略的原因：**`acceptCount` 不是"Tomcat 能接受多少连接"，而是"内核愿意替 Tomcat 排多长的队"。** 队列满之后的连接，Tomcat 根本感知不到——它连日志都不会有。

> 所以遇到"服务正常、CPU 不高、但客户端偶发连接失败"，先看两处：`ss -lnt` 输出里 `Send-Q` 列的积压量，以及 Tomcat 的 `acceptCount`。查 Tomcat 的日志是查不到的，因为它压根没收到。

### 2.2 请求到底有没有到你的服务

这一步的结论很朴素，但值得单独强调，因为它是所有排查的第一个分岔口：

**在确认请求是否到达服务之前，不要打开 IDE。**

```bash
# 1. 绕过所有中间设施，直连 Pod IP + 端口
curl -v http://10.244.1.37:8080/actuator/health

# 2. 看 Nginx 是否真的转发了这个请求
tail -f /var/log/nginx/access.log | grep '/api/order'

# 3. 看谁在监听、积压多少
ss -lntp | grep 8080
```

三条命令对应三个不同的结论：

| 结果 | 结论 |
|:---|:---|
| `curl` 直连也不通 | 问题在服务本身或网络策略，与 Nginx、网关无关 |
| `curl` 直连通、经 Nginx 不通 | 问题在中间设施（转发规则、上游配置） |
| Nginx access log 里没有这条请求 | 请求在更外层就被拦下了（CDN、WAF、Ingress） |

等到请求确认能到达进程，再进入下一节——因为它接下来要面对的，是 Tomcat 内部的三拨线程。

## 三、Tomcat 边界：三拨线程，各管一段

### 3.1 Acceptor、Poller、Worker 的真实分工

请求进入进程后，会被三拨线程依次接手。先把它们摆清楚，因为**很多"服务没死但请求进不来"的问题，答案就在"哪一拨线程满了"**。

```text
                    ┌──────────────┐
   TCP 连接到达 ───▶ │  Acceptor 线程 │  accept() 取出连接
                    └──────┬───────┘
                           │  包装成 NioSocketWrapper，注册到 Poller
                           ▼
                    ┌──────────────┐
                    │  Poller 线程  │  Selector 轮询，等数据可读
                    └──────┬───────┘
                           │  可读 → 提交 SocketProcessor
                           ▼
                    ┌──────────────┐
                    │  Worker 线程池 │  解析 HTTP、跑业务、写响应
                    │  exec-1..200 │
                    └──────────────┘
```

三者的真实创建位置如下。

**Acceptor：全进程只有一个。** 它甚至不是"线程池"，就是一根线程：

```java
// AbstractEndpoint.java:1379-1385（Tomcat 10.1.24）
protected void startAcceptorThread() {
    acceptor = new Acceptor<>(this);
    String threadName = getName() + "-Acceptor";
    acceptor.setThreadName(threadName);
    Thread t = new Thread(acceptor, threadName);
    t.setPriority(getAcceptorThreadPriority());
    t.setDaemon(getDaemon());
    t.start();
}
```

**Poller：也只有一个。** 这个结论和很多资料相反，值得单独强调：

```java
// NioEndpoint.java:277-282（Tomcat 10.1.24）
// Start poller thread
poller = new Poller();
Thread pollerThread = new Thread(poller, getName() + "-Poller");
pollerThread.setPriority(threadPriority);
pollerThread.setDaemon(true);
pollerThread.start();
```

> 网上流传的"Poller 线程数通常与 CPU 核心数相关"，在 Tomcat 10.1 里是**错的**。`pollerThreadCount` 和 `acceptorThreadCount` 这两个配置项**已经不存在了**——全源码树里搜这两个名字是 0 命中。
>
> 这个变化有明确的三段演进，可以直接在源码里看到：
>
> | 版本 | `acceptorThreadCount` | `pollerThreadCount` |
> |:---|:---|:---|
> | Tomcat 9.0.x | 存在但已无意义：字段注释写 `Unused`，"hard-coded to 1 and is no longer configurable. It will be removed in Tomcat 10.1" | `setPollerThreadCount` 的 javadoc 直接写 `@param pollerThreadCount Unused`，是个空实现 |
> | Tomcat 10.1.x | **字段与 setter/getter 全部删除** | **字段与 setter/getter 全部删除** |
>
> 换句话说，"Poller 是一个单线程"不是默认值问题，而是**现代 Tomcat 的结构性设计**——一个 `Selector` 单线程轮询成千上万个连接，正是 NIO 的意义所在;真正会形成并发瓶颈的是后面的 Worker 池。

**Worker：唯一一个真正的线程池。** 创建时用的是 Tomcat 自己的 `TaskQueue`：

```java
// AbstractEndpoint.java:1068-1078（Tomcat 10.1.24）
public void createExecutor() {
    internalExecutor = true;
    if (getUseVirtualThreads()) {
        executor = new VirtualThreadExecutor(getName() + "-virt-");
    } else {
        TaskQueue taskqueue = new TaskQueue();
        TaskThreadFactory tf = new TaskThreadFactory(getName() + "-exec-", daemon, getThreadPriority());
        executor = new ThreadPoolExecutor(getMinSpareThreads(), getMaxThreads(), getThreadsMaxIdleTime(),
                TimeUnit.MILLISECONDS, taskqueue, tf);
        taskqueue.setParent((ThreadPoolExecutor) executor);
    }
}
```

### 3.2 关键参数的真实默认值

这些数字都能在 `AbstractEndpoint` 里逐个查到：

| 参数 | 默认值 | 含义 | 出处 |
|:---|:---|:---|:---|
| `maxThreads` | **200** | Worker 线程池上限 | `AbstractEndpoint.java:794` |
| `minSpareThreads` | **10** | 核心线程数（不会回收直到空闲超时） | `AbstractEndpoint.java:767` |
| `acceptCount` | **100** | 内核全连接队列长度（backlog） | `AbstractEndpoint.java:693` |
| `maxConnections` | **8192** | 同时打开的**连接数**上限（不是并发请求数） | `AbstractEndpoint.java:548` |
| `connectionTimeout` | **60000 ms** | 读到请求行/头的超时 | `AbstractHttp11Protocol.java:64` |
| `maxKeepAliveRequests` | **100** | 单连接最多复用多少次 | `AbstractEndpoint.java:859` |
| `threadsMaxIdleTime` | **60000 ms** | 线程空闲回收时间 | `AbstractEndpoint.java:819` |

**`connectionTimeout` 这个值本身就是一个"不读源码就会答错"的例子。** 如果你去翻 `SocketProperties`，会看到：

```java
// SocketProperties.java:148-151
/** SO_TIMEOUT option. default is 20000. */
protected Integer soTimeout = Integer.valueOf(20000);
```

很容易就此得出"默认 20 秒"的结论——**但那是 HTTP/1.1 协议用不到的默认值**，它在构造时就被覆盖了：

```java
// AbstractHttp11Protocol.java:62-65
public AbstractHttp11Protocol(AbstractEndpoint<S,?> endpoint) {
    super(endpoint);
    setConnectionTimeout(Constants.DEFAULT_CONNECTION_TIMEOUT);
}
```

```java
// Constants.java:28
public static final int DEFAULT_CONNECTION_TIMEOUT = 60000;
```

所以对 HTTP/1.1 连接器来说，**默认是 60000 ms**。这类"字段初值 ≠ 生效值"的情况，正是本文坚持逐行对照源码、而不是背结论的原因。

另外，**`maxConnections` 和 `maxThreads` 管的是两件事**，别混：`maxConnections` 通过一个 `LimitLatch` 限制**同时打开的连接数**（在 `accept` 之前就先卡住），`maxThreads` 限制的是**同时干活的线程数**。一个连接可以长期空闲地挂着，不占 Worker 线程。

**`maxThreads = 200` 是真正的硬墙，而它比看上去更硬。** 原因在 `TaskQueue.offer()`——它不是普通的队列：

```java
// TaskQueue.java
public boolean offer(Runnable o) {
    //we can't do any checks
    if (parent==null) {
        return super.offer(o);
    }
    //we are maxed out on threads, simply queue the object
    if (parent.getPoolSizeNoLock() == parent.getMaximumPoolSize()) {
        return super.offer(o);
    }
    //we have idle threads, just add it to the queue
    if (parent.getSubmittedCount() <= parent.getPoolSizeNoLock()) {
        return super.offer(o);
    }
    //if we have less threads than maximum force creation of a new thread
    if (parent.getPoolSizeNoLock() < parent.getMaximumPoolSize()) {
        return false;
    }
    //if we reached here, we need to add it to the queue
    return super.offer(o);
}
```

一个普通的 `ThreadPoolExecutor` + 无界队列，会把任务先堆进队列、只开核心线程数——那才是"明明配置了 200 个线程却只用 10 个"的经典事故。Tomcat 的 `TaskQueue` 特意反着来：**只要池子没到 `maxThreads`，`offer` 就返回 `false` 逼线程池新建线程**。所以 200 这个数字会被真正用满，然后队列才开始堆积。

> 于是"Worker 打满"的表现很有欺骗性：CPU 可能很低（线程都在等数据库、等外部 API），接口却大面积超时，而且**日志会显现出一条清晰的分界线**——某几个时刻之后所有请求的耗时一起抬升。`jstack` 里数一下 `http-nio-8080-exec-*` 的数量，就能立刻确认。

### 3.3 线程名是怎么拼出来的

上面三处代码里的 `getName()` 最终来自协议前缀加端口：

```java
// Http11NioProtocol.java:71-74
protected String getNamePrefix() {
    if (isSSLEnabled()) {
        return "https-" + getSslImplementationShortName() + "-nio";
    } else {
        return "http-nio";
    }
}
```

`AbstractProtocol.getNameInternal()` 把它和端口拼起来，于是得到 `http-nio-8080`，三拨线程的名字分别是：

| 线程 | 名字 | 数量 |
|:---|:---|:---|
| Acceptor | `http-nio-8080-Acceptor` | 1 |
| Poller | `http-nio-8080-Poller` | 1 |
| Worker | `http-nio-8080-exec-1` … `-200` | 至多 200 |

**这三个名字是最便宜的排查工具。** 线上抓一份 `jstack`，或者看某条慢日志的线程名，就能直接判断卡在哪一拨：

| 线程名出现在哪 | 说明 |
|:---|:---|
| `-Acceptor` 里阻塞 | 几乎不会发生；若出现，看 `accept` 相关系统调用 |
| `-Poller` 里阻塞 | 端口/连接层异常，业务代码根本还没被碰到 |
| `-exec-*` 里大量堆积且状态是 `WAITING`/`TIMED_WAITING` | 业务阻塞（慢 SQL、外部调用），问题在代码里 |
| `-virt-*` | 开启了虚拟线程（`spring.threads.virtual.enabled=true`，Boot 3.2+） |

> 顺带一提，Java 21 之后 `createExecutor()` 里多了一个分支：`getUseVirtualThreads()` 为真时换成 `VirtualThreadExecutor`，线程名前缀变成 `-virt-`。此时"200 个线程"这个上限不再适用，但**"阻塞在数据库上"的问题依然存在**——虚拟线程解决的是线程开销，不是下游变慢。

### 3.4 一次读取：从可读事件到 HTTP 报文

连接已经建立、数据已经到达内核，接下来的流转是这样的（每一环都是真实方法名，可以按类名直接下断点）：

```text
Poller.run()                                 NioEndpoint.java:741
  └─ Selector 返回就绪的 SelectionKey
      └─ Poller.processKey(sk, socketWrapper)          NioEndpoint.java:801
          └─ AbstractEndpoint.processSocket(...)        AbstractEndpoint.java:1229
              └─ NioEndpoint.createSocketProcessor(...) NioEndpoint.java:549
                  └─ SocketProcessorBase.run()
                      └─ AbstractProtocol.ConnectionHandler.process(...)  AbstractProtocol.java:803
                          └─ AbstractProcessorLight.process(...)          AbstractProcessorLight.java:40
                              └─ Http11Processor.service(socketWrapper)   Http11Processor.java:243
                                  └─ 解析 HTTP 请求行 / 请求头 / 请求体
```

注意 `Poller.processKey()` 这一层做的事很轻：它只判断事件类型（`OPEN_READ` / `OPEN_WRITE` / `ERROR`），然后把 socket 丢给 Worker 池。**真正的 HTTP 解析发生在 Worker 线程上**，也就是 `Http11Processor.service()` 里面。这一点决定了：解析 HTTP 报文本身也消耗 Worker 线程，超大请求体（比如几百 MB 的 JSON）会直接占住一个 Worker。

> **这里要纠正一个流传很广的说法："Acceptor 收到连接后把 socket 交给业务线程池"。** 不是的。
>
> `Acceptor` 的职责只有两个：`accept()` 拿到连接，然后 `setSocketOptions()` 把它注册到 Poller。它**从不调用 `processSocket()`**。真正把 `SocketProcessor` 提交到线程池的是 `Poller`：
>
> ```java
> // NioEndpoint.java:823（Poller 线程内）
> } else if (!processSocket(socketWrapper, SocketEvent.OPEN_READ, true)) {
> ```
>
> ```java
> // AbstractEndpoint.java:1244-1249
> Executor executor = getExecutor();
> if (dispatch && executor != null) {
>     executor.execute(sc);
> } else {
>     sc.run();
> }
> ```
>
> 所以三类线程的接力关系是 **Acceptor 收 → Poller 等并分发 → Worker 干活**。把"分发"归给 Acceptor，会让你在排查"连接建立了但业务线程没动静"时找错方向。
>
> 还有一个细节值得知道：Servlet 里那种**阻塞式读取**（`request.getInputStream().read()`）并不是靠"线程一直转"实现的。它走 `registerReadInterest()` 注册读事件后 `readLock.wait()` 让出线程，由 Poller 检测到数据可读时 `notify` 唤醒。**这解释了为什么阻塞 IO 的 Tomcat 依然能把连接数撑到远大于线程数。**

还有一个容易忽略的分支：`AbstractProcessorLight.process()` 里有一个 `dispatches` 迭代器。

```java
// AbstractProcessorLight.java:40-52
public SocketState process(SocketWrapperBase<?> socketWrapper, SocketEvent status) throws IOException {

    SocketState state = SocketState.CLOSED;
    Iterator<DispatchType> dispatches = null;
    do {
        if (dispatches != null) {
            DispatchType nextDispatch = dispatches.next();
            ...
            state = dispatch(nextDispatch.getSocketStatus());
            if (!dispatches.hasNext()) {
                state = checkForPipelinedData(state, socketWrapper);
```

**这个循环是异步请求的入口。** 同步请求走一遍就结束了；异步请求在 `dispatch()` 里返回一个"待处理的 dispatch 类型"，下一次被唤醒时带着它重新进入循环。第十二节会回到这里。

### 3.5 CoyoteAdapter：Tomcat 对象变成 Servlet 对象

解析完 HTTP 报文后，Tomcat 手里拿到的是自己的 `org.apache.coyote.Request` / `Response`——**它们不是 Servlet 规范的类型**。跨过这条边界的是 `CoyoteAdapter.service()`：

```java
// CoyoteAdapter.java:303-307
public void service(org.apache.coyote.Request req, org.apache.coyote.Response res) throws Exception {

    Request request = (Request) req.getNote(ADAPTER_NOTES);
    Response response = (Response) res.getNote(ADAPTER_NOTES);
```

两个对象通过 `getNote(ADAPTER_NOTES)` 从 Coyote 层"取回" Catalina 层的实现——`org.apache.catalina.connector.Request` / `Response`，它们才实现了 `HttpServletRequest` / `HttpServletResponse`。

**所以第二次跨界不是类型转换，而是"换了一个对象"。** 这也是为什么在 Tomcat 内部拿不到 Spring 的东西，而在 `CoyoteAdapter` 之后、Filter 之前，Servlet 规范该有的一切就都齐了。

至此，请求第一次拿到了 `HttpServletRequest`。它接下来要面对的，是启动期就已经摆好的 Servlet 容器。

## 四、启动期：这些组件是什么时候装好的

运行期的一切"自动"，都是启动期就写好的静态布局。这一节回答：**Tomcat、`DispatcherServlet`、`HandlerMapping` 分别是谁、在什么时候装进容器的。**

### 4.1 自动配置选了 Tomcat

引入 `spring-boot-starter-web` 后，classpath 上就有了 Tomcat 的类。Spring Boot 据此做出选择：

```java
// ServletWebServerFactoryConfiguration.java:64-69
@ConditionalOnClass({ Servlet.class, Tomcat.class, UpgradeProtocol.class })
...
    @Bean
    TomcatServletWebServerFactory tomcatServletWebServerFactory(
```

`@ConditionalOnClass` 判断的是"类在不在 classpath 上"，而不是"配置里写了什么"。同文件里还有 Jetty 和 Undertow 两个分支，用的是同样手法——**谁在 classpath 上就选谁，这就是"换个 starter 就换了容器"的全部秘密**。

对应的入口自动配置是 `ServletWebServerFactoryAutoConfiguration`，它带 `@ConditionalOnWebApplication(Type.SERVLET)`。注意它此刻**只是注册了工厂 Bean**，Tomcat 还没有启动。

### 4.2 真正启动 Tomcat 的时机：`onRefresh()`

容器的 `refresh()` 过程中有一个专门给 Web 容器的扩展点：

```java
// ServletWebServerApplicationContext.java:159-167
protected void onRefresh() {
    super.onRefresh();
    try {
        createWebServer();
    }
    catch (Throwable ex) {
        throw new ApplicationContextException("Unable to start web server", ex);
    }
}
```

```java
// ServletWebServerApplicationContext.java:186-188
ServletWebServerFactory factory = getWebServerFactory();
...
this.webServer = factory.getWebServer(getSelfInitializer());
```

`TomcatServletWebServerFactory.getWebServer()` 里创建 Tomcat 与 Connector：

```java
// TomcatServletWebServerFactory.java:199-211
public WebServer getWebServer(ServletContextInitializer... initializers) {
    ...
    Tomcat tomcat = new Tomcat();
    File baseDir = (this.baseDirectory != null) ? this.baseDirectory : createTempDir("tomcat");
    tomcat.setBaseDir(baseDir.getAbsolutePath());
    ...
    Connector connector = new Connector(this.protocol);
    connector.setThrowOnFailure(true);
    tomcat.getService().addConnector(connector);
```

随后 `TomcatWebServer.initialize()` 调用 `this.tomcat.start()`，**端口在这个时刻真正开始监听，Acceptor 与 Poller 线程也是在这时启动的**。

> 这条时间线解释了一个常见困惑："接口还没准备好，端口就通了"或者反过来的"端口通了但请求 404"。因为 Tomcat 是在 `refresh()` 的中段启动的，而 `DispatcherServlet` 的初始化（`initStrategies()` 收集所有 `HandlerMapping`）紧随其后——**在 Servlet 初始化完成之前到达的请求，容器会先排队**。

### 4.3 `DispatcherServlet` 与 MVC 组件的注册

```java
// DispatcherServletAutoConfiguration.java:65-68
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
@AutoConfiguration(after = ServletWebServerFactoryAutoConfiguration.class)
...
@ConditionalOnClass(DispatcherServlet.class)
```

它做两件事，对应两个 Bean：

```java
// DispatcherServletAutoConfiguration.java:87-88
@Bean(name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
public DispatcherServlet dispatcherServlet(WebMvcProperties webMvcProperties) {
```

```java
// DispatcherServletAutoConfiguration.java:123-127
public DispatcherServletRegistrationBean dispatcherServletRegistration(DispatcherServlet dispatcherServlet,
        ...
    DispatcherServletRegistrationBean registration = new DispatcherServletRegistrationBean(dispatcherServlet,
            webMvcProperties.getServlet().getPath());
```

`webMvcProperties.getServlet().getPath()` 默认就是 `/`，所以 `DispatcherServlet` 被注册成一个映射到 `/` 的 Servlet。**这就是为什么 Spring MVC 能"接管一切请求"——它本身只是一个 Servlet 映射。**

`WebMvcAutoConfiguration` 接着创建 MVC 需要的组件；`DispatcherServlet` 自己在 `initStrategies()` 里按类型把它们收集起来，收集不到才回退到 `DispatcherServlet.properties` 里的默认实现。

最后还有一件容易被忽略的事：**`Filter` 的注册也发生在启动期**，通过 `FilterRegistrationBean` / `ServletContextInitializer` 完成。注册顺序决定了 Filter 链的顺序，而顺序错了，鉴权就可能跑在日志后面——这类问题在启动时就已经注定，运行期无从补救。

## 五、Servlet 边界：FilterChain 的真实结构

### 5.1 Filter 链不是 `for` 循环

一个直觉上的误解是：容器会遍历 Filter 数组，逐个调用。真实实现是**游标加递归**——所有 Filter 拿到的是**同一个 `ApplicationFilterChain` 实例**：

```java
// ApplicationFilterChain.java:144-165（Tomcat 10.1.24）
private void internalDoFilter(ServletRequest request, ServletResponse response)
        throws IOException, ServletException {

    // Call the next filter if there is one
    if (pos < n) {
        ApplicationFilterConfig filterConfig = filters[pos++];
        try {
            Filter filter = filterConfig.getFilter();
            ...
            } else {
                filter.doFilter(request, response, this);
            }
```

关键在最后一行：传下去的是 **`this`**，而不是"下一个 Filter"。Filter 内部调用 `chain.doFilter(...)` 时，会**重新进入 `internalDoFilter`**，而 `pos` 已经被自增过，于是轮到下一个 Filter。

这带来两个直接后果：

- **调用栈深度等于 Filter 数量。** 一个项目里挂 20 个 Filter，栈里就有 20 层 `internalDoFilter`。看堆栈时不要被这个"千层饼"吓到，它只是链式调用的正常形态。
- **`pos` 是共享状态，用完必须复位。** 复位发生在 `release()`：请求处理结束后，`pos` 和 `n` 归零，chain 对象被放回 `Request` 上复用。这也解释了为什么异步派发时 Filter 会"再跑一遍"（见第十二节）。

`pos >= n` 时才算走完链条，此时才调用真正的 Servlet：

```java
// ApplicationFilterChain.java:176-195
// We fell off the end of the chain -- call the servlet instance
...
        servlet.service(request, response);
```

我们的 `DispatcherServlet`，就是这里那个 `servlet`。

### 5.2 环绕式执行，与"直接写回"的中断

`chain.doFilter()` 是分界线：它之前是"请求进来"，之后是"响应出去"。

```java
public class LoggingFilter implements Filter {
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        long start = System.nanoTime();
        chain.doFilter(req, res);              // ← 分界线：请求继续向下
        long cost = System.nanoTime() - start;
        log.info("Request [{}] cost: {}ms", ((HttpServletRequest) req).getRequestURI(), cost);
    }
}
```

如果 Filter 在 `chain.doFilter()` 之前就写回了响应并 `return`，链条就断了——`DispatcherServlet` 不会被调用。Spring Security 认证失败直接返回 401、CORS 预检直接返回 204，走的都是这条路。

> 这也是一个重要的排查线索：**如果响应不是你写的代码产生的，先怀疑 Filter。** 断点没停、日志干净、状态码却已经返回，几乎都是 Filter 提前短路。

多个 Filter 的执行顺序是标准的栈语义：

```text
请求阶段：F1 → F2 → F3 → DispatcherServlet
响应阶段：        F3 → F2 → F1
```

### 5.3 Filter 还是 Interceptor

这是最常被问的问题，答案不在"哪个更强"，而在**它们活在两套不同的世界里**：

| 维度 | Filter | Interceptor |
|:---|:---|:---|
| 规范归属 | Servlet 规范（Tomcat 管理） | Spring MVC（Spring 管理） |
| 执行位置 | `DispatcherServlet` **之前** | `HandlerMapping` **之后**、Controller 之前 |
| 能否拿到目标 Handler | **不能**（此时还不知道要调哪个方法） | **能**（拿到 `HandlerMethod`） |
| 能否注入 Spring Bean | 能，但需要注册方式配合 | 能，天生就在容器里 |
| 单次请求执行次数 | 同步 1 次；异步派发可能 **2 次** | 同步 1 次；异步时 `postHandle`/`afterCompletion` 被跳过 |
| 典型用途 | 编码、跨域、安全、请求体缓存 | 鉴权、日志、业务级拦截 |

**"Filter 拿不到 Handler"是结构性的，不是能力问题。** Filter 在链条上更靠外，它执行的时候，Spring MVC 连 URL 都还没解析——`HandlerMethod` 这个对象根本不存在。反过来，如果需要在拦截器里拿到"即将执行的是哪个方法、上面有哪些注解"，那就必须用 Interceptor。

## 六、DispatcherServlet：把 doDispatch 完整读一遍

请求穿过滤镜链后，落到 `DispatcherServlet`。它是 Spring MVC 的前端控制器，继承自 `HttpServlet`，核心方法是 `doDispatch()`。

这个方法是整篇文章的枢纽——**前面所有铺垫是为了走到这里，后面所有细节都是从这里分出去的**。所以它值得逐行读。

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

这一节回到引子里的那个 400。

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

很多资料（包括本文的初版）说"自定义解析器排在所有内置解析器之后，所以会被内置的抢走"。**这个说法不准确。**

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

如果这篇文章讲的是 WebFlux，第十三节会是空白的——因为 **Netty 没有 `OutputBuffer`，也没有 `DispatcherServlet`**。

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
- **`addArgumentResolvers` 里传入的组件要来自容器**，不要 `new`——否则它内部的 `@Autowired`、`@Transactional`、AOP 全部失效（见 8.11 节）。

## 十五、Debug 指南

### 15.1 错误码 → 哪一层

| 错误码 | 所在层 | 常见原因 | 本文对应节 |
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

回到引子。那行 `Resolved [MissingServletRequestParameterException]` 之所以把人骗到 Nginx 和网关上去，是因为它看起来像"请求出了问题"，而实际上它在说"**请求已经成功走到第八节了**"。

现在，文章开头那张问题清单可以逐条回答了：

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
> - [Spring 接入点：跟着容器过一生](/2026/09/11/Spring-接入点：跟着容器过一生/)——本文第四节讲的是"装好了什么"，那篇讲的是"容器在哪个阶段把它们装好"。
> - [一次下载失败排查：被 Filter 吞掉的异步响应流](/2026/10/14/一次下载失败排查：被 Filter 吞掉的异步响应流/)——第十三节那个坑的完整排查过程。
> - [Spring MVC 中那些"不走寻常路"的返回类型](/2026/09/14/Spring-MVC-中那些“不走寻常路”的返回类型/)——第十一节那张返回值处理器表的展开版。
> - [深入理解 Spring WebFlux 与 Reactor Netty](/2026/09/15/深入理解-Spring-WebFlux-与-Reactor-Netty/)——把本文的 Tomcat 模型和 EventLoop 模型对照着看。
> - [@Transactional 为什么会失效？从 AOP 代理链看 Spring 事务机制](/2026/10/16/Transactional-为什么会失效-从-AOP-代理链看-Spring-事务机制/)——同样用"一条链 + 逐环定位"的方式，讲容器内部的另一条链路。

