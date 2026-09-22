---
title: 深入理解 Spring WebFlux 与 Reactor Netty
date: 2026-09-15 17:48:04
tags:
  - spring
  - spring webFlux
  - netty
categories:
  - develop
---

## 引子：从 Tomcat 说起

Spring MVC 的请求路径非常直观：

浏览器发请求 → Tomcat 接收 → 线程池分配线程 → `DispatcherServlet` 找到 Controller → 方法执行 → 返回响应。

在 Tomcat 的世界里，一切都有明确的"角色"：

- **Connector**：负责 TCP 连接、HTTP 解析。
- **线程池**：一个请求分配一个线程。
- **Filter**：在请求到达 `DispatcherServlet` 之前、响应返回客户端之前，做预处理和后处理。
- **CoyoteAdapter**：把底层请求适配成 Servlet 请求。
- **DispatcherServlet**：找到对应的 Controller。
- **Controller**：执行业务逻辑，阻塞等待数据库或远程调用，然后返回。

这个模型可以概括为：**一个请求，一个线程，从头跑到尾。**

换成 Spring WebFlux + Reactor Netty 之后，这些角色似乎消失了。

没有 Tomcat，没有 Servlet，没有 `DispatcherServlet`。取而代之的是：

- `EventLoop`
- `Channel`
- `ChannelPipeline`
- `ChannelHandler`
- `Mono`、`Flux`
- `subscribe`

这篇文章始终拿 Tomcat 作为参照物。每进入一个新概念，先回答一个问题：

> 如果是 Tomcat，这里会是谁？现在换成了谁？

---

## 一、Tomcat 的请求地图

假设浏览器访问：

```http
GET /hello
```

在 Spring MVC + Tomcat 中，大致发生以下步骤：

1. 浏览器与 Tomcat 建立 TCP 连接。
2. Tomcat 的 Connector 读取字节，解析成 HTTP 请求。
3. Tomcat 从线程池拿一个线程。
4. 这个线程执行 Filter 链，再执行 `DispatcherServlet`。
5. `DispatcherServlet` 根据 `/hello` 找到 `HelloController.hello()`。
6. 方法执行，可能阻塞等待数据库、远程调用。
7. 返回结果，反向经过 Filter 链，写回响应。
8. 线程归还线程池。

关键点：

> **请求处理是"一个线程包到底"。**

线程在等待数据库或远程服务时，什么也做不了，只能阻塞。为了支持更多并发，只能加线程。但线程是昂贵的，上下文切换也有成本。

Netty 想解决的就是这个问题。它不是"加更多线程"，而是换一种思路：

> **不要让线程等待。让一个线程管理很多连接，谁有数据就处理谁。**

这就是 EventLoop 的起点。

---

## 二、Netty 与 Tomcat 的角色对照

下面这张表不是严格的一一对应，但能帮助快速定位。

| Tomcat 世界 | Netty / Reactor Netty / WebFlux 世界 |
| --- | --- |
| Tomcat Connector | Netty 的 `ChannelPipeline` + HTTP 编解码器 |
| Tomcat 线程池 | `EventLoopGroup` |
| 一个请求一个线程 | 一个 `EventLoop` 管理多个 `Channel` |
| Servlet Filter | WebFlux 的 `WebFilter` |
| CoyoteAdapter | Reactor Netty 的桥接 `ChannelHandler` + 适配器 |
| Servlet | Reactor Netty 的 `HttpHandler` / WebFlux 的 `HttpHandler` |
| DispatcherServlet | WebFlux 的 `DispatcherHandler` |
| ServletRequest / Response | `ServerHttpRequest` / `ServerHttpResponse` |
| 阻塞等待远程调用 | 注册回调，立即返回，EventLoop 继续循环 |

---

## 三、Netty 的核心：一个线程，一个 Selector，一个 while(true)

在 Tomcat 里，线程池是核心。在 Netty 里，核心是 `EventLoop`。

`EventLoop` 的本质可以概括为：

> **一个线程 + 一个 Selector + 一个死循环。**

它大概长这样：

```java
while (true) {
    // 阻塞等待 IO 事件
    select();

    // 处理所有就绪的 Channel
    for (Channel channel : readyChannels) {
        handle(channel);
    }
}
```

一个 `EventLoop` 可以管理多个 `Channel`。但一个 `Channel` 只能绑定一个 `EventLoop`。

这意味着：

- 同一个 Channel 的读、写、连接事件，永远由同一个 EventLoop 线程处理。
- 不需要为每个 Channel 加锁。
- 一个线程就能扛住大量连接。

`EventLoopGroup` 就是一组 `EventLoop`。通常分：

- boss 组：接受连接。
- worker 组：处理已建立连接的 IO。

每个 `Channel` 都有自己的 `ChannelPipeline`。

`ChannelPipeline` 是一条 Handler 链表。数据从头部进来，经过一个个 `ChannelHandler`，最后到达业务 Handler。

