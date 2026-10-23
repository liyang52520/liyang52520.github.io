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

文中源码以 **Spring Framework 6.1.8 + Spring Boot 3.3.x**（Java 17）为基准，与 5.3 行为不同的地方会单独标出。如果你还没读过 [Spring 接入点：跟着容器过一生](/2026/09/11/Spring-接入点：跟着容器过一生/)，建议先扫一眼——那里讲了 `BeanPostProcessor` 在容器里的位置，这正是下面第三部分的地基。

---

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

注意 `case ASPECTJ` 是另一条完全不同的路（编译期/加载期织入，不需要代理）。这条线在第七部分单独说，默认情况下你走的是 `PROXY`。

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

**请记住这个 Bean 名字。** 后面第七部分会看到，事务和 `@Aspect` 抢的正是这一个名字。

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

（这段差异后面在 3.5 节展开，它正好解释了"`protected` 方法上的 `@Transactional` 到底生不生效"这个被传了很多年的错误结论。）

**第三，`advisor.setOrder(...)` 的值来自 `@EnableTransactionManagement` 的 `order` 属性，默认是 `Ordered.LOWEST_PRECEDENCE`：**

```java
// org.springframework.transaction.annotation.EnableTransactionManagement
int order() default Ordered.LOWEST_PRECEDENCE;
```

顺序问题在第七部分展开。

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

- 事务 Advisor 和你的 `@Aspect` Advisor 会被**同一个顾问链**收集、排序，所以它们的执行顺序是一个需要一起考虑的问题（第七部分）。
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

配了它，`AopAutoConfiguration` 和 `TransactionAutoConfiguration` 才会一起走各自的 `JdkDynamicAutoProxyConfiguration` 分支，此时不会有任何一方调用 `forceAutoProxyCreatorToUseClassProxying`。代价也直接：**目标类必须实现接口，且注解要能被解析到接口方法上**（3.5 节）。

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

---

## 第三部分：代理创建阶段——哪些 Bean 会被包一层

### 3.1 时机：Bean 初始化的最后一步

自动代理创建器的核心方法就两个，先看调用点：

```java
// org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator
@Override
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
    if (bean != null) {
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        if (this.earlyBeanReferences.remove(cacheKey) != bean) {
            return wrapIfNecessary(bean, beanName, cacheKey);
        }
    }
    return bean;
}
```

绕开 `BeanPostProcessor` 的整体列表，只看**单个 Bean 的一生**，顺序是这样的：

```text
实例化（构造器 / 工厂方法）
  → 属性填充（依赖注入）
  → BeanPostProcessor.postProcessBeforeInitialization
  → @PostConstruct / InitializingBean.afterPropertiesSet()      ← 你自己的初始化逻辑在这
  → BeanPostProcessor.postProcessAfterInitialization            ← 代理在这里诞生
  → 这个 Bean 进入容器，可被别人注入
```

**代理是在 `@PostConstruct`、`afterPropertiesSet()` 全部执行完之后才生成的。** 所以 Bean 自己的初始化方法里，`this` 永远不是代理——它拿到的是刚造好的目标对象；而别人注入你的 Bean 时，拿到的才是代理。

还有一层细节：注册 APC 时，`AopConfigUtils` 会把它的 `order` 设成 `Ordered.HIGHEST_PRECEDENCE`：

```java
// org.springframework.aop.config.AopConfigUtils#registerOrEscalateApcAsRequired
// （只保留"首次注册"这一段；已存在同名定义时走"取优先级更高者"的分支，见 2.4）
RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);
beanDefinition.setSource(source);
beanDefinition.getPropertyValues().add("order", Ordered.HIGHEST_PRECEDENCE);
beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
registry.registerBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME, beanDefinition);
```

这个值来自 `ProxyProcessorSupport`（`AbstractAutoProxyCreator` 的父类）的 `order` 字段，默认本是 `LOWEST_PRECEDENCE`。**调成最高优先级，是为了让自动代理创建器在 `BeanPostProcessor` 列表里排在前面**，这样它的 `postProcessAfterInitialization` 会先于其他后置处理器执行——后面 6.3 节会看到，`@Async` 正是利用"自己排在 APC 之后"这一点，往已经建好的代理里追加 advisor。

> 这也解释了为什么"在同一个 Bean 的方法里调用自己"必然绕过事务：`this` 在编译期就绑定到了目标对象，跟代理没有半毛钱关系。

关于 `BeanPostProcessor` 在容器里的完整位置，可以对照 [Spring 接入点：跟着容器过一生](/2026/09/11/Spring-接入点：跟着容器过一生/) 里的第三幕——那里用几十行手写代理演示了同一个位置。

### 3.2 `wrapIfNecessary`：三道闸门

```java
// org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
        return bean;
    }
    if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
        return bean;
    }
    if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }

    // Create proxy if we have advice.
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
    if (specificInterceptors != DO_NOT_PROXY) {
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
        Object proxy = createProxy(
                bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }

    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
}
```

三道闸门：

- **`targetSourcedBeans`**：这个 Bean 是通过 `TargetSource` 暴露的，不重复包装。
- **`advisedBeans` 缓存**：已经判断过的直接复用结论。注意 `advisedBeans.put(cacheKey, Boolean.FALSE)` 这个"负缓存"——**一个 Bean 一旦被判定为不需要代理，后面再有人问也是"不代理"**。
- **`isInfrastructureClass` / `shouldSkip`**：`Advice`、`Pointcut`、`Advisor`、`AopInfrastructureBean` 这些"基础设施类"本身不代理；`shouldSkip` 在 `AspectJAwareAdvisorAutoProxyCreator` 里还有额外职责——遍历候选 Advisor，**跳过 `@Aspect` 类自己**，否则切面会试图去代理它自己。

过了三道闸门，才是真正找 Advisor：

```java
// org.springframework.aop.framework.autoproxy.AbstractAdvisorAutoProxyCreator
protected Object[] getAdvicesAndAdvisorsForBean(
        Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {

    List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
    if (advisors.isEmpty()) {
        return DO_NOT_PROXY;
    }
    return advisors.toArray();
}
```

返回 `DO_NOT_PROXY` 的那一刻，**代理就不会产生**，这个 Bean 上所有 `@Transactional` 一起失效。这就是失效点 1。

### 3.3 事务 Advisor 的 Pointcut：本质是"方法上有没有事务属性"

`BeanFactoryTransactionAttributeSourceAdvisor` 本身很简单，全部逻辑都在它持有的 pointcut 里：

```java
// org.springframework.transaction.interceptor.BeanFactoryTransactionAttributeSourceAdvisor
public class BeanFactoryTransactionAttributeSourceAdvisor extends AbstractBeanFactoryPointcutAdvisor {

    private final TransactionAttributeSourcePointcut pointcut = new TransactionAttributeSourcePointcut();

    public void setTransactionAttributeSource(TransactionAttributeSource transactionAttributeSource) {
        this.pointcut.setTransactionAttributeSource(transactionAttributeSource);
    }

    @Override
    public Pointcut getPointcut() {
        return this.pointcut;
    }
}
```

