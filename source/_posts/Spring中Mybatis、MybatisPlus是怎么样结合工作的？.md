---
title: Spring中Mybatis、MybatisPlus是怎么样结合工作的？
date: 2026-09-10 15:29:29
tags: 
  - spring
  - mybatis
categories: 
  - develop
---

## 引子：一个常见的困惑

在 Spring Boot 项目中，我们几乎每天都在用 MyBatis 或 MyBatis-Plus 操作数据库。只要在 Mapper 接口上写几个方法，在 Service 里 `@Autowired` 注入，就能直接调用。一切看起来理所当然。

但当你开始追问下面这些问题时，事情就变得有意思了：

> - `UserMapper` 只是一个接口，没有实现类，为什么能被注入？
> - `BaseMapper` 里的 `insert`、`selectById` 方法，我从来没有写过 SQL，它们是怎么执行的？
> - `@MapperScan` 到底做了什么？它和 MyBatis-Plus 重写的 `MybatisMapperRegistry` 是怎么关联上的？
> - `MybatisPlusAutoConfiguration` 和 `@MapperScan` 是什么关系？
> - 多数据源时为什么还要配置 `sqlSessionFactoryRef` 和 `sqlSessionTemplateRef`？
> - MyBatis-Plus 的 `saveBatch` 为什么性能不理想？有没有真正的批量插入方案？

这篇文章就沿着这些问题，把 Spring、MyBatis、MyBatis-Plus 三者的协作关系一层层拆开，最后落到多数据源配置和批量插入的实战上。

---

## 第一部分：MyBatis 核心原理

### 1.1 MyBatis 的两个阶段

MyBatis 的工作可以分成两个阶段：

- **启动阶段**：解析所有 SQL（XML 或注解），把每条 SQL 翻译成一个 `MappedStatement` 对象，注册到全局容器 `Configuration` 中。
- **运行阶段**：根据 Mapper 接口的方法名，从 `Configuration` 中查出对应的 `MappedStatement`，执行 SQL，处理结果。

**理解这两个阶段的分界，是理解 MyBatis 的关键。**

### 1.2 `MappedStatement`：一条 SQL 的说明书

在 MyBatis 中，每个 `<select>`、`<insert>`、`<update>`、`<delete>` 标签（或注解），启动时都会被解析成一个 `MappedStatement` 对象。

它包含：

| 字段 | 含义 |
|:---|:---|
| `id` | 唯一标识，格式 `接口全限定名.方法名` |
| `sqlSource` | SQL 来源，含动态标签 |
| `resultMaps` | 结果映射规则 |
| `parameterMap` | 参数映射规则 |
| `statementType` | `STATEMENT` / `PREPARED` / `CALLABLE` |
| `keyGenerator` | 主键生成策略 |
| `cache` | 二级缓存配置 |

**容器在 `Configuration` 中**：

```java
public class Configuration {
    protected final Map<String, MappedStatement> mappedStatements 
        = new StrictMap<>("Mapped Statements collection");
}
```

`Configuration` 是整个 MyBatis 的**全局大脑**，所有元数据都在这里。执行 SQL 时，MyBatis 用 `id` 查出对应的 `MappedStatement`，按说明书执行。

### 1.3 启动阶段：`MapperRegistry` 与 `MapperAnnotationBuilder`

启动阶段的核心任务是：**把 XML 或注解中定义的 SQL 解析成 `MappedStatement`，注册到 `Configuration.mappedStatements` 容器里。**

#### `MapperRegistry`：接口与代理工厂的注册表

`MapperRegistry` 是 `Configuration` 的一个字段，维护一个 Map：

```java
public class MapperRegistry {
    private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap<>();
}
```

- **key**：Mapper 接口的 `Class` 对象，如 `UserMapper.class`
- **value**：`MapperProxyFactory`，用来为这个接口生成 JDK 动态代理对象

它的核心方法 `addMapper()` 做了两件事：

```java
public <T> void addMapper(Class<T> type) {
    if (type.isInterface()) {
        // 1. 注册代理工厂
        knownMappers.put(type, new MapperProxyFactory<>(type));
        // 2. 触发解析：生成 MappedStatement 并注册到 Configuration
        MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
        parser.parse();
    }
}
```

#### `MapperAnnotationBuilder`：SQL 注解的翻译器

`MapperAnnotationBuilder` 负责解析一个 Mapper 接口中所有方法上的 SQL 注解（或关联的 XML），生成对应的 `MappedStatement`。

它的核心方法 `parse()` 大致流程：

```java
public void parse() {
    // 1. 加载与接口同名的 XML（如 UserMapper.xml）
    loadXmlResource();
    // 2. 解析 @CacheNamespace 等注解
    parseCache();
    // 3. 遍历接口的所有方法
    for (Method method : type.getMethods()) {
        // 4. 处理 @Select、@Insert、@Update、@Delete 等注解
        parseStatement(method);
    }
}
```

在 `parseStatement()` 中，它会读取注解、构建 `SqlSource`、创建 `MappedStatement`，最终调用 `configuration.addMappedStatement(...)` 注册到全局容器。

**两者分工**：

- `MapperRegistry`：管"**接口 → 代理工厂**"的注册，并触发解析。
- `MapperAnnotationBuilder`：管"**接口方法 → MappedStatement**"的解析和注册。

### 1.4 运行阶段：动态代理与 SQL 执行

启动阶段完成后，`Configuration` 中已经存好了所有 SQL 描述。运行阶段要解决的是：**如何把接口方法调用翻译成对 `MappedStatement` 的执行？**

答案就是 **JDK 动态代理**。

#### `MapperProxyFactory`：代理对象的工厂

```java
public class MapperProxyFactory<T> {
    private final Class<T> mapperInterface;

    public T newInstance(SqlSession sqlSession) {
        final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
        return (T) Proxy.newProxyInstance(
            mapperInterface.getClassLoader(),
            new Class[]{mapperInterface},
            mapperProxy
        );
    }
}
```