入站事件（比如读到数据）从 pipeline 头部向尾部传播。

出站事件（比如写数据）从尾部向头部传播。

用原生 Netty 写 HTTP 服务器时，代码大致如下：

```java
pipeline.addLast(new HttpServerCodec());
pipeline.addLast(new HttpObjectAggregator(65536));
pipeline.addLast(new SimpleChannelInboundHandler<FullHttpRequest>() {
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, FullHttpRequest req) {
        // 业务逻辑
    }
});
```

这里的 `SimpleChannelInboundHandler` 就是一个 `ChannelHandler`。

当 EventLoop 读到数据、解码成 `FullHttpRequest` 后，就会调用它的 `channelRead0` 方法。

**所有业务代码，都跑在某个 EventLoop 线程上。**

---

## 四、Reactor Netty：给 Netty 穿上 HTTP 外衣

原生 Netty 太底层。HTTP 编解码、请求聚合、Keep-Alive、响应写回、异常处理、连接关闭，都需要自己处理。

Reactor Netty 没有改变 Netty 的线程模型。它把 Netty 的 `ChannelHandler` 世界，翻译成 Reactor 的 `Mono/Flux` 世界。

它对外暴露一个接口：

```java
public interface HttpHandler {
    Mono<Void> handle(HttpServerRequest request, HttpServerResponse response);
}
```

注意这个接口。它返回的是 `Mono<Void>`，不是 `void`，也不是 `FullHttpResponse`。

这意味着：

> **处理请求变成了一个响应式发布者。订阅它，才会真正执行。**

### Reactor Netty 内部的桥接

创建一个 Reactor Netty HTTP 服务器：

```java
HttpServer.create()
    .port(8080)
    .handle((request, response) -> {
        // 返回 Mono<Void>
    })
    .bindNow();
```

Reactor Netty 会为每个连接构建 Netty Pipeline。这个 Pipeline 里通常有：

- `HttpServerCodec`：HTTP 编解码。
- `HttpServerExpectContinueHandler`：处理 `Expect: 100-continue`。
- `HttpServerKeepAliveHandler`：处理 Keep-Alive。
- 以及最关键的末端 Handler：`ChannelOperationsHandler`。

这个 `ChannelOperationsHandler` 就是 Reactor Netty 的桥。

它扮演的角色，类似于 Tomcat 里的 **CoyoteAdapter**：负责把底层网络数据，翻译成上层可以理解的请求对象。

它仍然是 Netty 的 `ChannelHandler`，由 EventLoop 线程调用。

概念上，它的 `channelRead` 里会做以下事情：

```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    if (msg instanceof HttpRequest) {
        HttpServerOperations ops = new HttpServerOperations(ctx, msg);

        HttpHandler handler = serverConfig.httpHandler();

        Mono<Void> result = handler.handle(ops, ops);

        result.subscribe(
            null,
            error -> ops.onError(error),
            () -> ops.onComplete()
        );
    }
}
```

这段伪代码揭示了几个关键事实：

1. Netty 只认识 `ChannelOperationsHandler`。
2. Reactor Netty 的 `HttpHandler` **不是** Netty 的 `ChannelHandler`。
3. `HttpHandler.handle(...)` 返回一个 `Mono<Void>`。
4. `ChannelOperationsHandler` 会订阅这个 `Mono`，从而启动响应式链。
5. 订阅发生在 EventLoop 线程上。

从 Netty 的视角看：

> 业务逻辑藏在 `ChannelOperationsHandler` 的 `channelRead` 方法体里。

从 Reactor Netty 的视角看：

> 业务逻辑是一个 `HttpHandler`，它返回 `Mono<Void>`。

那么，Spring WebFlux 在哪里？它就是那个 `HttpHandler`。

但一个完整的 Web 框架，不可能只是一个 lambda。它需要路由、参数绑定、异常处理、过滤器。这些内容如何塞进 `HttpHandler` 这一个方法？这是下一章要回答的问题。

---

## 五、Spring WebFlux 如何坐进 Reactor Netty

### 5.1 从一个插槽说起

Reactor Netty 给外界留了一个插槽，就是 `HttpHandler`：

```java
public interface HttpHandler {
    Mono<Void> handle(HttpServerRequest request, HttpServerResponse response);
}
```

一个函数，输入请求和响应，输出一个 `Mono<Void>`。Reactor Netty 拿到这个 `Mono` 后会自行订阅，启动整个响应式链。

Spring WebFlux 要做的，就是实现这个接口，然后在这个实现内部，撑起一整套 Web 框架——路由、参数绑定、过滤器、异常处理。

但实现这个接口的过程需要解决两个不同层面的问题。

### 5.2 问题一：请求类型不匹配

Reactor Netty 递过来的是 `HttpServerRequest` 和 `HttpServerResponse`。WebFlux 内部不能直接使用这两个类型。