这个 pointcut 是整个事务机制里最"反直觉"的一段代码，因为它**只有两行有效逻辑**：

```java
// org.springframework.transaction.interceptor.TransactionAttributeSourcePointcut
final class TransactionAttributeSourcePointcut extends StaticMethodMatcherPointcut implements Serializable {

    @Nullable
    private TransactionAttributeSource transactionAttributeSource;

    public TransactionAttributeSourcePointcut() {
        setClassFilter(new TransactionAttributeSourceClassFilter());
    }

    @Override
    public boolean matches(Method method, Class<?> targetClass) {
        return (this.transactionAttributeSource == null ||
                this.transactionAttributeSource.getTransactionAttribute(method, targetClass) != null);
    }
}
```

**事务的"切点表达式"就是"能解析出事务属性的方法"。** 没有 `@annotation(Transactional)` 这样的表达式，没有 `execution(* com.example..*Service.*(..))`——它就是直接问属性源："这个方法的 `TransactionAttribute` 是 `null` 吗？"

这里藏着一个非常重要的事实：**判断"要不要切"和判断"用哪个事务配置"是同一件事**。所以属性解析为空 = 切不上 = 事务失效，一次调用就能确定因果，不需要分别排查。

类级过滤也做了一次排除：

```java
// org.springframework.transaction.interceptor.TransactionAttributeSourcePointcut 的内部类
private final class TransactionAttributeSourceClassFilter implements ClassFilter {

    @Override
    public boolean matches(Class<?> clazz) {
        if (TransactionalProxy.class.isAssignableFrom(clazz) ||
                TransactionManager.class.isAssignableFrom(clazz) ||
                PersistenceExceptionTranslator.class.isAssignableFrom(clazz)) {
            return false;
        }
        return (transactionAttributeSource == null || transactionAttributeSource.isCandidateClass(clazz));
    }
}
```

`TransactionalProxy`、`TransactionManager`、`PersistenceExceptionTranslator` 这三类被明确排除——**避免给事务基础设施自己套事务代理**，否则会无限套娃。最后还用 `isCandidateClass` 做一次廉价预筛，明显不可能有注解的类（比如各种 JDK 类）直接跳过注解解析。

### 3.4 属性解析：查找顺序决定了"注解写在哪才生效"

`matches` 里调的 `getTransactionAttribute` 是一个带缓存的装饰器，真正的解析在 `computeTransactionAttribute`：

```java
// org.springframework.transaction.interceptor.AbstractFallbackTransactionAttributeSource
@Override
public TransactionAttribute getTransactionAttribute(Method method, @Nullable Class<?> targetClass) {
    if (method.getDeclaringClass() == Object.class) {
        return null;
    }

    Object cacheKey = getCacheKey(method, targetClass);
    TransactionAttribute cached = this.attributeCache.get(cacheKey);
    if (cached != null) {
        return (cached != NULL_TRANSACTION_ATTRIBUTE ? cached : null);
    }
    // ... compute 之后写入 attributeCache，解析不到就写 NULL_TRANSACTION_ATTRIBUTE
}

protected TransactionAttribute computeTransactionAttribute(Method method, @Nullable Class<?> targetClass) {
    // Don't allow non-public methods, as configured.
    if (allowPublicMethodsOnly() && !Modifier.isPublic(method.getModifiers())) {
        return null;
    }

    // The method may be on an interface, but we need attributes from the target class.
    Method specificMethod = AopUtils.getMostSpecificMethod(method, targetClass);

    // First try is the method in the target class.
    TransactionAttribute txAttr = findTransactionAttribute(specificMethod);
    if (txAttr != null) {
        return txAttr;
    }

    // Second try is the transaction attribute on the target class.
    txAttr = findTransactionAttribute(specificMethod.getDeclaringClass());
    if (txAttr != null && ClassUtils.isUserLevelMethod(method)) {
        return txAttr;
    }

    if (specificMethod != method) {
        // Fallback is to look at the original method.
        txAttr = findTransactionAttribute(method);
        if (txAttr != null) {
            return txAttr;
        }
        // Last fallback is the class of the original method.
        txAttr = findTransactionAttribute(method.getDeclaringClass());
        if (txAttr != null && ClassUtils.isUserLevelMethod(method)) {
            return txAttr;
        }
    }

    return null;
}
```

查找顺序拆成一张表：

| 顺序 | 查找目标 | 备注 |
|---|---|---|
| 0 | 非 public 方法直接短路 | 由 `allowPublicMethodsOnly()` 控制（见 3.5） |
| 1 | 目标类的具体方法 | `AopUtils.getMostSpecificMethod` 把接口方法映射到实现类方法 |
| 2 | 该方法的**声明类** | 类级 `@Transactional` 生效点 |
| 3 | 原始方法（通常是接口方法） | 仅当 `specificMethod != method`，即代理接口调用时 |
| 4 | 原始方法的声明类 | 接口上的类级注解 |

这段代码有两个必须记住的推论：

**推论一：`attributeCache` 是 `MethodClassKey` 级别的缓存，而且缓存"没有事务"这个结论。** 所以属性解析只在第一次调用时算一次。你不可能在运行时"动态决定"某个方法要不要事务——除非换掉属性源实现。

**推论二：查找顺序 3、4 只在 `specificMethod != method` 时才走。** 这意味着注解写在哪，和**代理方式**强耦合，下一节展开。

### 3.5 代理方式：JDK 还是 CGLIB，直接决定注解写在哪才有效

先看**代理创建期的类级匹配**。`AopUtils.canApply` 会同时扫描目标类和它实现的所有接口：

```java
// org.springframework.aop.support.AopUtils
public static boolean canApply(Pointcut pc, Class<?> targetClass, boolean hasIntroductions) {
    // ...
    Set<Class<?>> classes = new LinkedHashSet<>();
    if (!Proxy.isProxyClass(targetClass)) {
        classes.add(ClassUtils.getUserClass(targetClass));
    }
    classes.addAll(ClassUtils.getAllInterfacesForClassAsSet(targetClass));

    for (Class<?> clazz : classes) {
        Method[] methods = ReflectionUtils.getAllDeclaredMethods(clazz);
        for (Method method : methods) {
            if (methodMatcher.matches(method, targetClass)) {
                return true;
            }
        }
    }
    return false;
}
```

注意 **`getAllInterfacesForClassAsSet`**——接口方法也会参与匹配。

再看**运行期的按方法匹配**。代理收到调用后，`AdvisedSupport` 按 `method` 去找拦截器链：

