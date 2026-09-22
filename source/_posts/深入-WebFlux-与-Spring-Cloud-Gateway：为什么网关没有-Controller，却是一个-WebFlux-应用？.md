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

大多数人第一次接触 WebFlux，都是从 `@Controller` 开始的——写一个注解，返回一个 `Mono` 或 `Flux`，请求就处理完了。于是很自然地，`@Controller` 成了 WebFlux 的代名词。

但真相是：**`@Controller` 只是 WebFlux 最上面的一层皮。Gateway 剥掉了这层皮，把下面的骨架完整地拿过来用了。**

这篇文章就沿着“一个请求进来之后到底发生了什么”这条线，把 WebFlux 的骨架拆开，再把 Gateway 套进去，看看它是怎么复用的。


## 一、从你熟悉的地方出发：Servlet 栈

在进入 WebFlux 之前，先回到你熟悉的 Spring MVC。

一个 HTTP 请求打进来，在 Servlet 栈里大概会经历这样的旅程：

```
请求 → Filter → DispatcherServlet → HandlerMapping → Interceptor → Controller → 响应
```

这里面有几个关键角色：

- **Filter**：最外层的关卡，做编码、鉴权、日志这些横切逻辑。
- **DispatcherServlet**：总入口，所有请求都先到它这里，由它负责调度。
- **HandlerMapping**：一张“请求 → 处理者”的通讯录。在 Spring MVC 里，它根据 `@RequestMapping` 找到对应的 Controller 方法。
- **Controller**：真正写业务逻辑的地方。

这个结构之所以好理解，是因为它回答了一个最朴素的问题：**一个请求进来，到底怎么找到该由谁来处理？**

WebFlux 的骨架，本质上就是在回答同一个问题。只是换了一套名字，换了一套编程模型。


## 二、WebFlux 的骨架：五个角色，各司其职

WebFlux 虽然建立在响应式编程之上，但它的请求处理结构和 Servlet 栈在**骨架层面是对称的**。

官方文档里有一句话点明了这一点：WebFlux 和 Spring MVC 一样，围绕前端控制器模式设计，核心的 `DispatcherHandler` 提供共享的请求处理算法，实际工作交给可配置的委托组件。

把这套骨架拆开，是五个角色：

### 2.1 DispatcherHandler：总调度

所有请求的第一站。

它自己不干活，只做一件事：**找到能处理当前请求的 Handler，然后调用它。**

你可以把它理解成 WebFlux 世界的 `DispatcherServlet`。

### 2.2 HandlerMapping：通讯录

`DispatcherHandler` 拿到请求后，第一件事就是问：**这个请求该由谁来处理？**

回答这个问题的就是 `HandlerMapping`。

在普通 WebFlux 应用里，主要的实现是 `RequestMappingHandlerMapping`——它扫描 `@RequestMapping` 注解，建立“URL → Controller 方法”的映射。

### 2.3 WebHandler：真正干活的

通讯录告诉你“找 X”，X 就是一个 `WebHandler`。

它拿到请求，开始真正执行。

在普通 WebFlux 应用里，X 最终是你的 Controller 方法。但 `WebHandler` 是一个更底层的抽象——**任何实现了 `Mono<Void> handle(ServerWebExchange exchange)` 的组件，都可以是一个 WebHandler。**

这句话很重要，后面讲 Gateway 时会用到。

### 2.4 ServerWebExchange：随身文件夹

请求从进来到处理完，会经过很多组件。每个组件都可能需要读请求信息、写响应，或者跟后面的组件传话。

这些信息不能丢，所以 WebFlux 把它们封装进一个对象里，一路传下去。这个对象就是 `ServerWebExchange`。

它装着三样东西：

- **请求**（`ServerHttpRequest`）
- **响应**（`ServerHttpResponse`）
- **一个可变的属性 Map**（`getAttributes()`）

那个属性 Map 是关键。它相当于一次请求生命周期内的共享存储——谁都可以往里放东西，谁都可以从里面取东西。

### 2.5 WebFilter：关卡

`WebFilter` 和 Servlet 的 `Filter` 定位类似，在 `DispatcherHandler` **之前**执行，可以拦截请求和响应。

它的执行顺序由 `@Order` 决定，形成一条链式调用。

### 一张表对齐两套栈

| Servlet 栈 | WebFlux 栈 | 位置 |
|---|---|---|
| Filter | WebFilter | DispatcherHandler 之前 |
| DispatcherServlet | DispatcherHandler | 总入口 |
| HandlerMapping | HandlerMapping | 查表 |
| Interceptor | （无直接对应） | — |
| Controller | WebHandler | 真正执行 |

到这里，WebFlux 的骨架就清楚了。**它不是一个注解框架，而是一套“请求进来 → 查通讯录 → 找到人 → 干活”的流水线。**

`@Controller` 只是这条流水线上“干活”那一环的一种实现方式。你可以换掉它，流水线照样运转。


## 三、Gateway 登场：它到底换了什么？

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

第 4 步很关键。**路由匹配的结果，是通过 `ServerWebExchange` 那个“随身文件夹”传给后面的组件的。**

### 3.2 换掉干活的人：FilteringWebHandler

普通 WebFlux 应用里，Handler 最终指向你的 Controller 方法。

