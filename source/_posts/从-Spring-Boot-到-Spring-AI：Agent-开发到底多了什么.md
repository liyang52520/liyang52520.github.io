---
title: 从 Spring Boot 到 Spring AI：Agent 开发到底多了什么
date: 2026-10-15 16:10:00
tags:
  - spring ai
  - agent
  - mcp
  - spring
categories:
  - develop
---

## 引子：一个只写过 Controller 的人，第一次听到 Agent

你已经能把 Spring Boot 用得很熟：`@RestController` 接请求，`@Service` 装业务，`@Transactional` 管事务，加一个 starter、写几行 `application.yml`，一个新功能就能上线。

现在需求变了。要做"一个能自己查订单、查库存、必要时还能给客户发邮件的智能助手"。

然后你打开搜索框，扑面而来的是这些词：**Agent、ReAct、MCP、RAG、Function Calling、Tool、Memory、LangChain**。每一个词都指向一套你没见过的东西。

你的第一反应是对的：调用大模型，不就是 POST 一个 JSON 到某个 HTTP 接口吗？`RestClient` 能不能干？能，二十行代码就能跑通。

但这个反应只对了一半。**调用模型确实是发 HTTP 请求，可让模型"干活"是一整套新的运行时结构。** 那些陌生名词不是营销词，它们各自对应一个真实的工程问题：模型没有记忆怎么办、模型不能执行代码怎么办、工具散落在十个应用里怎么复用、多步任务怎么收敛。

这篇文章沿着这条线走：先把 Agent 开发的三块地基——**工具调用、ReAct 循环、MCP**——用你熟悉的东西讲清楚，再看 Spring AI 如何用 Spring 的方式把它们包装起来。全程拿 Spring MVC、`RestClient`、Filter 链、Starter 自动配置当参照物。

文中代码以 **Spring AI 1.1.x + Spring Boot 3.5.x** 为基准，因为这是 Boot 3.x 项目当下能直接用的版本线。2.0 的 API 改动静不小，集中在第六节说明。

> 这篇文章要回答的问题：
>
> - 我直接用 `RestClient` 调大模型，到底差在哪？
> - 模型能执行我写的 Java 方法吗？一次工具调用的往返是怎么走的？
> - ReAct 是什么？为什么说 Agent 的本质是一个循环？这个循环谁在驱动？
> - MCP 解决什么问题？为什么被称作"工具界的 USB 接口"？
> - Spring AI 替我做了什么？它和 `DispatcherServlet`、Filter、Starter 怎么对应？
> - 1.0、1.1、2.0 三条版本线，我该选哪个？

---

## 一、先把大模型当成一个"远程服务"

### 1.1 第一反应：这不就是个 HTTP 接口吗

OpenAI 的对话接口长这样：向 `/v1/chat/completions` POST 一个 JSON，请求体里是 `messages` 数组，响应里拿 `choices[0].message.content`。

用 Spring 的 `RestClient` 写出来是这样：

```java
public String ask(String question) {
    ChatCompletionRequest request = new ChatCompletionRequest(
            "gpt-4o-mini",
            List.of(Map.of("role", "user", "content", question)));

    ChatCompletionResponse response = restClient.post()
            .uri("/v1/chat/completions")
            .header("Authorization", "Bearer " + apiKey)
            .body(request)
            .retrieve()
            .body(ChatCompletionResponse.class);

    return response.choices().get(0).message().content();
}
```

这段代码能跑。所以你可能会想：那这个框架还有什么用？

问题在于，这段代码只是"能跑"。把它放进真实业务里，你很快会连着撞上三堵墙。

### 1.2 它和普通的 REST 接口有三点不一样

| 维度 | 普通的 REST 接口 | 大模型对话接口 |
|------|----------------|--------------|
| 状态 | 通常无状态，服务端靠 session/token 识别上下文 | **完全无状态**，模型不记得上一句说了什么 |
| 出入参 | 结构化（JSON/XML，有 schema） | **入参和出参都是 token 序列**，出参是自然语言 |
| 计费 | 按调用次数或带宽 | **按 token 计费**，且上下文越长越贵 |

第一点最容易低估。你调一个订单接口，第二次调用不需要把第一次的响应再发一遍；但调模型，**想让模型"记得"前面聊过什么，就必须把前面的消息全部重新发一遍**。所谓"对话记忆"，本质是应用侧拼装请求体。

第二点决定了后面的所有麻烦。订单接口返回 `{"status": "PAID"}`，你能直接反序列化成枚举；模型返回的是"这个订单已经付款了，预计明天发货"，**你的代码没法用 `if` 判断它到底什么意思**。

第三点是最现实的约束：每多一轮对话，请求体就长一截，费用和延迟都会涨。而这个约束在 Agent 场景里会被放大很多倍——因为一轮 Agent 任务可能包含十几次模型调用。

### 1.3 这三点差异，各自长出了一块框架能力

把上面三点翻译成"框架该干什么"：

- **因为无状态**，所以需要**会话记忆**（Spring AI 的 `ChatMemory`）：按会话 ID 把历史消息存起来，每次调用前自动拼回去。
- **因为出参是自然语言**，所以需要**结构化输出**：让模型返回 JSON，再反序列化成 Java 对象——`ChatClient.call().entity(Order.class)` 干的就是这件事。方向正好和 `@RequestBody` 相反：`@RequestBody` 是把请求变成对象，`entity()` 是把模型的自然语言变成对象。
- **因为按 token 计费**，所以需要**上下文窗口管理**：控制历史消息保留多少条、什么时候截断（`MessageWindowChatMemory` 的默认窗口是 20 条）。

到这里，你手上还是一个"能聊天、能记住上下文"的服务。它离"能干活"还差一步：**它只能说话，不能动。**

---

## 二、工具调用：模型长出了手，但手长在你这边

### 2.1 一个必须先刻进脑子的前提：模型不执行任何东西

这是整个 Agent 开发里最容易被误解的一点，也是理解后面所有内容的分水岭。

**模型只会生成 token。它不能读你的数据库，不能调你的接口，不能执行你的 Java 方法。** 它能做的只有一件事：输出一段文本——或者，如果被要求，输出一段结构化的 JSON。

所谓"模型调用了工具"，完整的事实是：

1. 你先告诉模型"我这儿有个工具叫 `getOrderStatus`，接受一个 `orderId` 参数，作用是查订单状态"；
2. 模型判断"用户这个问题需要查订单"，于是**输出一段 JSON，内容是"请调用 `getOrderStatus`，参数是 `orderId=10086`"**；
3. **你的 Java 进程**收到这段 JSON，找到对应的方法，真正执行它；
4. 你把执行结果再发回给模型；
5. 模型基于结果，用自然语言回答用户。

第 3 步是重点：**执行发生在你的进程里，不在模型那边。** 模型只是"说出了它想调什么"，真正动手的是你的代码。

这也解释了一个常见现象：模型有时会"调用一个不存在的工具"。因为它只是在生成文本，它完全可以生成一个你从没定义过的工具名。**校验和分发必须由应用侧负责。**

### 2.2 一次工具调用的完整往返

把上面的流程落到 HTTP 层面，一次工具调用其实是**两轮请求中间夹一次本地方法调用**。

第一轮，应用把工具清单随消息一起发出去：

```json
{
  "model": "gpt-4o-mini",
  "messages": [{ "role": "user", "content": "订单 10086 发货了吗？" }],
  "tools": [{
    "type": "function",
    "function": {
      "name": "getOrderStatus",
      "description": "根据订单号查询订单当前状态与预计发货时间",
      "parameters": {
        "type": "object",
        "properties": {
          "orderId": { "type": "string", "description": "订单号，例如 10086" }
        },
        "required": ["orderId"]
      }
    }
  }]
}
```

模型这一轮的响应里没有自然语言，只有一个工具调用请求：

```json
{
  "choices": [{
    "message": {
      "role": "assistant",
      "tool_calls": [{
        "id": "call_abc123",
        "type": "function",
        "function": { "name": "getOrderStatus", "arguments": "{\"orderId\":\"10086\"}" }
      }]
    }
  }]
}
```

