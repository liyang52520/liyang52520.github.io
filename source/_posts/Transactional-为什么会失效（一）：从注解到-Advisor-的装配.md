---
title: "@Transactional 为什么会失效（一）：从注解到 Advisor 的装配"
date: 2026-10-16 16:25:31
tags:
  - spring
  - aop
  - transaction
categories:
  - develop
---

## 引子：注解挂上去了，容器里到底多出了什么

导读把四个阶段铺在了一张图上，这一篇只走第一步：**装配**。在任何业务 Bean 被创建之前，容器必须先准备好三件事——由谁来判断"这个 Bean 要不要包代理"、包上之后执行什么、以及怎么判断某个方法要不要被拦。

日常卡住的地方，恰恰都在这三件事上：老文章说要加 `@EnableTransactionManagement`，可从没加过它，事务照样能用；`@Aspect` 切面和事务写在同一个项目里，说不清它们是不是各有一套代理创建器；读到"`protected` 方法上的 `@Transactional` 不生效"，自己试了一下居然回滚了。这些疑问的答案都不在运行期，而在启动时多出来的那几个 Bean 里。

> 本篇接[系列导读](/2026/10/16/Transactional-为什么会失效-从-AOP-代理链看-Spring-事务机制/)那张四阶段图，覆盖第一、二部分：事务的三个组件分别对应 AOP 的哪个概念、`@EnableTransactionManagement` 导入的两个类各做什么、`AutoProxyRegistrar` 与 `ProxyTransactionManagementConfiguration` 注册了哪些 Bean，以及 Boot 自动配置为什么默认用 CGLIB。口径同导读：Spring Framework 6.1.8 + Spring Boot 3.3.x（Java 17）。

> 本篇要回答的问题：
>
> - 事务的 Advisor、Pointcut、Advice 分别是谁？它和你自己写的 `@Aspect` 是同一套机制吗？
> - Boot 项目里从不手写 `@EnableTransactionManagement`，事务是谁打开的？
> - `AutoProxyRegistrar` 到底注册了什么？为什么这个 Bean 名字值得记住？
> - 事务和 `@Aspect` 是"两个代理创建器并存"，还是"抢同一个 Bean 名"？
> - `spring.aop.proxy-target-class` 这一个开关，为什么能同时决定事务和切面的代理方式？
> - 只配了数据源却没生成 `TransactionManager` 时，`@Transactional` 为什么会全程静默失效？

## 第一部分：先看骨架——事务就是一条普通的 AOP 通知

### 1.1 三个组件，一次对照

Spring 的事务没有任何"底层魔法类"。它老老实实按 AOP 的标准分工拆成了三块，用的全是 AOP 的公开接口：

| AOP 概念 | 事务里的实现类 | 职责 |
|---|---|---|
| `Advisor` | `BeanFactoryTransactionAttributeSourceAdvisor` | 把"切哪里"和"切了做什么"绑在一起 |
| `Pointcut` | `TransactionAttributeSourcePointcut` | 判断某个方法要不要被切 |
| `Advice`（通知） | `TransactionInterceptor` | 真正干活的拦截器 |
| 装配者 | `ProxyTransactionManagementConfiguration` | 把上面三个注册成 Bean |

对比一下你自己写的切面：

```java
@Aspect
@Component
public class LogAspect {

    @Around("@annotation(com.example.Log)")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.nanoTime();
        try {
            return pjp.proceed();
        } finally {
            System.out.println(pjp.getSignature() + " 耗时 " + (System.nanoTime() - start) + " ns");
        }
    }
}
```

`@Aspect` 走的是"注解 → 解析 → 生成 `Advisor`"这条线，由 `AnnotationAwareAspectJAutoProxyCreator` 在启动时完成解析。

`@Transactional` **不走注解解析**。`@EnableTransactionManagement` 直接把一个写好的 `Advisor` 注册成 Bean，然后由自动代理创建器把它捞出来。

两条路，出口完全一样：**容器里多了一个 `Advisor`**。这是两者"高度相似"的真正原因——它们不是相似，它们是同一套机制的两个客户。