#### `MapperProxy`：方法调用的翻译器

```java
public class MapperProxy<T> implements InvocationHandler, Serializable {

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if (Object.class.equals(method.getDeclaringClass())) {
            return method.invoke(this, args);
        }
        if (method.isDefault()) {
            return invokeDefaultMethod(proxy, method, args);
        }
        return cachedInvoker(method).invoke(proxy, method, args, sqlSession);
    }
}
```

`MapperMethod` 在构造时会：

1. 解析方法签名（`select` / `insert` / `update` / `delete`）；
2. 从 `Configuration.mappedStatements` 中查找对应的 `MappedStatement`。

**完整调用链**：

```
userMapper.selectById(1L)
    ↓
MapperProxy.invoke()                        ← JDK 动态代理回调
    ↓
MapperMethod.execute(sqlSession, args)
    ↓
根据 SqlCommandType 调用 sqlSession.selectOne / insert / ...
    ↓
Executor → StatementHandler → JDBC → 数据库
```

---

## 第二部分：MyBatis-Plus 的增强机制

### 2.1 一个问题：`BaseMapper` 的方法为什么能调用？

到这里，MyBatis 的核心原理已经清楚了。但一个新的问题出现了：

**`UserMapper` 接口继承了 `BaseMapper<User>`，而 `BaseMapper` 里定义了一大堆方法（`insert`、`selectById`、`updateById` 等），这些方法在 XML 或注解里都没有对应的 SQL。它们是怎么能执行的？**

答案就在 MyBatis-Plus 的**动态注入**机制。

### 2.2 MyBatis-Plus 的切入点：重写两个核心类

MyBatis-Plus 重写了两个关键类：

| MyBatis 原生类 | MyBatis-Plus 重写类 | 目的 |
|:---|:---|:---|
| `MapperRegistry` | `MybatisMapperRegistry` | 在 `addMapper()` 中改用 MP 自己的解析器 |
| `MapperAnnotationBuilder` | `MybatisMapperAnnotationBuilder` | 在 `parse()` 中额外为 `BaseMapper` 通用方法注入 SQL |

`MybatisMapperRegistry.addMapper()` 的实现：

```java
@Override
public <T> void addMapper(Class<T> type) {
    if (type.isInterface()) {
        if (hasMapper(type)) {
            return;
        }
        boolean loadCompleted = false;
        try {
            // 1. 注册代理工厂
            knownMappers.put(type, new MapperProxyFactory<>(type));
            // 2. 使用 MyBatis-Plus 的解析器（不是原生的 MapperAnnotationBuilder）
            MybatisMapperAnnotationBuilder parser = new MybatisMapperAnnotationBuilder(config, type);
            parser.parse();   // ★ 触发解析和 SQL 注入
            loadCompleted = true;
        } finally {
            if (!loadCompleted) {
                knownMappers.remove(type);
            }
        }
    }
}
```

`MybatisMapperAnnotationBuilder.parse()` 的增强逻辑：

```java
@Override
public void parse() {
    // 1. 原生逻辑：解析 XML 和注解
    // ...

    // 2. MP 增强：如果接口继承了 BaseMapper
    if (BaseMapper.class.isAssignableFrom(type)) {
        List<AbstractMethod> methods = sqlInjector.getMethodList(type);
        for (AbstractMethod method : methods) {
            method.injectMappedStatement(type, ...);   // 生成并注册 MappedStatement
        }
    }
}
```

而 `AbstractMethod.injectMappedStatement()` 内部，会根据实体类的 `TableInfo`（表名、字段映射等元数据）拼出 SQL 模板，创建 `MappedStatement`，注册到 `Configuration.mappedStatements`。

**所以"动态注入"的本质是**：在启动阶段，把原本不存在的 SQL（以 `MappedStatement` 形式）**提前注册到 `Configuration` 中**。运行时调用 `userMapper.insert()` 时，MyBatis 就像执行普通 XML SQL 一样执行它。

### 2.3 这些重写类是怎么进入 MyBatis 启动流程的？

MyBatis-Plus 重写了 `MapperRegistry` 和 `MapperAnnotationBuilder`，但 MyBatis 启动流程是 MyBatis 自己的代码，它是怎么"偷梁换柱"的？

答案在于 **`MybatisConfiguration`**。

`MybatisConfiguration` 继承自 MyBatis 原生的 `Configuration`，但做了两件事：

1. **初始化自己的 `MybatisMapperRegistry`**：它用一个 `MybatisMapperRegistry` 的实例，**覆盖**了父类中默认的 `MapperRegistry` 字段。
2. **重写 `addMapper` 方法**：当 MyBatis 启动流程调用 `configuration.addMapper()` 时，实际调用的是 `MybatisConfiguration` 重写后的版本，该方法会将调用委托给 `MybatisMapperRegistry`。

```java
public class MybatisConfiguration extends Configuration {
    protected final MybatisMapperRegistry mapperRegistry;

    public MybatisConfiguration() {
        super();
        this.mapperRegistry = new MybatisMapperRegistry(this);  // ★ 替换原生 MapperRegistry
    }

    @Override
    public <T> void addMapper(Class<T> type) {
        this.mapperRegistry.addMapper(type);  // 委托给 MybatisMapperRegistry
    }
}
```

而 `MybatisConfiguration` 又是通过 `MybatisSqlSessionFactoryBean` 创建的。这个 FactoryBean 由 Spring Boot 的 `MybatisPlusAutoConfiguration` 创建，替换了原生的 `SqlSessionFactoryBean`。

**完整链路**：

