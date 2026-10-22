---
title: "SpringBoot 请求链路（一）：请求是怎么到 Tomcat 的"
date: 2026-08-26 13:51:16
tags:
  - spring
  - tomcat
categories:
  - develop
---

## 引子：端口是通的，请求却没进来

`nc -zv 10.0.1.7 8080` 通了，`curl` 却挂在那里，等满 30 秒超时。登录机器一看，`jstack` 里 `http-nio-8080-exec-*` 开了两百个线程，CPU 只有 12%，而应用日志里一行请求记录都没有。

这时候"服务挂了"和"服务好好的"两个结论都不对。**端口通只说明内核里有程序在 `accept()`，不说明请求被谁接住、更不说明它被处理了。** 从 socket 到 Controller，中间要跨两次边界，每一次都会换掉执行它的线程；搞不清这两次跨界，报错就只能靠猜。

> 本篇接[系列导读](/2026/08/26/SpringBoot的一次完整请求发生了什么？/)那张链路图，从第 ④ 步走到第 ⑥ 步（二~五节）：请求怎么从网卡进到 Tomcat 的 Worker 线程，`Filter` 链又是怎么把它送进 Spring 的。源码基准同导读——Spring Framework 6.1.8 / Spring Boot 3.3.x / Tomcat 10.1.x。

> 本篇要回答的问题：
>
> - 三次握手都成功了，请求为什么还是可能"没进来"？
> - Acceptor、Poller、Worker 三拨线程各管什么？为什么前两拨各只有一个？
> - `http-nio-8080-exec-3` 这个线程名是怎么拼出来的？
> - Tomcat、`DispatcherServlet`、`HandlerMapping` 是什么时候被装进容器的？
> - Filter 链为什么不是一个 `for` 循环？它凭什么能"环绕"执行？

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

**这个循环是异步请求的入口。** 同步请求走一遍就结束了；异步请求在 `dispatch()` 里返回一个"待处理的 dispatch 类型"，下一次被唤醒时带着它重新进入循环。第三篇的第十二节会回到这里。

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
- **`pos` 是共享状态，用完必须复位。** 复位发生在 `release()`：请求处理结束后，`pos` 和 `n` 归零，chain 对象被放回 `Request` 上复用。这也解释了为什么异步派发时 Filter 会"再跑一遍"（见第三篇第十二节）。

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

## 小结

这一篇走完了两次跨界：内核把连接交给 Tomcat（Acceptor → Poller → Worker 三拨线程），Tomcat 把对象交给 Servlet 容器（`CoyoteAdapter` 换对象、`FilterChain` 环绕执行）。线程归属、组件安装时机、Filter 链的栈式结构，这三件事是后面一切的前提。

请求此时已经拿到 `HttpServletRequest`，穿过整条 Filter 链，停在 `DispatcherServlet` 门口。**接下来发生的一切，都收拢在 `doDispatch()` 这一个方法里。**

> **下一篇**
>
> [SpringBoot 请求链路（二）：从 DispatcherServlet 到方法调用](/2026/08/26/SpringBoot请求链路2：从-DispatcherServlet-到方法调用/)——`doDispatch()` 逐行读完，再看路由表怎么建、27 个参数解析器按什么顺序认领参数、拦截器中断之后谁收到回调。

> **延伸阅读**
>
> - [一次下载失败排查：被 Filter 吞掉的异步响应流](/2026/10/14/一次下载失败排查：被 Filter 吞掉的异步响应流/)——Filter 把响应包起来却没写回的那类事故，这篇是一次完整复盘。