```java
// org.springframework.aop.framework.AdvisedSupport
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, @Nullable Class<?> targetClass) {
    // ...
    return this.methodCache.computeIfAbsent(new MethodCacheKey(method), k ->
            this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(this, method, targetClass));
}
```

关键在于这个 `method` 从哪来：

- **JDK 动态代理**：代理实现的是接口，回调拿到的 `method` 就是**接口方法**。属性解析走查找顺序 1 → 2 → 3，第 3 步回退到接口方法，**接口方法上的 `@Transactional` 能找到**。
- **CGLIB 代理**：代理是目标类的子类，回调拿到的 `method` 是**目标类（或父类）声明的方法**，`specificMethod == method`，于是查找顺序 3、4 根本不执行，**只有当实现类方法或实现类上有注解时才生效**。

这正好解释了 Spring 官方文档那句容易被忽略的提醒：`@Transactional` 推荐写在实现类上；写在接口上"只在基于接口的代理时按预期工作"。这不是风格建议，而是 `computeTransactionAttribute` 的分支结构决定的。

而且 CGLIB 这条路上有一个很坑的现象：**代理创建了，但事务没生效**。链条是这样断的：类级匹配扫描了接口方法（`canApply` 返回 true，代理照常生成），但运行期是按方法匹配的，而 CGLIB 回调收到的 `method` 是**被重写的实现类方法**：

```java
// org.springframework.aop.framework.CglibAopProxy.DynamicAdvisedInterceptor
public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
    // ...
    List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
```

于是 `AopUtils.getMostSpecificMethod(method, targetClass)` 返回的就是 `method` 本身，`specificMethod == method` 成立，`computeTransactionAttribute` 里那段"回退到接口方法"的分支根本不会执行 → 属性解析为 `null` → 拦截器链为空。此时你用 `AopUtils.isAopProxy(bean)` 检查，结果是 `true`——代理在，只是这条方法上没有任何通知。

代理方式还决定了哪些方法**在物理上**无法被拦截：

| 方法修饰符 | CGLIB | JDK 代理 | 说明 |
|---|---|---|---|
| `public` | ✅ | ✅ | 只要注解解析得出属性 |
| `protected` | ✅ 需代理类与被代理类同包（Spring 6+ 事务层不再拦截） | ❌ | 接口方法必然 public |
| 包级可见（默认） | ⚠️ 同类加载器下可重写 | ❌ | 跨 ClassLoader 时 CGLIB 无法代理 |
| `private` | ❌ | ❌ | 子类无法重写，直接调目标 |
| `static` | ❌ | ❌ | 不经过实例调用 |
| `final` 方法 | ❌ | — | CGLIB 会打 debug 日志："Final method cannot get proxied via CGLIB" |

关于 `protected`，Spring 6 的行为变化容易踩：

```java
// Spring 6.1.8 的调用点：显式传 false
return new AnnotationTransactionAttributeSource(false);
```

```java
// Spring 5.3.x 的调用点：无参构造 → this(true)，非 public 方法直接不认
return new AnnotationTransactionAttributeSource();
```

所以在 **Spring 6 + CGLIB + `protected` 方法**这个组合下，`@Transactional` 是**能生效的**（前提依旧是 CGLIB 能重写它：生成的代理类要与被代理类同包）；而 5.3 里同样的代码不生效。如果你照着老文章排查"`protected` 方法为什么不回滚"，先确认版本，别急着改代码。

顺带说明 CGLIB 是怎么"知道自己拦不住"的：

```java
// org.springframework.aop.framework.CglibAopProxy#doValidateClass
// （省略了紧随其后的"包级可见且跨 ClassLoader"分支）
if (!Modifier.isStatic(mod) && !Modifier.isPrivate(mod)) {
    if (Modifier.isFinal(mod)) {
        if (logger.isWarnEnabled() && implementsInterface(method, ifcs)) {
            logger.warn("Unable to proxy interface-implementing method [" + method + "] because " +
                    "it is marked as final, consider using interface-based JDK proxies instead.");
        }
        if (logger.isDebugEnabled()) {
            logger.debug("Final method [" + method + "] cannot get proxied via CGLIB: " +
                    "Calls to this method will NOT be routed to the target instance and " +
                    "might lead to NPEs against uninitialized fields in the proxy instance.");
        }
    }
}
```

把日志级别开到 `debug`，"哪些方法拦不住"其实是**Spring 自己会告诉你的**。这比到处加断点靠谱得多。

---

## 第四部分：运行阶段——一次调用穿过代理链

### 4.1 入口：代理对象如何找到拦截器链

以 JDK 代理为例，核心就三行：

```java
// org.springframework.aop.framework.JdkDynamicAopProxy
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    // ...
    if (this.advised.exposeProxy) {
        oldProxy = AopContext.setCurrentProxy(proxy);     // ← AopContext.currentProxy() 的来源
        setProxyContext = true;
    }

    target = targetSource.getTarget();
    Class<?> targetClass = (target != null ? target.getClass() : null);

    // Get the interception chain for this method.
    List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

    if (chain.isEmpty()) {
        // 链为空：直接反射调用目标方法
        Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
        retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
    }
    else {
        MethodInvocation invocation =
                new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
        retVal = invocation.proceed();                    // ← 进入通知链
    }
    // ...
}
```

两处和事务直接相关：

**`chain.isEmpty()` 就走 `invokeJoinpointUsingReflection`。** 这就是失效点 2 的物理形态——代理明明在，只是这个方法上一条通知都没有，于是"和没代理一样"。

**`exposeProxy` 决定了 `AopContext.currentProxy()` 能不能用。** 它是 `@EnableAspectJAutoProxy(exposeProxy = true)` 打开的，默认关闭。很多人靠 `((OrderService) AopContext.currentProxy()).saveOrder(...)` 解决自调用，如果没开这个开关，会直接抛 `IllegalStateException: Cannot find current proxy`——症状看起来像"事务问题"，其实是代理配置问题。

### 4.2 拦截器链的顺序

`ReflectiveMethodInvocation.proceed()` 是一个标准的递归：

```java
// org.springframework.aop.framework.ReflectiveMethodInvocation
public Object proceed() throws Throwable {
    // We start with an index of -1 and increment early...
    if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
        return invokeJoinpoint();
    }

    Object interceptorOrInterceptionAdvice =
            this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
    // ... 动态匹配则再判断一次是否 match
    return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
}
```

链的构建由 `DefaultAdvisorChainFactory` 完成，它拿到的 Advisor 顺序决定了拦截器的顺序，而顺序由每个 Advisor 的 `order` 决定：**升序执行，order 越小越靠外**。**事务 Advisor 的默认 order 是 `Ordered.LOWEST_PRECEDENCE`**，也就是 `int` 的最大值——但先别急着下"事务一定在最内层"的结论：**不写 `@Order` 的切面同样是这个值**，两者并列时相对顺序在源码层面没有任何保证，7.2 节展开。