### 1.2 一次事务调用的完整链路

把整条链路先画出来，后面的每一个小节都在往这张图上挂细节：

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

自调用（`this.saveOrder()`）、`new` 出来的对象、`final` 方法、`private` 方法，全都断在"运行阶段入口"之前——它们**根本没经过代理**。

### 1.3 把"失效"收敛成四种可能

有了这张图，所有"事务不生效"都能归到下面四类之一，排查顺序也就有了：

1. **代理没创建**：Bean 没被 `BeanPostProcessor` 加工，或者 Advisor 没匹配上这个类。
2. **调用没走代理**：方法内部调用自己、`new` 出来的实例、`final` / `private` / `static` 方法。
3. **走通了代理，但属性是 `null`**：注解所在的方法不是被调用时的方法（接口注解 + CGLIB 就是典型）。
4. **事务开了，但结果不符合预期**：异常类型不在回滚规则里、异常被自己吞了、传播行为选错、多线程丢了上下文。

下面按启动顺序，一环一环看。

---

## 第二部分：装配阶段——`@Transactional` 是怎么变成一个 Advisor 的

### 2.1 入口：`@EnableTransactionManagement` 到底导入了什么

先纠正一个常见误解：**Spring Boot 项目里不需要手写 `@EnableTransactionManagement`**，自动配置已经帮你做了（2.5 节）。但无论谁触发，最终都会走到这个选择器：

```java
// org.springframework.transaction.annotation.TransactionManagementConfigurationSelector
@Override
protected String[] selectImports(AdviceMode adviceMode) {
    return switch (adviceMode) {
        case PROXY -> new String[] {AutoProxyRegistrar.class.getName(),
                ProxyTransactionManagementConfiguration.class.getName()};
        case ASPECTJ -> new String[] {determineTransactionAspectClass()};
    };
}
```

两个类，对应两件事：

- `AutoProxyRegistrar`：注册"给 Bean 包代理"的自动代理创建器。
- `ProxyTransactionManagementConfiguration`：注册事务自己的 Advisor、属性源、拦截器。

注意 `case ASPECTJ` 是另一条完全不同的路（编译期/加载期织入，不需要代理）。这条线在第七部分（第四篇）单独说，默认情况下你走的是 `PROXY`。

### 2.2 `AutoProxyRegistrar`：注册那个"包代理"的 BeanPostProcessor

```java
// org.springframework.context.annotation.AutoProxyRegistrar#registerBeanDefinitions
// （只保留遍历注解那段循环体里 mode == PROXY 的一个分支）
if (mode == AdviceMode.PROXY) {
    AopConfigUtils.registerAutoProxyCreatorIfNecessary(registry);
    if ((Boolean) proxyTargetClass) {
        AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
        return;
    }
}
```

`registerAutoProxyCreatorIfNecessary` 注册的是 `InfrastructureAdvisorAutoProxyCreator`，Bean 名字是固定常量：

```java
// org.springframework.aop.config.AopConfigUtils
public static final String AUTO_PROXY_CREATOR_BEAN_NAME =
        "org.springframework.aop.config.internalAutoProxyCreator";
```

这个 Bean 就是第三部分的主角。它本质是个 `BeanPostProcessor`，在每个 Bean 完成初始化后决定"要不要包一层代理"。

**请记住这个 Bean 名字。** 后面第七部分（第四篇）会看到，事务和 `@Aspect` 抢的正是这一个名字。

### 2.3 `ProxyTransactionManagementConfiguration`：事务的三个核心 Bean