**注意这里：模型把参数写在了 `arguments` 字段里，但它并没有执行任何东西。**

接下来是应用侧的动作：解析 `arguments`、反序列化成参数、调用真正的 Java 方法、拿到返回值。然后把结果拼成一条 `role` 为 `tool` 的消息，**连着之前的所有消息再加一次请求**：

```json
{
  "model": "gpt-4o-mini",          // 与第一轮相同
  "tools": [ /* ... 与第一轮相同，此处省略 */ ],
  "messages": [
    { "role": "user", "content": "订单 10086 发货了吗？" },
    { "role": "assistant", "tool_calls": [ /* 上一轮模型的输出，原样回传 */ ] },
    { "role": "tool", "tool_call_id": "call_abc123", "content": "{\"status\":\"PAID\",\"shipAt\":\"2026-10-16\"}" }
  ]
}
```

第二轮，模型才输出给用户看的话："订单 10086 已付款，预计 10 月 16 日发货。"

还有一个细节在手写循环时最容易漏：**一次响应里的 `tool_calls` 可能不止一个。** 模型完全可能在同一轮里要求同时查订单、查库存、查物流（这就是并行工具调用）。你可以并行执行它们，也可以顺序执行，但**必须为每一个 `id` 都回填一条 `role=tool` 的消息**——少回一条，下一轮请求就会因为"有工具调用没有结果"而被拒绝。这是自己写循环时最常见的低级错误。

把这套流程记住，因为它就是后面所有"Agent 框架"的内核。**框架做的事，无非是把这个往返自动化、把 JSON 拼装藏起来、把这个循环接进你自己的运行时。**

### 2.3 换个角度看：工具就是"让模型来调用的 Controller"

如果你是 Spring MVC 开发者，这套东西其实已经有现成的心理模型——它就是一次 HTTP 请求，只不过**路由的决策者从 URL 变成了模型的自然语言理解**。

| Spring MVC | 工具调用 | 说明 |
|-----------|---------|------|
| `@GetMapping("/orders/{id}")` | 工具名 `getOrderStatus` | 都是"调用入口的标识" |
| 接口文档 / OpenAPI | `description` + `parameters` JSON Schema | 都是给调用方看的契约 |
| `HandlerMapping` 按 URL 找 handler | 模型输出 `tool_calls` 后由框架分发到方法 | **决策者从 URL 换成了模型** |
| `@RequestParam` + 类型转换 | JSON Schema + 反序列化 | 参数绑定 |
| 返回值序列化成 JSON | 返回值转成字符串回填 `role=tool` 消息 | 结果回传 |

这张表里最关键的是第三行。在 MVC 里，路由是确定性的：URL 是 `/orders/10086`，就一定进那个方法。在工具调用里，**是模型读了你的描述之后自己决定调不调、调哪个**。

所以有一条经验值得单独记住——**工具描述（`description`）的质量，直接决定调用准确率**。它不是注释，它是接口文档，而且是"唯一的调用方会逐字读它然后据此做决定"的接口文档。Anthropic 在 [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) 的附录里把这叫做"给工具做 prompt engineering"，建议写得像"给团队里的初级工程师写一段好 docstring"，并给出一个真实例子：他们在做代码 Agent 时发现模型总在相对路径上出错，于是把工具参数改成**只接受绝对路径**，模型立刻就用对了。这就是"把错误变得难以发生"（poka-yoke）。