这一点在第七部分会结合一个具体例子说明。

### 4.3 `TransactionInterceptor.invoke`：只剩转发

```java
// org.springframework.transaction.interceptor.TransactionInterceptor
public class TransactionInterceptor extends TransactionAspectSupport implements MethodInterceptor, Serializable {

    @Override
    @Nullable
    public Object invoke(MethodInvocation invocation) throws Throwable {
        // Work out the target class: may be {@code null}.
        Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

        // Adapt to TransactionAspectSupport's invokeWithinTransaction...
        return invokeWithinTransaction(invocation.getMethod(), targetClass, new CoroutinesInvocationCallback() {
            @Override
            public Object proceedWithInvocation() throws Throwable {
                return invocation.proceed();
            }
            // ...
        });
    }
}
```

它只做一件事：把 AOP 的 `MethodInvocation` 翻译成事务模块自己的 `InvocationCallback`，然后交给 `TransactionAspectSupport.invokeWithinTransaction`。

**这里有一个很容易看漏的细节：`targetClass` 用的是 `AopUtils.getTargetClass(invocation.getThis())`。** 传入属性解析的 `targetClass` 是**目标类**，不是代理类，也不是你声明的那个字段类型。所以 `computeTransactionAttribute` 里的 `getMostSpecificMethod` 才有意义——它要把接口方法映射到这个目标类上的具体实现。

### 4.4 `invokeWithinTransaction`：事务的心脏

```java
// org.springframework.transaction.interceptor.TransactionAspectSupport
protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
        final InvocationCallback invocation) throws Throwable {

    // If the transaction attribute is null, the method is non-transactional.
    TransactionAttributeSource tas = getTransactionAttributeSource();
    final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
    final TransactionManager tm = determineTransactionManager(txAttr);

    // ...（响应式分支见 7.3）

    PlatformTransactionManager ptm = asPlatformTransactionManager(tm);
    final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

    if (txAttr == null || !(ptm instanceof CallbackPreferringPlatformTransactionManager cpptm)) {
        // Standard transaction demarcation with getTransaction and commit/rollback calls.
        TransactionInfo txInfo = createTransactionIfNecessary(ptm, txAttr, joinpointIdentification);

        Object retVal;
        try {
            // This is an around advice: Invoke the next interceptor in the chain.
            retVal = invocation.proceedWithInvocation();
        }
        catch (Throwable ex) {
            // target invocation exception
            completeTransactionAfterThrowing(txInfo, ex);
            throw ex;
        }
        finally {
            cleanupTransactionInfo(txInfo);
        }

        // ...（Future / Vavr 的特判）
        commitTransactionAfterReturning(txInfo);
        return retVal;
    }

    else {
        // It's a CallbackPreferringPlatformTransactionManager: pass a TransactionCallback in.
        // ...（走 ptm.execute(txAttr, callback) 的形式）
    }
}
```

**这段代码就是"事务"的全部语义**，翻译成大白话：

1. 解析事务属性，拿到 `txAttr`。
2. 确定事务管理器 `tm`。
3. `createTransactionIfNecessary`：需要就开（或加入）事务。
4. 调用链上的下一个拦截器，最终执行你的业务方法。
5. 抛异常 → `completeTransactionAfterThrowing`（决定回滚还是提交，然后原样抛出）。
6. 正常返回 → `commitTransactionAfterReturning`。
7. `finally` 里 `cleanupTransactionInfo` 恢复 ThreadLocal 栈。

注意第 5 步：**异常一定会被重新抛出**（`throw ex`）。事务切面本身从不吞异常，你之所以"看不到异常"，是自己写的 `try-catch` 吞掉的——这属于失效点 5。

再看"需要就开"这个措辞的由来：

```java
// org.springframework.transaction.interceptor.TransactionAspectSupport
protected TransactionInfo createTransactionIfNecessary(@Nullable PlatformTransactionManager tm,
        @Nullable TransactionAttribute txAttr, final String joinpointIdentification) {

    // If no name specified, apply method identification as transaction name.
    if (txAttr != null && txAttr.getName() == null) {
        txAttr = new DelegatingTransactionAttribute(txAttr) {
            @Override
            public String getName() {
                return joinpointIdentification;
            }
        };
    }

    TransactionStatus status = null;
    if (txAttr != null) {
        if (tm != null) {
            status = tm.getTransaction(txAttr);
        }
        else {
            if (logger.isDebugEnabled()) {
                logger.debug("Skipping transactional joinpoint [" + joinpointIdentification +
                        "] because no transaction manager has been configured");
            }
        }
    }
    return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
}
```

**`tm == null` 时只打一行 `debug` 日志，然后照常执行业务方法，不报错、不告警。** 这是事务最危险的一种失效方式，也是失效点 4。生产环境通常开 `INFO`，这行日志永远不会出现。

要补一个前提：这个分支要求**拦截器已经装配好、并且事务属性非空**。如果容器里连 `TransactionManager` Bean 都没有，`TransactionAutoConfiguration` 的 `EnableTransactionManagementConfiguration` 根本不会生效（2.5 节），`TransactionInterceptor` 压根不会注册——那才是更彻底的静默，连这行 `debug` 都没有。

想在本地确认这种失效，把日志调成：

```properties
logging.level.org.springframework.transaction.interceptor=TRACE
```

`TRACE` 会打印 `Getting transaction for [全限定方法名]` / `Completing transaction for [...]`，有没有真的开事务一目了然。

### 4.5 事务管理器是怎么被选中的

```java
// org.springframework.transaction.interceptor.TransactionAspectSupport
protected TransactionManager determineTransactionManager(@Nullable TransactionAttribute txAttr) {
    // Do not attempt to lookup tx manager if no tx attributes are set
    if (txAttr == null || this.beanFactory == null) {
        return getTransactionManager();
    }

    String qualifier = txAttr.getQualifier();
    if (StringUtils.hasText(qualifier)) {
        return determineQualifiedTransactionManager(this.beanFactory, qualifier);
    }
    else if (StringUtils.hasText(this.transactionManagerBeanName)) {
        return determineQualifiedTransactionManager(this.beanFactory, this.transactionManagerBeanName);
    }
    else {
        TransactionManager defaultTransactionManager = getTransactionManager();
        if (defaultTransactionManager == null) {
            defaultTransactionManager = this.transactionManagerCache.get(DEFAULT_TRANSACTION_MANAGER_KEY);
            if (defaultTransactionManager == null) {
                defaultTransactionManager = this.beanFactory.getBean(TransactionManager.class);
                this.transactionManagerCache.putIfAbsent(DEFAULT_TRANSACTION_MANAGER_KEY, defaultTransactionManager);
            }
        }
        return defaultTransactionManager;
    }
}
```

优先级很清楚：**`@Transactional(transactionManager = "...")` → 拦截器上配的 Bean 名 → 容器里按类型取**。

