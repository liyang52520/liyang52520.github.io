---
title: "@Transactional 为什么会失效（四）：失效原因映射与收束"
date: 2026-10-16 16:25:34
tags:
  - spring
  - aop
  - transaction
categories:
  - develop
---

## 引子：从开头那段没回滚的代码说起

导读开头那段代码：

```java
public void createOrder(OrderDTO dto) {
    stockService.deduct(dto.getSkuId(), dto.getCount());
    this.saveOrder(dto);        // 同类内部调用
}
```

跑一遍，库存扣了、订单也落了——`saveOrder` 上那个 `@Transactional` 像没写一样。前面五个部分已把整条链拆过一遍，这一篇倒过来用：**从现象出发，反推断在哪一环。**

失效的样子千奇百怪，断点却只有几个：自调用、`private`、接口注解、被吞的异常、被换的线程——都会落回同一张图。

> 本篇接[第三篇](/2026/10/16/Transactional-为什么会失效（三）：事务管理器的-ThreadLocal-账本/)（账本已摊开），覆盖第六~八部分与延伸阅读：失效现象映射、自调用与 `@Async`、切面顺序、响应式与 AspectJ。口径同导读：Spring Framework 6.1.8 + Spring Boot 3.3.x（Java 17）。

> 本篇要回答的问题：
>
> - 十二种失效现象——自调用、`private`、接口上的注解……分别断在链条的哪一环？
> - 自调用到底怎么修？注入自己、`AopContext`、`TransactionTemplate` 各自代价是什么？
> - `@Async` 和 `@Transactional` 放同一个方法上，事务在哪个线程生效？
> - 事务切面和普通切面差在哪四处？谁先执行由什么决定？响应式与 AspectJ 又绕开了哪些限制？

## 第六部分：回到开头——把失效原因映射回链路

有了前五部分的链路，开头那张图上的"失效点"可以逐个填上具体原因了。下表的编号 A~L 是为了指代方便，和 1.2 节图（第一篇）里的"失效点 1~5"是两套编号。

| 现象 | 描述 | 断在哪一环 | 判断依据 |
|---|---|---|---|
| A | 同类内部方法调用，事务不生效 | 运行入口没经过代理 | `this` 不是代理 |
| B | `private` / `static` 方法不生效 | 代理无法重写 | CGLIB debug 日志 |
| C | `final` 方法不生效 | 代理无法重写 | CGLIB debug 日志（该 final 方法同时实现了接口方法时才会 `warn`） |
| D | 注解只写在接口上，CGLIB 下不生效 | 属性解析返回 null | 代理存在但链为空 |
| E | 受检异常不回滚 | `rollbackOn` 规则 | 异常不在回滚规则内 |
| F | 异常被 `catch` 吞掉 | 拦截器看不到异常 | 链上没收到 throw |
| G | 内层 `readOnly` / `isolation` 不生效 | `handleExistingTransaction` 忽略 | `validateExistingTransaction` 默认 false |
| H | 内层抛出、外层吞掉，最后 `UnexpectedRollbackException` | 参与已有事务只能标 rollback-only | 异常已吞，但全局标记还在 |
| I | 没有事务管理器，静默不生效 | `createTransactionIfNecessary` 只打 debug | TRACE 日志 |
| J | 多线程 / `@Async` 里事务不存在或各管各的 | ThreadLocal 不跨线程 | `isActualTransactionActive()` |
| K | `new` 出来的对象事务不生效 | 没经过容器 | `AopUtils.isAopProxy(obj)` 为 false |
| L | 多数据源下抛 `NoUniqueBeanDefinitionException` | 按类型取管理器 | `getBean(TransactionManager.class)` |

下面挑几条最容易误判的说透。

### 6.1 自调用：唯一的解法是"再走一次代理"

开头那段代码的问题在于 `this.saveOrder(dto)`。`this` 是目标对象，不是代理，调用不经过拦截器链。

三种可行的改法，各有代价：

```java
// 方案一：把方法拆到另一个 Bean —— 最干净，也是首选
@Service
public class OrderService {
    @Autowired private OrderTxService orderTxService;   // 注入的是代理

    public void createOrder(OrderDTO dto) {
        stockService.deduct(dto.getSkuId(), dto.getCount());
        orderTxService.saveOrder(dto);
    }
}
```

