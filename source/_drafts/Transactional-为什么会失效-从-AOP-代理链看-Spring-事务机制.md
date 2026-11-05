---
title: "@Transactional 为什么会失效？从 AOP 代理链看 Spring 事务机制"
date: 2026-10-16 16:25:30
tags:
  - spring
  - aop
  - transaction
categories:
  - develop
---

> 这是一个四篇的系列。本篇是导读：先把 AOP 的四个阶段和五类失效点铺在一张图上，再按篇给出入口。后续四篇的节号连续（第一~第八部分），跨篇引用"第七部分"时，翻到对应那一篇就能对上。

## 引子：注解明明在那，为什么没回滚

一个再普通不过的下午：订单服务要"扣库存 + 落订单"。

```java
@Service
public class OrderService {

    @Autowired
    private StockService stockService;

    // 没有 @Transactional
    public void createOrder(OrderDTO dto) {
        stockService.deduct(dto.getSkuId(), dto.getCount());
        this.saveOrder(dto);        // 同类内部调用
    }

    @Transactional
    public void saveOrder(OrderDTO dto) {
        orderMapper.insert(dto);
        throw new RuntimeException("模拟落库失败");
    }
}
```

跑一遍，库存扣了，订单也落了。`@Transactional` 就在 `saveOrder` 上，字面意义上没写错。

于是开始怀疑人生：注解是不是根本没生效？是不是 Spring 版本有问题？是不是要加 `@EnableTransactionManagement`？换了个写法，把 `@Transactional` 挪到 `createOrder` 上，回滚了。问题"解决"了，但为什么，说不清。

说不清的根源是：**大多数人把 `@Transactional` 当成一个"给方法打上的标记"，而它实际上是一个"要求 Spring 用代理对象包住这个 Bean"的请求。** 标记打上了，代理没生成，或者调用没走代理，标记就只是一段没人看的元数据。

换句话说：**事务问题，绝大多数时候就是 AOP 问题。** AOP 的那条链条在哪一环断了，事务就在哪一环失效——而且必然以同样的方式失效。把这条链条完整看清楚，事务失效就从一个"玄学问题"变成一个"定位问题"。

> 这篇文章要回答的问题：
>
> - 一个 `@Transactional` 注解，是怎么变成"会开事务的代理对象"的？
> - 它和 `@Aspect` 切面用的是同一套代理机制吗？会互相影响吗？
> - "当前事务"到底存在哪里？为什么换个线程就找不到了？
> - 七种传播行为，在源码里分别对应哪几行分支？
> - 内层方法标了 `readOnly = true`，为什么根本没生效？
> - `UnexpectedRollbackException` 到底是谁抛出来的？
> - 事务切面和普通切面，谁的代码先执行？
> - 每一种"事务失效"，对应链条里的哪一环？

文中源码以 **Spring Framework 6.1.8 + Spring Boot 3.3.x**（Java 17）为基准，与 5.3 行为不同的地方会单独标出。如果你还没读过 [Spring 接入点：跟着容器过一生](/2026/09/11/Spring-接入点：跟着容器过一生/)，建议先扫一眼——那里讲了 `BeanPostProcessor` 在容器里的位置，这正是本系列第三部分（[第二篇](/2026/10/16/Transactional-为什么会失效（二）：代理的生成与调用进链/)）的地基。

## 全链路图：一次事务调用的四个阶段

这张图是后面四篇共同的坐标系——**装配 → 代理创建 → 链的匹配 → 执行**。图中的五个"失效点"会在[第四篇](/2026/10/16/Transactional-为什么会失效（四）：失效原因映射与收束/)里逐条对应到具体现象。