```java
// org.springframework.transaction.annotation.ProxyTransactionManagementConfiguration
@Bean(name = TransactionManagementConfigUtils.TRANSACTION_ADVISOR_BEAN_NAME)
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public BeanFactoryTransactionAttributeSourceAdvisor transactionAdvisor(
        TransactionAttributeSource transactionAttributeSource, TransactionInterceptor transactionInterceptor) {

    BeanFactoryTransactionAttributeSourceAdvisor advisor = new BeanFactoryTransactionAttributeSourceAdvisor();
    advisor.setTransactionAttributeSource(transactionAttributeSource);
    advisor.setAdvice(transactionInterceptor);          // ← Advice 就是拦截器本身
    if (this.enableTx != null) {
        advisor.setOrder(this.enableTx.<Integer>getNumber("order"));
    }
    return advisor;
}

@Bean
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public TransactionAttributeSource transactionAttributeSource() {
    // Accept protected @Transactional methods on CGLIB proxies, as of 6.0.
    return new AnnotationTransactionAttributeSource(false);
}

@Bean
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public TransactionInterceptor transactionInterceptor(TransactionAttributeSource transactionAttributeSource) {
    TransactionInterceptor interceptor = new TransactionInterceptor();
    interceptor.setTransactionAttributeSource(transactionAttributeSource);
    if (this.txManager != null) {
        interceptor.setTransactionManager(this.txManager);
    }
    return interceptor;
}
```

三个关键点：

**第一，Advisor 的 `Advice` 就是 `TransactionInterceptor`。** 也就是说从 AOP 的角度看，事务只是一段 `MethodInterceptor`。它和你的 `@Around` 方法在结构上没有区别，区别只在方法体里干什么。

**第二，`new AnnotationTransactionAttributeSource(false)` 里的 `false` 是 Spring 6.0 的新行为。** 这个参数是 `publicMethodsOnly`：为 `true` 时，`computeTransactionAttribute` 遇到非 public 方法直接返回 `null`。5.3 的调用点是 `new AnnotationTransactionAttributeSource()`，即无参构造 → `this(true)`；Spring 6.0 把它改成显式传 `false`，源码注释写得很清楚："Accept protected @Transactional methods on CGLIB proxies, as of 6.0." **要注意变的是调用点，不是类本身的默认值——`AnnotationTransactionAttributeSource` 的无参构造至今仍然是 `this(true)`。**

（这段差异后面在第二篇 3.5 节展开，它正好解释了"`protected` 方法上的 `@Transactional` 到底生不生效"这个被传了很多年的错误结论。）

**第三，`advisor.setOrder(...)` 的值来自 `@EnableTransactionManagement` 的 `order` 属性，默认是 `Ordered.LOWEST_PRECEDENCE`：**

```java
// org.springframework.transaction.annotation.EnableTransactionManagement
int order() default Ordered.LOWEST_PRECEDENCE;
```

顺序问题在第七部分（第四篇）展开。

### 2.4 事务和 `@Aspect` 用的是同一个代理创建器

这是全文最关键的一段装配代码。`AopConfigUtils` 里维护了一张优先级表：

```java
// org.springframework.aop.config.AopConfigUtils
private static final List<Class<?>> APC_PRIORITY_LIST = new ArrayList<>(3);

static {
    APC_PRIORITY_LIST.add(InfrastructureAdvisorAutoProxyCreator.class);      // 优先级 0
    APC_PRIORITY_LIST.add(AspectJAwareAdvisorAutoProxyCreator.class);        // 优先级 1
    APC_PRIORITY_LIST.add(AnnotationAwareAspectJAutoProxyCreator.class);     // 优先级 2
}
```

而注册逻辑是"同名则取优先级更高者"：

```java
// org.springframework.aop.config.AopConfigUtils#registerOrEscalateApcAsRequired
private static BeanDefinition registerOrEscalateApcAsRequired(
        Class<?> cls, BeanDefinitionRegistry registry, @Nullable Object source) {

    Assert.notNull(registry, "BeanDefinitionRegistry must not be null");

    if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
        BeanDefinition apcDefinition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
        if (!cls.getName().equals(apcDefinition.getBeanClassName())) {
            int currentPriority = findPriorityForClass(apcDefinition.getBeanClassName());
            int requiredPriority = findPriorityForClass(cls);
            if (currentPriority < requiredPriority) {
                apcDefinition.setBeanClassName(cls.getName());    // ← 直接改写 BeanDefinition
            }
        }
        return null;
    }
    // ... 首次注册，order 设为 HIGHEST_PRECEDENCE
}
```

