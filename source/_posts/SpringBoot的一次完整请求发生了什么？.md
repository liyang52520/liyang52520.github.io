---
title: SpringBoot 的一次完整请求发生了什么？
date: 2026-08-26 13:51:15
tags:
  - spring
  - tomcat
categories:
  - develop
---

> 这是一个三篇的系列。本篇是导读：把整条链路先铺在图上，再按节给出三篇的入口。

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

> 这个系列要回答的问题：
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

文中源码以 **Spring Framework 6.1.8 + Spring Boot 3.3.x + Tomcat 10.1.x + Java 17** 为基准，全部对照本地 sources jar 逐行核对。与 Spring 5.3 / Tomcat 9 行为不同的地方会单独标出——**这条链路上有好几处"广为流传但已经过时"的说法，这个系列会逐个纠正。**

如果你还没读过 [Spring 接入点：跟着容器过一生](/2026/09/11/Spring-接入点：跟着容器过一生/)，建议先扫一眼：那里讲了容器启动期各类 `PostProcessor` 的执行位置，而第一篇第四节的"启动期装了什么"，正是那些机制的直接产物。

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

### 1.1 整个系列的主线：三次跨界

这张图里藏着一条主线，比"一层一层往下走"更有解释力：**请求每翻过一道边界，交出去的就不只是数据，还有执行它的线程。**

| 边界 | 从 | 到 | 谁在执行 |
|:---|:---|:---|:---|
| 第一次 | 内核协议栈 | Tomcat 用户态 | 内核软中断 → Acceptor 线程 |
| 第二次 | Tomcat 连接器 | Servlet 容器 | Poller 线程 → **Worker 线程** |
| 第三次 | Servlet 容器 | Spring MVC | 同一个 Worker 线程（同步场景） |

前两次跨界，线程都换了人。第三次没有——**同步请求下，从 Filter 到 Controller 到响应写回，全都跑在同一个 Worker 线程上**。这一点极其重要，因为它同时解释了两件事：

- 为什么 Controller 里能随手用 `ThreadLocal`（比如 `RequestContextHolder`、MDC 日志上下文）——因为整条链没换过线程；
- 为什么 Controller 里一旦阻塞（慢查询、外部 HTTP 调用），被卡住的是**整个 Worker 线程**，而不是某段代码。

> 异步请求会打破这条规律：Worker 线程会在 Controller 返回前就退出，业务逻辑跑到另一个线程上，之后再被"派回来"。这是第三篇第十二节的内容，也是 Spring MVC 里最容易被调试器骗到的地方。

### 1.2 每一层分别对应哪些错误

先给一张"反向索引"表，后面每一节讲完，都可以回到这张表上对一次。

| 现象 | 大概率所在层 | 对应节 |
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
| 响应头正确但 body 为空 | Filter 包装了响应却忘了写回 | 十三 |

## 这个系列怎么读

三篇的节号是连续的，没有重新编号：**一~五在第一篇，六~九在第二篇，十~十五和结语在第三篇**。这样做不是为了省事——这条链路本身就是连着的，跨篇引用"第八节"时，你翻回第二篇就能对上，不用在两套编号之间换算。

| 篇目 | 节 | 这一篇回答什么 |
|:---|:---|:---|
| [第一篇：请求是怎么到 Tomcat 的](/2026/08/26/SpringBoot请求链路1：请求是怎么到-Tomcat-的/) | 二~五 | 内核的两个队列、Tomcat 的 Acceptor / Poller / Worker 三拨线程、启动期装好的组件、`FilterChain` 的真实结构 |
| [第二篇：从 DispatcherServlet 到方法调用](/2026/08/26/SpringBoot请求链路2：从-DispatcherServlet-到方法调用/) | 六~九 | `doDispatch()` 逐行，路由表的建立与匹配、27 个参数解析器的顺序、拦截器的顺序与中断语义 |
| [第三篇：方法执行、异步与响应写回](/2026/08/26/SpringBoot请求链路3：从方法执行到响应写完/) | 十~十五 | 反射调用与异常解析、返回值处理器与内容协商、异步的三根线程、`OutputBuffer` 到 Socket，外加速查表与 Debug 指南 |

三个常用入口：

- **只想给一个报错定位**：直接去[第三篇](/2026/08/26/SpringBoot请求链路3：从方法执行到响应写完/)的第十五节，那张"错误码 → 哪一层"的表基本能一步到位。
- **想弄清线程模型**：[第一篇](/2026/08/26/SpringBoot请求链路1：请求是怎么到-Tomcat-的/)的第三、四节讲线程从哪来、什么时候建好，[第三篇](/2026/08/26/SpringBoot请求链路3：从方法执行到响应写完/)的第十二节讲异步为什么变成三根线程，对照着看最快。
- **想弄清"参数为什么取不到"**：[第二篇](/2026/08/26/SpringBoot请求链路2：从-DispatcherServlet-到方法调用/)的第八节。这是整条链路上最不直观的一环，也是引子里那个 400 的答案。

> 三篇各自独立成文，但都建立在这张地图上。如果某一段读起来突兀，多半是它依赖了前一节；顺着读一遍，再回头查表。

