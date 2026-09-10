---
title: Spring自定义Mybatis数据源，到底发生了什么？
date: 2026-09-10 18:15:00
tags:
  - spring
  - mybatis
  - mybatis plus
categories:
  - develop
---

在之前的文章中，我们为了在 Spring 中使用多数据源，引入了如下的数据源代码
```java
@Configuration
@MapperScan(sqlSessionTemplateRef = "pgsqlSqlSessionTemplate",
        basePackages = {"com.chinamobile.cmss.gzzx.framework.base.mapper",
                "com.chinamobile.cmss.gzzx.cmdb.console.mapper.postgresql"})
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
        bean.setPlugins(mybatisConfig.pageInterceptor());
        GlobalConfig globalConfig = GlobalConfigUtils.defaults();
        globalConfig.setMetaObjectHandler(mybatisConfig.MyMetaObjectHandler());
        bean.setGlobalConfig(globalConfig);
        return bean.getObject();
    }

    @Bean(name = "pgsqlSqlSessionTemplate")
    @Primary
    public SqlSessionTemplate sqlSessionTemplate(@Qualifier("pgsqlSqlSessionFactory") SqlSessionFactory sqlSessionFactory) {
        return new SqlSessionTemplate(sqlSessionFactory);
    }

    @Bean(name = "pgsqlTransactionManager")
    @Primary
    public DataSourceTransactionManager dataSourceTransactionManager(@Qualifier("pgsqlDataSource") DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
}
```

它到底是怎么样生效的呢？发生了什么？结合这篇能对之前的文章有更好的了解。

# 一、先看这个配置类做了什么
# 以 `PostgreSQLConfig` 为例：完整走一遍 MyBatis-Plus 的启动与运行流程

这个配置类非常典型，它把我们前面讨论的所有概念都串起来了。下面以它为线索，从 Spring 启动到运行时，完整走一遍。

---

## 一、先看这个配置类做了什么

**它做了四件事**：

1. 创建 PostgreSQL 的 `DataSource`（连接池）。
2. 创建 `SqlSessionFactory`，用 `MybatisSqlSessionFactoryBean`（不是原生 MyBatis 的 `SqlSessionFactoryBean`），并绑定 PostgreSQL 数据源。
3. 创建 `SqlSessionTemplate`，绑定上一步的 `SqlSessionFactory`。
4. 创建 `DataSourceTransactionManager`，绑定 PostgreSQL 数据源。

同时，`@MapperScan` 告诉 Spring：**指定包下的 Mapper 接口，使用名为 `pgsqlSqlSessionTemplate` 的 Template。**

---

## 二、启动阶段：Spring 容器的处理顺序

### 第 1 步：解析 `@MapperScan`

Spring 在解析配置类时，发现 `PostgreSQLConfig` 上有 `@MapperScan`，于是通过 `@Import` 机制引入 `MapperScannerRegistrar`：

```
ConfigurationClassPostProcessor 解析 PostgreSQLConfig
    ↓ 发现 @MapperScan
    ↓ 读取属性：sqlSessionTemplateRef = "pgsqlSqlSessionTemplate"
    ↓           basePackages = ["...framework.base.mapper", "...mapper.postgresql"]
    ↓ 通过 @Import 引入 MapperScannerRegistrar
MapperScannerRegistrar.registerBeanDefinitions()
    ↓ 创建 MapperScannerConfigurer 的 BeanDefinition
    ↓ 把 sqlSessionTemplateRef、basePackages 设置进去
    ↓ 注册到 Spring 容器
```

此时还没有任何 Mapper 被扫描，也没有任何 SQL 被解析。

### 第 2 步：`MapperScannerConfigurer` 扫描 Mapper

`MapperScannerConfigurer` 是 `BeanDefinitionRegistryPostProcessor`，在所有 BeanDefinition 加载完成后触发：