```java
// 方案二：注入自己，用代理调用自己
@Service
public class OrderService {

    @Autowired
    @Lazy                                  // ← 不加 @Lazy 会循环依赖
    private OrderService self;

    public void createOrder(OrderDTO dto) {
        self.saveOrder(dto);               // self 是代理
    }
}
```

```java
// 方案三：显式用编程式事务，绕开代理
@Service
public class OrderService {

    @Autowired private TransactionTemplate transactionTemplate;

    public void createOrder(OrderDTO dto) {
        transactionTemplate.execute(status -> {
            orderMapper.insert(toOrder(dto));
            return null;
        });
    }
}
```

方案三常被误解为"另一种事务机制"。其实不是：`TransactionTemplate` 内部拿的还是同一个 `PlatformTransactionManager`，只是把 `invokeWithinTransaction` 里那套 `getTransaction` / `commit` / `rollback` 手写了一遍。**代理是"声明式"的外观，`PlatformTransactionManager` 才是内核。**

```java
// org.springframework.transaction.support.TransactionTemplate
public <T> T execute(TransactionCallback<T> action) throws TransactionException {
    Assert.state(this.transactionManager != null, "No PlatformTransactionManager set");

    if (this.transactionManager instanceof CallbackPreferringPlatformTransactionManager cpptm) {
        return cpptm.execute(this, action);          // ← 和 invokeWithinTransaction 里那个分支同源
    }
    else {
        TransactionStatus status = this.transactionManager.getTransaction(this);
        T result;
        try {
            result = action.doInTransaction(status);
        }
        catch (RuntimeException | Error ex) {
            // Transactional code threw application exception -> rollback
            rollbackOnException(status, ex);           // ← 和拦截器同一套完成逻辑
            throw ex;
        }
        catch (Throwable ex) {
            // Transactional code threw unexpected exception -> rollback
            rollbackOnException(status, ex);
            throw new UndeclaredThrowableException(ex, "TransactionCallback threw undeclared checked exception");
        }
        this.transactionManager.commit(status);
        return result;
    }
}
```

### 6.2 `AopContext.currentProxy()`：开关没开就会炸

```java
// 需要 @EnableAspectJAutoProxy(exposeProxy = true)
public void createOrder(OrderDTO dto) {
    ((OrderService) AopContext.currentProxy()).saveOrder(dto);
}
```

对应 4.1 节（第二篇）里那段 `if (this.advised.exposeProxy)`。没打开时抛的是：

```text
java.lang.IllegalStateException: Cannot find current proxy:
    Set 'exposeProxy' property on Advised to 'true' to make it available,
    and ensure that AopContext.currentProxy() is invoked in the same thread as the AOP invocation context.
```

**注意最后半句 "in the same thread"**——`AopContext` 也是 `ThreadLocal`。这与事务的线程约束是同一个道理：AOP 的运行时上下文，本质上都是线程级的。

### 6.3 `@Async` 与 `@Transactional` 同框：最容易两头落空

```java
@Async
@Transactional
public void asyncSave(OrderDTO dto) { ... }
```

两个注解都要靠代理，但它们的**执行时机**完全不同：

- `@Transactional` 的拦截器在当前线程上开事务、绑定 `ConnectionHolder`。
- `@Async` 的拦截器把任务提交到线程池，业务代码在**另一个线程**执行。

关键在于：这两个 advisor 不是同一时刻挂上去的。自动代理创建器先给 Bean 包一层代理（此时链上只有事务 advisor），随后 `@EnableAsync` 注册的 `AsyncAnnotationBeanPostProcessor` 才处理这个 Bean。它发现 Bean 已经是 `Advised` 代理时，就**不重新包一层，而是把自己的 advisor 插进现有列表**：

```java
// org.springframework.aop.framework.AbstractAdvisingBeanPostProcessor
public Object postProcessAfterInitialization(Object bean, String beanName) {
    if (this.advisor == null || bean instanceof AopInfrastructureBean) {
        return bean;
    }

    if (bean instanceof Advised advised) {
        if (!advised.isFrozen() && isEligible(AopUtils.getTargetClass(bean))) {
            // Add our local Advisor to the existing proxy's Advisor chain.
            if (this.beforeExistingAdvisors) {
                advised.addAdvisor(0, this.advisor);          // ← 插到链首
            }
            // ...
            return bean;
        }
    }
    // ... 否则自己创建代理
}
```

