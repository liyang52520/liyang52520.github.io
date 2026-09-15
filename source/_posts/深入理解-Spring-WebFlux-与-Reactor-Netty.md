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

## 引子：当 Servlet 思维遇上 Netty

如果你习惯了 Spring MVC + Tomcat，第一次接触 Spring WebFlux + Reactor Netty 时，很容易感到困惑。

在 Tomcat 的世界里，一切都很直观：Tomcat 是 Servlet 容器，负责 TCP 连接、HTTP 解析、线程池。它把请求交给 `DispatcherServlet`，`DispatcherServlet` 再找到你的 Controller。你的代码跑在 Tomcat 的线程池里，阻塞等待数据库、远程调用，然后返回响应。

但 Netty 完全不同。它的核心是：

> **EventLoopGroup → EventLoop → Channel → ChannelPipeline → ChannelHandler**

一个 EventLoop 就是一个线程，它跑着 `while(true)` 循环，用 Selector 监听多个 Channel 的 IO 事件。每个 Channel 绑定唯一一个 EventLoop，拥有自己的 ChannelPipeline。所有业务逻辑都写在 ChannelHandler 里，由 EventLoop 线程调用。

那么问题来了：Spring WebFlux 在这个模型里处于什么位置？它是 Netty 的 ChannelHandler 吗？当我们在 Controller 里用 WebClient 发起异步调用时，线程到底发生了什么？回调是谁触发的？写回响应时又怎么回到原来的线程？

这篇文章将带你走完一次请求的完整生命周期。我们会从一行看似普通的回调代码出发，层层深入，最终看清整个响应式 Web 的线程模型。

## 一、Netty 核心模型：一切的基础

在深入之前，先明确几个 Netty 的基本事实。

**EventLoopGroup** 是一组 EventLoop 线程。通常分为 boss 组和 worker 组：boss 负责接受连接，worker 负责处理已建立连接的 IO。

**EventLoop** 是一个线程，内部有一个 Selector。它负责处理注册在其上的所有 Channel 的 IO 事件。一个 EventLoop 可以管理多个 Channel，但一个 Channel 只能绑定一个 EventLoop。

**Channel** 代表一个网络连接。它有自己的 ChannelPipeline。

**ChannelPipeline** 是 ChannelHandler 的链表。每个 ChannelHandler 负责处理入站或出站事件。入站事件（如读到数据）从 pipeline 头部向尾部传播；出站事件（如写数据）从尾部向头部传播。

**ChannelHandler** 是真正干活的地方。它的方法（如 `channelRead`）由 EventLoop 线程调用。

当你用原生 Netty 写一个 HTTP 服务器时，代码大致是这样：

```java
pipeline.addLast(new HttpServerCodec());
pipeline.addLast(new HttpObjectAggregator(65536));
pipeline.addLast(new SimpleChannelInboundHandler<FullHttpRequest>() {
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, FullHttpRequest req) {
        // 你的业务逻辑
    }
});
```

`SimpleChannelInboundHandler` 是一个 ChannelHandler，它被放在 Pipeline 里。当 EventLoop 读到数据、解码成 `FullHttpRequest` 后，就会调用它的 `channelRead0` 方法。**所有业务代码都跑在某个 EventLoop 线程上。**

## 二、Reactor Netty：在 Netty 之上加一层 HTTP 抽象

Reactor Netty 并没有改变 Netty 的线程模型，它只是把 Netty 的 ChannelHandler 细节封装起来，对外暴露一个更上层的接口：

```java
public interface HttpHandler {
    Mono<Void> handle(HttpServerRequest request, HttpServerResponse response);
}
```

Reactor Netty 内部会构建 Netty 的 Pipeline，添加 HTTP 编解码器、聚合器等 ChannelHandler。在 Pipeline 的末端，它放置一个自己的 ChannelHandler（我们称之为 `ReactorNettyBridge`）。这个 ChannelHandler 的 `channelRead0` 方法里，会做这样的事：

```java
protected void channelRead0(ChannelHandlerContext ctx, FullHttpRequest req) {
    HttpServerRequest request = wrapRequest(req);
    HttpServerResponse response = wrapResponse(ctx);
    
    // 调用外部传入的 HttpHandler
    Mono<Void> result = httpHandler.handle(request, response);
    
    // 订阅这个 Mono，启动响应式链
    result.subscribe();
}
```

