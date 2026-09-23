---
title: 深入 WebFlux 与 Spring Cloud Gateway：为什么网关没有 Controller，却是一个 WebFlux 应用？
date: 2026-09-21 17:28:55
tags:
categories:
---

## 一个真实的困惑

你第一次翻开 Spring Cloud Gateway 的源码，或者试着往里加一个自定义逻辑时，大概率会遇到这样一个瞬间：

> 我知道 Spring WebFlux 是用来写 `@Controller` 的响应式 Web 框架，可我翻遍了 Gateway 的代码，一个 Controller 都没找到。那它到底是怎么处理请求的？为什么所有人还说它“基于 WebFlux”？

这个困惑不是你的问题，而是“WebFlux”这个词被用得太窄了。

大多数人第一次接触 WebFlux，都是从 `@Controller` 开始的，于是很自然地把它当成了 WebFlux 的代名词。但真相是：**`@Controller` 只是 WebFlux 最上面的一层皮。Gateway 剥掉了这层皮，把下面的骨架完整地拿过来用了。**

这篇文章就沿着“一个请求进来之后到底发生了什么”这条线，把 WebFlux 的骨架拆开，再把 Gateway 套进去，看看它是怎么复用的。


## 一、从你熟悉的地方出发：Servlet 栈

在进入 WebFlux 之前，先回到你熟悉的 Spring MVC。

一个 HTTP 请求打进来，在 Servlet 栈里大概会经历这样的旅程：

```
请求 → Filter → DispatcherServlet → HandlerMapping → Interceptor → Controller → 响应
```

这里面有几个关键角色：

- **Filter**：最外层的关卡。它的特征是请求进来先过它，响应回去再过它，把后面的流程“包在中间”。
- **DispatcherServlet**：总入口，所有请求都先到它这里，由它负责调度。
- **HandlerMapping**：一张“请求 → 处理者”的通讯录，根据 `@RequestMapping` 找到对应的 Controller。
- **Controller**：真正写业务逻辑的地方。

这个结构回答了一个最朴素的问题：**一个请求进来，到底怎么找到该由谁来处理？**

WebFlux 的骨架，本质上就是在回答同一个问题。只是换了一套名字和编程模型。


## 二、WebFlux 的骨架：入口、关卡与干活的人

WebFlux 虽然建立在响应式编程之上，但它的请求处理结构和 Servlet 栈在**骨架层面对称**。官方文档明确指出：WebFlux 和 Spring MVC 一样，围绕前端控制器模式设计，核心的 `DispatcherHandler` 提供共享的请求处理算法，实际工作交给可配置的委托组件。

把这套骨架拆开，有四个关键角色。

### 2.1 DispatcherHandler：总调度

所有请求的第一站，WebFlux 的入口。

它自己不处理业务，只做一件事：**找到能处理当前请求的 Handler，然后调用它。**

你可以把它理解成 WebFlux 世界的 `DispatcherServlet`。它本身也实现了 `WebHandler` 接口，但它的角色是“调度者”，不是“执行者”。

### 2.2 HandlerMapping：通讯录

`DispatcherHandler` 拿到请求后，第一件事就是问：**这个请求该由谁来处理？**

回答这个问题的就是 `HandlerMapping`。

在普通 WebFlux 应用里，主要的实现是 `RequestMappingHandlerMapping`，它扫描 `@RequestMapping` 注解，建立“URL → Controller 方法”的映射。

### 2.3 WebFilter：关卡，以及驱动它的 FilteringWebHandler

在 `DispatcherHandler` 开始调度之前，请求会先经过一层过滤器。

这层过滤器是 `WebFilter`。它的执行模型和 Servlet 的 `Filter` 一样，请求进来先过一遍，响应回去再过一遍，把后面的整个流程“包在中间”：

```
请求
  ↓
WebFilter 1 → WebFilter 2 → WebFilter 3
  ↓
DispatcherHandler（调度开始）
  ↓
（处理请求）
  ↓
WebFilter 3 → WebFilter 2 → WebFilter 1（反向）
  ↓
响应
```

那么这条 WebFilter 链是谁来驱动的？

答案是 **WebFlux 的 `FilteringWebHandler`**，位于 `org.springframework.web.server.handler` 包下。它是一个 `WebHandlerDecorator`，职责是在调用最终的目标 `WebHandler`（通常是 `DispatcherHandler`）之前，先执行一串 `WebFilter`。

它的角色是 **WebFilter 链的装配器和启动器**——它本身不做过滤逻辑，只负责把过滤器串起来，然后启动这条链。

**记住这个名字。** 后面讲到 Gateway 时，会出现一个和它同名、但完全不同包、完全不同职责的类。到时候我们再回头对比。