最后那个 `beanFactory.getBean(TransactionManager.class)` 是多数据源场景下的经典陷阱：**按类型取，容器里有多个候选就直接抛 `NoUniqueBeanDefinitionException`**。所以多数据源时必须用 `transactionManager` 属性显式指定——这和后端文章 [Spring 中 MyBatis、MyBatis-Plus 是怎么样结合工作的？](/2026/09/10/Spring中Mybatis、MybatisPlus是怎么样结合工作的？/) 里"每个数据源一套 `DataSource` + `SqlSessionFactory` + `TransactionManager`"的结论是同一件事的两面。

到这里，调用确实进到事务了。接下来看事务管理器拿什么在记账。

---

## 第五部分：事务做了什么——`TransactionManager` 的 ThreadLocal 账本

### 5.1 "当前事务"存在哪里：六个 ThreadLocal

`TransactionSynchronizationManager` 用一组 `NamedThreadLocal` 保存线程级的事务状态：

```java
// org.springframework.transaction.support.TransactionSynchronizationManager
private static final ThreadLocal<Map<Object, Object>> resources =
        new NamedThreadLocal<>("Transactional resources");

private static final ThreadLocal<Set<TransactionSynchronization>> synchronizations =
        new NamedThreadLocal<>("Transaction synchronizations");

private static final ThreadLocal<String> currentTransactionName =
        new NamedThreadLocal<>("Current transaction name");

private static final ThreadLocal<Boolean> currentTransactionReadOnly =
        new NamedThreadLocal<>("Current transaction read-only status");

private static final ThreadLocal<Integer> currentTransactionIsolationLevel =
        new NamedThreadLocal<>("Current transaction isolation level");

private static final ThreadLocal<Boolean> actualTransactionActive =
        new NamedThreadLocal<>("Actual transaction active");
```

| ThreadLocal | 存什么 | 谁在用 |
|---|---|---|
| `resources` | `DataSource` → `ConnectionHolder` 的绑定 | `DataSourceTransactionManager` 判断"当前线程有没有连接" |
| `synchronizations` | 本次事务注册的同步回调 | `TransactionSynchronization` / `@TransactionalEventListener` |
| `currentTransactionName` | 事务名（默认是"全限定方法名"） | 日志、监控 |
| `currentTransactionReadOnly` | 只读标记 | 参与已有事务时的校验、`isCurrentTransactionReadOnly()` |
| `currentTransactionIsolationLevel` | 隔离级别 | 同上 |
| `actualTransactionActive` | **是否有"真正"的事务在跑** | 区分"空事务状态"和"真事务" |

**这张表解释了"为什么事务不跨线程"**：所有状态都在 `ThreadLocal` 里。`@Async`、`CompletableFuture`、线程池里跑的子任务，拿到的是另一个线程——`resources` 里没有 `ConnectionHolder`，`isExistingTransaction` 返回 false，于是要么开一个新事务（也就意味着新连接、新提交时机），要么在没有管理器的场景下静默跳过。

也解释了为什么 `@Transactional` 方法里 `TransactionSynchronizationManager.isActualTransactionActive()` 能当成"当前有没有真事务"的探针——排查时打印一次，比猜快得多。

### 5.2 `getTransaction`：先判断"有没有已有事务"

`AbstractPlatformTransactionManager.getTransaction` 是所有传播行为的唯一入口：

```java
// org.springframework.transaction.support.AbstractPlatformTransactionManager
public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
        throws TransactionException {

    // Use defaults if no transaction definition given.
    TransactionDefinition def = (definition != null ? definition : TransactionDefinition.withDefaults());

    Object transaction = doGetTransaction();
    boolean debugEnabled = logger.isDebugEnabled();

    if (isExistingTransaction(transaction)) {
        // Existing transaction found -> check propagation behavior to find out how to behave.
        return handleExistingTransaction(def, transaction, debugEnabled);
    }

    // Check definition settings for new transaction.
    if (def.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
        throw new InvalidTimeoutException("Invalid transaction timeout", def.getTimeout());
    }

    // No existing transaction found -> check propagation behavior to find out how to proceed.
    if (def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
        throw new IllegalTransactionStateException(
                "No existing transaction found for transaction marked with propagation 'mandatory'");
    }
    else if (def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
            def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
            def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
        SuspendedResourcesHolder suspendedResources = suspend(null);
        if (debugEnabled) {
            logger.debug("Creating new transaction with name [" + def.getName() + "]: " + def);
        }
        try {
            return startTransaction(def, transaction, false, debugEnabled, suspendedResources);
        }
        catch (RuntimeException | Error ex) {
            resume(null, suspendedResources);
            throw ex;
        }
    }
    else {
        // Create "empty" transaction: no actual transaction, but potentially synchronization.
        if (def.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT && logger.isWarnEnabled()) {
            logger.warn("Custom isolation level specified but no actual transaction initiated; " +
                    "isolation level will effectively be ignored: " + def);
        }
        boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
        return prepareTransactionStatus(def, null, true, newSynchronization, debugEnabled, null);
    }
}
```

结构非常清晰：**先判断"有没有已有事务"，再按传播行为分两路**。

`isExistingTransaction` 是同步器相关的抽象方法，JDBC 场景下由 `DataSourceTransactionManager` 实现：

```java
// org.springframework.jdbc.datasource.DataSourceTransactionManager
@Override
protected Object doGetTransaction() {
    DataSourceTransactionObject txObject = new DataSourceTransactionObject();
    txObject.setSavepointAllowed(isNestedTransactionAllowed());
    ConnectionHolder conHolder =
            (ConnectionHolder) TransactionSynchronizationManager.getResource(obtainDataSource());
    txObject.setConnectionHolder(conHolder, false);
    return txObject;
}

@Override
protected boolean isExistingTransaction(Object transaction) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
    return (txObject.hasConnectionHolder() && txObject.getConnectionHolder().isTransactionActive());
}
```

**"当前线程在这个 `DataSource` 上有没有一个处于事务状态的 `ConnectionHolder`"——这就是 `isExistingTransaction` 的全部含义。** 注意 `getResource(obtainDataSource())` 是以 `DataSource` 实例为 key 的，多数据源各自独立记账，这也正是多数据源事务互相看不见的原因。

### 5.3 开事务那一刻，`Connection` 上发生了什么