原因在于，WebFlux 不只跑在 Reactor Netty 上。它还要支持 Tomcat、Jetty、Undertow。如果 WebFlux 的核心逻辑依赖 `HttpServerRequest`，那它就跑不到 Tomcat 上——Tomcat 根本没有这个类。

这就像 Servlet 规范定义了 `HttpServletRequest` / `HttpServletResponse`，让所有容器去适配它，而不是让 Servlet 代码去适配 Tomcat。

WebFlux 定义了自己的抽象：

- `ServerHttpRequest`
- `ServerHttpResponse`

于是第一层翻译不可避免：Reactor Netty 的请求/响应，必须被翻译成 WebFlux 的请求/响应。

这个翻译工作，由 `ReactorHttpHandlerAdapter` 完成。它实现的是 **Reactor Netty 的 `HttpHandler`**：

```java
public class ReactorHttpHandlerAdapter implements HttpHandler {

    private final HttpHandler webFluxHttpHandler;

    @Override
    public Mono<Void> handle(HttpServerRequest request, HttpServerResponse response) {
        ServerHttpRequest serverRequest = ...;   // 翻译
        ServerHttpResponse serverResponse = ...; // 翻译
        return webFluxHttpHandler.handle(serverRequest, serverResponse);
    }
}
```

到这里，请求类型的问题解决了。但新的问题随之而来。

### 5.3 问题二：`DispatcherHandler` 需要的不只是请求和响应

WebFlux 内部的路由核心叫 `DispatcherHandler`，地位等同于 Spring MVC 的 `DispatcherServlet`。

`DispatcherServlet` 拿到的不只是 `HttpServletRequest`，而是一个完整的运行环境——请求对象上挂着属性、会话、输入流、语言环境。这些能力在 Servlet 世界里，是容器偷偷挂在 request 对象上的。

WebFlux 的设计者没有把这些能力继续硬塞进 `ServerHttpRequest`。他们引入了第三个概念：

> **`ServerWebExchange`。**

它不是"请求"，也不是"响应"，而是**一次 Web 请求的完整上下文**。它装着：

- `ServerHttpRequest`
- `ServerHttpResponse`
- 请求级属性（类似 `request.setAttribute`）
- 会话
- 表单数据
- 其他与这次请求相关的状态

WebFlux 内部的 `DispatcherHandler`、过滤器、参数解析器，全都只认 `ServerWebExchange`。

`ServerHttpRequest` + `ServerHttpResponse` 是原料，`ServerWebExchange` 是装好盘的菜。

所以，从请求/响应到 `ServerWebExchange`，是必须的一步。

### 5.4 为什么这一步不能塞进 `ReactorHttpHandlerAdapter`

一个自然的想法是：既然 `ReactorHttpHandlerAdapter` 已经拿到了 `ServerHttpRequest` 和 `ServerHttpResponse`，顺手把 `ServerWebExchange` 也构造出来即可。

这个想法的问题在于，它混淆了两件本质上不同的事。

`ReactorHttpHandlerAdapter` 的定位是一个**运行时适配器**。它的唯一职责，是把 Reactor Netty 的请求/响应，翻译成 WebFlux 抽象的请求/响应。这是一件**容器特有**的工作。

而构造 `ServerWebExchange`、初始化会话、装配过滤器、准备编解码器——这些是**所有容器共享**的逻辑。

这个适配器未来会被复制多份：

- `ReactorHttpHandlerAdapter`（Reactor Netty）
- `TomcatHttpHandlerAdapter`（Tomcat）
- `JettyHttpHandlerAdapter`（Jetty）
- `UndertowHttpHandlerAdapter`（Undertow）

每一个适配器都只负责把各自容器的请求/响应翻译成 `ServerHttpRequest` / `ServerHttpResponse`。到这一步，它们的任务就完成了。

如果把 `ServerWebExchange` 的构造也塞进去，那每个适配器都要重复实现一遍相同的逻辑。更糟的是，适配器会从一个"翻译层"变成"半 Web 框架"，职责严重越界。

所以，构造 `ServerWebExchange` 这一步必须从容器适配层里抽出来，放在一个所有容器共享的地方。这就是 `HttpWebHandlerAdapter` 存在的理由。

### 5.5 三层结构

把这两件事分开之后，WebFlux 接入任意运行时的结构就清晰了：

```text
┌──────────────────────────────────────────────────────────┐
│  第一层：容器适配层                                        │
│  ReactorHttpHandlerAdapter / TomcatHttpHandlerAdapter ...  │
│  职责：把容器原生请求/响应，翻译成 ServerHttpRequest /     │
│        ServerHttpResponse                                 │
└──────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────┐
│  第二层：Web 框架入口层                                    │
│  HttpWebHandlerAdapter                                    │
│  职责：把请求/响应组装成 ServerWebExchange，               │
│        交给 WebHandler 处理                                │
└──────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────┐
│  第三层：Web 框架核心                                      │
│  FilteringWebHandler → DispatcherHandler → Controller     │
│  职责：过滤器、路由、参数绑定、返回值处理、异常处理         │
└──────────────────────────────────────────────────────────┘
```