```
Spring Boot 启动
    ↓
MybatisPlusAutoConfiguration 创建 MybatisSqlSessionFactoryBean
    ↓
调用 buildSqlSessionFactory()
    ↓
创建 MybatisConfiguration（覆盖原生 Configuration）
    ↓
MyBatis 启动流程调用 configuration.addMapper(UserMapper.class)
    ↓
MybatisConfiguration.addMapper() 委托给 MybatisMapperRegistry
    ↓
MybatisMapperRegistry.addMapper() 创建 MybatisMapperAnnotationBuilder
    ↓
MybatisMapperAnnotationBuilder.parse() 执行
    ├── 1. 解析 XML 和注解 (原生逻辑)
    └── 2. 检查是否继承 BaseMapper，如果是则触发 SQL 注入器
            ↓
        SqlInjector.inspectInject() 为通用方法生成 MappedStatement 并注册
    ↓
启动完成，UserMapper 拥有了所有通用 CRUD 方法
```

**为什么是 `MybatisPlusAutoConfiguration` 而不是 `MybatisAutoConfiguration`？**

因为引入 `mybatis-plus-boot-starter` 时，`MybatisAutoConfiguration` 所在的 jar（`mybatis-spring-boot-autoconfigure`）根本不在 classpath 上。Spring Boot 只能扫描到 `MybatisPlusAutoConfiguration`，它创建的是 `MybatisSqlSessionFactoryBean`，从而把 MyBatis-Plus 的全套自定义实现注入了 MyBatis 的启动流程。

---

## 第三部分：Spring 集成层

### 3.1 核心问题：Mapper 接口没有实现类，怎么注入？

现在，MyBatis 核心层已经清楚了：**启动时解析 SQL，运行时通过动态代理执行。** MyBatis-Plus 也清楚了：**在启动解析阶段，多注册一批 `MappedStatement`。**

但还有一个问题：`UserMapper` 是一个接口，我们 `@Autowired` 注入它时，注入的到底是什么？谁负责创建它？

答案在 **Spring 集成层**。MyBatis 提供了 `SqlSession` 等核心对象，但要和 Spring 的容器、事务、依赖注入无缝整合，需要 mybatis-spring 这个桥梁。

### 3.2 `MapperFactoryBean`：Spring 层的 FactoryBean

`MapperFactoryBean` 是 mybatis-spring 提供的类，实现了 Spring 的 `FactoryBean<T>`：

```java
public class MapperFactoryBean<T> extends SqlSessionDaoSupport implements FactoryBean<T> {

    private Class<T> mapperInterface;

    @Override
    public T getObject() throws Exception {
        return getSqlSession().getMapper(this.mapperInterface);
    }

    @Override
    public Class<T> getObjectType() {
        return this.mapperInterface;
    }

    @Override
    public boolean isSingleton() {
        return true;
    }
}
```

**`FactoryBean` 是 Spring 的一个特殊接口**：当 Spring 容器需要注入 `UserMapper` 类型的 Bean 时，实际调用的是 `MapperFactoryBean.getObject()` 返回的对象，而不是 `MapperFactoryBean` 本身。

`getObject()` 内部调用 `getSqlSession().getMapper()`，最终会走到 MyBatis 核心层的 `MapperRegistry.getMapper()`，由 `MapperProxyFactory` 创建代理对象。

### 3.3 隐藏的关键：`MapperFactoryBean.checkDaoConfig()`

`MapperFactoryBean` 继承自 `SqlSessionDaoSupport`，后者继承自 Spring 的 `DaoSupport`。`DaoSupport` 实现了 `InitializingBean`，会在 Bean 属性注入完成后调用 `afterPropertiesSet()`，进而调用 `checkDaoConfig()`。

`MapperFactoryBean` 重写了 `checkDaoConfig()`：

```java
@Override
protected void checkDaoConfig() {
    super.checkDaoConfig();
    notNull(this.mapperInterface, "Property 'mapperInterface' is required");
    Configuration configuration = getSqlSession().getConfiguration();
    if (this.addToConfig && !configuration.hasMapper(this.mapperInterface)) {
        try {
            // ★★★ 关键调用：把 UserMapper 注册到 Configuration 中
            configuration.addMapper(this.mapperInterface);
        } catch (Exception e) {
            // ...
        }
    }
}
```

**这一步是 Spring 集成层和 MyBatis-Plus 增强层的连接点**：`MapperFactoryBean` 在初始化时，调用了 `configuration.addMapper(UserMapper.class)`，从而触发 MyBatis-Plus 的 `MybatisMapperRegistry`，完成 SQL 注入。

### 3.4 `SqlSessionTemplate`：线程安全的会话封装

`SqlSessionTemplate` 是 mybatis-spring 对 `SqlSession` 的线程安全封装。它本身**不持有真正的 `SqlSession`**，而是通过一个动态代理 `SqlSessionInterceptor` 来管理：

```java
public class SqlSessionTemplate implements SqlSession {

    private final SqlSession sqlSessionProxy;   // 动态代理

    public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType) {
        this.sqlSessionProxy = (SqlSession) newProxyInstance(
            SqlSessionFactory.class.getClassLoader(),
            new Class[]{SqlSession.class},
            new SqlSessionInterceptor());
    }
}
```

`SqlSessionInterceptor` 的 `invoke()` 方法每次都会：

1. 从 `SqlSessionFactory` 获取一个 `SqlSession`（如果当前有 Spring 事务，则复用事务中的 `SqlSession`）；
2. 执行方法调用；
3. 提交或回滚；
4. 关闭 `SqlSession`（如果是非事务场景）。

**所以 `SqlSessionTemplate` 是线程安全的，它可以作为单例 Bean 被所有 Mapper 共享。**

### 3.5 为什么同时定义 `SqlSessionFactory` 和 `SqlSessionTemplate`？

从技术上讲，**只定义 `SqlSessionFactory` 也能跑通**。`MapperFactoryBean` 继承自 `SqlSessionDaoSupport`，而 `SqlSessionDaoSupport.setSqlSessionFactory()` 会自动创建一个 `SqlSessionTemplate`。