Gateway 换成了 `FilteringWebHandler`。它实现了 `WebHandler` 接口，拿到请求后做的事情非常直接：**把过滤器组装成链，然后执行。**

```java
public class FilteringWebHandler implements WebHandler {
    @Override
    public Mono<Void> handle(ServerWebExchange exchange) {
        return new DefaultGatewayFilterChain(filters).filter(exchange);
    }
}
```

过滤器链的构建逻辑是：把当前路由的 `GatewayFilter` 和所有 `GlobalFilter` 合并，按 `Order` 排序，然后递归执行。

每个过滤器都可以在转发请求**之前**做处理，也可以调用 `chain.filter(exchange)` 之后，在**之后**做处理。这就是 Gateway 过滤器“pre”和“post”逻辑的来源。

### 3.3 完整的请求链路

把 Gateway 的请求处理链路完整画出来：

```
请求
  ↓
DispatcherHandler                    ← WebFlux 的调度器，原封不动
  ↓
RoutePredicateHandlerMapping        ← Gateway 替换的 HandlerMapping
  ↓  匹配 Route，存入 ServerWebExchange
FilteringWebHandler                  ← Gateway 替换的 WebHandler
  ↓
GatewayFilterChain
  ↓
NettyRoutingFilter                   ← 转发到下游
  ↓
下游服务
  ↓
响应回写
```

看这张图，你会发现一个事实：**Gateway 只替换了流水线上的两个工位，其余全部复用。**

`DispatcherHandler` 还是 WebFlux 的，`ServerWebExchange` 还是 WebFlux 的，整条响应式链路还是 WebFlux 的。

用一个比喻来总结：

> **WebFlux 是一套“请求进来 → 查通讯录 → 找到人 → 干活”的骨架。普通应用往里塞 Controller，Gateway 往里塞路由表和过滤器链。骨架是同一个。**

这就是“Gateway 基于 WebFlux”的真正含义。它不是说 Gateway 用了 WebFlux 的注解，而是说 **Gateway 复用了 WebFlux 的整个请求处理骨架**。


## 四、一个必须澄清的混淆：GatewayFilter 不是 WebFilter

理解了上面的结构之后，还有一个坑需要填。

Gateway 里有 `GlobalFilter` 和 `GatewayFilter`，WebFlux 里有 `WebFilter`。名字都带“Filter”，很容易让人以为它们是同一个东西。

**它们不是。**

| | WebFilter | GlobalFilter / GatewayFilter |
|---|---|---|
| 所属体系 | WebFlux | Gateway |
| 执行位置 | `DispatcherHandler` **之前** | `FilteringWebHandler` **内部** |
| 作用范围 | 所有请求 | GlobalFilter 作用于所有路由，GatewayFilter 绑定特定路由 |
| 和路由的关系 | 无关 | 强相关 |

用一张图表示它们的位置差异：

```
请求
  ↓
WebFilter                            ← WebFlux 的关卡，在调度器之前
  ↓
DispatcherHandler
  ↓
RoutePredicateHandlerMapping
  ↓
FilteringWebHandler
  ↓
GatewayFilterChain                   ← Gateway 的过滤器链，在 Handler 内部
  ↓
NettyRoutingFilter → 下游
```

所以 Gateway 的过滤器体系是**嵌套在 WebFlux 骨架内部的**，而不是替代它。两者层次不同，职责不同，不能混为一谈。


## 五、回到最初的问题

为什么 Gateway 没有 Controller，却是一个 WebFlux 应用？

因为 **WebFlux 从来就不等于 `@Controller`**。

WebFlux 是一套请求处理骨架：

- `DispatcherHandler` 负责调度；
- `HandlerMapping` 负责“请求 → 处理者”的映射；
- `WebHandler` 负责真正执行；
- `ServerWebExchange` 负责传递上下文；
- `WebFilter` 负责在外层拦截。

Gateway 做的事情只有一件：**往这套骨架里注册自己的 `HandlerMapping` 和 `WebHandler`，把“找 Controller”换成“找 Route”，把“执行业务方法”换成“执行过滤器链并转发”。**

其余的调度、上下文传递、响应式链式执行，全部复用 WebFlux。

所以，Gateway 确实是一个 WebFlux 应用。

**只是它没有 Controller，它有的是路由和过滤器。**


## 附：一张图记住全文

```
┌─────────────────────────────────────────────────────┐
│                  WebFlux 骨架                        │
│                                                     │
│   WebFilter（关卡）                                  │
│       ↓                                             │
│   DispatcherHandler（总调度）                        │
│       ↓                                             │
│   HandlerMapping（通讯录）                           │
│       ↓                                             │
│   WebHandler（干活的）                               │
│       ↓                                             │
│   ServerWebExchange（随身文件夹，全程传递）           │
│                                                     │
├─────────────────────────────────────────────────────┤
│                  Gateway 替换的部分                   │
│                                                     │
│   HandlerMapping  →  RoutePredicateHandlerMapping   │
│   WebHandler      →  FilteringWebHandler            │
│                       ↓                             │
│                   GatewayFilterChain                │
│                       ↓                             │
│                   NettyRoutingFilter → 下游          │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**WebFlux 提供骨架，Gateway 提供零件。这就是它们的关系。**