对应到 Servlet 世界：

- 第一层 ≈ `CoyoteAdapter`：把 Coyote 请求翻译成 Servlet 请求。
- 第二层 ≈ Servlet 容器给 `DispatcherServlet` 准备的运行环境。
- 第三层 ≈ `DispatcherServlet` + Filter 链本身。

第一层随容器而变，第二、三层完全复用。这就是启动装配里出现两个 Adapter 的原因。

### 5.6 关于第三层里的 `FilteringWebHandler`

第三层里有一个此前未展开的类：`FilteringWebHandler`。

在 Spring MVC 中：

- 写一个类实现 `javax.servlet.Filter`。
- 在 `web.xml` 或 Spring Boot 配置里注册它。
- Tomcat 在调用 `DispatcherServlet` 之前，按顺序执行这些 Filter。
- Filter 可以拿到 `HttpServletRequest` / `HttpServletResponse`，做鉴权、日志、跨域、请求包装等。

在 WebFlux 中：

- 写一个类实现 `org.springframework.web.server.WebFilter`。
- Spring 把它收集起来，交给 `FilteringWebHandler`。
- `FilteringWebHandler` 把 `DispatcherHandler` 包在中间，请求进来先过 Filter 链，再进 `DispatcherHandler`；响应回来再反向过 Filter 链。
- Filter 拿到的是 `ServerWebExchange`。

一个最小的 `WebFilter`：

```java
@Component
public class LoggingWebFilter implements WebFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        System.out.println("before: " + exchange.getRequest().getPath());
        return chain.filter(exchange)
                .doOnSuccess(v -> System.out.println("after"));
    }
}
```

`chain.filter(exchange)` 的返回类型是 `Mono<Void>`。这和 Servlet Filter 的 `chain.doFilter(req, res)` 是同一个概念：**把请求交给链上的下一个环节。**

区别只在于：

- Servlet Filter 是阻塞式的，`chain.doFilter` 返回时，响应已经写完。
- WebFilter 是响应式的，`chain.filter` 返回一个 `Mono<Void>`，表示"接下来的处理"这个动作。订阅它，才会真正执行。

#### WebFilter 与 Filter、Interceptor 的对比

一个常见的疑问是：WebFilter 更像 Servlet Filter，还是更像 Spring MVC 的 `HandlerInterceptor`？下面把三者放在一起对比：

| 维度 | Servlet Filter | Spring MVC Interceptor | WebFlux WebFilter |
|---|---|---|---|
| 执行位置 | `DispatcherServlet` **之前** | `DispatcherServlet` **之后** | `DispatcherHandler` **之前** |
| 能拿到什么 | `ServletRequest` / `ServletResponse` | `HttpServletRequest` / `HttpServletResponse` + `HandlerMethod` | `ServerWebExchange` |
| 知道目标方法吗 | 不知道 | 知道 | 不知道 |
| 典型用途 | 编码、跨域、安全、请求包装 | 鉴权、日志、性能统计、方法级切面 | 鉴权、日志、跨域、请求属性初始化 |

从**执行位置**和**能拿到的信息**这两个维度看：

> **WebFilter 更像 Servlet Filter，而不是 HandlerInterceptor。**

它执行在 `DispatcherHandler` 之前，此时路由还没发生，它**根本不知道请求最终会被哪个 Controller 方法处理**。

而 `HandlerInterceptor` 执行在 `DispatcherServlet` 之后，此时 `HandlerMapping` 已经找到了目标方法。所以 Interceptor 可以基于 `HandlerMethod` 做方法级逻辑。

WebFlux 里没有 Interceptor 的严格等价物。如果想做类似"根据 Controller 方法做拦截"的事情，通常要用 `@ControllerAdvice` 或自定义的 `HandlerMethodArgumentResolver` / `HandlerResultHandler`。

请记住这条分界线：

> **Interceptor 知道目标方法，WebFilter 不知道。**

### 5.7 启动时的装配

把所有零件串起来：

```java
// 第三层：WebFlux 内部，能找到 @Controller
DispatcherHandler dispatcherHandler = ...;
WebHandler webHandler = new FilteringWebHandler(dispatcherHandler);

// 第二层：把请求/响应组装成 ServerWebExchange
HttpHandler webFluxHttpHandler = new HttpWebHandlerAdapter(webHandler);

// 第一层：适配成 Reactor Netty 的 HttpHandler
ReactorHttpHandlerAdapter adapter = new ReactorHttpHandlerAdapter(webFluxHttpHandler);

// 注册到 Reactor Netty
HttpServer.create()
    .port(8080)
    .handle(adapter)
    .bindNow();
```

层级关系：