把这张表摊开看：

| 注册来源 | 注册的 APC 类 | 优先级 |
|---|---|---|
| `@EnableTransactionManagement` | `InfrastructureAdvisorAutoProxyCreator` | 0 |
| `@AspectJ` 相关（一般由上一个升上来） | `AspectJAwareAdvisorAutoProxyCreator` | 1 |
| `@EnableAspectJAutoProxy`（Boot 默认有） | `AnnotationAwareAspectJAutoProxyCreator` | 2 |

结论只有一句话：**它们不是"两个代理创建器并存"，而是"抢同一个 Bean 名，高优先级者胜出"。**

所以在一个正常的 Spring Boot 项目里，只要类路径上有 AspectJ（`spring-boot-starter-aop` 会带进来），最终生效的 `internalAutoProxyCreator` 就是 `AnnotationAwareAspectJAutoProxyCreator`——**事务用的也是它**。

这一点有两个直接后果，都是实战里会踩到的：

- 事务 Advisor 和你的 `@Aspect` Advisor 会被**同一个顾问链**收集、排序，所以它们的执行顺序是一个需要一起考虑的问题（第七部分，第四篇）。
- 代理方式（JDK 还是 CGLIB）是**全局的**，因为整个容器只有这一个自动代理创建器。

### 2.5 Boot 自动配置：默认 CGLIB，而且这个开关是共用的

Spring Boot 里负责这件事的是 `TransactionAutoConfiguration`：

```java
// org.springframework.boot.autoconfigure.transaction.TransactionAutoConfiguration
@AutoConfiguration
@ConditionalOnClass(PlatformTransactionManager.class)
public class TransactionAutoConfiguration {

    // ...

    @Configuration(proxyBeanMethods = false)
    @ConditionalOnBean(TransactionManager.class)
    @ConditionalOnMissingBean(AbstractTransactionManagementConfiguration.class)
    public static class EnableTransactionManagementConfiguration {

        @Configuration(proxyBeanMethods = false)
        @EnableTransactionManagement(proxyTargetClass = false)
        @ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "false")
        public static class JdkDynamicAutoProxyConfiguration {
        }

        @Configuration(proxyBeanMethods = false)
        @EnableTransactionManagement(proxyTargetClass = true)
        @ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "true",
                matchIfMissing = true)
        public static class CglibAutoProxyConfiguration {
        }
    }
}
```

三个细节值得停下来看：

**`@ConditionalOnBean(TransactionManager.class)`**——只有容器里已经存在事务管理器时，事务的自动代理才生效。反过来说：**如果你只加了 `spring-boot-starter-jdbc` 却自己配了奇怪的数据源、导致 `TransactionManager` Bean 没生成，那 `@EnableTransactionManagement` 这一整套都不会装配**，`@Transactional` 全程静默失效——连 `TransactionInterceptor` 都不会注册，日志里什么都看不到。

**`matchIfMissing = true`**——`spring.aop.proxy-target-class` 不配置时，走上面那个 `CglibAutoProxyConfiguration`，也就是 **`proxyTargetClass = true`，默认用 CGLIB**。

**同一个属性开关同时管着切面**。看 `AopAutoConfiguration`：

```java
// org.springframework.boot.autoconfigure.aop.AopAutoConfiguration
@AutoConfiguration
@ConditionalOnProperty(prefix = "spring.aop", name = "auto", havingValue = "true", matchIfMissing = true)
public class AopAutoConfiguration {

    @Configuration(proxyBeanMethods = false)
    @ConditionalOnClass(Advice.class)
    static class AspectJAutoProxyingConfiguration {

        @Configuration(proxyBeanMethods = false)
        @EnableAspectJAutoProxy(proxyTargetClass = false)
        @ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "false")
        static class JdkDynamicAutoProxyConfiguration {
        }

        @Configuration(proxyBeanMethods = false)
        @EnableAspectJAutoProxy(proxyTargetClass = true)
        @ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "true",
                matchIfMissing = true)
        static class CglibAutoProxyConfiguration {
        }
    }
}
```