而 `AsyncAnnotationBeanPostProcessor` 的构造函数里明确写了：

```java
// org.springframework.scheduling.annotation.AsyncAnnotationBeanPostProcessor
public AsyncAnnotationBeanPostProcessor() {
    setBeforeExistingAdvisors(true);
}
```

所以 `@Async` + `@Transactional` 同方法的实际链路是确定的：

```text
调用方线程
  → 异步拦截器（在链首）
       └─ 提交任务到线程池，返回 Future
线程池线程
  → 事务拦截器          ← getTransaction 在这里执行
       └─ 业务方法
```

**结论：事务确实生效了，只不过它是在线程池线程里开启的。** `@Async` 拦截器在链首，它先把任务丢进线程池，`@Transactional` 的 `getTransaction` 才在那个新线程里执行——账本自然也就记在新线程上，调用方线程完全不知道这件事。

这也解释了为什么 `@Transactional` 和 `@Async` 放同一个方法上不该被推荐：**不是"不生效"，而是生效的语义和绝大多数人的直觉相反**——"事务"覆盖的是线程池里的那段执行，而不是调用方的那次调用；调用方一旦在线程池任务提交之后、任务执行之前做别的写操作，两者根本不在一个事务里。事务边界和线程边界必须一眼看得出对齐关系。

想同时要异步和事务，把职责拆开，让事务边界显式落在异步线程内部：

```java
@Async
public void asyncHandle(OrderDTO dto) {
    // 在新线程里发起一次完整的、有事务的调用
    orderTxService.saveInTransaction(dto);
}
```

### 6.4 怎么快速定位

排查顺序建议固定成"从外往里"，每一步都能给出明确结论：

```java
// 1. 拿到的是代理吗？
System.out.println(AopUtils.isAopProxy(orderService));          // false → 失效点 1 / 现象 K

// 2. 这个代理上有几条通知？事务 Advisor 在里面吗？
if (orderService instanceof Advised advised) {
    for (Advisor advisor : advised.getAdvisors()) {
        System.out.println(advisor);
    }
}
// 输出里应该看到 BeanFactoryTransactionAttributeSourceAdvisor

// 3. 当前线程到底有没有事务？
System.out.println(TransactionSynchronizationManager.isActualTransactionActive());

// 4. 从日志看更省事
// logging.level.org.springframework.transaction.interceptor=TRACE
// logging.level.org.springframework.aop.framework.CglibAopProxy=DEBUG
```

第 2 步最有用：**`Advised.getAdvisors()` 直接告诉你这个 Bean 身上挂了哪些切面、在某些方法上会形成什么顺序。** 当"事务和自定义切面互相干扰"时，一行 `System.out.println` 比读十遍代码快。

---

## 第七部分：事务与普通切面，到底哪里不一样

前面所有的相似性都建立在"它们都是 `Advisor` + `MethodInterceptor`"这个基础上。但事务并不是一个平凡的切面，有四处本质差异。

### 7.1 事务有状态，切面通常无状态

普通切面的拦截器基本都是无状态的：进去、出来、打印日志，不依赖"当前线程上有没有另一个切面正在执行"。事务不一样，它依赖 ThreadLocal 里那份账本（`resources` / `synchronizations` / `actualTransactionActive`），而且**同一线程上嵌套调用时，内层必须感知外层**。

这一点带来的直接后果：

- 事务不可重入任意次——`getTransaction` 每次都要判断 `isExistingTransaction`，于是产生了七种传播行为。
- 事务不跨线程——ThreadLocal 的天然限制。
- 事务的回调顺序和提交时机强绑定，而普通切面没有"提交"这个概念。

### 7.2 顺序：谁包着谁，由 order 说了算

事务 Advisor 的默认 order：

```java
// org.springframework.transaction.annotation.EnableTransactionManagement
int order() default Ordered.LOWEST_PRECEDENCE;
```

`AbstractPointcutAdvisor.getOrder()` 在没有显式设置时同样返回 `Ordered.LOWEST_PRECEDENCE`。而拦截器链是**按 order 从小到大依次执行**的，order 越小越靠外。于是就有了一个很流行的说法——"事务默认在最内层"。这个说法只对了一半：