```text
Reactor Netty 的 HttpHandler
    ↑
ReactorHttpHandlerAdapter        ← 第一层：容器适配（类似 CoyoteAdapter）
    ↑
WebFlux 的 HttpHandler (HttpWebHandlerAdapter)
                                 ← 第二层：构造 ServerWebExchange
    ↑
WebHandler (FilteringWebHandler) ← 第三层：过滤器链（类似 Servlet Filter 链）
    ↑
DispatcherHandler                ← 第三层：路由（类似 DispatcherServlet）
    ↑
Controller
```

回到最初的问题：Spring WebFlux 在 Reactor Netty 里是什么？

> **它是一个实现了 Reactor Netty `HttpHandler` 接口的对象。这个对象内部，封装了一整套 Web 框架，通过三层结构把自己接进了 Netty 的 Pipeline。**

从 Netty 的视角看，它只是 Pipeline 末端某个 ChannelHandler 里调用的一个函数。

从开发者的视角看，它是 `@Controller`、`@GetMapping`、`WebFilter`。

这两件事，同时成立。

---

## 六、同步请求：GET /hello 的完整旅行

浏览器访问：

```http
GET /hello
```

Controller 返回：

```java
@GetMapping("/hello")
public Mono<String> hello() {
    return Mono.just("hello");
}
```

完整链路如下：

1. 客户端连接，Netty 创建 `SocketChannel`，注册到某个 worker EventLoop。
2. 该 Channel 拥有自己的 Pipeline。
3. 这个 EventLoop 线程从 Channel 读到字节。
4. 字节经过 HTTP 编解码器，最终到达 `ChannelOperationsHandler`。
5. `ChannelOperationsHandler` 创建 `HttpServerOperations`，调用 Reactor Netty 的 `HttpHandler`。
6. `ReactorHttpHandlerAdapter` 适配请求/响应，调用 WebFlux 的 `HttpHandler`。
7. `HttpWebHandlerAdapter` 组装 `ServerWebExchange`，进入 `FilteringWebHandler`。
8. `FilteringWebHandler` 按顺序执行 `WebFilter` 链，最后交给 `DispatcherHandler`。
9. `DispatcherHandler` 根据 `/hello` 找到 `HelloController.hello()`，得到 `Mono.just("hello")`。
10. WebFlux 订阅这个 `Mono`，把 `"hello"` 写入 `ServerHttpResponse`。
11. 最终写回 Netty Channel，由 EventLoop 发送给客户端。

整个过程中，所有操作都在同一个 EventLoop 线程上完成。没有线程切换。

---

## 七、Publisher 与 Subscriber：理解异步请求的前提

在讲异步请求之前，必须先补上 Reactor 的两个核心概念：**Publisher** 和 **Subscriber**。

前面已经多次出现 `Mono`、`subscribe` 这些词，但一直没有正式解释它们。这里一次性讲清楚。

### 7.1 Mono 和 Flux 是 Publisher

在 Reactor 里，`Mono<T>` 和 `Flux<T>` 都实现了同一个接口：

```java
public interface Publisher<T> {
    void subscribe(Subscriber<? super T> s);
}
```

所以：

- **`Mono<String>` 是一个 Publisher**，它承诺：订阅之后，会发出 0 或 1 个 String。
- **`Flux<String>` 是一个 Publisher**，它承诺：订阅之后，会发出 0 到 N 个 String。

Publisher 本身不干活。它只是一份"剧本"，描述了"如果有人订阅我，我会发出什么"。

### 7.2 Subscriber 是订阅者

`Subscriber` 是另一个接口：

```java
public interface Subscriber<T> {
    void onSubscribe(Subscription s);
    void onNext(T t);
    void onError(Throwable t);
    void onComplete();
}
```

一个 Subscriber 就是一个"接收数据的人"。它定义了四件事：

- `onSubscribe`：订阅关系建立时被调用，拿到一个 `Subscription`（可以理解为"请求更多数据"的凭据）。
- `onNext`：每收到一个数据时被调用。
- `onError`：发生错误时被调用。
- `onComplete`：数据发完时被调用。

### 7.3 subscribe() 把两者连起来

当调用：

```java
publisher.subscribe(subscriber);
```

Publisher 就开始工作，并通过上面四个方法，向 Subscriber 发信号。

**这就是 Reactor 的全部模型：**

> Publisher 是"数据源"，Subscriber 是"数据接收者"，`subscribe` 把两者连接起来。连接之后，Publisher 通过 `onNext` / `onError` / `onComplete` 三个信号，向 Subscriber 推送数据。

### 7.4 每个操作符都会产生一对新的 Publisher/Subscriber

这才是最容易让人糊涂的地方。

当你写：

```java
webClient.get().uri(...).retrieve().bodyToMono(String.class)
```

这一串链式调用**不是**一个 Publisher。它是**一串** Publisher。

每个操作符（`get`、`uri`、`retrieve`、`bodyToMono`）都会产生一个新的 Publisher，包装前一个。

