---
title: 深入理解 Spring WebFlux 与 Reactor Netty：从 Tomcat 到 EventLoop 的思维跃迁
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

换成 Spring WebFlux + Reactor Netty 之后，这些角色似乎消失了。取而代之的是：

- `EventLoop`
- `Channel`
- `ChannelPipeline`
- `ChannelHandler`
- `Mono`、`Flux`
- `subscribe`

这篇文章始终拿 Tomcat 作为参照物。每进入一个新概念，先回答一个问题：

> 如果是 Tomcat，这里会是谁？现在换成了谁？

通过这种方式，新概念可以被挂到已有的知识树上，而不是孤立地堆砌。

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
    selector.select();

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

这就是同步、非阻塞的 WebFlux 请求。

但这个例子太简单了。它的最上游 Publisher 是 `Mono.just("hello")`——数据就在内存里，压根没有异步的成分。

现实中的请求，往往需要访问外部服务。这时候，整条链路的行为就完全不同了。

---

## 七、异步请求：从一行代码到一次完整的往返

### 7.1 场景：Controller 里调用了其他服务

假设 Controller 里用 WebClient 调用另一个服务：

```java
@GetMapping("/hello")
public Mono<String> hello() {
    return webClient.get()
            .uri("http://other-service/api")
            .retrieve()
            .bodyToMono(String.class);
}
```

先不急着深入代码。先在脑子里过一遍几个问题：

- 这段代码看起来是"顺序执行"的，但真的是这样吗？
- 它调用了 other-service，Server EventLoop 会被阻塞吗？
- other-service 的响应回来时，谁来处理？
- 处理完后，怎么把结果写回浏览器？

这四个问题，是理解整条异步链路的关键。

### 7.2 第一个问题：这段代码不是"顺序执行"的

很多人第一眼看到：

```java
return webClient.get()
        .uri("http://other-service/api")
        .retrieve()
        .bodyToMono(String.class);
```

会下意识地理解成三段顺序执行的代码：

1. `get().uri().retrieve()`：发起请求。
2. `bodyToMono(String.class)`：处理响应。
3. `return`：返回结果。

**这个理解是错的。**

在 Reactor 中，这一整段链式调用，在 Controller 方法返回时，**一行都没执行**。

它只是构建了一个 `Mono`——一个描述"将来要做什么"的对象。

在这个阶段：

- 没有连接 other-service。
- 没有发送 HTTP 请求。
- 没有注册任何回调。
- 没有分配任何线程。

它只是一份**蓝图**。

真正让蓝图变成行动的，是订阅。

### 7.3 subscribe 是分界线

Reactor Netty 在 `ChannelOperationsHandler` 里对 WebFlux 返回的 `Mono` 调用了 `subscribe()`。

这一行，就是"启动引擎"的动作。

但 `subscribe()` 到底做了什么？要回答这个问题，必须先理解响应式流模型。

### 7.4 前置知识：响应式流的五个角色和四个信号

Reactive Streams 规范定义了五个关键角色：

| 角色 | 通俗理解 | 代码中 |
|---|---|---|
| Publisher | 数据源，负责生产数据 | `Flux` / `Mono` |
| Subscriber | 消费者，负责接收数据 | `subscribe(...)` 里的 Lambda 被包装成它 |
| Subscription | 订阅凭证，连接生产者和消费者 | 包含 `request` 和 `cancel` |
| `request` | 消费者说：我要几个数据 | `subscription.request(n)` |
| `onNext` | 生产者说：给你一个数据 | `subscriber.onNext(data)` |

还有几个关键信号：

- `onSubscribe`：生产者把订阅凭证交给消费者。
- `onNext`：生产者发一个数据。
- `onComplete`：数据发完了。
- `onError`：出错了。

这些信号的方向非常重要：

```text
消费者 Subscriber                数据源 Publisher
      |                                |
      |---- subscribe(subscriber) --->|  我要订阅你
      |                                |
      |<--- onSubscribe(subscription)--|  给你订阅凭证
      |                                |
      |---- subscription.request(n) -->|  我要 n 个数据
      |                                |
      |<--- onNext(data1) -------------|  给你一个
      |<--- onNext(data2) -------------|  再给你一个
      |<--- onComplete() --------------|  发完了
```