```java
// org.springframework.jdbc.datasource.DataSourceTransactionManager
@Override
protected void doBegin(Object transaction, TransactionDefinition definition) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
    Connection con = null;

    try {
        if (!txObject.hasConnectionHolder() ||
                txObject.getConnectionHolder().isSynchronizedWithTransaction()) {
            Connection newCon = obtainDataSource().getConnection();
            // ...
            txObject.setConnectionHolder(new ConnectionHolder(newCon), true);
        }

        txObject.getConnectionHolder().setSynchronizedWithTransaction(true);
        con = txObject.getConnectionHolder().getConnection();

        Integer previousIsolationLevel = DataSourceUtils.prepareConnectionForTransaction(con, definition);
        txObject.setPreviousIsolationLevel(previousIsolationLevel);
        txObject.setReadOnly(definition.isReadOnly());

        // Switch to manual commit if necessary. This is very expensive in some JDBC drivers,
        // so we don't want to do it unnecessarily (for example if we've explicitly
        // configured the connection pool to set it already).
        if (con.getAutoCommit()) {
            txObject.setMustRestoreAutoCommit(true);
            if (logger.isDebugEnabled()) {
                logger.debug("Switching JDBC Connection [" + con + "] to manual commit");
            }
            con.setAutoCommit(false);
        }

        prepareTransactionalConnection(con, definition);
        txObject.getConnectionHolder().setTransactionActive(true);

        int timeout = determineTimeout(definition);
        if (timeout != TransactionDefinition.TIMEOUT_DEFAULT) {
            txObject.getConnectionHolder().setTimeoutInSeconds(timeout);
        }

        // Bind the connection holder to the thread.
        if (txObject.isNewConnectionHolder()) {
            TransactionSynchronizationManager.bindResource(obtainDataSource(), txObject.getConnectionHolder());
        }
    }
    catch (Throwable ex) {
        if (txObject.isNewConnectionHolder()) {
            DataSourceUtils.releaseConnection(con, obtainDataSource());
            txObject.setConnectionHolder(null, false);
        }
        throw new CannotCreateTransactionException("Could not open JDBC Connection for transaction", ex);
    }
}
```

**"开启事务"在 JDBC 层面的动作，其实就是把连接拿过来、把 `autoCommit` 关掉、把这个连接绑定到当前线程。** 后面所有 SQL 通过 `DataSourceUtils.getConnection(dataSource)` 拿到的都是这个被绑定的连接，所以它们自然在同一个事务里。

注意顺序：先 `con.setAutoCommit(false)`，再 `bindResource`。绑定发生在连接准备好之后，所以"事务可见"的标志是 `ConnectionHolder` 出现在 ThreadLocal 里。

### 5.4 传播行为：源码里就这几行分支

已有事务时的处理，全在 `handleExistingTransaction` 里：

```java
// org.springframework.transaction.support.AbstractPlatformTransactionManager
private TransactionStatus handleExistingTransaction(
        TransactionDefinition definition, Object transaction, boolean debugEnabled)
        throws TransactionException {

    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NEVER) {
        throw new IllegalTransactionStateException(
                "Existing transaction found for transaction marked with propagation 'never'");
    }

    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NOT_SUPPORTED) {
        if (debugEnabled) {
            logger.debug("Suspending current transaction");
        }
        Object suspendedResources = suspend(transaction);
        boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
        return prepareTransactionStatus(
                definition, null, false, newSynchronization, debugEnabled, suspendedResources);
    }

    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW) {
        // ... 挂起当前事务，起一个全新的
        SuspendedResourcesHolder suspendedResources = suspend(transaction);
        try {
            return startTransaction(definition, transaction, false, debugEnabled, suspendedResources);
        }
        catch (RuntimeException | Error beginEx) {
            resumeAfterBeginException(transaction, suspendedResources, beginEx);
            throw beginEx;
        }
    }

    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
        if (!isNestedTransactionAllowed()) {
            throw new NestedTransactionNotSupportedException(
                    "Transaction manager does not allow nested transactions by default - " +
                    "specify 'nestedTransactionAllowed' property with value 'true'");
        }
        if (useSavepointForNestedTransaction()) {
            DefaultTransactionStatus status = newTransactionStatus(
                    definition, transaction, false, false, true, debugEnabled, null);
            status.createAndHoldSavepoint();
            return status;
        }
        else {
            return startTransaction(definition, transaction, true, debugEnabled, null);
        }
    }

    // PROPAGATION_REQUIRED, PROPAGATION_SUPPORTS, PROPAGATION_MANDATORY:
    // regular participation in existing transaction.
    if (debugEnabled) {
        logger.debug("Participating in existing transaction");
    }
    if (isValidateExistingTransaction()) {
        // ...（默认关闭，见 5.5）
    }
    boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
    return prepareTransactionStatus(definition, transaction, false, newSynchronization, debugEnabled, null);
}
```

整理成一张表：

| 传播行为 | 无已有事务 | 有已有事务 | 关键源码动作 |
|---|---|---|---|
| `REQUIRED`（默认） | 新建 | 加入 | 最后一段："Participating in existing transaction" |
| `SUPPORTS` | 空事务状态 | 加入 | 同上，不走 `startTransaction` |
| `MANDATORY` | 抛 `IllegalTransactionStateException` | 加入 | `getTransaction` 开头那个 `throw` |
| `REQUIRES_NEW` | 新建 | **挂起旧的，新建** | `suspend(transaction)` + `startTransaction` |
| `NOT_SUPPORTED` | 空事务状态 | **挂起旧的，不建新的** | `suspend` + `prepareTransactionStatus(..., null, ...)` |
| `NEVER` | 空事务状态 | **抛异常** | `handleExistingTransaction` 第一段 |
| `NESTED` | 新建 | **建 savepoint** | `status.createAndHoldSavepoint()` |

两个必须记住的细节：

**`REQUIRES_NEW` 的"挂起"是双向的。** `suspend` 会把当前线程 `resources` 里的 `ConnectionHolder` 解绑，`startTransaction` 再绑定一个新的；外层结束时 `resume` 把旧的绑回去。所以：

- `REQUIRES_NEW` **需要第二条数据库连接**。连接池被打满时，`REQUIRES_NEW` 是经典的死锁来源（外层占着连接不放，内层等不到新连接）。
- `REQUIRES_NEW` 内层提交后，外层回滚**不会**撤销内层已经提交的数据。这一点经常被误解为"事务没生效"。
- 挂起和恢复都作用于当前线程，所以 `REQUIRES_NEW` 里的状态变化对同线程的其他代码是不可见的——这正是它想要的效果，也是"内层查不到外层未提交数据"的原因。

**`NESTED` 在 JDBC 上默认是 savepoint，不是新事务。** 看分支里的 `useSavepointForNestedTransaction()`（`AbstractPlatformTransactionManager` 默认 true）和 `status.createAndHoldSavepoint()`。所以：

- 内层回滚是 `rollback to savepoint`，外层还能继续提交；
- 内层失败不会把外层标成 rollback-only，除非异常穿透到外层拦截器；
- savepoint 需要数据库和驱动支持（MySQL InnoDB 支持）。

### 5.5 参与已有事务时，内层的属性被静默忽略

这是被引用最少、踩坑最多的一段。上面源码最后那个分支里：