```
MapperScannerConfigurer.postProcessBeanDefinitionRegistry()
    ↓ 创建 ClassPathMapperScanner
    ↓ 传入 sqlSessionTemplateBeanName = "pgsqlSqlSessionTemplate"
    ↓ 调用 scanner.scan(basePackages)
    ↓ 扫描到 N 个 Mapper 接口
        ├── com.xxx.framework.base.mapper.UserMapper
        ├── com.xxx.cmdb.console.mapper.postgresql.CmdbMapper
        └── ...
    ↓ 对每个接口，改写 BeanDefinition：
        beanClass = MapperFactoryBean
        constructorArgs = [UserMapper.class]
        propertyValues["sqlSessionTemplate"] = RuntimeBeanReference("pgsqlSqlSessionTemplate")
    ↓ 注册到 Spring 容器
```

**关键点**：因为 `@MapperScan` 里指定了 `sqlSessionTemplateRef`，所以每个 `MapperFactoryBean` 的 `BeanDefinition` 里都会写入这条属性。**它不会走"只指定 Factory 时自己 new Template"的路径，而是直接用容器里共享的 `pgsqlSqlSessionTemplate`。**

### 第 3 步：创建 `pgsqlDataSource`

Spring 开始实例化 Bean。先创建 `pgsqlDataSource`：

```java
DataSourceBuilder.create().build() 
    → 读取 spring.datasource.pgsql.* 配置
    → 创建 HikariCP DataSource
```

这一步产出的是一个 PostgreSQL 连接池。

### 第 4 步：创建 `pgsqlSqlSessionFactory`（核心）

```java
MybatisSqlSessionFactoryBean bean = new MybatisSqlSessionFactoryBean();
bean.setDataSource(dataSource);
bean.setPlugins(mybatisConfig.pageInterceptor());
bean.setGlobalConfig(globalConfig);
return bean.getObject();
```

`bean.getObject()` 内部会执行 `buildSqlSessionFactory()`，这是整个 MyBatis-Plus 增强机制真正生效的地方：

```
MybatisSqlSessionFactoryBean.buildSqlSessionFactory()
    ↓
创建 MybatisConfiguration（继承自 MyBatis 的 Configuration）
    ↓ 初始化时，用一个 MybatisMapperRegistry 替换原生的 MapperRegistry
    ↓
把 DataSource 设置到 Environment 中
    ↓
把分页插件设置到 interceptorChain 中
    ↓
把 GlobalConfig（含 MetaObjectHandler）设置到 MybatisConfiguration
    ↓
解析 <mappers> 中的 XML（如果有）
    ↓ 对每个 XML，解析出 MappedStatement 注册到 Configuration
    ↓
返回 DefaultSqlSessionFactory（持有 MybatisConfiguration）
```

**注意：这一步完成时，`pgsqlSqlSessionFactory` 内部的 `MybatisConfiguration` 里已经存好了 XML 中定义的 SQL，但 `BaseMapper` 的通用方法还没注册，注解中的 SQL 也没注册。**

### 第 5 步：创建 `pgsqlSqlSessionTemplate`

```java
return new SqlSessionTemplate(pgsqlSqlSessionFactory);
```

创建出来的 `SqlSessionTemplate` 内部持有 `pgsqlSqlSessionFactory`，并通过它间接持有 `MybatisConfiguration`。

**这个 Template 是共享单例的，后面所有指定包下的 Mapper 都用它。**

### 第 6 步：创建每个 `MapperFactoryBean`，触发 SQL 解析

Spring 开始实例化每个 Mapper 对应的 `MapperFactoryBean`：