```text
启动阶段
  @EnableTransactionManagement
    ├─ AutoProxyRegistrar
    │    └─ 注册 InfrastructureAdvisorAutoProxyCreator（APC，一个 BeanPostProcessor）
    └─ ProxyTransactionManagementConfiguration
         ├─ BeanFactoryTransactionAttributeSourceAdvisor   ← 事务 Advisor
         ├─ AnnotationTransactionAttributeSource           ← 判断"方法上有没有事务属性"
         └─ TransactionInterceptor                         ← 事务拦截器

  Bean 创建
    └─ APC.postProcessAfterInitialization(bean)
         ├─ wrapIfNecessary()
         │    ├─ isInfrastructureClass / shouldSkip ?       → 是，直接返回原对象
         │    ├─ getAdvicesAndAdvisorsForBean()             → 用 Advisor 匹配这个类
         │    │    └─ 不匹配 → DO_NOT_PROXY，返回原对象      ← 失效点 1：代理没创建
         │    └─ createProxy()
         │         └─ JDK 动态代理 / CGLIB 子类              ← 换成代理对象放回容器

运行阶段（外部调用入口）
  Controller → 注入的其实是被代理过的 OrderService
    └─ JdkDynamicAopProxy.invoke() / CglibAopProxy 的拦截器
         ├─ methodCache 里找这个方法对应的拦截器链
         │    └─ 链为空 → 直接调用目标方法                    ← 失效点 2：没走到事务
         └─ ReflectiveMethodInvocation.proceed()
              └─ TransactionInterceptor.invoke()
                   └─ invokeWithinTransaction()
                        ├─ getTransactionAttribute(method, targetClass)
                        │    └─ 为 null → 不开事务                 ← 失效点 3：属性解析为空
                        ├─ determineTransactionManager()
                        │    └─ 为 null → 只打一行 debug 日志       ← 失效点 4：没有事务管理器
                        ├─ tm.getTransaction(txAttr)               ← 开事务 / 加入已有事务
                        ├─ invocation.proceedWithInvocation()      ← 执行你的业务代码
                        ├─ 抛异常 → completeTransactionAfterThrowing()
                        │    └─ rollbackOn(ex) ? rollback : commit ← 失效点 5：异常不回滚
                        └─ 正常返回 → commitTransactionAfterReturning()
```

## 这个系列怎么读

四个阶段的边界就是拆篇的边界，节号没有重新编号：

| 篇目 | 节 | 这一篇回答什么 |
|:---|:---|:---|
| [第一篇：从注解到 Advisor 的装配](/2026/10/16/Transactional-为什么会失效（一）：从注解到-Advisor-的装配/) | 一、二 | 事务的三个组件分别对应 AOP 的哪个概念；`@EnableTransactionManagement` 导入了什么；`AutoProxyRegistrar` 与 `ProxyTransactionManagementConfiguration` 注册了哪些 Bean；Boot 自动配置为什么默认 CGLIB |
| [第二篇：代理的生成与调用进链](/2026/10/16/Transactional-为什么会失效（二）：代理的生成与调用进链/) | 三、四 | `wrapIfNecessary` 的三道闸门、pointcut 怎么判断"方法上有没有事务属性"、JDK 与 CGLIB 如何决定注解写在哪才有效、调用如何穿过拦截器链进入 `invokeWithinTransaction` |
| [第三篇：事务管理器的 ThreadLocal 账本](/2026/10/16/Transactional-为什么会失效（三）：事务管理器的-ThreadLocal-账本/) | 五 | "当前事务"存在哪六个 ThreadLocal 里、七种传播行为对应哪几行分支、内层属性为什么被静默忽略、提交与回滚的完成逻辑、事务生命周期回调 |
| [第四篇：失效原因映射与收束](/2026/10/16/Transactional-为什么会失效（四）：失效原因映射与收束/) | 六、七、八 | 十二种失效现象分别断在哪一环、自调用/`@Async` 的定位与修法、事务切面与普通切面的四处本质差异、响应式与 AspectJ 两条支线、全系列收束 |

三个常用入口：

- **只想给一个失效现象定位**：直接去[第四篇](/2026/10/16/Transactional-为什么会失效（四）：失效原因映射与收束/)第六部分那张"现象 → 断在哪一环"的表。
- **想弄清"当前事务到底存在哪"**：[第三篇](/2026/10/16/Transactional-为什么会失效（三）：事务管理器的-ThreadLocal-账本/) 5.1 节那张六行 ThreadLocal 表是全系列的账本。
- **想弄清"注解为什么没变成代理"**：[第一篇](/2026/10/16/Transactional-为什么会失效（一）：从注解到-Advisor-的装配/)的第二部分讲怎么装配出 Advisor，[第二篇](/2026/10/16/Transactional-为什么会失效（二）：代理的生成与调用进链/)的第三部分讲它怎么匹配到具体的 Bean。

> 四篇各自独立成文，但都建立在这张图上。如果某一段读起来突兀，多半是它依赖了前一节；顺着读一遍，再回头查表。