```java
// org.springframework.transaction.support.AbstractPlatformTransactionManager
// （handleExistingTransaction 的最后一个分支，即"加入已有事务"）
if (isValidateExistingTransaction()) {
    if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT) {
        Integer currentIsolationLevel = TransactionSynchronizationManager.getCurrentTransactionIsolationLevel();
        if (currentIsolationLevel == null || currentIsolationLevel != definition.getIsolationLevel()) {
            throw new IllegalTransactionStateException("Participating transaction with definition [" +
                    definition + "] specifies isolation level which is incompatible with existing transaction: " +
                    (currentIsolationLevel != null ?
                            DefaultTransactionDefinition.getIsolationLevelName(currentIsolationLevel) :
                            "(unknown)"));
        }
    }
    if (!definition.isReadOnly()) {
        if (TransactionSynchronizationManager.isCurrentTransactionReadOnly()) {
            throw new IllegalTransactionStateException("Participating transaction with definition [" +
                    definition + "] is not marked as read-only but existing transaction is");
        }
    }
}
```

而 `isValidateExistingTransaction` 的默认值是 `false`：

```java
// org.springframework.transaction.support.AbstractPlatformTransactionManager
private boolean validateExistingTransaction = false;
```

于是默认行为是：**内层方法的 `isolation`、`readOnly`、`timeout` 在"加入已有事务"时全部被忽略，而且连一条 `WARN` 日志都没有。**

具体表现：

```java
@Transactional
public void outer() {
    innerReadOnly();      // 内层 @Transactional(readOnly = true) 完全不起作用
    // ... 这里继续写库照样成功
}

@Transactional(readOnly = true)
public void innerReadOnly() {
    // 期望只读，实际跟着外层的事务走
}
```

如果确实想让这类不一致暴露出来，可以打开校验：

```java
@Configuration
public class TxConfig {

    @Bean
    public TransactionManagerCustomizer<AbstractPlatformTransactionManager> validateExistingTx() {
        return tm -> tm.setValidateExistingTransaction(true);
    }
}
```

打开后，上面那种"外层只读、内层要写"的组合会直接抛 `IllegalTransactionStateException`，而不是安静地按外层来。

另外要记住：**`readOnly = true` 本身也只是提示。** 对 `DataSourceTransactionManager`，它体现在 `txObject.setReadOnly(...)` 和 `DataSourceUtils.prepareConnectionForTransaction(con, definition)` 上——最终是把连接设为只读（前提是驱动和数据库支持），并且让 JPA/Hibernate 之类的使用者据此调整刷新策略。它**不是**数据库级别的写保护，能不能拦住写操作取决于底层实现。

### 5.6 提交、回滚与 `UnexpectedRollbackException`

`rollbackOn` 决定"这个异常该不该回滚"：

```java
// org.springframework.transaction.interceptor.TransactionAspectSupport
protected void completeTransactionAfterThrowing(@Nullable TransactionInfo txInfo, Throwable ex) {
    if (txInfo != null && txInfo.getTransactionStatus() != null) {
        // ...
        if (txInfo.transactionAttribute != null && txInfo.transactionAttribute.rollbackOn(ex)) {
            txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
        }
        else {
            // We don't roll back on this exception.
            // Will still roll back if TransactionStatus.isRollbackOnly() is true.
            txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
        }
    }
}
```

默认规则是"`RuntimeException` 和 `Error` 回滚，受检异常不回滚"。带 `rollbackFor` 时走的是规则匹配：

```java
// org.springframework.transaction.interceptor.RuleBasedTransactionAttribute
@Override
public boolean rollbackOn(Throwable ex) {
    RollbackRuleAttribute winner = null;
    int deepest = Integer.MAX_VALUE;

    if (this.rollbackRules != null) {
        for (RollbackRuleAttribute rule : this.rollbackRules) {
            int depth = rule.getDepth(ex);
            if (depth >= 0 && depth < deepest) {
                deepest = depth;
                winner = rule;
            }
        }
    }

    // User superclass behavior (rollback on unchecked) if no rule matches.
    if (winner == null) {
        return super.rollbackOn(ex);
    }

    return !(winner instanceof NoRollbackRuleAttribute);
}
```

匹配规则是"**继承层级最深的那条获胜**"（`depth < deepest` 不断收敛）。所以同一个 `@Transactional` 上同时写了 `rollbackFor = Exception.class` 和 `noRollbackFor = IllegalStateException.class` 时，`IllegalStateException` 命中的是更深的那条——**两种规则并列时，谁更具体谁说了算**，而不是"后写的覆盖先写的"。

真正让无数人抓头的是这个：

```text
org.springframework.transaction.UnexpectedRollbackException:
    Transaction silently rolled back because it has been marked as rollback-only
```

抛出它的地方在 `processCommit`：

```java
// org.springframework.transaction.support.AbstractPlatformTransactionManager
private void processCommit(DefaultTransactionStatus status) throws TransactionException {
    try {
        boolean beforeCompletionInvoked = false;
        // ...
        try {
            boolean unexpectedRollback = false;
            prepareForCommit(status);
            triggerBeforeCommit(status);
            triggerBeforeCompletion(status);
            beforeCompletionInvoked = true;

            if (status.hasSavepoint()) {
                unexpectedRollback = status.isGlobalRollbackOnly();
                status.releaseHeldSavepoint();
            }
            else if (status.isNewTransaction()) {
                // ...
                unexpectedRollback = status.isGlobalRollbackOnly();
                doCommit(status);
            }
            else if (isFailEarlyOnGlobalRollbackOnly()) {
                unexpectedRollback = status.isGlobalRollbackOnly();
            }

            // Throw UnexpectedRollbackException if we have a global rollback-only
            // marker but still didn't get a corresponding exception from commit.
            if (unexpectedRollback) {
                throw new UnexpectedRollbackException(
                        "Transaction silently rolled back because it has been marked as rollback-only");
            }
        }
        // ...
    }
    finally {
        cleanupAfterCompletion(status);
    }
}
```

要理解它，需要把 `processRollback` 的"参与已有事务"分支接上：

```java
// org.springframework.transaction.support.AbstractPlatformTransactionManager#processRollback
else {
    // Participating in larger transaction
    if (status.hasTransaction()) {
        if (status.isLocalRollbackOnly() || isGlobalRollbackOnParticipationFailure()) {
            if (status.isDebug()) {
                logger.debug("Participating transaction failed - marking existing transaction as rollback-only");
            }
            doSetRollbackOnly(status);
        }
        // ...
    }
    else {
        logger.debug("Should roll back transaction but cannot - no transaction available");
    }
}
```

把两段拼起来，就是那个经典现象：

```java
@Transactional
public void outer() {
    try {
        inner();                       // 内层加入外层事务
    } catch (Exception e) {
        // 内层异常在这里被吞掉，outer 正常返回
    }
}

@Transactional
public void inner() {
    throw new RuntimeException("boom");
}
```