结构如下：

```text
bodyToMono(String.class)   ← 最下游 Publisher
      ↑ 包装
retrieve()                 ← 上游 Publisher
      ↑ 包装
uri(...)                   ← 更上游 Publisher
      ↑ 包装
get()                      ← 最上游 Publisher
```

当你对这个链条的最下游 Publisher 调用 `subscribe(subscriber)` 时，会发生什么？

**订阅信号会一层一层往上传播**：

1. 最下游 Publisher 收到订阅，它调用自己上游 Publisher 的 `subscribe`，并把自己包装成一个 Subscriber 传进去。
2. 上游 Publisher 收到订阅，又调用它的上游 Publisher 的 `subscribe`，再包装一个 Subscriber 传进去。
3. 一直传最上游。
4. 最上游开始干活（比如发起 HTTP 请求）。当数据产生时，通过 Subscriber 链，把数据一层一层往下游推。

所以，**每个 Publisher 都对应一个 Subscriber**。它们成对出现，串成一条链。

这就是为什么前面说：

> 订阅链，实际上是一串 Publisher 和一串 Subscriber，一一对应地连接起来。

### 7.5 回到异步请求

现在回头看异步请求里发生的事：

```java
@GetMapping("/hello")
public Mono<String> hello() {
    return webClient.get()
            .uri("http://other-service/api")
            .retrieve()
            .bodyToMono(String.class);
}
```

`hello()` 返回的，是这个链条的**最下游 Publisher**。它是一个 `Mono<String>`。

当 Reactor Netty 在 `ChannelOperationsHandler` 里对它调用 `subscribe()` 时：

1. 订阅信号沿着链条往上传播，一层一层建立 Publisher/Subscriber 配对。
2. 最上游的 Publisher 被激活，从连接池拿 ClientChannel，把 HTTP 请求写出去。
3. 同时在 ClientChannel 的 Pipeline 上，注册一个 `ResponseHandler`。
4. `ResponseHandler` 里保存的，正是这条订阅链上**最上游的那个 Subscriber**。

当响应到达时：

1. ClientChannel 的 EventLoop 唤醒。
2. `ResponseHandler.channelRead` 被调用。
3. 它调用保存的那个 Subscriber 的 `onNext(data)`。
4. 数据沿着 Subscriber 链，一层一层往下游传。
5. 每经过一层，对应的操作符执行自己的逻辑（比如 `bodyToMono` 会把 `ByteBuf` 转成 `String`）。
6. 最终到达链条末端——WebFlux 在 `subscribe()` 时挂上去的那个 Subscriber。

链条末端的那个 Subscriber 是什么？

它是在 `ChannelOperationsHandler` 调用 `subscribe()` 时创建的。它做的事很简单：把数据写回响应。

伪代码：

```java
Mono<Void> result = handler.handle(exchange);

result.subscribe(new Subscriber<String>() {
    @Override
    public void onSubscribe(Subscription s) { s.request(Long.MAX_VALUE); }

    @Override
    public void onNext(String data) {
        // 把 data 写回响应
        exchange.getResponse().writeWith(...);
    }

    @Override
    public void onError(Throwable t) {
        exchange.getResponse().setComplete();
    }

    @Override
    public void onComplete() {
        exchange.getResponse().setComplete();
    }
});
```

注意这里的 `exchange`。

它是**闭包捕获**的。换句话说：

> 在装配阶段，WebFlux 就把 `ServerWebExchange` 塞进了末端 Subscriber 的闭包里。这个 `ServerWebExchange` 内部持有 `ServerHttpResponse`，`ServerHttpResponse` 内部持有 ServerChannel。

当数据到达链条末端时，末端 Subscriber 用闭包里早就记着的 `exchange`，把数据写回去：

```java
exchange.getResponse().writeWith(...);
```

整个过程中，没有任何"找"的动作。

**一句话总结**：

> 数据流回 ServerChannel，靠的不是"找"，而是"记"。WebFlux 在装配阶段就把 `ServerWebExchange` 记进了链条末端 Subscriber 的闭包里。数据到达末端时，末端 Subscriber 自然知道该往哪里写。

---

## 八、异步请求的完整时间线

把前面所有内容串成一条时间线。每一步都标注"跑在哪个线程"。