记住：

- `subscribe`、`request` 这两个动作，从**下游往上游**走。
- `onSubscribe`、`onNext`、`onComplete`、`onError` 这四个信号，从**上游往下游**走。

**一个默认行为要记住**：`subscribe(Consumer)` 这类方法，内部会帮你调用 `subscription.request(Long.MAX_VALUE)`。意思是"有多少给我多少，我不做背压控制"。这就是为什么后面看到"request 被调用"时，其实就是"启动了数据发射"。

### 7.5 一个操作符，一对新的 Publisher/Subscriber

但事情没有那么简单。

刚才那行代码：

```java
webClient.get()
        .uri("http://other-service/api")
        .retrieve()
        .bodyToMono(String.class)
```

它**不是**一个 Publisher。它是**一串** Publisher。

每个操作符（`get`、`uri`、`retrieve`、`bodyToMono`）都会产生一个新的 Publisher，包装前一个：

```text
bodyToMono(String.class)   ← 最下游 Publisher
      ↑ 包装
retrieve()                 ← 上游 Publisher
      ↑ 包装
uri(...)                   ← 更上游 Publisher
      ↑ 包装
get()                      ← 最上游 Publisher
```

装配阶段构造的，就是这条 **Publisher 链**。每个 Publisher 都持有它的上游。

而订阅阶段，会构造一条方向相反的 **Subscriber 链**。订阅信号从最下游往上传播，每经过一个 Publisher，就创建一个包装下游 Subscriber 的新 Subscriber：

```text
最上游 Publisher
     │ 对应
     ↓
Subscriber A  （由最上游 Publisher 创建，包装下游）
     │ 持有下游
     ↓
Subscriber B  （由上游 Publisher 创建，包装下游）
     │ 持有下游
     ↓
Subscriber C  （由下游 Publisher 创建，包装下游）
     │ 持有下游
     ↓
末端 Subscriber（由 subscribe(...) 传入的 Lambda 包装而来）
```

每个 Publisher 都对应一个 Subscriber。两条链，方向相反，一一对应。

**数据流动的方向**：从最上游 Publisher 出发，沿 Subscriber 链往下游传。

数据经过每一层 Subscriber 时，对应的操作符逻辑就会执行一次。比如 `map` 对应的 Subscriber 会把数据加工一下再传给下游。

### 7.6 同步源的 subscribe：Flux.just 走一遍

先看一个最朴素的同步例子：

```java
Flux.just("A", "B")
    .subscribe(System.out::println);

System.out.println("subscribe 返回");
// 输出：
// A
// B
// subscribe 返回
```

数据先被打印，然后才是"subscribe 返回"。这说明 **subscribe() 返回时，整条链已经跑完了**。

为什么？因为 `Flux.just` 底层是 `FluxArray`，它的数据就在内存数组里，行为是同步发射。用伪代码模拟一下 `FluxArray` 的行为：

```java
class JustPublisher implements Publisher<String> {
    String[] data = {"A", "B"};

    public void subscribe(Subscriber<String> subscriber) {
        Subscription subscription = new Subscription() {
            boolean done = false;

            public void request(long n) {
                for (String d : data) {
                    if (!done) {
                        subscriber.onNext(d);
                    }
                }
                done = true;
                subscriber.onComplete();
            }

            public void cancel() {
                done = true;
            }
        };

        subscriber.onSubscribe(subscription);
    }
}
```

`subscriber.onSubscribe(...)` 被调用时，Subscriber 会去调 `subscription.request(Long.MAX_VALUE)`。而这个 `request` 方法里做的事情，就是**当场循环发射**：`onNext("A")`、`onNext("B")`、`onComplete()`。

所以整个执行顺序是：

1. 消费者调用 `JustPublisher.subscribe(...)`。
2. `JustPublisher` 创建 `Subscription`，调用 `subscriber.onSubscribe(subscription)`。
3. Subscriber 在 `onSubscribe` 里调用 `subscription.request(MAX)`。
4. `request` 当场发完数据，包括 `onComplete()`。
5. `subscribe()` 返回。

**subscribe() 返回时，数据已经处理完了。**

### 7.7 异步源的 subscribe：WebClient 走一遍

再看异步源。用 `Mono.fromFuture` 做例子：

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    sleep(1000);
    return "Hello";
});