**关键点：Reactor Netty 的 `HttpHandler` 并不是 Netty 的 `ChannelHandler`。它是被 ChannelHandler 调用的一个函数。** 从 Netty 的视角看，WebFlux 藏在 `ReactorNettyBridge` 的 `channelRead0` 方法体里。

## 三、Spring WebFlux 的接入点

Spring WebFlux 不依赖 Servlet API，它有自己的抽象：`HttpHandler`、`WebHandler`、`DispatcherHandler`（类似 MVC 的 `DispatcherServlet`）。

当 Spring Boot 使用 WebFlux + Reactor Netty 时，启动过程大致如下：

```java
DispatcherHandler dispatcherHandler = ...; // 能找到 @Controller
WebHandler webHandler = new FilteringWebHandler(dispatcherHandler);
HttpHandler webFluxHttpHandler = new HttpWebHandlerAdapter(webHandler);

// 将 WebFlux 的 HttpHandler 适配成 Reactor Netty 的 HttpHandler
ReactorHttpHandlerAdapter adapter = new ReactorHttpHandlerAdapter(webFluxHttpHandler);

HttpServer.create()
    .port(8080)
    .handle(adapter)   // 注册 Reactor Netty 的 HttpHandler
    .bindNow();
```

`ReactorHttpHandlerAdapter` 实现了 Reactor Netty 的 `HttpHandler` 接口。它的 `handle` 方法会把 Reactor Netty 的请求/响应适配成 WebFlux 的 `ServerHttpRequest` / `ServerHttpResponse`，然后调用 `webFluxHttpHandler.handle(...)`，最终进入 `DispatcherHandler`，再到你的 Controller。

**所以，Spring WebFlux 不是 Netty 的 ChannelHandler，而是被适配成 Reactor Netty 的 HttpHandler，在 Pipeline 末端的某个 ChannelHandler 里被调用。**

## 四、一次同步请求的完整链路

为了建立整体认知，我们先看一次简单的同步请求。浏览器访问 `GET /hello`，Controller 返回 `Mono.just("hello")`。

1. 客户端连接，Netty 创建 `SocketChannel`，注册到某个 worker EventLoop。Channel 拥有自己的 Pipeline。
2. 该 EventLoop 线程从 Channel 读到字节。
3. 字节经过 `HttpServerCodec` 解码，`HttpObjectAggregator` 聚合，最终到达 `ReactorNettyBridge.channelRead0`。
4. `ReactorNettyBridge` 调用 `adapter.apply(request, response)`。
5. `ReactorHttpHandlerAdapter` 适配后调用 `HttpWebHandlerAdapter`，再调用 `DispatcherHandler`。
6. `DispatcherHandler` 根据 `/hello` 找到 `HelloController.hello()`，得到 `Mono.just("hello")`。
7. WebFlux 将 `"hello"` 写入 `ReactorServerHttpResponse`，最终写回 Netty Channel，由 EventLoop 发送给客户端。

整个过程中，所有操作都在同一个 EventLoop 线程上完成，没有线程切换。

## 五、异步场景：从一行回调代码开始

现在，真正的挑战来了。假设 Controller 里用 WebClient 调用另一个服务：

```java
@GetMapping("/hello")
public Mono<String> hello() {
    return webClient.get()
            .uri("http://other-service/api")
            .retrieve()
            .bodyToMono(String.class);
}
```

`hello()` 返回的是一个 `Mono<String>`，它只是一个**描述**，并没有执行。Reactor Netty 在 `channelRead0` 里对这个 Mono 调用了 `subscribe()`，这才启动了整个响应式链。

`subscribe()` 触发后，WebClient 会发起 HTTP 请求。这个请求是异步的。它内部会做几件事：

1. 获取一个连接 other-service 的 Channel（由 WebClient 的 EventLoopGroup 管理）。
2. 把 HTTP 请求写入这个 Channel。
3. 注册一个回调：当这个 Channel 收到响应时，调用回调。
4. 立即返回。