把工具比作 Controller、把描述比作接口文档，是一个好用的入门类比，不过要说清楚这是**类比**，官方并没有这样表述过。官方更强调的一层含义是"契约"：Anthropic 在 [Writing effective tools for agents](https://www.anthropic.com/engineering/writing-tools-for-agents) 里把工具称为"**a contract between deterministic systems and non-deterministic agents**"，MCP 的文档则说"**Tools are schema-defined interfaces that LLMs can invoke**"。这两种说法都点出了关键：**契约的另一端是一个不确定的调用方**。在 MVC 里，URL 是 `/orders/10086` 就一定进那个方法；在这里，进不进来、带什么参数进来，都由模型临场判断。

由此还有一条很容易踩的边界：**不要把现有接口一比一包成工具。** 上面那篇文章明确提醒，不要按"给其他开发者写函数和 API 的方式"来写工具，并把"仅仅包装了现有软件功能或 API 端点（tools that merely wrap existing software functionality or API endpoints）"点名为常见错误。原因回想一下机制就明白了：工具是给模型"读"的，不是给编译器"编译"的。一个接收十几个必填参数、内部还隐含调用顺序的接口，人看着都头疼，模型只会更容易调错。

### 2.4 Spring AI 的做法：`@Tool` + 注册

理解了机制，Spring AI 的注解就非常好懂了。它做的事就是：**把标注了 `@Tool` 的 Java 方法，翻译成上文的 `tools` JSON Schema**。

```java
@Component
public class OrderTools {

    private final OrderService orderService;

    public OrderTools(OrderService orderService) {
        this.orderService = orderService;
    }

    @Tool(description = "根据订单号查询订单当前状态与预计发货时间。当用户询问某笔订单的发货进度时调用。")
    public String getOrderStatus(
            @ToolParam(description = "订单号，纯数字，例如 10086") String orderId) {
        return orderService.findStatus(orderId).toJson();
    }

    @Tool(description = "给指定客户发送邮件通知")
    public String sendMail(
            @ToolParam(description = "客户邮箱") String to,
            @ToolParam(description = "邮件正文", required = false) String body) {
        return mailService.send(to, body);
    }
}
```

属性不多，但都有明确用途：

| 元素 | 作用 |
|------|------|
| `@Tool(name = ...)` | 覆盖工具名，默认取方法名 |
| `@Tool(description = ...)` | 工具描述，**就是写给模型看的接口文档** |
| `@Tool(returnDirect = true)` | 工具结果直接返回给调用方，不再回传给模型 |
| `@ToolParam(description = ...)` | 单个参数的说明 |
| `@ToolParam(required = false)` | 声明参数可选，默认全部必填 |

注册给 `ChatClient` 只需要一行：

```java
ChatClient chatClient = ChatClient.builder(chatModel)
        .defaultTools(new OrderTools(orderService))
        .build();

String answer = chatClient.prompt()
        .user("订单 10086 发货了吗？")
        .call()
        .content();
```

这里有一个很容易踩的坑：**1.1.x 里 `.defaultTools(...)` 和 `.tools(...)` 只接受带 `@Tool` 方法的对象实例。** 这两个方法最终都会走到 `MethodToolCallbackProvider`，而它对每一个入参都断言"必须能找到 `@Tool` 方法"，否则直接抛 `IllegalStateException`——异常信息把答案都写好了：

```text
No @Tool annotated methods found in ... Did you mean to pass a ToolCallback or
ToolCallbackProvider? If so, you have to use .toolCallbacks() instead of .tool()
```

所以要传 `ToolCallback` 或 `ToolCallbackProvider`，得用 `.defaultToolCallbacks(...)` / `.toolCallbacks(...)`。**从 2.0 起 `.tools()` 才支持异构参数**，POJO、`ToolCallback`、`ToolCallbackProvider` 可以混着给。

如果你只想在单次调用里临时加一个工具，用 `prompt().tools(...)`。它是**追加**语义，不会替换掉默认工具：源码里请求级工具会与默认工具合并（走的是 `mergeToolCallbacks`）。这里要提一句——**1.1 的官方文档在这一点上写的是"runtime tools will completely override the default tools"，与源码行为不符，2.0 文档才改成 "appends"**。以源码为准，但读文档时别被绕进去。

**最容易看漏的一点**：这里没有任何一行代码在"写循环"。可模型要先调工具、再回答，这个往返是谁完成的？

答案是：在 1.x 里，**它藏在 `ChatModel` 内部**。我们下一节就把这件事挖出来。

---

## 三、ReAct：把一次调用变成一场循环

### 3.1 从一篇论文说起

2022 年 10 月，Shunyu Yao 等人在 arXiv 上挂出 [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629)（arXiv:2210.03629；2023 年 3 月的 v3 是 ICLR camera-ready 版本）。

论文要解决的问题很具体：当时的研究把大模型的两种能力**分开**看待——"推理"（比如思维链 CoT）和"行动"（比如生成动作计划）。前者让模型想得好，但想出来的东西可能不符合事实；后者让模型能接触外部环境，但缺少规划和纠错。

ReAct 的贡献是把两者**交错**起来，让模型一边想一边做。落到交互格式上，就是三段式循环：

```text
Thought 1:  我需要先查这个订单的状态，才能回答发货问题。
Action 1:   getOrderStatus[10086]
Observation 1: {"status":"PAID","shipAt":"2026-10-16"}
Thought 2:  状态是已付款，预计明天发货，可以回答了。
Action 2:   finish[订单 10086 已付款，预计 10 月 16 日发货。]
```

- **Thought**：模型的推理，写在明面上，让计划可追踪；
- **Action**：模型决定要做的动作；
- **Observation**：环境（也就是你的应用）返回的结果，喂回去进入下一轮 Thought。

有一个细节容易被忽略：**Thought 不产生 Observation。** 论文的原话是，思考属于语言空间的动作，"does not affect the external environment, thus leading to no observation feedback"。也就是说，真正让循环往前推进的是 Action——Thought 只影响模型接下来选哪个 Action。这也顺带解释了纯 CoT 为什么会幻觉：它只有 Thought、没有 Observation，模型没有机会用外部事实验证自己的推理。

论文的实验说明了两件事。在问答任务上，ReAct 通过与 Wikipedia API 交互，**缓解了纯 CoT 容易出现的幻觉和错误传播**——论文对失败案例做了人工标注，纯 CoT 的失败中有 56% 源于幻觉，ReAct 是 0；代价是 ReAct 的推理错误比例更高（47% 对 16%）。**外部的 Observation 换来了事实性，但不保证推理质量**，这个权衡后面还会反复遇到。在 ALFWorld 和 WebShop 两个交互决策任务上，ReAct 比模仿学习和强化学习基线分别高出 34% 和 10% 的成功率。

这里最重要的不是数字，而是那个结构：**Thought → Action → Observation，然后回到 Thought。这是一个循环。**

### 3.2 文本式 ReAct 与原生工具调用

ReAct 发表时，"工具调用"还不是模型 API 的一等公民，所以论文里的 `Action 1: getOrderStatus[10086]` 是**模型用文本写出来的**，应用侧再用正则去解析。

今天我们用的 function calling 是厂商把它**结构化**了：`Action` 变成响应里的 `tool_calls` 字段，参数变成 `arguments` 里的 JSON。区别只在"表达形式"，循环的骨架一模一样。

| | 文本式 ReAct | 原生工具调用（function calling） |
|---|---|---|
| 模型如何表达动作 | 在文本里写 `Action: Search[...]` | 在 `tool_calls` 字段里给结构化 JSON |
| 应用如何解析 | 正则/模板匹配，容易解析失败 | 直接反序列化，稳定 |
| 工具定义 | 写在 prompt 里 | 单独 `tools` 字段，带 JSON Schema |
| 参数校验 | 靠模型自觉 | 有 schema，可校验 |
| 循环结构 | Thought→Action→Observation | 完全一样 |

也就是说，**function calling 不是 ReAct 的替代品，它是 ReAct 的"工程化实现"**。你在 2.2 节看到的两轮往返，就是 ReAct 循环跑了一圈。

### 3.3 关键问题：这个循环由谁驱动

现在回到 2.4 节留下的问题。模型每次只完成"一轮"——它输出 `tool_calls` 就结束了，它不会自己再发一轮请求。**所以循环必须由应用侧驱动。**

用最朴素的方式写，它就是一个 `while`：

```java
Prompt prompt = new Prompt(List.of(new UserMessage(question)), options);

ChatResponse response = chatModel.call(prompt);
while (response.hasToolCalls()) {
    // 执行模型要求的所有工具，把结果拼进对话历史
    ToolExecutionResult result = toolCallingManager.executeToolCalls(prompt, response);
    prompt = new Prompt(result.conversationHistory(), options);
    response = chatModel.call(prompt);   // 带着工具结果再问一次
}
return response.getResult().getOutput().getText();
```

**这个 `while` 就是 Agent 的全部秘密。** 这不是我的概括——Anthropic 在 [Writing effective tools for agents](https://www.anthropic.com/engineering/writing-tools-for-agents) 里把 agent loop 直接描述成"**`while`-loops wrapping alternating LLM API and tool calls**"；Spring AI 官方文档的表述则是"ToolCallingManager executes them; **the loop continues until the model produces a response without tool calls**"。剩下的都是工程细节：循环怎么终止、历史怎么裁剪、工具怎么分发、出错怎么重试。

顺带提醒一句：在 Spring AI 1.x 里想自己跑这个循环，得先把模型内部的工具执行关掉，否则它会在 `ChatModel` 内部先跑一遍，你的循环拿到的是已经跑完的结果。这件事谁来做，正是下一段的主角。

而 Spring AI 做的选择很有意思，也值得单独讲——**它把这个循环的位置挪过一次，而且这次挪动正好解释了"1.1 和 2.0 有什么区别"。**

**在 Spring AI 1.0/1.1 里，循环藏在 `ChatModel` 内部。** 以 `OpenAiChatModel` 为例，它的 `internalCall` 方法在拿到响应后会判断要不要执行工具，然后**递归调用自己**：

```java
if (this.toolExecutionEligibilityPredicate.isToolExecutionRequired(prompt.getOptions(), response)) {
    var toolExecutionResult = this.toolCallingManager.executeToolCalls(prompt, response);
    if (toolExecutionResult.returnDirect()) {
        return ChatResponse.builder()        // 工具结果直接返回给调用方
                .from(response)
                .generations(ToolExecutionResult.buildGenerations(toolExecutionResult))
                .build();
    }
    // 否则把工具结果发回模型，递归——这就是循环
    return this.internalCall(new Prompt(toolExecutionResult.conversationHistory(), prompt.getOptions()),
            response);
}
```

好处是你什么都不用写，`.defaultTools(...)` 之后循环自动发生。坏处也很明显：**循环在 `ChatModel` 内部，你的代码插不进去。** 想在每一轮循环前后做点事——记日志、加缓存、拦一道安全校验、把中间过程推给前端——没有干净的切入点。

**1.1 引入了一个新组件把循环搬到链上**：`ToolCallAdvisor`。它的类注释写得很直白：

> Recursive Advisor that **disables the internal tool execution flow** and instead implements the tool calling loop **as part of the advisor chain**. This enables **intercepting the tool calling loop** by the rest of the advisors next in the chain.

看它怎么做到的：先把选项复制一份，**把模型内部的工具执行关掉**，然后自己用 `do/while` 驱动循环：

```java
// Overwrite the ToolCallingChatOptions to disable internal tool execution.
var optionsCopy = (ToolCallingChatOptions) chatClientRequest.prompt().getOptions().copy();
optionsCopy.setInternalToolExecutionEnabled(false);

boolean isToolCall = false;
do {
    // ... 调用链上的下一个 advisor（最终打到模型）
    chatClientResponse = callAdvisorChain.copy(this).nextCall(processedChatClientRequest);
    // ... 判断响应里还有没有 tool calls
} while (isToolCall);   // 循环直到没有工具调用
```

用起来就是把它加进 advisor 链：

```java
@Bean
ChatClient chatClient(ChatClient.Builder builder, ToolCallingManager toolCallingManager) {
    return builder
            .defaultToolCallbacks(MethodToolCallbackProvider.builder()
                    .toolObjects(new OrderTools(orderService))
                    .build())
            .defaultAdvisors(ToolCallAdvisor.builder()
                    .toolCallingManager(toolCallingManager)
                    .build())
            .build();
}
```

**2.0 把这件事做彻底了**：`ChatModel` 内部的循环被移除，`ChatModel.call(prompt)` 只返回模型的原始响应；循环完全由 advisor 承担，并且 `ChatClient` **默认就会自动注册**它。同时 `ToolCallAdvisor` 改名为 `ToolCallingAdvisor`（旧的作为 `@Deprecated(forRemoval = true)` 的兼容类保留）。

为什么这个设计值得单独说？因为它正好呼应你熟悉的一件事：**把循环放进链式结构，就等于把它变成可组合的。** 就像 Servlet Filter 链——每个 Filter 都能决定放行、短路、或者在前后加逻辑，而"接下来会发生什么"由链上的下一个节点负责。循环成为链上的一环之后，"记忆放在循环内还是循环外""日志记在每轮还是整体"这类问题，就变成了一个 order 问题。这是后文讲解 Advisor 的基础。

### 3.4 Agent 与 Workflow：不是所有场景都要循环

既然循环这么强，是不是所有大模型应用都该做成 Agent？

Anthropic 在 [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) 里给了明确的区分：

> **Workflows** are systems where LLMs and tools are orchestrated through **predefined code paths**.
>
> **Agents**, on the other hand, are systems where LLMs **dynamically direct their own processes and tool usage**, maintaining control over how they accomplish tasks.

用 Java 的话说：Workflow 是**你写死的编排**（`if`、顺序调用、并行调用），Agent 是**把编排权交给模型**。

Anthropic 归纳的几种 Workflow 模式——prompt chaining（串联）、routing（分流）、parallelization（并行/投票）、orchestrator-workers（主从）、evaluator-optimizer（评估-优化）——其实都是你写惯了的编排代码，只不过节点换成了模型调用。

同一篇文章还给了一段很实在的提醒：

> Agentic systems often **trade latency and cost for better task performance**... find the simplest solution possible, and only increasing complexity when needed... For many applications, optimizing single LLM calls with retrieval and in-context examples is usually enough.

翻译成工程判断：**如果一个任务的路数是确定的（先查订单、再查物流、然后发邮件），写死编排更便宜、更快、更可靠。** 只有当步骤数无法预先确定、必须靠模型边走边判断时，才值得上循环。这也是"上 Agent"最常见的过度设计——**能写成 Workflow 的，就别写成 Agent。**

### 3.5 循环跑通之后，新的麻烦出现了

假设你现在有一批好用的工具：查订单、查物流、发邮件、查知识库。它们分别写在不同的服务里。

很快你会发现两件事：

1. **每个想用大模型的应用，都要把这批工具重新实现一遍**，或者至少重新写一遍工具描述。三个应用乘十套工具，就是三十份胶水代码。
2. **生态里的工具是别人的**：GitHub、文件系统、数据库、Slack……你不可能为每一个都写一遍适配。而且它们是 Python 写的、Node 写的，你在 Java 进程里怎么接？

这就是 MCP 要解决的问题。

---

## 四、MCP：工具的"USB 接口"

### 4.1 把工具做成"可插拔的"

MCP（Model Context Protocol）由 Anthropic 在 2024 年 11 月提出，现在是开放标准，规范站点在 [modelcontextprotocol.io](https://modelcontextprotocol.io/)。

官方文档里有一句非常好用的类比：

> Think of MCP like a **USB-C port for AI applications**. Just as USB-C provides a standardized way to connect electronic devices, MCP provides a standardized way to connect AI applications to external systems.

规范正文则给了另一个更有工程味的类比：

> MCP takes some inspiration from the **Language Server Protocol**... In a similar way, MCP standardizes how to integrate additional context and tools into the ecosystem of AI applications.

LSP 这个类比对 Java 开发者尤其贴切：LSP 让"编辑器"和"语言支持"解耦，MCP 让"AI 应用"和"工具提供方"解耦。Anthropic 发布时对这个痛点的表述是：

> Every new data source requires its own custom implementation, making truly connected systems difficult to scale.

社区常把它概括成"M × N 问题"（M 个模型/应用 × N 个工具）。**这个说法是社区通俗版，官方原文里并没有出现"M×N"的字样**，但它准确表达了动机：与其维持 M×N 份连接器，不如双方都对着一个协议写，变成 M+N。

### 4.2 三个角色：Host、Client、Server

MCP 的架构里有三个角色，官方的定义是：

| 角色 | 官方原文 | 通俗解释 |
|------|---------|---------|
| **Host** | "The AI application that coordinates and manages one or multiple MCP clients" | **你的 Spring Boot 应用**（或者 Claude Desktop、VS Code） |
| **Client** | "A component that maintains a connection to an MCP server and obtains context from an MCP server for the MCP host to use" | Host 内部为**每个** server 创建的连接组件 |
| **Server** | "Servers provide specialized context and capabilities... Expose resources, tools and prompts via MCP primitives" | 工具/数据的提供方，可以是本地进程，也可以是远程服务 |

（Host 与 Client 两条取自官方文档的架构页，Server 一条取自规范正文——官网的文档区和规范区会有措辞差异，引用时值得留意。）

数量关系需要记准，因为它决定了你的部署形态：

- **Host : Client = 1 : N** —— 原文："each host can run multiple client instances"
- **Client : Server = 1 : 1** —— 原文："each client having a 1:1 relationship with a particular server"
- **Server : Client = 1 : N** —— 一个 server 可以同时服务很多 client

规范里还有一条设计原则值得做安全评审时引用：**"Servers should not be able to read the whole conversation, nor 'see into' other servers."** 也就是说，MCP 在架构上就限制了一个 server 能看到的上下文范围。

### 4.3 三个原语：Tools、Resources、Prompts

MCP server 能暴露三类东西。规范正文给了一张对照表，**这张表是理解 MCP 的关键**：

| 原语 | 控制权 | 规范描述 | 典型 UI |
|------|-------|---------|--------|
| **Prompts** | **User-controlled** | "Interactive templates invoked by user choice" | 斜杠命令、菜单 |
| **Resources** | **Application-controlled** | "Contextual data attached and managed by the client" | 文件内容、git 历史 |
| **Tools** | **Model-controlled** | "Functions exposed to the LLM to take actions" | API 调用、写文件 |

三者的差别，用一句话概括就是**谁决定它被使用**：

- **Tools 由模型决定**——模型自己判断该不该调、调哪个。这是唯一会"产生副作用"的那类，也是 Agent 的能力来源。
- **Resources 由应用决定**——你的代码决定要不要把这份数据塞进上下文。它是**只读**的上下文来源。
- **Prompts 由用户决定**——用户在界面上主动选一个模板。注意规范特意澄清了一句：**"This refers to who decides when the prompt is used, not who authors its content. Prompt content is defined by the server."** 即"用户控制"说的是触发时机，不是内容作者。

有一点需要澄清：**规范划分三者的依据是控制权**（model / application / user），而不是"有没有副作用"。官方文档在描述时确实会带出差异——Resources 是 "read-only" 的被动数据源，Tools 则 "can write to databases, call external APIs, modify files"——但这属于描述性说法，不是规范里的分类标准。别把社区总结当规范条文引用。

### 4.4 协议底座：JSON-RPC 2.0 与两种传输

MCP 的底层协议很朴素。规范原文：

> All messages between MCP clients and servers **MUST** follow the JSON-RPC 2.0 specification.

传输方式则是**绑定（binding）**，与协议语义解耦。规范原文："Protocol semantics are identical on every transport." 目前两个标准绑定：

| 传输 | 规范描述 | 适用场景 |
|------|---------|---------|
| **stdio** | "newline-delimited messages over the standard streams of a client-launched subprocess" | 本地进程，比如你 `npx` 起一个文件系统 server |
| **Streamable HTTP** | "each message is an HTTP POST to a single MCP endpoint; replies arrive as a JSON object or a request-scoped SSE stream" | 远程 server |

还有一段历史需要知道，因为它解释了为什么你看到的教程写法差异很大：**旧的 HTTP+SSE 传输在 `2025-03-26` 版被 Streamable HTTP 取代，并被标记为 deprecated，新实现不应再采用。**

### 4.5 MCP 和工具调用是什么关系

这可能是最容易混淆的一点，值得说清楚：**MCP 不替代 function calling，它只是工具定义的一个"供给侧标准"。**

理由很直接：模型 API 只认它自己的 `tools` 数组（OpenAI）、`tools` 字段（Anthropic）……模型不认识 MCP。所以一个 MCP server 暴露出来的工具，**最终仍然要被转换成模型侧的 tool 定义**，才能被模型看见和调用。

官方教程把这一步演示得非常露骨——它的"构建 MCP client"文档里，就是逐字段搬运：

```python
tool_list = await client.list_tools()
available_tools = [{
    "name": tool.name,
    "description": tool.description,
    "input_schema": tool.input_schema
} for tool in tool_list.tools]
```

然后把这些定义原样传给模型 API 的 `tools` 参数。

**谁负责这次转换？** 是 Host 应用（或者它用的 SDK 封装），**不是协议规定的**。规范对此的说法是："MCP focuses solely on the protocol for context exchange—it does not dictate how AI applications use LLMs or manage the provided context."

所以分层应该是这样：

```text
模型 API  ←(tool 定义/调用请求)→  Host 应用  ←(MCP/JSON-RPC)→  MCP Server
                                  ↑
                      应用侧负责两侧的翻译与循环
```

用 Java 的视角总结：**MCP 之于工具，很像 JDBC 之于数据库。** 你对着 JDBC 接口写代码，底下换 MySQL 还是 PostgreSQL 由驱动决定；你（或你的 SDK）对着 MCP 连工具，工具是 Python 写的还是 Node 写的、跑在本地还是远端，由传输方式决定。

**但类比的边界也要说清**：JDBC 是"同一套 SQL 语义 + 不同驱动"，而 MCP 里工具的能力、描述、参数 schema 都是各 server 自定义的，**协议统一的是"怎么发现和调用"，不是"工具有什么语义"**。所以把几十个 MCP server 接进来之后，"工具太多、描述互相冲突"依然是你自己要处理的问题。

### 4.6 一个正在快速演进的协议

MCP 规范迭代很快，而且最近一次改动幅度很大。截至本文写作时，规范最新版本是 **`2026-07-28`**，它的变更清单里有几条会直接影响你抄到的示例能不能跑通：

- **MCP 改为无状态**：移除了 `initialize` / `notifications/initialized` 握手，每个请求在 `_meta` 里自带协议版本与客户端能力（`io.modelcontextprotocol/protocolVersion`、`io.modelcontextprotocol/clientCapabilities`）。版本不匹配返回 `UnsupportedProtocolVersionError`。
- **新增 `server/discover`**，且服务端 **MUST** 实现，用于声明支持的协议版本、能力与身份。
- **引入 MRTR（Multi Round-Trip Requests）**，取代"服务端主动发请求"（`roots/list`、`sampling/createMessage`、`elicitation/create`）。
- **所有 result 必须带 `resultType`**（`"complete"` 或 `"input_required"`）。
- **Roots、Sampling、Logging 被标记为废弃**（Deprecated），新实现不应再采用。
- **旧的 HTTP+SSE 传输**被归入生命周期废弃政策。

> 如果你从网上抄了一份 MCP 教程，代码里出现了 `initialize` 握手和 `Mcp-Session-Id`，那它讲的是 `2025-11-25` 及更早的流程——**这就是为什么"照着最新规范写"和"照着示例代码写"经常对不上。**

在 Java 侧，官方 SDK 是 [modelcontextprotocol/java-sdk](https://github.com/modelcontextprotocol/java-sdk)，坐标是 `io.modelcontextprotocol.sdk:mcp`，在官方 SDK 分级里属于 **Tier 2**（TypeScript、Python、C#、Go、Rust、Ruby 是 Tier 1）。而且要注意：**Spring 专属的 WebFlux / WebMVC 传输实现已经从该 SDK 移出，现在由 Spring AI 提供。**

所以在 Spring Boot 里，你不必直接用 Java SDK，用 Spring AI 的 starter 就够了——这正是下一节的内容。

---

## 五、Spring AI 登场：它到底给了你什么

前面四节讲了机制，现在看框架。Spring AI 的定位可以用一句话概括：**把上面那些手写 JSON、手拼消息、手写循环的活儿，变成 Spring 开发者熟悉的那套东西。**

### 5.1 对照表：传统 Spring 与 Spring AI

这张表是本文的核心，建议对照着记：

| 传统 Spring | Spring AI | 说明 |
|------------|----------|------|
| `spring-boot-starter-web` | `spring-ai-starter-model-openai` | 都是 starter，换个坐标换一家模型厂商 |
| `DataSource` + JDBC 驱动 | `ChatModel` + 模型 starter | 抽象 + 厂商实现，屏蔽差异 |
| `RestClient` / `JdbcTemplate` | `ChatClient` | 同一个 fluent 风格 |
| `@RequestBody` 反序列化 | `.call().entity(Foo.class)` | **方向相反**：把自然语言变对象 |
| `DispatcherServlet` 的 `HandlerMapping` | 工具分发（`ToolCallingManager`） | 都由框架把"调用"路由到方法 |
| `Filter` / AOP 环绕通知 | `Advisor` | 链式拦截、可短路、可前后增强 |
| `HttpSession` / 会话表 | `ChatMemory` + `ChatMemoryRepository` | 对话历史存储 |
| JPA `Repository` | `VectorStore` | RAG 检索的数据访问抽象 |
| `RestClientCustomizer` | `ChatClientCustomizer` | 统一定制自动配置出来的客户端（2.0 改为 `ChatClientBuilderCustomizer`） |

`ChatClient` 与 `RestClient` 的对应不是我的臆测，官方文档的原话是：

> **ChatClient API** - Fluent API for communicating with AI Chat Models, **idiomatically similar to the WebClient and RestClient APIs**.

具体到写法上，两者的结构确实同构——**入口方法 → 规格构建 → 终结方法**：

```java
// RestClient：入口 get() → 构建请求 → 终结 retrieve()/body()
restClient.get()
        .uri("/orders/{id}", orderId)
        .retrieve()
        .body(Order.class);

// ChatClient：入口 prompt() → 构建请求 → 终结 call()/content()
chatClient.prompt()
        .user("查询订单 10086 的状态")
        .call()
        .entity(Order.class);
```

有一点要注意：官方文档里明确提醒，**`call()` 本身不会触发模型调用**，它只是告诉 Spring AI 用同步还是流式；真正的调用发生在 `content()`、`chatResponse()`、`responseEntity()` 这些终结方法被调用时（`entity()` 同理）。这和 `RestClient` 的 `retrieve()`/`body()` 分两步是一个道理。

### 5.2 Starter 与自动配置：还是你熟悉的那一套

引入依赖：

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>1.1.8</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-model-openai</artifactId>
    </dependency>
</dependencies>
```

**注意：Spring AI 的版本需要你自己通过 `spring-ai-bom` 显式导入**，官方文档没有"交给 Spring Boot 的 dependency management 自动接管"这种说法。这和很多人第一次用 MyBatis-Plus starter 时的预期不太一样。

配置还是你熟悉的样子：

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o-mini
          temperature: 0.7
```

属性名不是我编的，它们来自 starter 自带的 `spring-configuration-metadata.json`（`spring.ai.openai.chat.api-key`、`spring.ai.openai.chat.base-url`、`spring.ai.openai.chat.options.model`、`spring.ai.openai.chat.options.temperature`……），所以**你在 IDE 里有自动补全**。

自动配置的链路，和你在 [Spring 接入点：跟着容器过一生](/2026/09/11/Spring-接入点：跟着容器过一生/) 里看到的那套机制完全一致。`spring-ai-starter-model-openai` 的 POM 会拉进这几个模块：

```text
spring-ai-starter-model-openai 的直接依赖（摘自其 POM）
├─ spring-ai-openai                              →  OpenAI 厂商实现（OpenAiChatModel 等）
├─ spring-ai-autoconfigure-model-openai          →  自动配置出 OpenAiChatModel
├─ spring-ai-autoconfigure-model-chat-client     →  自动配置出 ChatClient.Builder
│    └─ spring-ai-client-chat                    →  ChatClient 本身（传递依赖）
└─ spring-ai-autoconfigure-model-chat-memory     →  自动配置出 ChatMemory

另有传递依赖 spring-ai-autoconfigure-model-tool →  自动配置出 ToolCallingManager
```

注意 `spring-ai-client-chat`（`ChatClient` 所在的模块）**不是 starter 的直接依赖**，而是经 `spring-ai-autoconfigure-model-chat-client` 传递进来的；`ToolCallingManager` 也一样来自另一个模块。看依赖树时别被这两层绕晕。

`ChatClientAutoConfiguration` 产出的 `ChatClient.Builder` 是 **prototype 作用域**的（官方 Javadoc 原文："each injection point will receive a newly cloned instance of the builder"），所以你可以在注入点各自 `build()` 出自己的客户端，互不影响。这和 `RestClient.Builder` 的用法是一个套路：

```java
@RestController
public class ChatController {

    private final ChatClient chatClient;

    public ChatController(ChatClient.Builder builder) {   // 注入 Builder，而不是 ChatClient
        this.chatClient = builder.build();
    }
}
```

还有一个值得知道的扩展点：想统一改所有 `ChatClient` 的默认行为，实现 `ChatClientCustomizer` 比在每个注入点手工配置更合适——它就是 `RestClientCustomizer` 的对应物。注意 2.0 里这个接口已被标记为 `@Deprecated(forRemoval = true)`，替代者叫 `ChatClientBuilderCustomizer`。

### 5.3 Advisor：Filter 链，也是 Agent 循环的容器

`Advisor` 是 Spring AI 里最"Spring"的设计。核心方法只有一个，但它继承自 `Advisor`，而 `Advisor extends Ordered`——所以实现类除了 `adviseCall`，还得提供 `getName()` 和 `getOrder()`：

```java
public interface CallAdvisor extends Advisor {
    ChatClientResponse adviseCall(ChatClientRequest chatClientRequest, CallAdvisorChain callAdvisorChain);
}

// 继承链：CallAdvisor → Advisor → Ordered
// 因此还需实现：String getName();   int getOrder();
```

流式场景对应 `StreamAdvisor.adviseStream(...)`，两者的链式继续方法分别是 `callAdvisorChain.nextCall(...)` 和 `streamAdvisorChain.nextStream(...)`。

它的行为模式你在 [SpringBoot 的一次完整请求发生了什么？](/2026/08/26/SpringBoot的一次完整请求发生了什么？/) 的 Filter 一节里见过：**拿到请求 → 决定做点事 → 调下一个 → 拿到响应 → 决定再做点事**。官方文档对 Advisor 能力的描述里，"可以不往下走"这条最像 Filter：

> it can choose to **block the request** by not making the call to invoke the next entity. In the latter case, the advisor is responsible for filling out the response.

链的执行模型也是栈式的：

> The advisor chain operates as a **stack**: The first advisor in the chain is the first to process the request. It is also the last to process the response.

不过要诚实一点：**"Advisor 就是 Filter"是我的类比，官方文档并没有这么写。** 官方唯一提到 Servlet 的地方，是引用 Spring `Ordered` 接口自己的 Javadoc（`load-on-startup`），说的是排序语义，不是 Advisor。有意思的是，公开接口也没有采用 AOP 那套命名——它们是 `CallAdvisor` 和 `StreamAdvisor`，"around" 只残留在内部类 `DefaultAroundAdvisorChain` 的名字里。所以更准确的说法是：**从行为上看，Advisor 兼具 Filter 的链式拦截与可短路语义，以及 AOP 环绕通知的前后增强语义，但官方并没有把它定义成这两者中的任何一个。**

内置的 Advisor 覆盖了大部分常用能力：

| Advisor | 作用 |
|---------|------|
| `MessageChatMemoryAdvisor` | 把对话历史读写到 `ChatMemory` |
| `PromptChatMemoryAdvisor` | 记忆以单条 prompt 形式注入 |
| `VectorStoreChatMemoryAdvisor` | 用向量库做长期记忆 |
| `QuestionAnswerAdvisor` | RAG：检索到文档后拼进 prompt |
| `SafeGuardAdvisor` | 内容安全护栏 |
| `SimpleLoggerAdvisor` | 打印请求/响应，调试用 |
| `ToolCallAdvisor` | **工具调用循环（1.1 起；2.0 改名 `ToolCallingAdvisor`）** |
| `ChatModelCallAdvisor` | 链条末端，真正发起模型调用 |

最后一行值得注意：`ChatModelCallAdvisor` 是链的末尾，`getOrder()` 固定为 `Ordered.LOWEST_PRECEDENCE`，主体就是一次 `this.chatModel.call(...)`（之前会按需把结构化输出指令拼进消息，之后包装成 `ChatClientResponse`）。**这和你熟悉的 Filter 链末端最终打到 `DispatcherServlet` 是同一个结构**——链本身不干活，末端才是真正的执行者。

而 `ToolCallAdvisor` 放在这张表里，就说明了一件很重要的事：**在 Spring AI 1.1 里，Agent 循环不是一个特殊的"Agent API"，它就是链上的一个 Advisor。** 这也解释了 3.3 节那个设计的用意——循环一旦成为链上的一环，"记忆放循环内还是循环外"就变成了一个 order 配置问题。

### 5.4 记忆与 RAG：`ChatMemory` 和 `VectorStore`

会话记忆在 Spring AI 里是两层的：`ChatMemory` 是给 `ChatClient` 用的门面，`ChatMemoryRepository` 是存储抽象。1.1.x 内置的实现有 `InMemoryChatMemoryRepository`（默认）、`JdbcChatMemoryRepository`、`CassandraChatMemoryRepository`、`Neo4jChatMemoryRepository`、`MongoChatMemoryRepository`。**`RedisChatMemoryRepository` 要等到 2.0 才有**（对应的 starter 在 Maven Central 上最早只到 2.0.0-M1）——这是查版本时容易被 BOM 误导的一个点，BOM 里列了名字不代表该版本真的发布了这个产物。

用法上有一个极易踩的坑：

```java
ChatMemory chatMemory = MessageWindowChatMemory.builder().build();   // 默认保留 20 条消息

ChatClient chatClient = ChatClient.builder(chatModel)
        .defaultAdvisors(MessageChatMemoryAdvisor.builder(chatMemory).build())
        .build();

chatClient.prompt()
        .user("我刚才说的订单号是多少？")
        .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, "user-42"))   // 必填！
        .call()
        .content();
```

**`ChatMemory.CONVERSATION_ID` 是必填的**，官方文档两处都强调了这一点：没有默认会话 ID，漏了它会在运行时抛 `IllegalArgumentException`。这个设计其实很合理——多用户环境下"默认会话"是个危险的默认值。

RAG（检索增强生成）则是另一条路：用 `VectorStore` 存文档向量，检索出相关片段后拼进 prompt。它的用法和 JPA `Repository` 的思路相通——`VectorStore` 是接口，底下可以是 PGVector、Redis、Milvus、Elasticsearch 等，对应 `spring-ai-starter-vector-store-*` 系列 starter。最省事的接法是挂一个 Advisor：

```java
chatClient.prompt()
        .advisors(QuestionAnswerAdvisor.builder(vectorStore).build())
        .user("我们的退款政策是什么？")
        .call()
        .content();
```

注意 `QuestionAnswerAdvisor` 在独立模块 `spring-ai-advisors-vector-store` 里，包名是 `org.springframework.ai.chat.client.advisor.vectorstore`，需要单独加依赖。（本文 5.3 节表格里的 `VectorStoreChatMemoryAdvisor` 也在同一个模块。）

### 5.5 一个能跑的最小 Agent

把前面的东西拼起来。依赖：

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-model-openai</artifactId>
    </dependency>
</dependencies>
```

工具（就是普通的 Spring Bean）：

```java
@Component
public class OrderTools {

    @Tool(description = "根据订单号查询订单状态与预计发货时间")
    public String getOrderStatus(@ToolParam(description = "订单号，纯数字") String orderId) {
        return orderService.findStatus(orderId).toJson();
    }

    @Tool(description = "查询某件商品当前库存")
    public int getStock(@ToolParam(description = "商品 SKU 编码") String sku) {
        return stockService.available(sku);
    }
}
```

装配 Agent（工具 + 记忆 + 显式挂上循环）：

```java
@Configuration
public class AgentConfig {

    @Bean
    ChatClient agentChatClient(ChatClient.Builder builder,
                               OrderTools orderTools,
                               ToolCallingManager toolCallingManager) {
        ChatMemory chatMemory = MessageWindowChatMemory.builder()
                .maxMessages(30)
                .build();

        return builder
                .defaultSystem("你是订单助手。回答前先查数据，不要凭记忆猜测订单状态。")
                .defaultToolCallbacks(MethodToolCallbackProvider.builder()
                        .toolObjects(orderTools)
                        .build())
                .defaultAdvisors(
                        MessageChatMemoryAdvisor.builder(chatMemory).build(),
                        ToolCallAdvisor.builder()
                                .toolCallingManager(toolCallingManager)
                                .build())
                .build();
    }
}
```

对外暴露接口。想要流式（对前端体验很重要，因为 Agent 一轮循环可能要好几秒），用 `Flux<String>`：

```java
@RestController
public class AgentController {

    private final ChatClient agentChatClient;

    public AgentController(ChatClient agentChatClient) {
        this.agentChatClient = agentChatClient;
    }

    @PostMapping(value = "/agent", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> chat(@RequestBody ChatRequest request,
                             @RequestHeader("X-Session-Id") String sessionId) {
        return agentChatClient.prompt()
                .user(request.message())
                .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, sessionId))
                .stream()
                .content();
    }
}
```

这段代码里有两处值得停下来看一眼：

- **`defaultSystem` 是一段 prompt，不是配置。** 它相当于给模型写的"岗位说明书"，也是约束模型行为最直接的手段。
- **`.stream()` 返回 `Flux<String>`，所以需要 Reactive 栈。** 这一点和 Spring MVC 里的 `SseEmitter` 是同一件事的两种写法。区别在于模型侧的流式数据源本身就是 `Flux`，所以 Reactive 栈在这里更自然。如果你对 `Flux` 的订阅模型还不太放心，可以先看 [深入理解 Spring WebFlux 与 Reactor Netty](/2026/09/15/深入理解-Spring-WebFlux-与-Reactor-Netty/) 里"链式调用只是构建蓝图，订阅才让它活过来"那一节——流式接口的调试问题，八成出在订阅时机上。

---

## 六、版本选型、坑与边界

### 6.1 三条版本线，怎么选

Spring AI 目前同时维护三条 GA 线，选择依据是**你的 Spring Boot 版本**，而不是"哪个数字大"：

| 版本线 | 最新版 | 适配的 Spring Boot | 说明 |
|-------|-------|------------------|------|
| 1.0.x | 1.0.9 | Spring Boot 3.4.x / 3.5.x | 维护线 |
| 1.1.x | 1.1.8 | Spring Boot 3.4.x / 3.5.x | **Boot 3.x 项目的现实选择** |
| 2.0.x | 2.0.1 | Spring Boot 4.0.x / 4.1.x | 当前最新稳定版 |

这个对应关系可以直接从 starter 的 POM 里验证：`spring-ai-starter-model-openai:1.1.8` 依赖 `spring-boot-starter:3.5.15`，而 `2.0.1` 依赖 `spring-boot-starter-webclient:4.1.1`。

选型建议很直接：

- **项目在 Spring Boot 3.x**：用 1.1.x，这是唯一的 GA 选项（1.0.x 也在维护，但 1.1.x 才有循环上移到 advisor 的能力）。
- **准备升 Spring Boot 4**：用 2.0.x。
- **`2.1.0-M1` 是 milestone，别在生产用。** 另外提醒一句：Maven Central 的 `maven-metadata.xml` 里 `<release>` 指向的是**最新发布的非 SNAPSHOT 版本**——milestone 和 RC 都不算 SNAPSHOT，所以同样会出现在这个字段里。它不是"最新稳定版"的意思，别被它误导。

### 6.2 2.0 与 1.1 的关键差异

如果你在升级或读两套文档，下面这几处差异最容易让人对不上号：

| 项目 | 1.1.x | 2.0.x |
|------|-------|-------|
| 工具循环的位置 | `ChatModel` 内部（可用 `internalToolExecutionEnabled` 关闭） | **从 `ChatModel` 移除，全部由 advisor 承担** |
| 循环 Advisor 类名 | `ToolCallAdvisor` | `ToolCallingAdvisor`（旧名 `@Deprecated(forRemoval = true)` 保留） |
| 是否自动注册 | **不自动**，需要你自己加进 advisor 链 | `ChatClient` 默认自动注册 |
| `ChatModel.call(prompt)` | 自动执行工具并递归 | **只返回模型原始响应**，不执行工具 |
| 自定义 `ChatClient` 的扩展点 | `ChatClientCustomizer` | `ChatClientBuilderCustomizer`（旧接口已废弃） |

最后一行是升级时最容易踩的坑：如果你的代码直接用 `ChatModel`（而不是 `ChatClient`），2.0 里它**不会**再自动跑循环了，工具调用会"消失"。

### 6.3 七个容易踩的坑

**1. 循环没有终止条件，就是死循环。** Anthropic 那篇文章里专门提过要设"stopping conditions (such as a maximum number of iterations)"。模型完全可能反复调同一个工具、拿同一个结果，然后继续调。**上线前一定要有迭代上限**——OpenAI 的 Agents SDK 把这件事做成了框架级约束（超过 `max_turns` 直接抛 `MaxTurnsExceeded`）；2.0 给 `ToolCallingManager` 加了调用次数上限（同名工具默认 40 次、总量默认 150 次，超限默认抛异常），1.1 里则需要你自己兜底。

**2. 工具描述写得含糊，模型就调不准。** 回到 2.3 节：`description` 是接口文档，而且是调用方逐字阅读后据此决策的文档。工具一多，描述之间的边界尤其重要——`getOrderStatus` 和 `getOrderDetail` 如果不写清区别，模型只能猜。

**3. 工具执行是阻塞的，而且会打乱可观测性链路。** 官方文档说得很直接："Tool calling is imperative, leading to blocking workflows. This also results in partial/interrupted Micrometer observations"。线程归属则要分两条路：**非流式（`adviseCall`）在调用方线程里同步跑完，流式路径下不是**——`ToolCallAdvisor.adviseStream` 的最后一步是 `subscribeOn(Schedulers.boundedElastic())`，工具方法跑在 Reactor 的弹性线程池上。所以别假设工具的执行线程与调用链一致，`ThreadLocal`、MDC、事务上下文都别指望能透传；工具里如果去调远程服务，超时时间必须自己设。

**4. 上下文长度和费用会被循环放大。** 每多一轮工具调用，消息历史就长一截，而历史是要整段重发的。一个五轮循环的任务，输入 token 可能是单轮问答的十几倍。`MessageWindowChatMemory` 的窗口大小（默认 20 条）和系统提示的长度，都会直接体现在账单上。

**5. `ChatMemory.CONVERSATION_ID` 漏了会在运行时抛异常**，而且没有默认值。多实例部署时，`InMemoryChatMemoryRepository` 会导致"上一句还在、这一句就忘了"，需要换成 `JdbcChatMemoryRepository`（或 Neo4j / Cassandra / Mongo 的实现）。

**6. MCP server 默认没有鉴权。** 2.0.x 的参考文档明确警告：基于 HTTP 的 server 传输（SSE、Streamable HTTP）**默认暴露一个未鉴权的 JSON-RPC 端点**，starter 本身不做任何认证授权——"any client that can reach the endpoint can list and invoke every registered tool, resource, and prompt"。默认端点是 `POST /mcp`。官方给了一句很值得贴在代码评审里的话：**"Treat the registration of a tool, resource, or prompt as a decision to expose it."** 工具就是真正的权限边界——如果你的 `deleteOrder` 被注册成工具，那就是把删除能力开放给了任何能连上端点的人。（顺带一提：这段警告出现在 2.0.x 文档里，1.1.x 的同名页面还没有这句话——但漏洞不分版本，暴露在公网上的 server 一样危险。）

**7. 用 `ChatClient.builder(chatModel)` 建客户端，会静默丢掉自动配置。** 这条和 Spring Boot 里"绕过自动配置自己 `new` 一个 Bean"是同一类问题：官方文档明确说明，这样做会让**可观测性和 `ChatClientBuilderCustomizer` 全部失效**。想自定义又不想丢东西，正确做法是注入 `ChatClientBuilderConfigurer` 来构建，它会把所有 customizer 和你需要的观测组件都接上。另外，当容器里存在多个 `ChatModel` 时，自动配置的 `ChatClient.Builder` 会因为依赖歧义启动失败，需要给其中一个标 `@Primary`。

### 6.4 上生产之前必须想清楚的四件事

**可观测性。** Spring AI 基于 Micrometer Observation 提供了完整埋点（`ChatClient`、`Advisor`、`ChatModel`、`Tool`、`VectorStore`）。但要特别注意：**默认不记录 prompt 和响应内容**，需要显式开启——这也意味着一旦开启，敏感数据就可能进日志：

```yaml
spring:
  ai:
    chat:
      client:
        observations:
          log-prompt: false        # 开启前先想清楚数据合规
          log-completion: false
```

**重试。** 模型接口的失败率比普通内部接口高，超时和限流都很常见。Spring AI 有 `spring.ai.retry.*` 系列属性（`max-attempts`、`backoff.initial-interval`、`backoff.multiplier` 等），但要注意：**重试的是模型调用，不是工具调用**。工具的幂等性得你自己保证。

**成本上限。** 除了 token 计费，还要考虑循环次数、并发数、单用户配额。这三个量里任何一个失控，账单都会失控。

**权限模型。** 这一条最容易被忽略，也最危险。工具是模型"能动的手"，而模型可能被诱导去调它。**不要在工具实现里信任模型传进来的参数**——`orderId` 要校验属主，`sendMail` 要限制收件人范围，写操作要有二次确认。把工具当成一个暴露给外部的 API 来设计，而不是当成内部方法随手开放。

---

## 结语：回到那个"用 `RestClient` 不就行了"的问题

现在可以正面回答了。

用 `RestClient` 直接调模型，你得到的是一次"问答"。它没有记忆，输出无法可靠地反序列化，也不能触发任何业务动作。**你缺的不是一个 HTTP 客户端，而是围绕这次调用长出来的三样东西**：

- **工具调用**：让模型能"说"出它想调什么，而你的进程负责真的去调；
- **循环**：让"调工具 → 看结果 → 再决定"能重复进行，直到任务收敛；
- **协议**：让工具可以在应用之间、语言之间复用，而不是每个应用重写一遍。

Spring AI 做的事，是用你熟悉的方式把这三样交到你手上：`@Tool` 对应"接口 + 文档"，`Advisor` 对应 Filter 链（循环本身就是链上的一环），Starter 与自动配置对应你写了多年的 Boot 那一套。

### 核心结论

1. **模型不执行任何东西。** 它只生成 token；工具执行永远发生在你的进程里。这是 Agent 开发的第一道分水岭。
2. **一次工具调用 = 两轮 HTTP 请求 + 一次本地方法调用。** 模型返回 `tool_calls`，应用执行，结果以 `role=tool` 回填，再问一次。
3. **工具描述就是接口文档**，而且是被调用方逐字阅读后据此决策的文档。写得含糊，模型就调不准。
4. **ReAct 的核心是循环**：Thought → Action → Observation，然后回到 Thought。function calling 是它的工程化实现，不是替代品。
5. **模型每轮只完成一轮，循环必须由应用侧驱动。** 那个 `while` 就是 Agent 的全部秘密。
6. **循环的位置决定了可组合性。** Spring AI 1.x 把循环藏在 `ChatModel` 内部，1.1 用 `ToolCallAdvisor` 把它搬到 Advisor 链上，2.0 则彻底移除模型内部循环、由 `ChatClient` 自动注册 advisor——目的都是让循环能被别的组件包住。
7. **不是所有场景都要 Agent。** 能写成确定编排（Workflow）的，就别交给模型自由决策：更便宜、更快、更可靠。
8. **MCP 是工具的供给侧标准**：Host : Client = 1 : N，Client : Server = 1 : 1；Tools 由模型控制、Resources 由应用控制、Prompts 由用户控制。
9. **MCP 不替代 function calling。** MCP server 暴露的工具最终仍要由 Host 侧翻译成模型 API 的 tool 定义。
10. **MCP 规范迭代很快。** 最新版 `2026-07-28` 已改为无状态、移除 `initialize` 握手、新增 `server/discover`、废弃 Roots/Sampling/Logging；而 Java SDK 的文档口径仍停留在旧版流程。抄示例代码前先确认它对应的规范版本。
11. **选版本看 Spring Boot，不看数字大小。** Boot 3.x 用 1.1.x，Boot 4.x 用 2.0.x；`2.1.0-M1` 是 milestone。
12. **上生产的四个必答题**：循环迭代上限、工具执行阻塞与超时、token 成本、工具即权限边界。

### 延伸阅读

- [Spring 接入点：跟着容器过一生](/2026/09/11/Spring-接入点：跟着容器过一生/)——本文 5.2 节讲的 starter 与自动配置，那里有完整的容器启动时间线；想搞清 `ChatClient.Builder` 这类 Bean 是何时被造出来的，看它。
- [SpringBoot 的一次完整请求发生了什么？](/2026/08/26/SpringBoot的一次完整请求发生了什么？/)——本文把 Advisor 类比成 Filter 链，那篇文章的第四节和第十一节可以对着看，理解"链式拦截 + 末端执行"这个结构。
- [深入理解 Spring WebFlux 与 Reactor Netty](/2026/09/15/深入理解-Spring-WebFlux-与-Reactor-Netty/)——本文 5.5 节的流式接口返回 `Flux<String>`，订阅模型、线程归属、为什么不能手动 `subscribe`，都在那里。
- [一次下载失败排查：被 Filter 吞掉的异步响应流](/2026/10/14/一次下载失败排查：被 Filter 吞掉的异步响应流/)——同样是流式响应，同样卡在 Filter/Advisor 这一层。Agent 的流式接口出问题时，排查思路可以复用。
- [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629)（arXiv:2210.03629）——3.1 节引用的论文原文与失败案例分析都在这里。
- [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)（Anthropic，2024-12-19，页面标题现已改为 "Building Effective AI Agents"）——3.4 节 workflow 与 agent 的区分、以及"什么时候不该用 Agent"的原文出处。
- [Writing effective tools for agents](https://www.anthropic.com/engineering/writing-tools-for-agents)（Anthropic，2025-09-11）——2.3 节"工具即契约"与"不要一比一包装现有接口"的出处，做工具设计时值得整篇读。
- [Model Context Protocol 规范](https://modelcontextprotocol.io/)——第四节所有协议细节的一手来源，注意确认你读的是哪个版本。
- [Spring AI Reference Documentation](https://docs.spring.io/spring-ai/reference/)——注意页面顶部可以切换 1.0 / 1.1 / 2.0 的文档版本，读 API 前先对准版本。