Mono.fromFuture(future)
    .subscribe(System.out::println);

System.out.println("subscribe 返回");
// 输出：
// subscribe 返回
// （一秒后）
// Hello
```

顺序反了。为什么？

关键在异步源的 `request` **不立即发数据**。它做的事情，只是"登记需求 + 注册回调"。用伪代码模拟：

```java
class FutureSubscription<T> implements Subscription {
    Subscriber<? super T> actual;
    CompletableFuture<T> future;
    volatile long requested;
    volatile boolean cancelled;

    public void request(long n) {
        requested = n;

        if (future.isDone()) {
            emit();
        } else {
            future.whenComplete((value, error) -> {
                if (cancelled) return;
                if (error != null) {
                    actual.onError(error);
                } else {
                    if (requested > 0) {
                        actual.onNext(value);
                        actual.onComplete();
                    }
                }
            });
        }
    }

    public void cancel() {
        cancelled = true;
    }
}
```

看 `request` 方法：

- 如果 Future 已经完成，直接发。
- 如果 Future 没完成，**只注册一个 `whenComplete` 回调，然后返回**。

所以 subscribe 的执行顺序是：

1. 消费者调用 `FuturePublisher.subscribe(...)`。
2. `FuturePublisher` 创建 `FutureSubscription`，调用 `subscriber.onSubscribe(fs)`。
3. Subscriber 调用 `fs.request(MAX)`。
4. `request` 发现 Future 没完成，**注册一个回调**，然后返回。
5. `subscribe()` 返回。

**subscribe() 返回时，响应还没到，数据还没产生。**

一秒钟后，Future 完成。`whenComplete` 回调在**完成 Future 的那个线程**上被触发：

```java
actual.onNext("Hello");
actual.onComplete();
```

数据从这一刻起，才开始沿 Subscriber 链向下游流动。

**把 WebClient 套进这个模型，就是同样的道理。** WebClient 的 Publisher 在 `request` 时做的事情，本质上和上面一样：不是"当场发数据"，而是"发起 HTTP 请求 + 在 ClientChannel 的 Pipeline 上注册一个 ResponseHandler"。

`request` 返回，`subscribe` 返回，Server EventLoop 回到循环。**响应什么时候到，由 Client EventLoop 通过 `select()` 感知。**

### 7.8 subscribe 到底阻不阻塞？

到这里，可以回答一个最常见的问题了：

> **subscribe() 是不是阻塞的？**

准确的答案是一句话：

> **subscribe() 本身不阻塞。但如果数据源是同步的，它会在当前线程一直执行到 `onComplete` 才返回。如果是异步的，它会注册回调后立即返回。**

用一个对比表把它说清楚：

| 对比点 | 同步源 | 异步源 |
|---|---|---|
| 例子 | `Flux.just`、`Flux.range` | `Mono.fromFuture`、`Flux.interval`、WebClient |
| `subscribe` 调用线程 | 常常跑完整条流才返回 | 通常只注册回调/启动任务就返回 |
| `request` 做什么 | 当场循环发数据 | 登记需求、注册回调、启动定时任务、发起 IO |
| `onNext` 在哪个线程 | 调用 `subscribe` 的线程 | 异步线程、调度器线程、Netty 线程 |
| `subscribe` 返回时 | 数据通常已经发完 | 数据通常还没到 |
| 是否占住调用线程 | 是，直到完成 | 通常不占，快速返回 |

真正的显式阻塞是 `block()` / `blockLast()`。`subscribe()` 不是这个语义。

**一句话概括**：

> **subscribe 是"触发执行"，不是"等待执行"。但同步源会让"触发过程"本身就执行完，从而占住调用线程。**

### 7.9 分水岭：NIO 与 BIO

现在回到那个关键问题：

> **为什么 WebClient 可以"注册回调后立即返回"，而 RestTemplate 不行？**

答案不在 API 层，也不在 Mono 这一层。答案在**操作系统的 IO 模型**里。

**RestTemplate 用的是 BIO（阻塞 IO）。**

```java
String result = restTemplate.getForObject("http://other-service/api", String.class);
```

这行代码背后，当前线程做的是：

```java
socket.getOutputStream().write(requestBytes);  // 写请求
socket.getOutputStream().flush();