`subscribe()` 返回，`channelRead0` 返回，EventLoop 线程回到 `while(true)`，继续处理其他 Channel。

**此时，原来的 EventLoop 线程被释放了，没有阻塞等待远程响应。**

那么，当 other-service 的响应到达时，发生了什么？这就是异步回调的核心。

## 六、回调触发：谁调用了你的 lambda？

当 other-service 的响应到达时，是由 **WebClient 的 EventLoop 线程** 感知到的。

- 该 EventLoop 在自己的 `while(true)` 循环中 `select()` 到数据可读。
- 它读取字节，解码成 HTTP 响应。
- 然后触发 Pipeline，最终调用到 WebClient 内部注册的回调。
- 这个回调就是你写在 `.bodyToMono(String.class)` 后面的那些操作符，它们最终会形成一个 lambda，被调用时执行。

**“触发回调”听起来很神秘，但实际上就是一次普通的方法调用：**

```java
callback.onSuccess(result);
```

没有魔法，只是另一个 EventLoop 线程在它的循环中执行了这一行代码。

让我们把这个过程展开成最原始的调用栈：

```text
WebClient 的 EventLoop 线程：
  |
  |-- while(true)
  |     |
  |     |-- select() 返回，发现连接 other-service 的 Channel 可读
  |     |
  |     |-- read() 读到字节
  |     |
  |     |-- decode() 解码成 HTTP 响应
  |     |
  |     |-- pipeline.fireChannelRead(response)
  |     |     |
  |     |     |-- ResponseHandler.channelRead(ctx, response)
  |     |     |     |
  |     |     |     |-- Callback cb = pendingCallbacks.get(channel);
  |     |     |     |-- cb.onSuccess(result)
  |     |     |     |     |
  |     |     |     |     |-- 你的 lambda 代码开始执行
  |     |     |     |     |-- 比如 map、flatMap 等操作符
  |     |     |     |     |
  |     |     |     |     |-- 返回
  |     |     |     |
  |     |     |     |-- 返回
  |     |     |
  |     |     |-- 返回
  |     |
  |     |-- 继续 while(true)
```

**看到没有？`cb.onSuccess(result)` 这一行，就是“触发回调”。它是一次普通的方法调用，和其他任何方法调用没有区别。**

那么，后续的操作（比如 `map`、`flatMap`）在哪个线程上执行？默认情况下，它们继续在这个 WebClient 的 EventLoop 线程上执行。直到需要写回原响应时，才会发生线程调度。

## 七、线程调度：写回响应时发生了什么

回调在 WebClient 的 EventLoop 线程上执行。如果你的后续操作没有显式切换线程，它们会继续在这个线程上运行。

但最终，你需要把结果写回**原来那个服务器的 Channel**。而服务器的 Channel 绑定在它自己的 EventLoop 上。Netty 规定：Channel 的写操作必须在它绑定的 EventLoop 上执行。

所以，Reactor Netty 在写响应时（`ReactorServerHttpResponse`）会自动检查当前线程：

```java
if (currentThread == serverChannel.eventLoop()) {
    serverChannel.writeAndFlush(data);
} else {
    serverChannel.eventLoop().execute(() -> {
        serverChannel.writeAndFlush(data);
    });
}
```

它把写任务丢进服务器 EventLoop 的任务队列，然后返回。服务器 EventLoop 在下次循环时执行这个任务，将响应发送给客户端。

**这就是所谓的“调度回原 EventLoop”。框架帮你做了，但底层就是 `eventLoop().execute(task)`。**

## 八、EventLoopGroup 共享：同一个线程的惊喜

你可能听过一种说法：“WebClient 会把自己新建的 Channel 注册到当前的 Selector 上。” 这听起来与“服务器和客户端各自维护 EventLoop”矛盾，但其实两者描述的是不同层面。

- **每个 EventLoop 拥有独立的 Selector**，这是 Netty 的基础。一个 EventLoop 管理的所有 Channel 都注册在它自己的 Selector 上，不会跨 EventLoop 注册。
- **Reactor Netty 默认让客户端和服务器共享同一个 EventLoopGroup**。这是通过 `ColocatedEventLoopGroup` 实现的。