```text
时刻   发生了什么                                             跑在哪个线程
──────────────────────────────────────────────────────────────────────────────
T0    浏览器请求到达 ServerChannel                           Server EventLoop
      │
T1    ServerChannel 的 Pipeline 触发                          Server EventLoop
      │
T2    ChannelOperationsHandler 调用 WebFlux 的 HttpHandler     Server EventLoop
      │
T3    WebFlux 层层处理，进入 Controller                        Server EventLoop
      │
T4    Controller 返回 Mono（Publisher 链，什么都没执行）        Server EventLoop
      │
T5    Reactor Netty 对 Mono 调用 subscribe()                  Server EventLoop
      │
T6    订阅信号沿 Publisher 链向上传播，逐层建立 Subscriber 配对  Server EventLoop
      │
T7    最上游 Publisher 拿 ClientChannel，写入请求，             Server EventLoop
      注册 ResponseHandler（内部保存最上游 Subscriber）
      │
T8    装配完成，EventLoop 回到 while(true)                    Server EventLoop
      │
      │   ... 线程继续处理其他 Channel，没有阻塞 ...
      │
T9    ClientChannel 上的响应到达，select() 唤醒               Client EventLoop
      │
T10   ClientChannel 的 Pipeline 触发                          Client EventLoop
      │
T11   ResponseHandler 调用最上游 Subscriber 的 onNext          Client EventLoop
      │
T12   onNext 信号沿 Subscriber 链向下游传播，                   Client EventLoop
      bodyToMono 等操作符在这时执行
      │
T13   到达末端 Subscriber，用闭包捕获的 exchange 写回           Client EventLoop
      │
T14   serverChannel.writeAndFlush(data)                       Client EventLoop
      │
T15   writeAndFlush 检查 eventLoop：                           —
      │   - 如果 Client EventLoop == Server EventLoop：直接写
      │   - 否则：把写任务排进 Server EventLoop 的队列
      │
T16   Server EventLoop 执行写任务，数据发送给浏览器             Server EventLoop
```

**关键点回顾**：

- T6 到 T8 之间，没有任何线程被阻塞。
- `.bodyToMono(String.class)` 是 T12 执行的，跑在 Client EventLoop 上。
- T13 写回响应，靠的是末端 Subscriber 闭包捕获的 `exchange`，不需要"找" ServerChannel。
- T14 的 `writeAndFlush` 会在 T15 决定是否要调度回 Server EventLoop。
- T16 数据真正发送给浏览器。

---

## 九、`writeAndFlush`：Netty 是怎么把数据真正发出去的

`ServerHttpResponse.writeWith(...)` 最终会调用：

```java
serverChannel.writeAndFlush(data);
```

`writeAndFlush` 是 Netty 发送数据的核心方法。有两点需要理解：

**第一点：write 和 flush 是两步**

- **write**：把数据放进一个叫 `ChannelOutboundBuffer` 的出站缓冲区。
- **flush**：把缓冲区里的数据真正通过 socket 发出去。

`writeAndFlush` 就是这两步的组合。

**第二点：它是线程安全的**

它的内部会做一次检查：

```java
if (eventLoop.inEventLoop()) {
    // 当前就是目标 EventLoop，直接走 Pipeline
    doWriteAndFlush(data);
} else {
    // 不是，包装成一个任务，丢进 EventLoop 的任务队列
    eventLoop.execute(() -> {
        doWriteAndFlush(data);
    });
}
```

也就是说：

- 如果当前线程就是 ServerChannel 的 EventLoop，直接写。
- 如果不是，把写任务排进 ServerChannel 的 EventLoop 队列。

**为什么必须这样？**

因为 EventLoop 是单线程模型，出站缓冲区不是线程安全的。如果允许多个线程同时写同一个 Channel，就必须加锁，性能会急剧下降。

所以 Netty 选择：**所有对 Channel 的操作，最终都归到它的 EventLoop 上执行。**

**"回到原 EventLoop"的真正含义**：不是数据"找"回去了，而是 `writeAndFlush` 内部帮你把任务排进了正确的队列。

---

## 十、为什么常常不需要线程切换

Server EventLoop 和 Client EventLoop 不一定是不同线程。

Reactor Netty 默认会使用全局的 `HttpResources`，其中包含 `LoopResources` 和 `ConnectionProvider`。

这意味着：

- 服务器和客户端可以共享同一套 EventLoopGroup。
- 它们可以共享连接池。
- 它们可以减少线程数量，降低上下文切换。

Reactor Netty 还提供了共址机制。在某些集成场景中，当在 Server EventLoop 线程里调用 WebClient 时，WebClient 创建的 ClientChannel 可以"就地"注册到当前 EventLoop。

于是：

- T9 的响应到达，Server EventLoop 直接 `select()` 到。
- T10 到 T14 全都在 Server EventLoop 上。
- T15 检查发现当前已经是目标 EventLoop，直接写。
- 没有任何线程切换。

但不要把这句话绝对化。更准确的说法是：

> Reactor Netty 默认共享全局资源，并提供共址机制，让客户端连接尽量复用当前 EventLoop。具体是否同线程，取决于配置、连接建立时机和 EventLoop 的选择策略。

不共享也没关系。T15 的 `writeAndFlush` 会帮助调度到正确的 EventLoop。共享只是优化。

---

## 结语：从 Tomcat 到 Netty，思维需要一次跳跃

回到最开始。

在 Tomcat 里：

> 一个请求，一个线程，阻塞等待，返回响应。

在 Netty 里：