执行过程是：

1. 内层拦截器捕获异常，`rollbackOn` 判定要回滚，调用 `tm.rollback(innerStatus)`。
2. 内层的 `status` 不是新事务（`isNewTransaction() == false`），所以走 `doSetRollbackOnly`——**它不能真正回滚，只能把外层事务标记成 rollback-only**。
3. 异常被 `outer` 的 `catch` 吞了，`outer` 正常返回，外层拦截器调 `commit`。
4. `processCommit` 发现 `isGlobalRollbackOnly()` 为真，于是回滚，并抛 `UnexpectedRollbackException`。

**所以这个异常的含义是："事务确实回滚了，但回滚的原因你没告诉我。"** 它不是 Bug，是 Spring 在提醒你"内层已经判定必须回滚，外层却打算提交，两者矛盾"。

排查方向也很明确：

- 内层**本就不该回滚** → 内层加 `propagation = Propagation.REQUIRES_NEW`，让内层有自己的事务。注意这会让内层的提交对外层不可撤销。
- 内层**确实该回滚** → 别吞异常，让外层也知道。
- 只想让外层决定 → `propagation = Propagation.NESTED`，内层用 savepoint 回滚，不污染外层。

### 5.7 事务的生命周期回调

提交不是"commit 一下就完事"，`processCommit` 里有一整套回调顺序：

```text
prepareForCommit
  → TransactionSynchronization.beforeCommit()
  → TransactionSynchronization.beforeCompletion()
  → doCommit()                                  ← 真正写库
  → TransactionSynchronization.afterCommit()
  → TransactionSynchronization.afterCompletion(STATUS_COMMITTED)
```

回滚路径对称：`triggerBeforeCompletion` → `doRollback` → `afterCompletion(STATUS_ROLLED_BACK)`。

**关键点：`beforeCommit` 里抛异常会导致回滚；`afterCommit` 里抛异常不会回滚——事务已经提交了。** 这个区别决定了很多"消息发出去了但数据没落库"的线上事故，因为把"发消息/发事件"放在 `afterCommit` 里，就是在赌"提交一定成功"。

Spring 提供了和这套回调绑定的注解：

```java
@Component
public class OrderCreatedListener {

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onOrderCreated(OrderCreatedEvent event) {
        // 事务提交后才执行；事务回滚时这里根本不会被调用
    }
}
```

`@TransactionalEventListener` 默认就是 `AFTER_COMMIT`。它平时"注册"在 `synchronizations` 这个 ThreadLocal 里——也就是 5.1 节那张表的第二行——因此它天然受线程约束：**没有事务时默认不执行**（除非把 `fallbackExecution` 设为 `true`）。

这条线和事件机制的交汇，在 [Spring 接入点：跟着容器过一生](/2026/09/11/Spring-接入点：跟着容器过一生/) 里已经出现过：代理负责拦截，事件负责解耦，而事务负责"什么时候才算真的成功"。

---

## 第六部分：回到开头——把失效原因映射回链路

有了前五部分的链路，开头那张图上的"失效点"可以逐个填上具体原因了。下表的编号 A~L 是为了指代方便，和 1.2 节图里的"失效点 1~5"是两套编号。

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

对应 4.1 节里那段 `if (this.advised.exposeProxy)`。没打开时抛的是：

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

**一个特别注意点**：如果项目里手写 `@EnableTransactionManagement`，`@ConditionalOnMissingBean(AbstractTransactionManagementConfiguration.class)` 会让 Boot 的事务自动配置退让，事务侧的配置以你的注解为准。但这**不会**让 `spring.aop.proxy-target-class` 失效——`AopAutoConfiguration` 是独立的自动配置，默认仍然强制 CGLIB（2.5 节）。所以"我写了 `@EnableTransactionManagement(proxyTargetClass = false)`，就切到 JDK 代理了"这种判断在很多项目里站不住：注解里写 `false` 只是"自己不要求 CGLIB"，而另一条路还在要求。

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

反过来说也成立：**理解了 AOP 的四个阶段——装配（注册 Advisor）、代理创建（`wrapIfNecessary`）、链的匹配（pointcut + methodCache）、执行（`proceed`）——你手里就有了一把能同时打开"事务失效"和"切面失效"两把锁的钥匙。** 这也是为什么本文一开始要用一张调用链图把两个话题合并：它们本来就是一回事，只是在 Spring 的代码里被拆到了 `spring-aop` 和 `spring-tx` 两个模块。

回到开头那段代码，现在可以准确地说出它错在哪：

- `this.saveOrder(dto)` 是目标对象内部调用，不经过代理——**失效点 2**，链条在"运行阶段入口"就断了。
- 把它挪到 `createOrder` 上能生效，是因为 `createOrder` 是被外部（Controller）通过代理调用的。
- 如果坚持要拆两个方法，正确做法是把 `saveOrder` 挪到另一个 Bean（6.1 节方案一），而不是靠 `AopContext` 或者给类加 `@Transactional` 来"糊"过去。

---

## 延伸阅读

- [Spring 接入点：跟着容器过一生](/2026/09/11/Spring-接入点：跟着容器过一生/) —— `BeanPostProcessor` 在容器生命周期里的位置，是理解第三部分"代理在什么时候生成"的前置知识。
- [Spring 中 MyBatis、MyBatis-Plus 是怎么样结合工作的？](/2026/09/10/Spring中Mybatis、MybatisPlus是怎么样结合工作的？/) —— 事务和 `SqlSession` 的复用关系：**没有 Spring 事务时，每次 Mapper 调用都是独立的 `SqlSession`；有事务时，`SqlSession` 跟着事务走。** 这是"事务影响的不只是 SQL 提交"的最好例子。
- [Spring 自定义 MyBatis 数据源，到底发生了什么？](/2026/09/10/Spring自定义Mybatis数据源，到底发生了什么？/) —— 多数据源场景下，每个数据源要配一套 `DataSource` + `SqlSessionFactory` + `TransactionManager`，正好对应 4.5 节"按类型取管理器会炸"的问题。
- [深入理解 Spring WebFlux 与 Reactor Netty](/2026/09/15/深入理解-Spring-WebFlux-与-Reactor-Netty/) —— 理解 7.3 节"为什么响应式事务必须换一套上下文"，得先理解 Reactor 的线程模型。

接下来值得深挖的两个方向：

- **`TransactionSynchronization` 的完整生命周期**：`beforeCommit` 抛异常会发生什么、`afterCommit` 抛异常为什么不会回滚、多个同步器的执行顺序。
- **`AbstractRoutingDataSource` + 事务**：动态数据源切换为什么在"事务开启后"失效——因为 `ConnectionHolder` 在 `doBegin` 时就已经按 `DataSource` 绑定了，切换发生在拿到连接之后。这个坑的本质，仍然是本文第五部分那张 ThreadLocal 表。