```
实例化 MapperFactoryBean（以 UserMapper 为例）
    ↓
注入属性：setSqlSessionTemplate(pgsqlSqlSessionTemplate)
    ↓
调用 afterPropertiesSet() → checkDaoConfig()
    ↓
MapperFactoryBean.checkDaoConfig()
    ↓ 从 sqlSessionTemplate 拿到 Configuration（实际是 MybatisConfiguration）
    ↓ 调用 configuration.addMapper(UserMapper.class)
    ↓
MybatisConfiguration.addMapper()
    ↓ 委托给 MybatisMapperRegistry（不是原生 MapperRegistry）
    ↓
MybatisMapperRegistry.addMapper()
    ↓ 注册 MapperProxyFactory 到 knownMappers
    ↓ 创建 MybatisMapperAnnotationBuilder，调用 parse()
    ↓
MybatisMapperAnnotationBuilder.parse()
    ├── 1. 解析 XML（已在上一步完成，跳过）
    ├── 2. 解析注解 SQL（如 @Select）→ 注册 MappedStatement
    └── 3. 检查是否继承 BaseMapper
            ↓ 是！触发 SqlInjector
            ↓
        SqlInjector.inspectInject()
            ↓ 遍历所有 AbstractMethod（Insert、SelectById、InsertBatchSomeColumn 等）
            ↓ 为每个方法生成 MappedStatement 并注册到 Configuration
    ↓
UserMapper 拥有了 insert、selectById、updateById 等所有通用方法
```

**这一步是 `@MapperScan` 和 `MybatisMapperRegistry` 的连接点。** 没有 `@MapperScan`，就不会有 `MapperFactoryBean`；没有 `MapperFactoryBean.checkDaoConfig()`，就不会触发 `addMapper()`，也就不会触发 SQL 注入。

### 第 7 步：`MapperFactoryBean.getObject()` 创建代理对象

依赖注入阶段，Service 需要 `UserMapper`：

```
Spring 调用 MapperFactoryBean.getObject()
    ↓
getSqlSession().getMapper(UserMapper.class)
    ↓ getSqlSession() 返回 pgsqlSqlSessionTemplate
    ↓
SqlSessionTemplate.getMapper(UserMapper.class)
    ↓
getConfiguration().getMapper(UserMapper.class, this)
    ↓ getConfiguration() 转调到 pgsqlSqlSessionFactory.getConfiguration()
    ↓ 拿到 MybatisConfiguration
    ↓
MybatisConfiguration.getMapper()
    ↓ 委托给 MybatisMapperRegistry
    ↓
MybatisMapperRegistry.getMapper()
    ↓ 从 knownMappers 中取出 MapperProxyFactory
    ↓
MapperProxyFactory.newInstance(pgsqlSqlSessionTemplate)
    ↓ new MapperProxy(pgsqlSqlSessionTemplate, UserMapper.class, ...)
    ↓ Proxy.newProxyInstance(...) → JDK 动态代理
    ↓
返回 UserMapper 代理对象
```

**代理对象内部持有的 `SqlSessionTemplate` 就是 `pgsqlSqlSessionTemplate`。** 这个绑定关系在代理对象创建时就固定了，后续无论谁调用，都会走这个 Template。

---

## 三、运行时：一次查询的完整链路

假设 Service 中注入了 `UserMapper`，现在调用：

```java
User user = userMapper.selectById(1L);
```

执行链：

```
UserMapper 代理对象.selectById(1L)
    ↓
MapperProxy.invoke()                              ← JDK 动态代理回调
    ↓
MapperMethod.execute(sqlSession, args)
    ↓ sqlSession 就是 pgsqlSqlSessionTemplate
    ↓ 从 Configuration.mappedStatements 中查找 id = "UserMapper.selectById"
    ↓
SqlSessionTemplate.selectOne("UserMapper.selectById", 1L)
    ↓ 内部走动态代理 SqlSessionInterceptor
    ↓
SqlSessionInterceptor.invoke()
    ↓ 从 pgsqlSqlSessionFactory 获取 SqlSession
    ↓
SqlSessionFactory.openSession()
    ↓ 从 Environment 取 DataSource（pgsqlDataSource）
    ↓
SqlSession.selectOne()
    ↓
Executor.query()
    ↓
Transaction.getConnection()
    ↓ 从 pgsqlDataSource 借一个 PostgreSQL Connection
    ↓
PreparedStatement.executeQuery()
    ↓
执行 SQL，操作 PostgreSQL 数据库
    ↓
ResultSetHandler 映射结果
    ↓
返回 User 对象
```