byte[] responseBytes = readFully(socket.getInputStream());
//    ↑ 关键：当前线程在这里阻塞。
//      数据还没到，操作系统就把线程挂起。
//      线程从"运行"变成"等待"，什么都做不了。
//      直到响应到达，操作系统才把它唤醒。
```

BIO 的 `read()` 是阻塞的。没有数据，线程就挂起。**线程必须亲自去读，读了就得等。**

**WebClient 用的是 NIO（非阻塞 IO）。**

最上游 Publisher 做的是：

```java
channel.writeAndFlush(requestBytes);   // 写请求

channel.pipeline().addLast(new ResponseHandler(callback));
//    ↑ 关键：注册一个 Handler，不读。

return;  // 立即返回
```

它没有调用任何阻塞的 `read()`。它注册完 Handler，就返回了。

那么响应什么时候被读？答案是：**由 EventLoop 在未来的某个时刻，通过 `select()` 感知到"这个 Channel 有数据可读"，然后代它读。**

**分水岭就在这里：**

> **BIO 的 read() 会阻塞线程，所以线程必须亲自等。**
>
> **NIO 的 read() 不阻塞，响应的读取交给 `select()` 通知，所以线程可以注册完 Handler 就走。**

Mono 和 `subscribe()` 只是把 NIO 的这个能力，包装成了"订阅式 API"的样子。真正决定同步还是异步的，是底层的 IO 模型。

**一个常见的误解**：

有人以为"WebClient 是异步的，是因为它返回 Mono"。

这是错的。如果把一个阻塞的 RestTemplate 调用包在一个 Mono 里：

```java
Mono<String> fakeAsync() {
    return Mono.fromCallable(() -> restTemplate.getForObject(...));
}
```

它确实返回了 Mono。但 `subscribe()` 之后，还是会一路走到阻塞的 `read()`。它只是把阻塞"藏"在了另一个地方。

**Mono 不是异步的原因，NIO 才是。**

### 7.10 响应到达：回调触发，数据开始流动

other-service 的响应回来了。

它的字节到达的是 **ClientChannel 绑定的那个 EventLoop**。

因为这个 Channel 的所有 IO 事件只能由它绑定的 EventLoop 处理。

于是：

- 这个 EventLoop 在自己的 `while(true)` 中 `select()` 到数据可读。
- 它读取字节，解码成 HTTP 响应。
- 它触发 ClientChannel 的 Pipeline。
- 它走到 `ResponseHandler.channelRead`。
- `ResponseHandler` 调用它保存的那个最上游 Subscriber 的 `onNext(data)`。

"触发回调"听起来很神秘，但实际上就是一次普通的方法调用：

```java
callback.onNext(responseData);
```

没有魔法。只是另一个 EventLoop 线程在它的循环中执行了这一行代码。

从这一刻起，数据开始沿 Subscriber 链往下游流动。这就是 `.bodyToMono(String.class)` 真正执行的地方：

- 它把响应体（`ByteBuf` / `DataBuffer`）转换成 `String`。
- 然后把 String 交给下游的 Subscriber。

**这一步跑在 ClientChannel 的 EventLoop 上。**

`.bodyToMono(String.class)` 是在装配阶段就"就位"的一节。它执行的时间点是"响应到达，数据流经它的时候"。它执行的环境是 ClientChannel 的 EventLoop。

### 7.11 数据怎么流回 ServerChannel：靠的是"记"，不是"找"

数据一节一节地沿着 Subscriber 链往下游走。最终会到达链的末端。

链的末端是什么？是 **WebFlux 在订阅 Controller 返回的 Mono 时挂上去的那个末端 Subscriber**。

这个末端 Subscriber 是在什么时候挂上去的？

在 `ChannelOperationsHandler` 调用 `subscribe()` 的那一刻，WebFlux 内部会做这样一件事：

```java
Mono<Void> result = handler.handle(exchange);