但把它独立成 Bean 有明确的好处：

| 好处 | 说明 |
|:---|:---|
| **单例共享** | 整个应用只有一个 `SqlSessionTemplate`，所有 `MapperFactoryBean` 共用同一个实例 |
| **统一配置 `ExecutorType`** | 可以通过配置项统一指定 `ExecutorType`，影响所有 Mapper 的执行方式 |
| **可被其他组件注入** | 任何需要直接执行 SQL 的组件都可以 `@Autowired SqlSessionTemplate` |
| **支持用户覆盖** | `@ConditionalOnMissingBean` 允许用户自定义 `SqlSessionTemplate` 来替换默认实现 |

**一句话**：`SqlSessionFactory` 是"造车的工厂"，`SqlSessionTemplate` 是"可以共享的车"。自动配置两个都定义，是为了让整个应用共用同一辆"车"。

---

## 第四部分：`@MapperScan` 的生效原理

### 4.1 核心问题

前面提到，`MapperFactoryBean` 负责创建代理对象，但它是怎么被注册到 Spring 容器中的？我们从来没有手动写过 `@Bean MapperFactoryBean`，为什么 `UserMapper` 就能被注入？

答案就是 `@MapperScan`。

### 4.2 第一环：`@MapperScan` 是个"伪装"的 `@Import`

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(MapperScannerRegistrar.class)  // ← 关键在这里
@Repeatable(MapperScans.class)
public @interface MapperScan {
    String[] basePackages() default {};
    String sqlSessionFactoryRef() default "";
    String sqlSessionTemplateRef() default "";
    // ...
}
```

它的核心是 `@Import(MapperScannerRegistrar.class)`。Spring 在解析配置类时，会递归查找所有注解上的 `@Import`，把其中的类当作"额外要处理的组件"。

**所以 `@MapperScan` 本身不干活，它只是告诉 Spring："请把 `MapperScannerRegistrar` 这个类也纳入你的处理流程。"**

### 4.3 第二环：`MapperScannerRegistrar` 注册 `MapperScannerConfigurer`

`MapperScannerRegistrar` 实现了 Spring 的 **`ImportBeanDefinitionRegistrar`** 接口。这个接口是 Spring 为第三方框架提供的**"手动注册 BeanDefinition"** 的扩展点。

它的核心方法 `registerBeanDefinitions()` 做了一件事：

1. 读取 `@MapperScan` 注解上的属性（`basePackages`、`sqlSessionFactoryRef`、`sqlSessionTemplateRef` 等）。
2. 创建一个 **`MapperScannerConfigurer`** 的 `BeanDefinition`，把这些属性设置进去。
3. 把这个 `BeanDefinition` 注册到 Spring 容器中。

### 4.4 第三环：`MapperScannerConfigurer` 触发扫描

`MapperScannerConfigurer` 实现了 **`BeanDefinitionRegistryPostProcessor`** 接口。这个接口比普通的 `BeanFactoryPostProcessor` 更早执行——它在**所有 Bean 定义加载完成之后、Bean 实例化之前**，允许你**继续向容器中注册新的 BeanDefinition**。

它的 `postProcessBeanDefinitionRegistry()` 方法做了两件事：

1. **创建 `ClassPathMapperScanner`**：这是一个专门用于扫描 Mapper 接口的扫描器，继承自 Spring 的 `ClassPathBeanDefinitionScanner`。
2. **把 `@MapperScan` 中的属性传递给它**：包括 `sqlSessionFactoryBeanName`、`sqlSessionTemplateBeanName`、`annotationClass` 等。
3. **调用 `scanner.scan(basePackages)`**：开始扫描指定包路径下的接口。

### 4.5 第四环：`ClassPathMapperScanner` 改写 BeanDefinition

这是整个流程中**最核心的一步**。

当 `doScan()` 找到 `UserMapper` 这个接口后，它生成的原始 `BeanDefinition` 的 `beanClass` 是 `UserMapper.class`。但 `ClassPathMapperScanner` 会**立刻改写这个 BeanDefinition**：

```java
// 在 processBeanDefinitions() 中
definition.setBeanClass(this.mapperFactoryBeanClass);  // 默认是 MapperFactoryBean.class
definition.getConstructorArgumentValues().addGenericArgumentValue(beanClassName);  // 把 UserMapper 作为构造参数传入
```

**这意味着**：Spring 容器中名为 `userMapper` 的 Bean，其 `beanClass` 不再是 `UserMapper`，而是 **`MapperFactoryBean`**，并且 `UserMapper` 被作为构造参数传了进去。

同时，`ClassPathMapperScanner` 还会根据 `@MapperScan` 中是否指定了 `sqlSessionFactoryRef` 或 `sqlSessionTemplateRef`，把对应的 Bean 名称也设置到 `BeanDefinition` 的属性中，后续注入时会用到这些名称。

### 4.6 第五环：`MapperFactoryBean` 初始化，触发 SQL 注入

Spring 容器启动到实例化阶段，开始创建 `userMapper` 这个 Bean。它发现 `beanClass` 是 `MapperFactoryBean`，于是：

1. 实例化 `MapperFactoryBean`。
2. 注入 `sqlSessionFactory` 和 `sqlSessionTemplate`。
3. 调用 `afterPropertiesSet()` → `checkDaoConfig()`。

**关键就在这里**：`checkDaoConfig()` 调用了 `configuration.addMapper(UserMapper.class)`，触发了第二部分的 MyBatis-Plus 增强逻辑：

```
MapperFactoryBean.checkDaoConfig()
    ↓ 调用 configuration.addMapper(UserMapper.class)
    ↓
MybatisConfiguration.addMapper()
    ↓ 委托给 MybatisMapperRegistry（不是原生 MapperRegistry）
    ↓
MybatisMapperRegistry.addMapper()
    ↓ 创建 MybatisMapperAnnotationBuilder
    ↓ 调用 parser.parse()
    ↓