### 2.4 ServerWebExchange：随身文件夹

请求从进来到处理完，会经过很多组件。每个组件都可能需要读请求信息、写响应，或者跟后面的组件传话。

这些信息不能丢，所以 WebFlux 把它们封装进一个对象里，一路传下去。这个对象就是 `ServerWebExchange`。

它装着三样东西：

- **请求**（`ServerHttpRequest`）
- **响应**（`ServerHttpResponse`）
- **一个可变的属性 Map**（`getAttributes()`）

那个属性 Map 是关键。它相当于一次请求生命周期内的共享存储——谁都可以往里放东西，谁都可以从里面取东西。后面你会看到，Gateway 的路由匹配结果就是通过这个 Map 传递给后续组件的。

### 一张表对齐两套栈

| Servlet 栈 | WebFlux 栈 | 位置 |
|---|---|---|
| Filter | WebFilter | DispatcherHandler 之前 |
| DispatcherServlet | DispatcherHandler | 总入口 |
| HandlerMapping | HandlerMapping | 查表 |
| Interceptor | （无直接对应） | — |
| Controller | 目标 WebHandler | 真正执行 |

到这里，WebFlux 的骨架就清楚了。**它不是一个注解框架，而是一套“请求进来 → 过 WebFilter 链 → 查通讯录 → 找到人 → 干活”的流水线。**

`@Controller` 只是这条流水线上“干活”那一环的一种实现方式。你可以换掉它，流水线照样运转。


## 三、Gateway 登场：它替换了什么，保留了什么

现在回到最初的问题。

Gateway 的官方文档里有一句很直白的话：**Spring Cloud Gateway 匹配路由的机制，是作为 Spring WebFlux 的 `HandlerMapping` 基础设施的一部分来工作的。**

这句话翻译过来就是：Gateway 没有重写 WebFlux 的流水线，它只是往流水线里**塞了两个自己的零件**。

### 3.1 换掉通讯录：RoutePredicateHandlerMapping

普通 WebFlux 应用里，通讯录是 `RequestMappingHandlerMapping`，它根据注解找 Controller。

Gateway 换成了 `RoutePredicateHandlerMapping`。它继承自 `AbstractHandlerMapping`，本质就是一个 WebFlux 的 `HandlerMapping` 实现。

它的逻辑很直接：

1. 遍历所有已加载的 `Route`；
2. 对每条路由的 `Predicate` 求值；
3. 返回第一个匹配的路由；
4. 把匹配到的 `Route` 对象存入 `ServerWebExchange` 的属性 Map 中；
5. 返回 `FilteringWebHandler`。

这里有两个关键点。

**第一**：路由匹配的结果，是通过 `ServerWebExchange` 那个“随身文件夹”传给后面的组件的。

**第二**：无论匹配到哪条路由，返回的 Handler 始终是同一个 `FilteringWebHandler` 实例。它不区分路由，只负责执行过滤器链，路由信息由 exchange 提供。

用一个餐厅的比喻：前台（`RoutePredicateHandlerMapping`）确认你找 3 号桌的王先生，然后把“3 号桌王先生”写在纸条上，把你带到**一个统一的服务台**（`FilteringWebHandler`）。服务台是同一个，不专属于任何一桌客人。它拿到什么纸条，就服务哪一桌。

### 3.2 换掉干活的人：Gateway 的 FilteringWebHandler

普通 WebFlux 应用里，目标 Handler 最终指向你的 Controller 方法。

Gateway 换成了它自己的 `FilteringWebHandler`——位于 `org.springframework.cloud.gateway.handler` 包下。

**注意，这里出现了一个和第二章同名但完全不同的类。**

WebFlux 的 `FilteringWebHandler` 在 `org.springframework.web.server.handler` 包下，负责装配并启动 `WebFilter` 链，包在 `DispatcherHandler` 外面。

Gateway 的 `FilteringWebHandler` 在 `org.springframework.cloud.gateway.handler` 包下，负责组装并执行 `GatewayFilter` 链，运行在 `DispatcherHandler` 内部。

同名，不同包，不同职责，不同位置。记住这个区别，它会在后面反复被用到。

它实现了 `WebHandler` 接口，拿到请求后做的事情非常直接：

```java
public class FilteringWebHandler implements WebHandler {

    private final List<GatewayFilter> globalFilters;

    @Override
    public Mono<Void> handle(ServerWebExchange exchange) {
        Route route = exchange.getRequiredAttribute(GATEWAY_ROUTE_ATTR);
        List<GatewayFilter> gatewayFilters = route.getFilters();
        List<GatewayFilter> combined = new ArrayList<>(this.globalFilters);
        combined.addAll(gatewayFilters);
        AnnotationAwareOrderComparator.sort(combined);
        return new DefaultGatewayFilterChain(combined).filter(exchange);
    }
}
```

