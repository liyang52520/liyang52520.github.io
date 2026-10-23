---
title: "@Transactional 为什么会失效（二）：代理的生成与调用进链"
date: 2026-10-16 16:25:32
tags:
  - spring
  - aop
  - transaction
categories:
  - develop
---

## 引子：代理明明在，方法上却没有事务

上一篇结束时，容器里备好了三样东西：自动代理创建器、事务 Advisor、`TransactionInterceptor`。但"备好"不等于"用上"，它们得先落到具体 Bean 上，才可能真的拦到一次调用。

于是有了本篇特有的两类困惑：有的 Bean 被包了一层，有的没有，判断标准是什么？更隐蔽的是 `AopUtils.isAopProxy(bean)` 返回 `true`，`@Transactional` 却没生效——代理在，只是这个方法上一条通知都没有。

> 本篇接[第一篇](/2026/10/16/Transactional-为什么会失效（一）：从注解到-Advisor-的装配/)，覆盖第三、四部分：`wrapIfNecessary` 的三道闸门、pointcut 如何判断"方法上有没有事务属性"、JDK 与 CGLIB 决定注解写在哪才有效、调用怎么进链。口径同导读：Spring Framework 6.1.8 + Spring Boot 3.3.x（Java 17）。

> 本篇要回答的问题：
>
> - 代理在哪一步生成？为什么 `@PostConstruct` 里的 `this` 不是代理？
> - 事务的"切点表达式"为什么只有两行逻辑？注解写在接口和实现类上，结果为什么不一样？
> - 代理在、方法上却没有通知，断在哪一环？

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

这个值来自 `ProxyProcessorSupport`（`AbstractAutoProxyCreator` 的父类）的 `order` 字段，默认本是 `LOWEST_PRECEDENCE`。**调成最高优先级，是为了让自动代理创建器在 `BeanPostProcessor` 列表里排在前面**，这样它的 `postProcessAfterInitialization` 会先于其他后置处理器执行——后面 6.3 节（第四篇）会看到，`@Async` 正是利用"自己排在 APC 之后"这一点，往已经建好的代理里追加 advisor。

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

链的构建由 `DefaultAdvisorChainFactory` 完成，它拿到的 Advisor 顺序决定了拦截器的顺序，而顺序由每个 Advisor 的 `order` 决定：**升序执行，order 越小越靠外**。**事务 Advisor 的默认 order 是 `Ordered.LOWEST_PRECEDENCE`**，也就是 `int` 的最大值——但先别急着下"事务一定在最内层"的结论：**不写 `@Order` 的切面同样是这个值**，两者并列时相对顺序在源码层面没有任何保证，7.2 节（第四篇）展开。

这一点在第七部分（第四篇）会结合一个具体例子说明。

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

要补一个前提：这个分支要求**拦截器已经装配好、并且事务属性非空**。如果容器里连 `TransactionManager` Bean 都没有，`TransactionAutoConfiguration` 的 `EnableTransactionManagementConfiguration` 根本不会生效（第一篇 2.5 节），`TransactionInterceptor` 压根不会注册——那才是更彻底的静默，连这行 `debug` 都没有。

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

## 小结

代理的生成与进链在这里走完：`wrapIfNecessary` 的三道闸门决定"包不包"，`TransactionAttributeSourcePointcut` 用"能不能解析出事务属性"决定"切不切"，而 `AopUtils.canApply` 扫接口、运行期却按方法匹配，于是有了"代理在、事务不在"。进链后 `TransactionInterceptor` 只做转发，`tm.getTransaction(txAttr)` 之后的"当前事务"记在哪，留给下一篇。

> **下一篇**
>
> [（三）：事务管理器的 ThreadLocal 账本](/2026/10/16/Transactional-为什么会失效（三）：事务管理器的-ThreadLocal-账本/)。系列入口见[导读](/2026/10/16/Transactional-为什么会失效-从-AOP-代理链看-Spring-事务机制/)。