MybatisMapperAnnotationBuilder.parse()
    ├── 1. 解析 XML 和注解（原生逻辑）
    └── 2. 检查是否继承 BaseMapper
            ↓ 如果是，触发 SqlInjector
            ↓
        AbstractMethod.injectMappedStatement()
            ↓ 生成 MappedStatement 并注册到 Configuration.mappedStatements
    ↓
UserMapper 拥有了 insert、selectById 等通用方法
```

### 4.7 `@MapperScan` 与 `MybatisMapperRegistry` 的关联

**这是整个体系中最关键的连接点，之前分开讲容易让人困惑。**

- `@MapperScan` 负责在 Spring 层注册 `MapperFactoryBean`；
- `MapperFactoryBean` 初始化时调用 `configuration.addMapper()`；
- 这个调用进入 MyBatis-Plus 重写的 `MybatisMapperRegistry`，触发 SQL 注入。

**`@MapperScan` 是"触发器"，`MybatisMapperRegistry` 是"执行者"，连接它们的是 `MapperFactoryBean` 的初始化过程。**

- 没有 `@MapperScan`，`MapperFactoryBean` 不会被注册，`addMapper()` 不会被调用，`MybatisMapperRegistry` 永远不会被触发。
- 没有 `MybatisMapperRegistry`，`addMapper()` 会走原生逻辑，`BaseMapper` 的通用方法不会被注入。

两者缺一不可。

### 4.8 完整链路

```text
Spring 容器 refresh()
    ↓
ConfigurationClassPostProcessor 解析配置类上的 @MapperScan
    ↓ 发现 @Import(MapperScannerRegistrar.class)
MapperScannerRegistrar.registerBeanDefinitions()
    ↓ 注册 MapperScannerConfigurer 的 BeanDefinition
    ↓
BeanDefinitionRegistryPostProcessor 执行阶段
    ↓ MapperScannerConfigurer.postProcessBeanDefinitionRegistry()
    ↓ 创建 ClassPathMapperScanner，调用 scan()
    ↓
ClassPathMapperScanner.doScan()
    ↓ 扫描到 UserMapper 接口
    ↓ 把 BeanDefinition 的 beanClass 改写为 MapperFactoryBean
    ↓ 把 UserMapper 作为构造参数传入
    ↓ 把 sqlSessionFactoryRef / sqlSessionTemplateRef 设置到属性中
    ↓ 注册到 Spring 容器
    ↓
Spring 实例化 userMapper Bean
    ↓ 发现是 MapperFactoryBean
    ↓ 注入 SqlSessionFactory / SqlSessionTemplate
    ↓ 调用 afterPropertiesSet() → checkDaoConfig()
    ↓
MapperFactoryBean.checkDaoConfig()
    ↓ configuration.addMapper(UserMapper.class)
    ↓
MybatisConfiguration.addMapper()
    ↓ 委托给 MybatisMapperRegistry
    ↓
MybatisMapperRegistry.addMapper()
    ↓ 创建 MybatisMapperAnnotationBuilder
    ↓ parser.parse()
    ↓
MybatisMapperAnnotationBuilder.parse()
    ├── 1. 解析 XML 和注解（原生逻辑）
    └── 2. 检查是否继承 BaseMapper，触发 SQL 注入
    ↓
UserMapper 拥有了所有通用 CRUD 方法
    ↓
MapperFactoryBean.getObject()
    ↓
SqlSessionTemplate.getMapper(UserMapper.class)
    ↓
Configuration.getMapper() → MapperRegistry.getMapper()
    ↓
MapperProxyFactory.newInstance() → JDK 动态代理
    ↓
返回 UserMapper 代理对象，放入 Spring 容器
    ↓
