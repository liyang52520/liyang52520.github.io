---
title: "@Transactional 为什么会失效（三）：事务管理器的 ThreadLocal 账本"
date: 2026-10-16 16:25:33
tags:
  - spring
  - aop
  - transaction
categories:
  - develop
---

## 引子：事务是开了，可它到底开在哪

上一篇的结尾停在 `TransactionInterceptor.invokeWithinTransaction` 拿到事务属性、选出事务管理器的那一刻。注意，到这里为止**还没有一条 SQL 被执行，也还没有连接被占用**——真正"开事务"，发生在 `PlatformTransactionManager.getTransaction` 里面。而在那之前，Spring 必须先回答一个更底层的问题：所谓的"当前事务"，究竟是个什么东西，又存在哪里？

换个说法：`@Transactional` 让方法"进了事务"，可事务本身既不是对象，也不是变量，它散落在当前线程的几个 `ThreadLocal` 里——连接、同步回调、只读标记、隔离级别、事务名。**这些账目一旦记错线程，事务就会在人眼皮底下消失**，不报错、不打日志。这一篇要做的，就是把这份账本翻开。

> 本篇接[第二篇](/2026/10/16/Transactional-为什么会失效（二）：代理的生成与调用进链/)（事务属性已解析、调用已进链），覆盖第五部分：`TransactionManager` 的 ThreadLocal 账本、七种传播行为、内层属性的静默忽略、提交与回滚的完成逻辑、事务生命周期回调。口径同导读：Spring Framework 6.1.8 + Spring Boot 3.3.x（Java 17）。

> 本篇要回答的问题：
>
> - "当前事务"到底存在哪里？为什么换个线程就找不到了？
> - `isExistingTransaction` 判断的到底是什么？多数据源为什么互相看不见对方的事务？
> - 七种传播行为在源码里各对应哪几行分支？`REQUIRES_NEW` 与 `NESTED` 的差别在哪？
> - 内层的 `readOnly = true` 为什么被静默忽略？打开校验之后会发生什么？
> - `UnexpectedRollbackException` 到底是谁抛出来的？回滚与不回滚的规则并列时谁说了算？
> - 提交与回滚各按什么顺序触发回调？`beforeCommit` 和 `afterCommit` 抛异常的差别在哪？

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

## 小结

第五部分走完，事务的账本也就摊开了：所谓"当前事务"，不过是当前线程在某个 `DataSource` 上绑定的 `ConnectionHolder`，外加 `synchronizations`、`actualTransactionActive` 这几个 `ThreadLocal`；七种传播行为的差别，也只是 `getTransaction` 里"有没有已有事务"这一个判断分出的两路；`UnexpectedRollbackException`、内层 `readOnly` 失效这些现象，都能在 `handleExistingTransaction` 与 `processCommit` 里找到对应行。

到这里，从注解到 Advisor、从代理到拦截器、从事务管理器到 ThreadLocal 账本，每一环都看过了。下一篇把这条链倒过来用：**不再顺着链路往下走，而是拿十二种真实的失效现象，逐个回答"它断在哪一环"**，再补上自调用与 `@Async` 的修法、事务切面与普通切面的差异，以及响应式和 AspectJ 两条支线。

> **下一篇**
>
> [第四篇：失效原因映射与收束](/2026/10/16/Transactional-为什么会失效（四）：失效原因映射与收束/)——第六~八部分：十二种失效现象分别断在哪一环、自调用与 `@Async` 怎么修、事务切面与普通切面的四处本质差异、响应式与 AspectJ 两条支线。系列入口见[导读](/2026/10/16/Transactional-为什么会失效-从-AOP-代理链看-Spring-事务机制/)。