它的职责是：**把当前路由的 `GatewayFilter` 和所有 `GlobalFilter` 合并、排序，组装成一条链，然后执行。**

### 3.3 过滤器链里到底在发生什么：一次真实的转发

到这里，`FilteringWebHandler` 的职责已经说清楚了——它负责组装并启动过滤器链。但“过滤器链”这四个字仍然很抽象。它到底长什么样？执行起来是什么感觉？

看一个最简单的自定义过滤器：

```java
public class AddHeaderFilter implements GatewayFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 在转发之前，给下游请求加一个 header
        exchange.getRequest().mutate()
                .header("X-Gateway-Trace", "gateway-01")
                .build();

        return chain.filter(exchange);
    }
}
```

这个过滤器做的事情只有一件：在请求被转发到下游之前，往它的 header 里塞一个 `X-Gateway-Trace`。然后调用 `chain.filter(exchange)`，把控制权交给链中的下一个过滤器。

再看一个更完整的、带“前后逻辑”的过滤器：

```java
public class TimingFilter implements GatewayFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        long start = System.currentTimeMillis();

        return chain.filter(exchange)
                .then(Mono.fromRunnable(() -> {
                    long cost = System.currentTimeMillis() - start;
                    log.info("request to {} cost {} ms",
                            exchange.getRequest().getURI(), cost);
                }));
    }
}
```

`chain.filter(exchange)` 之前是 pre 逻辑，之后（通过 `.then()` 串联）是 post 逻辑。

那么，真正把请求转发到下游的那个过滤器是谁？

是 `NettyRoutingFilter`。它也是链中的一员，但它不是简单的“加个 header”或“记个耗时”，它做的事情是**真正发起对下游服务的 HTTP 请求**。它内部使用 Reactor Netty 的 `HttpClient`，把当前 `ServerWebExchange` 里的请求信息转成一个下游请求，发出去，拿到响应，再写回 `ServerWebExchange` 的响应里。

把这三个过滤器按顺序串起来，一次完整转发大概是这样：

```
请求进入 Gateway
  ↓
AddHeaderFilter          → 给下游请求加 header
  ↓  chain.filter(exchange)
TimingFilter             → 记录开始时间
  ↓  chain.filter(exchange)
NettyRoutingFilter       → 真正转发到下游
  ↓  拿到下游响应
TimingFilter             → 计算耗时，打日志（post 逻辑）
  ↓
AddHeaderFilter          → （如果有 post 逻辑）
  ↓
响应返回给客户端
```

每一个过滤器都拿到同一个 `ServerWebExchange`，都可以读写它，都可以在 `chain.filter(exchange)` 前后插入自己的逻辑。**这就是 Gateway 过滤器链的本质：一条由 `FilteringWebHandler` 组装、由 `ServerWebExchange` 贯穿的响应式处理链。**

而这条链的位置，是在 `DispatcherHandler` 内部，在路由匹配完成之后。它和第二章讲的 `WebFilter` 链，是两条完全不同层面上的链。

### 3.4 调用关系：两个 WebHandler，一外一内

把调用链路画出来：

```
DispatcherHandler（调度者）
  ↓ 调用 HandlerMapping.getHandler(exchange)
RoutePredicateHandlerMapping
  ↓ 返回 Gateway 的 FilteringWebHandler
DispatcherHandler
  ↓ 调用 FilteringWebHandler.handle(exchange)
Gateway 的 FilteringWebHandler
  ↓ 从 exchange 取出 Route，组装并执行过滤器链
GatewayFilterChain
  ↓
NettyRoutingFilter → 下游
```

**`DispatcherHandler` 是调度者，它在外面；Gateway 的 `FilteringWebHandler` 是被调用的 Handler，它在里面。**

### 3.5 完整链路：两个 FilteringWebHandler 各就各位

现在把两个 `FilteringWebHandler` 在完整链路中的位置标出来：