你的 Service 中 @Autowired UserMapper 注入的是这个代理对象
```

---

## 第五部分：`MybatisPlusAutoConfiguration` 与 `@MapperScan` 的关系

### 5.1 两条独立的路径

很多人在学到这里时会感到困惑：`MybatisPlusAutoConfiguration` 和 `@MapperScan` 到底什么关系？怎么感觉没关联上？

**核心答案：它们是两条独立的路径，最终在 `MapperFactoryBean` 这里汇合。`MybatisPlusAutoConfiguration` 负责"造引擎"，`@MapperScan` 负责"装轮子"，两者通过 `SqlSessionFactory` / `SqlSessionTemplate` 这两个 Bean 连接起来。**

#### 路径一：`MybatisPlusAutoConfiguration` —— 造引擎

它的职责是**创建基础设施 Bean**：

```java
@Configuration
@ConditionalOnClass({SqlSessionFactory.class, MybatisSqlSessionFactoryBean.class})
@ConditionalOnSingleCandidate(DataSource.class)
public class MybatisPlusAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public SqlSessionFactory sqlSessionFactory(DataSource dataSource) {
        MybatisSqlSessionFactoryBean factory = new MybatisSqlSessionFactoryBean();
        // ...
        return factory.getObject();
    }

    @Bean
    @ConditionalOnMissingBean
    public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
        return new SqlSessionTemplate(sqlSessionFactory);
    }
}
```

它产出两个关键 Bean：

- **`SqlSessionFactory`**：内部持有 `MybatisConfiguration`，启动时负责解析 SQL、注册 `MappedStatement`。
- **`SqlSessionTemplate`**：线程安全的 `SqlSession` 封装，后续被 `MapperFactoryBean` 使用。

**注意**：`MybatisPlusAutoConfiguration` **不负责注册任何 Mapper 接口**。它不知道 `UserMapper` 的存在。

#### 路径二：`@MapperScan` —— 装轮子

它的职责是**把 Mapper 接口注册为 `MapperFactoryBean`**：

```java
@MapperScan("com.xx.mapper")
```

它通过 `MapperScannerRegistrar` → `MapperScannerConfigurer` → `ClassPathMapperScanner`，把 `UserMapper` 接口的 `BeanDefinition` 改写为 `MapperFactoryBean`。

**注意**：`@MapperScan` **不负责创建 `SqlSessionFactory`**。它只管扫描接口、注册 BeanDefinition。

### 5.2 汇合点：`MapperFactoryBean`

两条路径在 `MapperFactoryBean` 这里汇合。

`MapperFactoryBean` 继承自 `SqlSessionDaoSupport`，需要注入 `SqlSessionFactory` 或 `SqlSessionTemplate`：

- 单数据源下：容器里只有一个 `SqlSessionFactory` 和一个 `SqlSessionTemplate`（由 `MybatisPlusAutoConfiguration` 创建），Spring 按类型自动注入。
- 多数据源下：容器里有多个同类型 Bean，必须通过 `@MapperScan` 的 `sqlSessionFactoryRef` 和 `sqlSessionTemplateRef` 显式指定。

### 5.3 为什么会产生"没关联上"的感觉

因为**它们确实没有直接的代码调用关系**：

- `MybatisPlusAutoConfiguration` 里没有任何一行代码提到 `@MapperScan`。
- `@MapperScan` 的源码里也没有任何一行代码提到 `MybatisPlusAutoConfiguration`。

它们是通过 **Spring 容器的依赖注入机制** 间接关联的：

- `MybatisPlusAutoConfiguration` 把 `SqlSessionFactory` 和 `SqlSessionTemplate` 注册到容器中。
- `@MapperScan` 把 `MapperFactoryBean` 注册到容器中。
- Spring 在实例化 `MapperFactoryBean` 时，发现它需要 `SqlSessionFactory`，于是从容器中查找并注入。

**这就像"发电厂"和"冰箱"的关系**：发电厂不关心谁在用它的电，冰箱也不关心电从哪个发电厂来，它们通过"电网"（Spring 容器）连接在一起。

### 5.4 多数据源时，`MybatisPlusAutoConfiguration` 还需要吗？

**答案是：仍然会被加载，但它创建的 `SqlSessionFactory` 和 `SqlSessionTemplate` Bean 会被跳过。**

只要 `mybatis-plus-boot-starter` 在 classpath 上，`MybatisPlusAutoConfiguration` 就会被 Spring Boot 的自动配置机制发现并处理。但它的关键 Bean 方法上都标了 `@ConditionalOnMissingBean`：

```java
@Bean
@ConditionalOnMissingBean
public SqlSessionFactory sqlSessionFactory(DataSource dataSource) { ... }

@Bean
@ConditionalOnMissingBean
public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory sqlSessionFactory) { ... }
```

当你在多数据源配置中手动声明了 `pgsqlSqlSessionFactory` 和 `pgsqlSqlSessionTemplate` 后：

- `MybatisPlusAutoConfiguration.sqlSessionFactory()` 发现容器中已有 `SqlSessionFactory` 类型的 Bean，**跳过**。
- `MybatisPlusAutoConfiguration.sqlSessionTemplate()` 发现容器中已有 `SqlSessionTemplate` 类型的 Bean，**跳过**。

**但自动配置类仍然会被加载**，它还有其他作用：

| 作用 | 说明 |
|:---|:---|
| **`@EnableConfigurationProperties(MybatisPlusProperties.class)`** | 绑定 `mybatis-plus.*` 配置项 |
| **注册 `MybatisPlusInterceptor` 等扩展 Bean** | 分页、乐观锁等插件 |
| **注册 `SqlSessionFactory` 的定制器** | 如 `ConfigurationCustomizer`、`MybatisPlusPropertiesCustomizer` |

**最关键的问题**：你手动创建的 `SqlSessionFactory` 用的是什么 `FactoryBean`？

- 如果用 `MybatisSqlSessionFactoryBean`（正确）：MyBatis-Plus 的动态注入功能全部生效。
- 如果用原生的 `SqlSessionFactoryBean`（错误）：`BaseMapper` 的通用方法不会被注入，调用 `userMapper.insert()` 会报 `Invalid bound statement` 错误。

---

## 第六部分：多数据源实战

### 6.1 问题的引出

单数据源场景下，`@MapperScan` 不需要指定 `sqlSessionFactoryRef` 和 `sqlSessionTemplateRef`，因为容器里只有一个 `SqlSessionFactory` 和一个 `SqlSessionTemplate`，Spring 可以按类型自动注入。

但在多数据源场景下，容器里存在多个同类型的 Bean，自动注入就会因为**类型不唯一**而报错。这时就必须显式指定。

### 6.2 一个典型的多数据源配置（有问题的版本）

```java
@Configuration
@MapperScan(
    sqlSessionFactoryRef = "pgsqlSqlSessionFactory",
    basePackages = {
        "com.chinamobile.cmss.gzzx.framework.base.mapper",
        "com.chinamobile.cmss.gzzx.cmdb.console.mapper.postgresql"
    }
)
public class PostgreSQLConfig {

    @Bean(name = "pgsqlDataSource")
    @Primary
    @ConfigurationProperties(prefix = "spring.datasource.pgsql")
    public DataSource dataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean(name = "pgsqlSqlSessionFactory")
    @Primary
    public SqlSessionFactory sqlSessionFactory(@Qualifier("pgsqlDataSource") DataSource dataSource) throws Exception {
        MybatisSqlSessionFactoryBean bean = new MybatisSqlSessionFactoryBean();
        bean.setDataSource(dataSource);
        // ...
        return bean.getObject();
    }