共享的效果是：当你在服务器的 EventLoop 线程中调用 WebClient 时，WebClient 创建的连接 Channel 会“就地”注册到**当前服务器线程自己的 Selector** 上。这样，远程响应的数据到达时，仍然是同一个 EventLoop 线程通过 `select()` 感知到，并直接在该线程上触发回调。

**所以，之前关于“回调在 client 线程上执行”的说法，在共享 EventLoopGroup 的情况下，client 线程和 server 线程其实是同一个线程。** 如果不共享，它们就是不同的线程，需要显式调度。Reactor Netty 默认共享，因此通常不需要线程切换，性能也更好。

让我们用一张图来展示共享情况下的完整线程之旅：

```text
服务器 EventLoop 线程（也是 WebClient 的 EventLoop）：
  |
  |-- 处理浏览器请求
  |-- 进入 Controller，返回 Mono
  |-- Reactor Netty 订阅 Mono
  |     |-- WebClient 发起请求（写任务到同一 EventLoop）
  |     |-- 返回
  |-- EventLoop 继续 while(true)
  |
  |        ... 等待远程响应 ...
  |
  |-- select() 发现 WebClient 的连接可读
  |-- 读取、解码 other-service 响应
  |-- 触发 Reactor 回调（在当前线程）
  |     |-- 执行 map/flatMap 等
  |     |-- 写回原响应（Reactor Netty 自动调度，但同线程直接写）
  |-- 响应发送给浏览器
```

## 九、全景总结：从一行代码到整个宇宙

让我们回到最初的那行代码：

```java
callback.onSuccess(result);
```

这行代码背后，是整个 Netty 的 EventLoop 体系在运转：

- 一个 EventLoop 线程，在 `while(true)` 中不断 `select()`。
- 当它管理的某个 Channel 可读时，它读取数据、解码、触发 Pipeline。
- Pipeline 末端的某个 ChannelHandler 从 Map 中取出回调对象，调用它的方法。
- 回调执行你的响应式链，最终写回原 Channel。
- 如果 EventLoop 是共享的，这一切都在同一个线程上完成，没有线程切换开销。

Spring WebFlux 并没有脱离 Netty 的线程模型。它只是通过 `ReactorHttpHandlerAdapter` 把自己嵌入到 Reactor Netty 的 `HttpHandler` 中，而 Reactor Netty 又把 `HttpHandler` 嵌入到 Netty 的 ChannelHandler 中。层层嵌套，但核心始终是 EventLoop 线程在驱动一切。

**核心结论：**

1. Spring WebFlux 不是 Netty 的 ChannelHandler，而是被适配成 Reactor Netty 的 HttpHandler，在 Pipeline 末端的 ChannelHandler 中被调用。
2. `Mono` 是描述，`subscribe()` 才启动执行。异步操作立即返回，EventLoop 线程被释放。
3. 回调由持有连接的 EventLoop 线程触发，触发就是一次普通的方法调用。
4. 写回原响应时，需要确保在正确的 EventLoop 上执行。Reactor Netty 自动处理线程调度，底层是 `eventLoop().execute(task)`。
5. Reactor Netty 默认共享 EventLoopGroup，使得 WebClient 的 Channel 注册到当前服务器的 EventLoop 上，从而减少线程切换，提高性能。

理解这些，你就能看透 WebFlux + Reactor Netty 的线程模型：它没有魔法，只有 EventLoop 的循环、Selector 的监听、Pipeline 的传播和回调的调用。响应式编程的优雅，正建立在这些朴素而强大的机制之上。

## 结语

从 Tomcat 的 Servlet 线程池，到 Netty 的 EventLoop 循环，思维需要一次跳跃。但一旦你理解了 EventLoop 如何驱动一切，理解了回调不过是方法调用，理解了线程调度如何通过任务队列实现，整个响应式 Web 的线程模型就会变得清晰而自然。

希望这篇文章能帮你完成这次跳跃。当你下次写下 `webClient.get().retrieve().bodyToMono(...)` 时，你会知道，在那行代码背后，是一个 EventLoop 线程正在安静地等待，然后在数据到达时，毫不犹豫地继续它的旅程。