**关键点：整条链路操作的是 PostgreSQL 数据库，因为 `pgsqlSqlSessionTemplate` 绑定的是 `pgsqlSqlSessionFactory`，而 `pgsqlSqlSessionFactory` 绑定的是 `pgsqlDataSource`。**

---

## 四、回到你的配置，回答几个关键问题

### Q1：为什么只指定 `sqlSessionTemplateRef` 就够了？

因为 `pgsqlSqlSessionTemplate` 内部已经持有 `pgsqlSqlSessionFactory`，而 `pgsqlSqlSessionFactory` 内部持有 `MybatisConfiguration` 和 `pgsqlDataSource`。**指定 Template 就等于指定了 Factory、Configuration 和 DataSource。**

### Q2：如果没有 `sqlSessionTemplateRef` 会怎样？

每个 `MapperFactoryBean` 会用注入的 `pgsqlSqlSessionFactory` 自己 `new` 一个私有的 `SqlSessionTemplate`。功能上能跑，但：

- 每个 Mapper 持有一个私有 Template，浪费内存；
- 无法统一配置 `ExecutorType`；
- 如果容器里还有其他 `SqlSessionTemplate`，可能因类型不唯一而报错。

### Q3：`@MapperScan` 和 `MybatisMapperRegistry` 是怎么关联的？

连接点是 `MapperFactoryBean.checkDaoConfig()` 中的这一行：

```java
configuration.addMapper(this.mapperInterface);
```

- `@MapperScan` 注册 `MapperFactoryBean`；
- `MapperFactoryBean` 初始化时调用 `addMapper()`；
- `addMapper()` 进入 `MybatisMapperRegistry`，触发 SQL 注入。

**两者缺一不可。**

### Q4：SQL 解析是在 `getObject()` 时做的吗？

不是。SQL 解析发生在 `MapperFactoryBean.checkDaoConfig()` 中，这个方法的调用时机是 Spring 的 `afterPropertiesSet()`，**比 `getObject()` 早**。

- XML 中的 SQL：在 `pgsqlSqlSessionFactory` 创建时（`buildSqlSessionFactory()`）解析；
- 注解 SQL 和 `BaseMapper` 通用方法：在 `MapperFactoryBean.checkDaoConfig()` 中解析；
- `getObject()` 只负责从已经解析好的 `Configuration` 中创建代理对象。

### Q5：`Configuration` 绑定在 `SqlSessionTemplate` 上吗？

不是直接绑定。持有链是：

```
pgsqlSqlSessionTemplate
    ↓ 持有
pgsqlSqlSessionFactory
    ↓ 持有
MybatisConfiguration（核心容器，含 mappedStatements、mapperRegistry、DataSource 等）
```

`SqlSessionTemplate.getConfiguration()` 是转调 `sqlSessionFactory.getConfiguration()`，自己并不持有 `Configuration`。

### Q6：`MybatisPlusAutoConfiguration` 在这个配置里还起作用吗？

起作用，但它创建的 `SqlSessionFactory` 和 `SqlSessionTemplate` 因为 `@Primary` 和 `@ConditionalOnMissingBean` 被跳过了。它剩下的作用：

- 绑定 `mybatis-plus.*` 配置项；
- 提供 `ConfigurationCustomizer`、`MybatisPlusPropertiesCustomizer` 等扩展点；
- 如果你的项目里没有手动配置 `MybatisPlusInterceptor`，它也可以注册（但你的配置里手动传了 `mybatisConfig.pageInterceptor()`，所以自动配置的会被跳过）。