    @Bean(name = "pgsqlTransactionManager")
    @Primary
    public DataSourceTransactionManager dataSourceTransactionManager(
            @Qualifier("pgsqlDataSource") DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
}
```

### 6.3 这个配置有什么问题？

**问题在于：没有指定 `sqlSessionTemplateRef`，也没有定义 `pgsqlSqlSessionTemplate` Bean。**

根据前面的原理，`MapperScannerConfigurer` 在创建 `MapperFactoryBean` 时，会根据 `@MapperScan` 的属性决定如何注入 `SqlSessionTemplate`：

- **如果指定了 `sqlSessionTemplateRef`**：`MapperFactoryBean` 直接使用容器中已存在的 `SqlSessionTemplate` Bean。
- **如果只指定了 `sqlSessionFactoryRef`**：`MapperFactoryBean` 会用这个 Factory **自己创建一个私有的 `SqlSessionTemplate`**。
- **如果两者都没指定**：按类型自动注入，多个同类型 Bean 时报错。
- **如果两者同时指定**：启动后会提示` Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.`，所以，最佳实践是只指定``sqlSessionTemplateRef``

所以，只指定 `sqlSessionFactoryRef` 会导致：**每个 Mapper 接口都会持有一个私有的 `SqlSessionTemplate` 实例。**

这会导致两个问题：

1. **无法复用与统一管理**：每个 Mapper 都持有一个私有的 `SqlSessionTemplate`，无法享受共享 Bean 带来的内存优化和统一配置（如 `ExecutorType`）的好处。
2. **潜在的启动失败风险**：如果容器中还存在其他 `SqlSessionTemplate` Bean，你的 Mapper 在自动注入时可能会因**类型不唯一**而导致启动失败。

### 6.4 正确的多数据源配置

一个完整的多数据源配置，需要为每个数据源提供 **`DataSource`**、**`SqlSessionFactory`**、**`SqlSessionTemplate`** 和 **`TransactionManager`** 这一整套 Bean。

```java
@Configuration
@MapperScan(
    sqlSessionTemplateRef = "pgsqlSqlSessionTemplate",  // 显式指定要使用的模板
    basePackages = {
        "com.chinamobile.cmss.gzzx.framework.base.mapper",
        "com.chinamobile.cmss.gzzx.cmdb.console.mapper.postgresql"
    }
)
public class PostgreSQLConfig {

    @Bean(name = "pgsqlDataSource")
    @Primary
    @ConfigurationProperties(prefix = "spring.datasource.pgsql")
    public DataSource dataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean(name = "pgsqlSqlSessionFactory")
    @Primary
    public SqlSessionFactory sqlSessionFactory(@Qualifier("pgsqlDataSource") DataSource dataSource) throws Exception {
        MybatisSqlSessionFactoryBean bean = new MybatisSqlSessionFactoryBean();
        bean.setDataSource(dataSource);
        // ...
        return bean.getObject();
    }

    /**
     * 2. 新增 SqlSessionTemplate Bean，并引用对应的 SqlSessionFactory
     */
    @Bean(name = "pgsqlSqlSessionTemplate")
    @Primary
    public SqlSessionTemplate pgsqlSqlSessionTemplate(
            @Qualifier("pgsqlSqlSessionFactory") SqlSessionFactory sqlSessionFactory) {
        return new SqlSessionTemplate(sqlSessionFactory);
    }