result.subscribe(
    data -> writeBackToServer(exchange, data),
    error -> handleError(exchange, error),
    () -> completeResponse(exchange)
);
```

注意这里的 `exchange`。

它是一个被闭包捕获的变量。换句话说：

> 在装配阶段，WebFlux 就已经把 `ServerWebExchange` 塞进了末端 Subscriber 的闭包里。这个 `ServerWebExchange` 内部持有 `ServerHttpResponse`，`ServerHttpResponse` 内部持有 ServerChannel。

于是，当数据最终流到链的末端时，末端 Subscriber 不需要"找" ServerChannel。它只需要用自己闭包里早就记着的 `exchange`，把数据写回去：

```java
exchange.getResponse().writeWith(...);
```

整个过程中，没有任何"找"的动作。

**一句话总结**：

> 数据流回 ServerChannel，靠的不是"找"，而是"记"。WebFlux 在装配阶段就把 `ServerWebExchange` 记进了链条末端 Subscriber 的闭包里。数据到达末端时，末端自然知道该往哪里写。

用一个比喻：这就像寄快递。寄件的时候，收件地址就已经写在单子上了。包裹在路上怎么辗转，都不影响单子上已经写好的收件地址。到了末端，直接按单子上的地址投递即可。

### 7.12 最后一公里：writeAndFlush

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

### 7.13 整条时间线

把上面所有步骤串成一条时间线。每一步都标注"跑在哪个线程"。

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
T8    subscribe() 在异步边界处返回，EventLoop 回到 while(true)  Server EventLoop
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

- T5 到 T8 之间，subscribe() 执行了装配，遇到异步边界后返回。
- T8 到 T9 之间，没有任何线程被阻塞。
- `.bodyToMono(String.class)` 是 T12 执行的，跑在 Client EventLoop 上。
- T13 写回响应，靠的是末端 Subscriber 闭包捕获的 `exchange`，不需要"找" ServerChannel。
- T14 的 `writeAndFlush` 会在 T15 决定是否要调度回 Server EventLoop。
- T16 数据真正发送给浏览器。

### 7.14 为什么常常不需要线程切换

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

## 八、在 WebFlux 里，不要手动 subscribe

理解了 subscribe 的机制之后，有一条实践准则值得单独拿出来讲：

> **在 Spring WebFlux 的 Controller 里，不要手动调用 `subscribe()`。**

**正确写法**：

```java
@GetMapping("/users/{id}")
public Mono<User> getUser(@PathVariable String id) {
    return userService.findById(id)
            .doOnNext(user -> log.info("found: {}", user));
}
```

返回 `Mono` 或 `Flux`。Spring WebFlux 会替你订阅：它会把你返回的 Publisher 接到 WebFlux 内部的订阅链上，`onNext` 写成 HTTP 响应体，`onComplete` 结束响应，`onError` 交给异常处理机制。

**错误写法**：

```java
@GetMapping("/users/{id}")
public Mono<User> getUser(@PathVariable String id) {
    userService.findById(id)
            .subscribe(user -> log.info("found: {}", user));

    return Mono.empty();
}
```

手动订阅会带来一堆问题：

1. **框架无法管理响应**：结果不会自动写回 HTTP。
2. **可能双订阅**：冷流会被执行两次。
3. **错误丢失**：错误不会进入 WebFlux 的 `@ExceptionHandler`。
4. **上下文丢失**：Reactor Context、安全上下文可能丢失。
5. **资源泄漏**：请求结束或取消时，手动订阅的流可能继续运行。
6. **可能阻塞 Netty EventLoop**：如果流中包含同步阻塞操作。

需要副作用，用 `doOnNext`、`doOnError`、`doFinally`：

```java
return userService.findById(id)
        .doOnNext(user -> log.info("found: {}", user));
```

需要组合异步操作，用 `flatMap`、`then`、`zip`、`merge`：

```java
return userService.findById(id)
        .flatMap(user -> anotherService.doSomething(user))
        .then();
```

测试时用 `StepVerifier`：

```java
StepVerifier.create(service.findById("1"))
        .expectNextCount(1)
        .verifyComplete();