两边读的是同一个配置项，写的是同一个 `internalAutoProxyCreator` BeanDefinition 的 `proxyTargetClass` 属性（`forceAutoProxyCreatorToUseClassProxying` 里就是 `definition.getPropertyValues().add("proxyTargetClass", Boolean.TRUE)`）。

这就意味着：**代理方式是全局唯一的，事务和切面共用一个开关。**

但要把机制说准，这里有两个容易想错的点。

**第一，这个属性只会被写成 `TRUE`，没有任何代码把它写成 `FALSE`。** `forceAutoProxyCreatorToUseClassProxying` 的全部内容就是 `definition.getPropertyValues().add("proxyTargetClass", Boolean.TRUE)`；而 `AutoProxyRegistrar` 和 `AspectJAutoProxyRegistrar` 都只在注解属性为 `true` 时才调用它。所以不存在"后写覆盖先写、取值看注册先后"的竞争——**只要任何一处要求 CGLIB，结果就是 CGLIB。**

**第二，手写 `@EnableAspectJAutoProxy(proxyTargetClass = false)` 不足以切回 JDK 代理。** `AopAutoConfiguration` 与事务自动配置彼此独立，只要 `spring.aop.auto` 没关（默认就是开的），它就仍然按 `spring.aop.proxy-target-class` 决定行为，而这个属性默认是 `true`（`matchIfMissing = true`），照样强制 CGLIB。真正能整套切到 JDK 代理的只有显式配置：

```properties
spring.aop.proxy-target-class=false
```

配了它，`AopAutoConfiguration` 和 `TransactionAutoConfiguration` 才会一起走各自的 `JdkDynamicAutoProxyConfiguration` 分支，此时不会有任何一方调用 `forceAutoProxyCreatorToUseClassProxying`。代价也直接：**目标类必须实现接口，且注解要能被解析到接口方法上**（第二篇 3.5 节）。

**没有 AspectJ 依赖时的那条路更能说明问题**——`AopAutoConfiguration` 里还有第三个分支：

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnMissingClass("org.aspectj.weaver.Advice")
@ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "true",
        matchIfMissing = true)
static class ClassProxyingConfiguration {

    @Bean
    static BeanFactoryPostProcessor forceAutoProxyCreatorToUseClassProxying() {
        return (beanFactory) -> {
            if (beanFactory instanceof BeanDefinitionRegistry registry) {
                AopConfigUtils.registerAutoProxyCreatorIfNecessary(registry);
                AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
            }
        };
    }
}
```

**没有 AspectJ 依赖时，Boot 会主动帮你注册一个 `InfrastructureAdvisorAutoProxyCreator` 并强制 CGLIB。** 这解释了一个常见现象：项目里明明没有 `@Aspect`，事务照样能用。

到这里装配阶段就结束了：容器里多了三样东西——一个 APC、一个事务 Advisor、一个 `TransactionInterceptor`。接下来看它们怎么作用到具体的 Bean 上。

## 小结

这一篇走完了装配阶段：`@EnableTransactionManagement` 经由 `TransactionManagementConfigurationSelector` 导入两个类，`AutoProxyRegistrar` 注册出那个名为 `internalAutoProxyCreator` 的自动代理创建器，`ProxyTransactionManagementConfiguration` 把事务 Advisor、属性源与 `TransactionInterceptor` 注册成 Bean。到这一步，容器里只多了三样东西——**它们都还没有作用到任何业务 Bean 上**。

真正把 Advisor 匹配到具体 Bean、并决定"用 JDK 还是 CGLIB 包一层"的，是下一篇：`wrapIfNecessary` 的三道闸门、`TransactionAttributeSourcePointcut` 那两行判断，以及"代理在、事务却没生效"是怎么发生的。

> **下一篇**
>
> [（二）：代理的生成与调用进链](/2026/10/16/Transactional-为什么会失效（二）：代理的生成与调用进链/)——从 `wrapIfNecessary` 一路读到 `invokeWithinTransaction`。系列入口见[导读](/2026/10/16/Transactional-为什么会失效-从-AOP-代理链看-Spring-事务机制/)。