```text
order 更小 = 更靠外（先进、后出）
order 更大 = 更靠内（后进、先出）

例如切面声明了 @Order(0)：
  自定义切面（order = 0）
    └─ TransactionInterceptor（order = LOWEST_PRECEDENCE）
         └─ 业务方法
```

这里有两个容易踩的点：

**影响一：不写 `@Order` 的切面，和事务是并列的，相对顺序没有保证。** 这一点最容易想当然。事务 Advisor 的 order 来自 `@EnableTransactionManagement.order`，默认 `LOWEST_PRECEDENCE`；而一个普通的 `@Aspect` 类如果不写 `@Order`，它的 Advisor order 同样落在默认值上：

```java
// org.springframework.aop.aspectj.annotation.BeanFactoryAspectInstanceFactory
@Override
public int getOrder() {
    Class<?> type = this.beanFactory.getType(this.name);
    if (type != null) {
        if (Ordered.class.isAssignableFrom(type) && this.beanFactory.isSingleton(this.name)) {
            return ((Ordered) this.beanFactory.getBean(this.name)).getOrder();
        }
        return OrderUtils.getOrder(type, Ordered.LOWEST_PRECEDENCE);   // ← 不写 @Order 就是它
    }
    return Ordered.LOWEST_PRECEDENCE;
}
```

两者同阶时，`AspectJPrecedenceComparator` 给出的结论是"不关心顺序"，最终怎么排只取决于排序的稳定性。所以**"事务一定在最内层"并不成立**：只有切面显式声明了更小的 order，事务才确定在它之内。回到那个"写审计日志 + `pjp.proceed()`"的 `@Around`——如果它碰巧排在事务外面，这段审计写入就不会被回滚。

**影响二：要让"切面里的写库"确定进事务，就把切面放到事务内层。** 做法是把事务的 order 显式调小（`LOWEST_PRECEDENCE` 已经是 `int` 的最大值，往大调没有空间），让事务移到外层：

```java
@Configuration
@EnableTransactionManagement(order = 0)        // 事务移到外层
public class TxOrderConfig { }
```

此时链变成 `事务 → 审计切面 → 业务方法`，切面的写库和外层处于同一个事务。**代价也要一并接受**：切面里抛出的异常同样会触发整个事务回滚——把切面拉进事务，就等于把它的失败也纳入了事务语义。

**一个特别注意点**：如果项目里手写 `@EnableTransactionManagement`，`@ConditionalOnMissingBean(AbstractTransactionManagementConfiguration.class)` 会让 Boot 的事务自动配置退让，事务侧的配置以你的注解为准。但这**不会**让 `spring.aop.proxy-target-class` 失效——`AopAutoConfiguration` 是独立的自动配置，默认仍然强制 CGLIB（2.5 节，第一篇）。所以"我写了 `@EnableTransactionManagement(proxyTargetClass = false)`，就切到 JDK 代理了"这种判断在很多项目里站不住：注解里写 `false` 只是"自己不要求 CGLIB"，而另一条路还在要求。

### 7.3 响应式：账本从 ThreadLocal 搬到 Reactor Context

响应式事务（`ReactiveTransactionManager`）不能再用 ThreadLocal，因为一次响应式调用会在不同线程上被调度。Spring 的做法是在 `invokeWithinTransaction` 的最前面分叉：

```java
// org.springframework.transaction.interceptor.TransactionAspectSupport
final TransactionManager tm = determineTransactionManager(txAttr);

if (this.reactiveAdapterRegistry != null && tm instanceof ReactiveTransactionManager rtm) {
    boolean isSuspendingFunction = KotlinDetector.isSuspendingFunction(method);
    // ...
    ReactiveTransactionSupport txSupport = this.transactionSupportCache.computeIfAbsent(method, key -> {
        Class<?> reactiveType =
                (isSuspendingFunction ? (hasSuspendingFlowReturnType ? Flux.class : Mono.class) : method.getReturnType());
        ReactiveAdapter adapter = this.reactiveAdapterRegistry.getAdapter(reactiveType);
        if (adapter == null) {
            throw new IllegalStateException("Cannot apply reactive transaction to non-reactive return type [" +
                    method.getReturnType() + "] with specified transaction manager: " + tm);
        }
        return new ReactiveTransactionSupport(adapter);
    });
    // ...
    return txSupport.invokeWithinTransaction(method, targetClass, callback, txAttr, rtm);
}
```