    @Bean(name = "pgsqlTransactionManager")
    @Primary
    public DataSourceTransactionManager dataSourceTransactionManager(
            @Qualifier("pgsqlDataSource") DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
}
```

### 6.5 多数据源配置的要点

| 要点 | 说明 |
|:---|:---|
| **每个数据源一套完整 Bean** | `DataSource` + `SqlSessionFactory` + `SqlSessionTemplate` + `TransactionManager` |
| **`@MapperScan` 显式引用** | 指定  `sqlSessionTemplateRef`，明确绑定 |
| **`@Primary` 标记主数据源** | 避免自动注入时类型不唯一 |
| **`@Qualifier` 精确注入** | 在 Bean 方法参数上使用，避免歧义 |

---

## 第七部分：MyBatis-Plus 批量插入实战

### 7.1 `saveBatch` 的局限：伪批量

MyBatis-Plus 的 `ServiceImpl` 中提供了 `saveBatch` 方法，用起来很方便：

```java
userService.saveBatch(userList);
```

但它的底层实现是：

```java
for (T entity : entityList) {
    sqlSession.insert(sqlStatement, entity);   // 每次都是一条独立 INSERT
    if (i % batchSize == 0) {
        sqlSession.flushStatements();          // 定期刷入
    }
}
```

**它并没有合并 SQL**，只是减少了事务提交次数。每条记录仍然是一条独立的 `INSERT` 语句，网络往返次数并没有减少。

### 7.2 `rewriteBatchedStatements=true`：驱动层的"打包员"

要真正提升 `saveBatch` 的性能，必须配合 JDBC 驱动参数 `rewriteBatchedStatements=true`。它的原理是：MySQL JDBC 驱动的 `executeBatchInternal()` 方法中，当 `batchHasPlainStatements == false` 且 `rewriteBatchedStatements == true` 时，会走 `executeBatchedInserts()` 路径，把多条结构相同的 `INSERT` 在内存中拼成一条多值 SQL：

```sql
INSERT INTO user (name, age) VALUES (?, ?), (?, ?), (?, ?), ...;
```

这样一次网络往返就能插入所有数据。但这种方式依赖驱动层重写，且只对 `INSERT` 等特定语句有效。

### 7.3 `insertBatchSomeColumn`：真批量，动态注入的产物

MyBatis-Plus 提供了一个官方扩展方法 `insertBatchSomeColumn`，它从 **SQL 生成阶段** 就直接构建多值 `INSERT`，不依赖 JDBC 驱动的重写。

**它的原理**：回顾第二部分，MyBatis-Plus 通过重写 `MybatisMapperAnnotationBuilder`，在启动时遍历 `BaseMapper` 的通用方法，调用每个 `AbstractMethod` 的 `injectMappedStatement()` 方法，把生成的 `MappedStatement` 注册到 `Configuration` 中。

`insertBatchSomeColumn` 正是 `AbstractMethod` 的一个子类。它的 `injectMappedStatement()` 会：

1. 通过 `tableInfo.getAllInsertSqlColumnMaybeIf()` 获取列名脚本；
2. 通过 `tableInfo.getAllInsertSqlPropertyMaybeIf()` 获取属性表达式；
3. 拼接出带 `<foreach>` 的 SQL 模板：

```sql
INSERT INTO user (name, age) VALUES
<foreach collection="list" item="item" separator=",">
    (#{item.name}, #{item.age})
</foreach>
```

4. 创建 `MappedStatement` 并注册到 `Configuration.mappedStatements` 中。

运行时，MyBatis 的 `<foreach>` 标签会把 `list` 展开，**最终只生成一条多值 SQL**，一次网络往返发送给数据库。

**使用示例**：

**步骤 1：自定义 SQL 注入器**

```java
@Component
public class MySqlInjector extends DefaultSqlInjector {
    @Override
    public List<AbstractMethod> getMethodList(Class<?> mapperClass) {
        List<AbstractMethod> methodList = super.getMethodList(mapperClass);
        methodList.add(new InsertBatchSomeColumn());
        return methodList;
    }
}
```

**步骤 2：定义自定义 BaseMapper**

```java
public interface MyBaseMapper<T> extends BaseMapper<T> {
    int insertBatchSomeColumn(List<T> entityList);
}
```

**步骤 3：业务 Mapper 继承**

```java
public interface UserMapper extends MyBaseMapper<User> {}
```

**步骤 4：调用**

```java
userMapper.insertBatchSomeColumn(userList);
```

### 7.4 两种方案对比

| 特性 | `saveBatch` + `rewriteBatchedStatements` | `insertBatchSomeColumn` |
|:---|:---|:---|
| **谁在合并** | JDBC 驱动 | MyBatis-Plus（SQL 生成阶段） |
| **合并时机** | SQL 发送前 | SQL 生成时 |
| **是否依赖 JDBC 参数** | 是 | 否 |
| **SQL 形式** | 多条独立 INSERT（驱动重写） | 始终一条多值 INSERT |
| **性能** | 中 | 高 |

**一句话总结**：`saveBatch` 是"伪批量"，需要驱动帮忙才能合并；`insertBatchSomeColumn` 是"真批量"，从一开始就只生成一条 SQL。后者正是 MyBatis-Plus 动态注入机制的绝佳案例。

### 7.5 多数据源下的批量插入

在多数据源配置中，我们为每个数据源都定义了独立的 `SqlSessionFactory` 和 `SqlSessionTemplate`。`insertBatchSomeColumn` 作为 `AbstractMethod`，同样会被注入到每个 `SqlSessionFactory` 对应的 `Configuration` 中。因此，只要你的 Mapper 继承了自定义的 `MyBaseMapper`，批量插入方法就能正常工作。

---

## 总结

回到最初的问题，我们把整个体系串起来：

**MyBatis 核心层**：
- `Configuration` 是全局容器，`MappedStatement` 是每条 SQL 的说明书；
- 启动时 `MapperRegistry` + `MapperAnnotationBuilder` 把 SQL 解析成 `MappedStatement` 注册到 `Configuration`；
- 运行时 `MapperProxyFactory` + `MapperProxy` 通过 JDK 动态代理把接口方法调用翻译成对 `MappedStatement` 的执行。

**MyBatis-Plus 增强层**：
- 重写 `MapperRegistry` 和 `MapperAnnotationBuilder`，在启动解析阶段把 `BaseMapper` 通用方法的 `MappedStatement` 也注册进去；
- 通过 `MybatisConfiguration` 和 `MybatisSqlSessionFactoryBean` 替换原生实现，无缝嵌入 MyBatis 启动流程；
- `insertBatchSomeColumn` 是动态注入机制的典型应用，从 SQL 生成阶段就构建多值 INSERT，实现真·批量插入。

**Spring 集成层**：
- `MapperFactoryBean` 实现 Spring 的 `FactoryBean`，让 Mapper 接口能作为 Bean 被注入；
- `MapperFactoryBean.checkDaoConfig()` 调用 `configuration.addMapper()`，是连接 Spring 层和 MyBatis-Plus 增强层的关键；
- `SqlSessionTemplate` 提供线程安全的 `SqlSession` 封装；
- `@MapperScan` 通过 `@Import` + `ImportBeanDefinitionRegistrar` + `BeanDefinitionRegistryPostProcessor` 三个扩展点，把 Mapper 接口批量注册为 `MapperFactoryBean`。

**两条路径的汇合**：
- `MybatisPlusAutoConfiguration` 造引擎（`SqlSessionFactory`、`SqlSessionTemplate`）；
- `@MapperScan` 装轮子（`MapperFactoryBean`）；
- 两者通过 Spring 容器的依赖注入在 `MapperFactoryBean` 处汇合。

**多数据源场景**：
- 每个数据源需要一套完整的 `DataSource` + `SqlSessionFactory` + `SqlSessionTemplate` + `TransactionManager`；
- `@MapperScan` 中同时指定 `sqlSessionFactoryRef` 和 `sqlSessionTemplateRef`，明确绑定；
- 只指定 `sqlSessionFactoryRef` 会导致每个 Mapper 持有私有 `SqlSessionTemplate`，虽然能跑，但不够规范。

**一句话概括整个体系**：MyBatis 在启动时把 SQL 解析成 `MappedStatement` 存入 `Configuration`，运行时通过 JDK 动态代理把接口方法调用翻译成对 `MappedStatement` 的执行；MyBatis-Plus 在启动解析阶段"偷偷"多注册一批 `MappedStatement`，并用这套机制实现了真·批量插入等实用功能；Spring 则通过 `@MapperScan` 和 `MapperFactoryBean` 把这些接口装配成可注入的 Bean。三者各司其职，共同构成了我们日常使用的持久层框架。