**你没有排除它，也不需要排除它。**

---

## 五、完整时序图

```
┌─────────────────────────────────────────────────────────────────────┐
│ 阶段一：BeanDefinition 加载                                          │
│                                                                     │
│  解析 @MapperScan → 注册 MapperScannerConfigurer                     │
│  MapperScannerConfigurer 扫描 → 注册 N 个 MapperFactoryBean          │
│      每个 BeanDefinition 里写入：                                     │
│          sqlSessionTemplate = "pgsqlSqlSessionTemplate"             │
└─────────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────────┐
│ 阶段二：Bean 实例化                                                  │
│                                                                     │
│  创建 pgsqlDataSource（PostgreSQL 连接池）                            │
│       ↓                                                             │
│  创建 pgsqlSqlSessionFactory                                          │
│      → MybatisSqlSessionFactoryBean.buildSqlSessionFactory()         │
│      → 创建 MybatisConfiguration（含 MybatisMapperRegistry）         │
│      → 解析 XML → 注册 XML 中的 MappedStatement                      │
│       ↓                                                             │
│  创建 pgsqlSqlSessionTemplate                                        │
│      → new SqlSessionTemplate(pgsqlSqlSessionFactory)               │
│       ↓                                                             │
│  创建每个 MapperFactoryBean                                          │
│      → setSqlSessionTemplate(pgsqlSqlSessionTemplate)               │
│      → afterPropertiesSet() → checkDaoConfig()                       │
│      → configuration.addMapper(UserMapper.class)                    │
│      → MybatisMapperRegistry.addMapper()                            │
│      → MybatisMapperAnnotationBuilder.parse()                       │
│          ├── 解析注解 SQL → 注册 MappedStatement                     │
│          └── 继承 BaseMapper → 触发 SQL 注入 → 注册通用方法           │
└─────────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────────┐
│ 阶段三：依赖注入                                                     │
│                                                                     │
│  Service 需要 UserMapper                                             │
│      → MapperFactoryBean.getObject()                                │
│      → pgsqlSqlSessionTemplate.getMapper(UserMapper.class)          │
│      → MybatisMapperRegistry.getMapper()                            │
│      → MapperProxyFactory.newInstance(pgsqlSqlSessionTemplate)      │
│      → 返回代理对象，注入 Service                                     │
└─────────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────────┐
│ 运行时                                                               │
│                                                                     │
│  userMapper.selectById(1L)                                          │
│      → MapperProxy.invoke()                                         │
│      → MapperMethod.execute(pgsqlSqlSessionTemplate, args)          │
│      → SqlSessionTemplate.selectOne()                               │
│      → SqlSessionInterceptor → pgsqlSqlSessionFactory.openSession() │
│      → Environment → pgsqlDataSource → Connection                   │
│      → 执行 SQL，操作 PostgreSQL                                     │
│      → 返回结果                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 六、一句话总结

**你的 `PostgreSQLConfig` 做了一件很清晰的事：定义了一套 PostgreSQL 的持久层基础设施（DataSource → SqlSessionFactory → SqlSessionTemplate → TransactionManager），然后用 `@MapperScan` 把指定包下的 Mapper 绑定到 `pgsqlSqlSessionTemplate` 上。**

启动时，`MapperFactoryBean` 在 `checkDaoConfig()` 中调用 `configuration.addMapper()`，触发 MyBatis-Plus 的 `MybatisMapperRegistry`，完成注解 SQL 和 `BaseMapper` 通用方法的注入；运行时，代理对象通过 `pgsqlSqlSessionTemplate` 找到 `pgsqlSqlSessionFactory`，最终从 `pgsqlDataSource` 获取 PostgreSQL 连接执行 SQL。

**整个链路是：`@MapperScan` → `MapperFactoryBean` → `SqlSessionTemplate` → `SqlSessionFactory` → `Configuration` → `DataSource` → `Connection` → PostgreSQL。**