> 一个 EventLoop 线程，管理很多 Channel，谁有数据就处理谁。
>
> 链式调用不是"顺序执行"，而是"构建 Publisher 链"。订阅才让 Publisher 链变成行动。
>
> 订阅的过程，是 Publisher 链和 Subscriber 链一一配对的过程。数据从上游 Publisher 流出，经过 Subscriber 链，一层一层往下游传。
>
> 数据流回 ServerChannel，靠的是"记"，不是"找"。WebFlux 在装配阶段就把 `ServerWebExchange` 记进了末端 Subscriber 的闭包里。数据到达末端，末端自然知道该往哪里写。
>
> 真正发送数据时，`writeAndFlush` 会保证落到正确的 EventLoop 上。

Spring WebFlux 并没有脱离 Netty 的线程模型。它只是通过三层结构把自己嵌入到 Reactor Netty 的 `HttpHandler` 中。而 Reactor Netty 又把 `HttpHandler` 嵌入到 Netty 的 `ChannelOperationsHandler` 中。

层层嵌套，但核心始终是：

> EventLoop 线程在驱动一切。

### 核心结论

1. **Spring WebFlux 不是 Netty 的 ChannelHandler。**
   它是被适配成 Reactor Netty 的 `HttpHandler`，在 Pipeline 末端的 `ChannelOperationsHandler` 中被调用。

2. **Reactor Netty 不是另一个网络框架。**
   它把 Netty 的 ChannelHandler 世界翻译成 Reactor 的 Mono/Flux 世界。

3. **WebFlux 接入任意运行时，靠的是三层结构。**
   容器适配层、框架入口层、框架核心层。第一层随容器而变，第二、三层可以复用。

4. **为什么需要 `HttpWebHandlerAdapter`？**
   因为"翻译请求类型"和"构造 Web 请求上下文"是两件不同的事。

5. **`WebFilter` 更像 Servlet Filter，而不是 HandlerInterceptor。**
   它执行在 `DispatcherHandler` 之前，拿不到 HandlerMethod。WebFlux 里没有 Interceptor 的严格等价物。

6. **Mono 和 Flux 是 Publisher。**
   Publisher 是"数据源"，Subscriber 是"数据接收者"，`subscribe` 把两者连接起来。Publisher 通过 `onNext` / `onError` / `onComplete` 向 Subscriber 推送信号。

7. **链式调用构建的是一串 Publisher。**
   每个操作符产生一个新的 Publisher，包装前一个。订阅时，订阅信号沿 Publisher 链向上传播，逐层建立 Subscriber 配对。

8. **链式调用不是"顺序执行"，而是"构建蓝图"。**
   `webClient.get().uri(...).retrieve().bodyToMono(...)` 在 Controller 返回时，一行都没执行。它只构建了一条 Publisher 链。

9. **subscribe() 是分界线。**
   它启动装配：订阅信号向上传播，最上游 Publisher 拿 ClientChannel、写入请求、注册 ResponseHandler。装配过程跑在 Server EventLoop 上。

10. **装配完成后，没有线程被阻塞。**
    Server EventLoop 回到循环，继续处理其他 Channel。

11. **响应到达时，Client EventLoop 唤醒。**
    `ResponseHandler` 调用最上游 Subscriber 的 `onNext`，数据沿 Subscriber 链向下游流动，`.bodyToMono` 等操作符在这时执行。

12. **写回响应，靠的是"记"，不是"找"。**
    WebFlux 在装配阶段就把 `ServerWebExchange`（内含 `ServerChannel` 引用）记进了末端 Subscriber 的闭包。数据到达末端，末端用它写回响应。

13. **`writeAndFlush` 负责把数据交给正确的 EventLoop。**
    它内部检查当前线程，不是目标 EventLoop 就把任务排进队列。底层是 `eventLoop().execute(task)`。

14. **Reactor Netty 默认共享全局资源，并支持共址机制。**
    这能让客户端连接尽量复用当前 EventLoop，减少线程切换。但具体是否同线程，取决于配置和场景。

下次写下：

```java
webClient.get()
        .uri("http://other-service/api")
        .retrieve()
        .bodyToMono(String.class);
```

在这行代码背后：

- 这段链式调用本身什么都没做，只是在构建一条 Publisher 链；
- 订阅让它活过来，Publisher 链和 Subscriber 链一一配对；
- 装配完成后，线程回到循环，等待响应；
- 响应到达时，Client EventLoop 唤醒，数据沿 Subscriber 链向下游流动；
- 数据到达末端时，末端 Subscriber 用闭包里早就记下的 `ServerWebExchange` 写回 ServerChannel；
- 最后 `writeAndFlush` 保证数据落到正确的 EventLoop 上。

没有魔法。只有 EventLoop 的循环、Publisher/Subscriber 的配对、闭包的捕获、`writeAndFlush` 的线程调度，以及装配（Assembly）和执行（Execution）两个阶段的分野。