响应式事务的上下文由 `org.springframework.transaction.reactive.TransactionSynchronizationManager` 承载，它是一个**持有 `TransactionContext` 的实例**（而不是静态 ThreadLocal），`TransactionContext` 被写进 Reactor 的 Context 里随之传播。

这带来两条硬约束：

**约束一：`@Transactional` 的返回值必须是响应式类型。** 上面那段 `throw new IllegalStateException("Cannot apply reactive transaction to non-reactive return type ...")` 就是防线。`WebFlux` 控制器里写一个返回 `void`、内部用 `JdbcTemplate` 的 `@Transactional` 方法，拿到的不是"事务跨线程丢了"，而是一开始就选错了事务管理器。

**约束二：阻塞式的事务管理器和响应式的是两套账本，不能混用。** 线程池 + `ThreadLocal` 的组合在响应式链路里必然出错。这也是为什么 [深入理解 Spring WebFlux 与 Reactor Netty](/2026/09/15/深入理解-Spring-WebFlux-与-Reactor-Netty/) 里反复强调的"线程模型"在这里又一次成为决定因素——**事务问题的本质，一半是 AOP，另一半是线程模型。**

### 7.4 AspectJ 模式：不走代理的那条路

`@EnableTransactionManagement(mode = AdviceMode.ASPECTJ)` 时，`TransactionManagementConfigurationSelector` 返回的不再是 `AutoProxyRegistrar`，而是配置类 `org.springframework.transaction.aspectj.AspectJTransactionManagementConfiguration`，由它注册 `spring-aspects` 里的 `AnnotationTransactionAspect`（因此需要编译期或加载期织入）。

这条路绕开了代理，于是前面提到的限制大多消失：

| 限制 | PROXY（默认） | ASPECTJ |
|---|---|---|
| 自调用 | 失效 | **生效**（代码被织入，不依赖代理） |
| `private` 方法 | 失效 | **生效**（可织入 private） |
| `protected` 方法 | Spring 6 可生效（CGLIB） | 生效 |
| `final` 方法 | 失效 | 生效 |
| 额外依赖 | 无 | `spring-aspects` + 织入配置 |
| 运行期开销 | 有代理调用开销 | 编译期已织入，无代理开销 |

代价也很明确：构建链路变复杂（`aspectj-maven-plugin` 或 `-javaagent`），调试时栈帧里看不到代理层，团队里"为什么本地生效、线上不生效"的经典问题也常来自织入没配好。**除非自调用确实是架构上躲不开的，否则不值得为了省一次方法拆分而切到 ASPECTJ。**

---

## 第八部分：把两张图合成一张

最后回到那个最初的困惑：为什么"事务"和"AOP"总让人感觉是两块知识，却又怎么都拆不开？

因为它们是同一套机制的两种视角。

| 你在事务里看见的 | 它在 AOP 里叫什么 |
|---|---|
| `@EnableTransactionManagement` | `@EnableAspectJAutoProxy` 的同级角色（注册 APC） |
| `BeanFactoryTransactionAttributeSourceAdvisor` | 一个 `Advisor` Bean |
| `TransactionAttributeSourcePointcut` | 一个 `StaticMethodMatcherPointcut` |
| "方法上有没有 `@Transactional`" | pointcut 的匹配条件 |
| `TransactionInterceptor` | 一个 `MethodInterceptor`（即 `@Around`） |
| `proxyTargetClass` / `spring.aop.proxy-target-class` | 代理方式，全局唯一那份配置 |
| `transactionManager` 属性 | 拦截器里的依赖选择 |
| `TransactionSynchronizationManager` | AOP 之外的"线程级运行时上下文" |

于是可以给出一句概括：

**事务 = 一个 Advisor + 一个 Interceptor + 一份 ThreadLocal 账本。前两者可以直接用 AOP 的机制描述，唯独第三者不属于 AOP——这也是为什么"用 AOP 手动实现一个 `@Transactional`"总差一点：你能复制代理和拦截，但复制不了跨方法、跨 Bean、可嵌套传播的事务语义。**