```

**记住这一条**：`subscribe` 是"由框架负责"的。业务代码只负责"构建 Publisher 链"，由框架在合适的时机把它接进自己的订阅链。

---

## 结语：从 Tomcat 到 Netty，思维需要一次跳跃

回到最开始。

在 Tomcat 里：

> 一个请求，一个线程，阻塞等待，返回响应。
>
> 线程必须亲自去 `read()` 结果。`read()` 会阻塞线程，所以线程只能等。这就是 BIO 的宿命。

在 Netty 里：

> 一个 EventLoop 线程，管理很多 Channel，谁有数据就处理谁。
>
> 链式调用不是"顺序执行"，而是"构建 Publisher 链"。订阅才让 Publisher 链变成行动。
>
> 订阅信号从下游往上游走，建立 Subscriber 链。数据信号从上游往下游走，沿 Subscriber 链流动。
>
> subscribe() 本身不阻塞。但如果数据源是同步的，它会在当前线程执行到 `onComplete` 才返回；如果是异步的，它注册回调后立即返回。
>
> 异步的本质不是"多线程"，而是"当前线程不亲自等，把等待和后续处理交给一个回调"。
>
> 数据流回 ServerChannel，靠的是"记"，不是"找"。WebFlux 在装配阶段就把 `ServerWebExchange` 记进了末端 Subscriber 的闭包里。
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

6. **响应式流有五个角色：Publisher、Subscriber、Subscription、request、onNext。**
   `subscribe` 和 `request` 从下游往上游走；`onSubscribe`、`onNext`、`onComplete`、`onError` 从上游往下游走。

7. **一个操作符，一对新的 Publisher/Subscriber。**
   装配阶段构造 Publisher 链（从上往下持有），订阅阶段构造 Subscriber 链（从下往上配对）。两条链方向相反，一一对应。

8. **链式调用不是"顺序执行"，而是"构建蓝图"。**
   `webClient.get().uri(...).retrieve().bodyToMono(...)` 在 Controller 返回时，一行都没执行。它只构建了一条 Publisher 链。

9. **subscribe() 本身不阻塞。**
   如果数据源是同步的，它会在当前线程执行到 `onComplete` 才返回。如果是异步的，它注册回调后立即返回。真正的显式阻塞是 `block()`。

10. **同步源和异步源的 `request` 行为不同。**
    同步源的 `request` 是"当场发货"；异步源的 `request` 是"登记需求 + 注册回调"。

11. **同步与异步的分水岭在 IO 层，不在 API 层。**
    RestTemplate 底层是 BIO，当前线程必须亲自 `read()`，`read()` 会阻塞线程。WebClient 底层是 NIO，注册 Handler 后立即返回，响应由 EventLoop 在将来通过 `select()` 感知并读取。

12. **不是 Mono 让 WebClient 异步的。**
    如果底层是 BIO，返回 Mono 也还是同步阻塞。真正让 WebClient 异步的，是它底层用的 NIO。

13. **响应到达时，Client EventLoop 唤醒。**
    `ResponseHandler` 调用最上游 Subscriber 的 `onNext`，数据沿 Subscriber 链向下游流动，`.bodyToMono` 等操作符在这时执行。

14. **写回响应，靠的是"记"，不是"找"。**
    WebFlux 在装配阶段就把 `ServerWebExchange`（内含 `ServerChannel` 引用）记进了末端 Subscriber 的闭包。数据到达末端，末端用它写回响应。

15. **`writeAndFlush` 负责把数据交给正确的 EventLoop。**
    它内部检查当前线程，不是目标 EventLoop 就把任务排进队列。底层是 `eventLoop().execute(task)`。

16. **在 WebFlux 里，不要手动 subscribe。**
    业务代码只负责"构建 Publisher 链"，由框架在合适的时机订阅。需要副作用用 `doOnXxx`，需要组合用 `flatMap` / `then` / `zip`。

17. **Reactor Netty 默认共享全局资源，并支持共址机制。**
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
- 最上游 Publisher 底层用 NIO 发出请求，注册 Handler，立即返回；
- subscribe() 在异步边界处返回，EventLoop 回到循环，等待响应；
- 响应到达时，Client EventLoop 通过 `select()` 感知，触发 ResponseHandler，数据沿 Subscriber 链向下游流动；
- 数据到达末端时，末端 Subscriber 用闭包里早就记下的 `ServerWebExchange` 写回 ServerChannel；
- 最后 `writeAndFlush` 保证数据落到正确的 EventLoop 上。

没有魔法。只有 EventLoop 的循环、NIO 的 `select()`、Publisher/Subscriber 的配对、闭包的捕获、`writeAndFlush` 的线程调度，以及装配（Assembly）和执行（Execution）两个阶段的分野。