```
请求
  ↓
┌─────────────────────────────────────────────────────┐
│  WebFlux 的 FilteringWebHandler                      │
│  （装配并启动 WebFilter 链）                          │
│       ↓                                             │
│  WebFilter 链                                        │
│       ↓                                             │
│  DispatcherHandler                                   │
│       ↓                                             │
│  RoutePredicateHandlerMapping                        │
│       ↓  匹配 Route，存入 ServerWebExchange           │
│  ┌───────────────────────────────────────────────┐  │
│  │  Gateway 的 FilteringWebHandler               │  │
│  │  （组装并执行 GatewayFilter 链）               │  │
│  │       ↓                                       │  │
│  │  GatewayFilterChain                           │  │
│  │       ↓                                       │  │
│  │  NettyRoutingFilter → 下游                     │  │
│  └───────────────────────────────────────────────┘  │
│       ↓                                             │
│  响应回写                                            │
│       ↓                                             │
│  DispatcherHandler                                   │
│       ↓                                             │
│  WebFilter 链（反向）                                 │
│       ↓                                             │
│  WebFlux 的 FilteringWebHandler                      │
└─────────────────────────────────────────────────────┘
```

几个关键事实：

**第一**：WebFlux 的 `FilteringWebHandler` 在**最外层**，装配并启动的是包住 `DispatcherHandler` 的那条 `WebFilter` 链。请求进来先过它，响应回去再过它。

**第二**：Gateway 的 `FilteringWebHandler` 在**最内层**，是 `DispatcherHandler` 在路由匹配之后找到并调用的 Handler，执行的是一条在 `DispatcherHandler` 内部运行的 `GatewayFilter` 链。

**第三**：两条链长得像，位置完全不同。WebFilter 链在外，GatewayFilter 链在内。前者影响路由匹配之前的一切，后者只在路由匹配之后执行。

用一个表把两套过滤器彻底区分开：

| | WebFilter | GlobalFilter / GatewayFilter |
|---|---|---|
| 所属体系 | WebFlux | Gateway |
| 由谁装配和启动 | WebFlux 的 `FilteringWebHandler` | Gateway 的 `FilteringWebHandler` |
| 位置 | `DispatcherHandler` **外面** | `DispatcherHandler` **内部** |
| 能否影响路由匹配 | 能（在匹配之前执行） | 不能（路由已经匹配完了） |
| 典型用途 | 编码、CORS、安全、日志 | 路由转发、限流、重写、熔断 |

WebFilter 链由 WebFlux 的 `FilteringWebHandler` 装配启动，包住 `DispatcherHandler`。GatewayFilter 链由 Gateway 的 `FilteringWebHandler` 组装执行，在 `DispatcherHandler` 内部运行。

**两个同名类，各自驱动一条过滤器链，一个在外，一个在内。**


## 四、回到最初的问题

为什么 Gateway 没有 Controller，却是一个 WebFlux 应用？

因为 **WebFlux 从来就不等于 `@Controller`**。

WebFlux 是一套请求处理骨架：

- `WebFilter` 负责在外层拦截，由 WebFlux 的 `FilteringWebHandler` 装配启动；
- `DispatcherHandler` 负责调度；
- `HandlerMapping` 负责“请求 → 处理者”的映射；
- 目标 `WebHandler` 负责真正执行；
- `ServerWebExchange` 负责传递上下文。

Gateway 做的事情只有一件：**往这套骨架里注册自己的 `HandlerMapping`（`RoutePredicateHandlerMapping`）和目标 `WebHandler`（Gateway 的 `FilteringWebHandler`），把“找 Controller”换成“找 Route”，把“执行业务方法”换成“执行过滤器链并转发”。**

其余的调度、上下文传递、响应式链式执行，全部复用 WebFlux。

所以，Gateway 确实是一个 WebFlux 应用。

**只是它没有 Controller，它有的是路由和过滤器。**


## 附：一张图记住全文

```
┌──────────────────────────────────────────────────────────────┐
│                    WebFlux 骨架                               │
│                                                              │
│   WebFlux 的 FilteringWebHandler（装配并启动 WebFilter 链）     │
│       ↓                                                      │
│   WebFilter 链（包住 DispatcherHandler）                      │
│       ↓                                                      │
│   DispatcherHandler（入口 WebHandler，负责调度）               │
│       ↓                                                      │
│   HandlerMapping（通讯录）                                    │
│       ↓                                                      │
│   目标 WebHandler（被调用者，负责执行）                        │
│       ↓                                                      │
│   ServerWebExchange（随身文件夹，全程传递）                    │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│                    Gateway 替换的部分                          │
│                                                              │
│   HandlerMapping  →  RoutePredicateHandlerMapping            │
│                       ↓  匹配 Route，存入 exchange            │
│   目标 WebHandler  →  Gateway 的 FilteringWebHandler         │
│                       ↓  从 exchange 取出 Route              │
│                   GatewayFilterChain                         │
│                       ↓                                      │
│                   NettyRoutingFilter → 下游                   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**WebFlux 提供骨架，Gateway 提供零件。两个同名的 FilteringWebHandler，一个在最外面管 WebFilter，一个在最里面管 GatewayFilter。这就是它们的关系。**