反过来说也成立：**理解了 AOP 的四个阶段——装配（注册 Advisor）、代理创建（`wrapIfNecessary`）、链的匹配（pointcut + methodCache）、执行（`proceed`）——你手里就有了一把能同时打开"事务失效"和"切面失效"两把锁的钥匙。** 这也是为什么系列导读一开始要用一张调用链图把两个话题合并：它们本来就是一回事，只是在 Spring 的代码里被拆到了 `spring-aop` 和 `spring-tx` 两个模块。

回到开头那段代码，现在可以准确地说出它错在哪：

- `this.saveOrder(dto)` 是目标对象内部调用，不经过代理——**失效点 2**，链条在"运行阶段入口"就断了。
- 把它挪到 `createOrder` 上能生效，是因为 `createOrder` 是被外部（Controller）通过代理调用的。
- 如果坚持要拆两个方法，正确做法是把 `saveOrder` 挪到另一个 Bean（6.1 节方案一），而不是靠 `AopContext` 或者给类加 `@Transactional` 来"糊"过去。

---

## 延伸阅读

- [Spring 接入点：跟着容器过一生](/2026/09/11/Spring-接入点：跟着容器过一生/) —— `BeanPostProcessor` 在容器生命周期里的位置，是理解第三部分（第二篇）"代理在什么时候生成"的前置知识。
- [Spring 中 MyBatis、MyBatis-Plus 是怎么样结合工作的？](/2026/09/10/Spring中Mybatis、MybatisPlus是怎么样结合工作的？/) —— 事务和 `SqlSession` 的复用关系：**没有 Spring 事务时，每次 Mapper 调用都是独立的 `SqlSession`；有事务时，`SqlSession` 跟着事务走。** 这是"事务影响的不只是 SQL 提交"的最好例子。
- [Spring 自定义 MyBatis 数据源，到底发生了什么？](/2026/09/10/Spring自定义Mybatis数据源，到底发生了什么？/) —— 多数据源场景下，每个数据源要配一套 `DataSource` + `SqlSessionFactory` + `TransactionManager`，正好对应 4.5 节（第二篇）"按类型取管理器会炸"的问题。
- [深入理解 Spring WebFlux 与 Reactor Netty](/2026/09/15/深入理解-Spring-WebFlux-与-Reactor-Netty/) —— 理解 7.3 节"为什么响应式事务必须换一套上下文"，得先理解 Reactor 的线程模型。

接下来值得深挖的两个方向：

- **`TransactionSynchronization` 的完整生命周期**：`beforeCommit` 抛异常会发生什么、`afterCommit` 抛异常为什么不会回滚、多个同步器的执行顺序。
- **`AbstractRoutingDataSource` + 事务**：动态数据源切换为什么在"事务开启后"失效——因为 `ConnectionHolder` 在 `doBegin` 时就已经按 `DataSource` 绑定了，切换发生在拿到连接之后。这个坑的本质，仍然是第三篇第五部分那张 ThreadLocal 表。

## 小结

第六到第八部分做的是"回头查账"：把十二种现象钉回链路，给自调用和 `@Async` 可落地的修法，最后确认事务与 AOP 本是同一套机制。定位顺序也固定了——**先问"是不是代理"，再问"链上有没有事务 Advisor"，最后问"当前线程有没有真事务"。**

> **系列入口**
>
> - [系列导读](/2026/10/16/Transactional-为什么会失效-从-AOP-代理链看-Spring-事务机制/)
> - [第一篇](/2026/10/16/Transactional-为什么会失效（一）：从注解到-Advisor-的装配/)、[第二篇](/2026/10/16/Transactional-为什么会失效（二）：代理的生成与调用进链/)、[第三篇](/2026/10/16/Transactional-为什么会失效（三）：事务管理器的-ThreadLocal-账本/)、[第四篇](/2026/10/16/Transactional-为什么会失效（四）：失效原因映射与收束/)（本篇）
>
> 导读开头那 8 个问题到此合拢：注解怎么变成代理、事务与 `@Aspect` 是否同源，看第一、二篇；"当前事务"存在哪、内层 `readOnly` 为何失效，看第三篇；切面与事务谁先执行，见 7.2 节；失效断在哪一环，看第六部分那张表。
