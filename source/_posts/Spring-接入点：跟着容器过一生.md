---
title: Spring 接入点：跟着容器过一生
date: 2026-09-11 14:42:31
tags:
  - spring
categories:
  - develop
---

我总是记不住 Spring 的接入点，一个字，乱！

`BeanFactoryPostProcessor`、`BeanPostProcessor`、`ApplicationListener`、`ApplicationContextInitializer`、`SmartInitializingSingleton`、`SmartLifecycle`……名字都认识，但一到用的时候就乱：  
到底谁先谁后？谁改配置？谁改 Bean？谁监听启动？谁负责关闭？

后来我接了一个多租户任务平台，叫 **TaskPilot**，需求一串：

1. 启动前从配置中心拉租户配置；
2. 每个租户动态注册一个数据源；
3. 给任务处理器加审计代理；
4. 所有任务处理器创建完后，校验任务类型不能重复；
5. 应用启动完成后预热缓存；
6. MQ 消费者要随容器启动，关闭时优雅停机。

做完这个项目，我突然发现：  
**Spring 接入点根本不用背，它们就是容器一生中的一个个插槽。**

而且除了“插槽”，还有两条暗线贯穿始终：

- **Aware 线**：Bean 在生命周期里认领容器资源；
- **事件线**：容器一生都在广播，谁想听就听。

下面跟着 TaskPilot 的生命周期走一遍。

---

## 先看总图：容器的一生

四条线一起看：外层是 Spring Boot 启动流程，内层是 Spring 容器 `refresh()` 流程，横线是事件广播，微线是 Bean 创建过程。

最重要的还是 Spring Boot 启动主流程，尤其是前 5 步。把这 5 步记住，后面的一系列问题都会自然而然地对应上。

### 1. Spring Boot 启动主流程

```text
SpringApplication.run()
  │
  ├─ 0. 发布 ApplicationStartingEvent
  │
  ├─ 1. prepareEnvironment()              // ★ 先于 ApplicationContext 创建
  │        ├── 创建/获取 Environment
  │        ├── 加载配置文件、命令行参数
  │        ├── 触发 EnvironmentPostProcessor
  │        └── 发布 ApplicationEnvironmentPreparedEvent
  │
  ├─ 2. createApplicationContext()
  │      └── 创建 ApplicationContext
  │          └── 内部构造 DefaultListableBeanFactory   // BeanFactory 在这里诞生
  │
  ├─ 3. prepareContext()
  │        ├── 设置 Environment
  │        ├── 执行 ApplicationContextInitializer
  │        ├── 发布 ApplicationContextInitializedEvent
  │        ├── 注册主配置类 primarySources
  │        ├── 注册监听器 ApplicationListener
  │        └── 发布 ApplicationPreparedEvent
  │
  ├─ 4. refreshContext()  →  refresh()
  │
  ├─ 5. afterRefresh()
  ├─ 6. 发布 ApplicationStartedEvent
  ├─ 7. 调用 Runners（ApplicationRunner / CommandLineRunner）
  ├─ 8. 发布 ApplicationReadyEvent
  └─ 9. run() 返回 → 应用运行

异常时：
  └─ 发布 ApplicationFailedEvent

关闭时：
  └─ 发布 ContextClosedEvent
```

### 2. refresh() 内部 12 步

```text
refresh()
  ├─ prepareRefresh()
  ├─ obtainFreshBeanFactory()
  ├─ prepareBeanFactory()
  ├─ postProcessBeanFactory()
  ├─ invokeBeanFactoryPostProcessors()   // 含 BeanDefinitionRegistryPostProcessor
  ├─ registerBeanPostProcessors()
  ├─ initMessageSource()
  ├─ initApplicationEventMulticaster()   // 初始化事件多播器
  ├─ onRefresh()                          // 子类扩展，如内嵌容器启动
  ├─ registerListeners()                  // 注册 ApplicationListener
  ├─ finishBeanFactoryInitialization()    // 实例化 + Aware + 初始化前后 + SmartInitializingSingleton
  └─ finishRefresh()                      // SmartLifecycle.start + 发布 ContextRefreshedEvent
```

### 3. 事件时间线

```text
ApplicationStartingEvent             ← Bean 不存在
ApplicationEnvironmentPreparedEvent  ← Bean 不存在
ApplicationContextInitializedEvent   ← Bean 不存在
ApplicationPreparedEvent             ← Bean 不存在
────────────────────── refresh() ──────────────────────
ContextRefreshedEvent                ← Bean 已存在
ApplicationStartedEvent              ← Bean 已存在
ApplicationReadyEvent                ← Bean 已存在
ApplicationFailedEvent               ← 任意阶段抛异常
────────────────────── 关闭 ──────────────────────
ContextClosedEvent                   ← Bean 已存在
```

一句话记住：

> **改环境、初始化上下文、加图纸、改图纸、改成品、拿资源、做校验、听事件、管启停、做销毁。**

---

## 第一幕：容器还没出生，先改环境

TaskPilot 启动前，需要从配置中心拉取租户列表。

这个时候 Spring 容器还没创建，你不可能拿到 `ApplicationContext`，更不可能拿 Bean。  
你能改的只有 `Environment`。

接入点：`EnvironmentPostProcessor`。

```java
public class TenantEnvironmentPostProcessor implements EnvironmentPostProcessor {

    @Override
    public void postProcessEnvironment(ConfigurableEnvironment environment,
                                       SpringApplication application) {
        Map<String, Object> config = new HashMap<>();
        config.put("taskpilot.tenant.names", "alpha,beta");
        environment.getPropertySources()
                .addFirst(new MapPropertySource("tenantConfig", config));
    }
}
```

注册方式，传统写法：

```properties
# META-INF/spring.factories
org.springframework.boot.env.EnvironmentPostProcessor=\
com.example.taskpilot.TenantEnvironmentPostProcessor
```

Spring Boot 2.7+ 更推荐：

```text
# META-INF/spring/org.springframework.boot.env.EnvironmentPostProcessor.imports
com.example.taskpilot.TenantEnvironmentPostProcessor
```

它适合做：

- 从配置中心拉配置；
- 解密配置；
- 添加自定义 `PropertySource`；
- 根据环境变量决定激活哪些配置。

再往后一点，`ApplicationContext` 已经创建，但 Bean 还没开始创建。  
这时候用 `ApplicationContextInitializer`。

```java
public class TenantContextInitializer
        implements ApplicationContextInitializer<ConfigurableApplicationContext> {

    @Override
    public void initialize(ConfigurableApplicationContext context) {
        context.getEnvironment().setActiveProfiles("tenant");

        context.addApplicationListener(event ->
                System.out.println("收到事件：" + event.getClass().getSimpleName()));
    }
}
```

一句话区分：

| 接入点 | 时机 | 能做什么 |
|---|---|---|
| `EnvironmentPostProcessor` | ApplicationContext 之前 | 改 Environment |
| `ApplicationContextInitializer` | context 创建后，refresh 前 | 初始化 context、激活 profile、加监听器 |

---

## 第二幕：图纸阶段，动态注册 BeanDefinition

Spring 启动时会先读配置类、扫描注解，得到一堆 `BeanDefinition`。

你可以把 `BeanDefinition` 理解成“造 Bean 的图纸”。  
这时候 Bean 还没造出来。

TaskPilot 需要给每个租户注册一个数据源。  
这种“往图纸柜里加图纸”的事，用 `BeanDefinitionRegistryPostProcessor`：

```java
@Component
public class TenantDataSourceRegistrar
        implements BeanDefinitionRegistryPostProcessor {

    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
        String tenants = "alpha,beta";
        for (String tenant : tenants.split(",")) {
            GenericBeanDefinition bd = new GenericBeanDefinition();
            bd.setBeanClass(HikariDataSource.class);
            bd.getPropertyValues().add("jdbcUrl",
                    "jdbc:mysql://localhost:3306/task_" + tenant);
            bd.getPropertyValues().add("username", "root");
            bd.getPropertyValues().add("password", "root");

            registry.registerBeanDefinition("dataSource_" + tenant, bd);
        }
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        // 这里可以留空
    }
}
```

它比 `BeanFactoryPostProcessor` 更早执行，因为它能注册新的 `BeanDefinition`。

如果你只是想修改已经存在的图纸，比如把所有 `Service` 改成懒加载，用 `BeanFactoryPostProcessor`：

```java
@Component
public class LazyServicePostProcessor implements BeanFactoryPostProcessor {

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        for (String name : beanFactory.getBeanDefinitionNames()) {
            BeanDefinition bd = beanFactory.getBeanDefinition(name);
            if (bd.getBeanClassName() != null
                    && bd.getBeanClassName().endsWith("Service")) {
                bd.setLazyInit(true);
            }
        }
    }
}
```

记住这个比喻：

| 接入点 | 比喻 | 作用 |
|---|---|---|
| `BeanDefinitionRegistryPostProcessor` | 往图纸柜里加图纸 | 注册新的 BeanDefinition |
| `BeanFactoryPostProcessor` | 改图纸 | 修改已有 BeanDefinition |
| `BeanPostProcessor` | 改成品 | 修改已经创建出来的 Bean 实例 |

---

## 第三幕：造 Bean：Aware 是补给站，BeanPostProcessor 是加工厂

图纸有了，Spring 开始实例化 Bean。

一个 Bean 的完整生命周期，大致是这样：

```text
构造方法
  -> 属性注入 populateBean
  -> Aware 回调
  -> BeanPostProcessor.postProcessBeforeInitialization
  -> @PostConstruct
  -> InitializingBean.afterPropertiesSet
  -> init-method
  -> BeanPostProcessor.postProcessAfterInitialization
  -> 使用
  -> @PreDestroy
  -> DisposableBean.destroy
  -> destroy-method
```

这里最容易被忽略的是 **Aware**。

### Aware：Bean 的补给站

Aware 是一组回调接口。  
Bean 实现它们以后，Spring 会在创建 Bean 的过程中，把容器里的资源“塞”给它。

常见 Aware：

| Aware 接口 | 注入什么 | 典型用途 |
|---|---|---|
| `BeanNameAware` | 当前 Bean 名称 | 日志、调试 |
| `BeanClassLoaderAware` | ClassLoader | 类加载相关 |
| `BeanFactoryAware` | BeanFactory | 拿 BeanFactory |
| `EnvironmentAware` | Environment | 读配置 |
| `EmbeddedValueResolverAware` | 占位符解析器 | 解析 `${}` |
| `ResourceLoaderAware` | ResourceLoader | 加载资源文件 |
| `ApplicationEventPublisherAware` | 事件发布器 | 发布自定义事件 |
| `MessageSourceAware` | MessageSource | 国际化 |
| `ApplicationContextAware` | ApplicationContext | 拿整个容器 |
| `ServletContextAware` | ServletContext | Web 环境 |

看个例子：

```java
@Component
public class TaskExecutor implements ApplicationContextAware,
        ApplicationEventPublisherAware, InitializingBean {

    private ApplicationContext context;
    private ApplicationEventPublisher publisher;

    @Override
    public void setApplicationContext(ApplicationContext context) {
        this.context = context;
    }

    @Override
    public void setApplicationEventPublisher(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }

    @PostConstruct
    public void postConstruct() {
        System.out.println("@PostConstruct");
    }

    @Override
    public void afterPropertiesSet() {
        System.out.println("InitializingBean.afterPropertiesSet");
    }

    @PreDestroy
    public void preDestroy() {
        System.out.println("@PreDestroy");
    }
}
```

Aware 不是用来“改变容器流程”的，它更像是 Bean 在说：

> 我准备好了，把容器里的工具给我一份，我后面要用。

### BeanPostProcessor：Bean 的加工厂

TaskPilot 有个需求：  
所有带 `@Audit` 的任务处理器，调用时都要记录耗时。

这种“对 Bean 实例做包装、做代理”的事情，用 `BeanPostProcessor`：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Audit {
}
```

```java
@Component
public class AuditBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (bean.getClass().isAnnotationPresent(Audit.class)) {
            return Proxy.newProxyInstance(
                    bean.getClass().getClassLoader(),
                    bean.getClass().getInterfaces(),
                    (proxy, method, args) -> {
                        long start = System.nanoTime();
                        try {
                            return method.invoke(bean, args);
                        } finally {
                            long cost = (System.nanoTime() - start) / 1_000_000;
                            System.out.printf("[Audit] %s.%s 耗时 %d ms%n",
                                    beanName, method.getName(), cost);
                        }
                    });
        }
        return bean;
    }
}
```

真实项目里，Spring AOP 的代理也主要发生在 `BeanPostProcessor` 阶段。  
你熟悉的 `@Transactional`、`@Async`，背后都有它的影子。

如果想在代理里发事件，让被代理对象实现 `ApplicationEventPublisherAware` 即可。这就是 Aware 线和事件线的交汇：

> Aware 让 Bean 拿到事件发布器，Bean 再通过事件发布器把消息广播出去。

---

## 第四幕：所有单例就绪，做全局校验

TaskPilot 有很多 `TaskHandler`，每个负责一种任务类型。

我们希望启动时检查：**任务类型不能重复。**

这个校验不能太早做，因为 Handler 可能还没创建完；也不能太晚，最好在应用真正对外服务前发现错误。

接入点：`SmartInitializingSingleton`。

```java
public interface TaskHandler {
    String type();
    void handle(String task);
}
```

```java
@Component
public class TaskHandlerValidator implements SmartInitializingSingleton {

    private final Map<String, TaskHandler> handlers;

    public TaskHandlerValidator(Map<String, TaskHandler> handlers) {
        this.handlers = handlers;
    }

    @Override
    public void afterSingletonsInstantiated() {
        Set<String> types = new HashSet<>();
        handlers.forEach((name, handler) -> {
            if (!types.add(handler.type())) {
                throw new IllegalStateException(
                        "重复的任务类型: " + handler.type() + "，Bean: " + name);
            }
        });
    }
}
```

`afterSingletonsInstantiated()` 执行时：

- 所有非懒加载单例都已经创建完；
- 但 `ContextRefreshedEvent` 还没发布；
- 适合做全局校验、收集所有实现、建立映射关系。

---

## 第五幕：事件——一个判断就够了

Spring 里事件很多，但真正需要记住的只有一个判断：

> **事件发布的时候，Bean 存在吗？**

存在 → `@EventListener` 随便用。
不存在 → `@EventListener` 根本收不到，必须用 `SpringApplication.addListeners` 或 `spring.factories`。

为什么？因为 `@EventListener` 依赖 Bean 被创建、事件多播器初始化、监听器被注册，这些都在 `refresh()` 里发生。而 Spring Boot 的早期事件，发生在 `refresh()` 之前。

```text
ApplicationStartingEvent             ← Bean 不存在，@EventListener 收不到
ApplicationEnvironmentPreparedEvent  ← Bean 不存在，@EventListener 收不到
ApplicationContextInitializedEvent   ← Bean 不存在，@EventListener 收不到
ApplicationPreparedEvent             ← Bean 不存在，@EventListener 收不到
────────────────────── refresh() ──────────────────────
ContextRefreshedEvent                ← Bean 已存在，@EventListener 能收到
ApplicationStartedEvent              ← Bean 已存在，@EventListener 能收到
ApplicationReadyEvent                ← Bean 已存在，@EventListener 能收到
────────────────────── 关闭 ──────────────────────
ContextClosedEvent                   ← Bean 已存在，@EventListener 能收到
```

早期事件只能用 `ApplicationListener` + 手动注册：

```java
public class TenantEnvListener
        implements ApplicationListener<ApplicationEnvironmentPreparedEvent> {

    @Override
    public void onApplicationEvent(ApplicationEnvironmentPreparedEvent event) {
        // 这里能拿到 Environment，但拿不到 Bean
        ConfigurableEnvironment env = event.getEnvironment();
        System.out.println("租户配置可以读了");
    }
}
```

注册：

```java
SpringApplication app = new SpringApplication(TaskPilotApplication.class);
app.addListeners(new TenantEnvListener());
app.run(args);
```

框架级扩展用 `spring.factories`：

```properties
org.springframework.context.ApplicationListener=\
com.example.taskpilot.TenantEnvListener
```

后期事件直接用 `@EventListener`：

```java
@Component
public class CacheWarmer {

    @EventListener(ApplicationReadyEvent.class)
    public void warmUp() {
        System.out.println("应用已就绪，开始预热缓存");
    }
}
```

要发事件，注入 `ApplicationEventPublisher`：

```java
@Component
public class AuditPublisher {

    private final ApplicationEventPublisher publisher;

    public AuditPublisher(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }

    public void publish(String beanName, String method, long cost) {
        publisher.publishEvent(new TaskAuditEvent(this, beanName, method, cost));
    }
}
```

事件默认同步。想异步，`@Async` 加在 `@EventListener` 上，或换一个带 `TaskExecutor` 的 `ApplicationEventMulticaster`。

一句话：

> 事件能不能被 `@EventListener` 收到，只看发布时 Bean 出生了没有。

---

## 第六幕：容器刷新完成，开始对外服务

所有单例就绪后，Spring 会启动 `SmartLifecycle`，然后发布 `ContextRefreshedEvent`。

再往后是 Spring Boot 的启动事件：

```text
SmartLifecycle.start
  -> ContextRefreshedEvent
  -> afterRefresh()
  -> ApplicationStartedEvent
  -> ApplicationRunner / CommandLineRunner
  -> ApplicationReadyEvent
```

TaskPilot 需要在应用就绪后预热缓存。

用 `@EventListener`：

```java
@Component
public class CacheWarmer {

    @EventListener(ApplicationReadyEvent.class)
    public void warmUp() {
        System.out.println("应用已就绪，开始预热缓存");
    }
}
```

也可以用 `ApplicationRunner`：

```java
@Component
public class AdminInitializer implements ApplicationRunner {

    @Override
    public void run(ApplicationArguments args) {
        System.out.println("初始化管理员账号");
    }
}
```

区别：

| 接入点 | 时机 | 适合 |
|---|---|---|
| `ApplicationRunner` | `ApplicationReadyEvent` 之前 | 启动后执行一次，带结构化参数 |
| `CommandLineRunner` | 同上 | 启动后执行一次，参数是 `String...` |
| `@EventListener(ApplicationReadyEvent.class)` | `ApplicationRunner` 之后 | 应用完全就绪后做事 |

如果只是监听 Spring 容器事件，用 `ContextRefreshedEvent`；如果是 Spring Boot 应用，推荐 `ApplicationReadyEvent`。

---

## 第七幕：运行与关闭，体面离场

TaskPilot 有个 MQ 消费者，需要：

- 容器启动后开始消费；
- 容器关闭时先停止消费；
- 等待正在执行的任务完成；
- 再关闭线程池。

这种“随容器启停”的组件，用 `SmartLifecycle`：

```java
@Component
public class MqConsumerLifecycle implements SmartLifecycle {

    private volatile boolean running = false;

    @Override
    public void start() {
        running = true;
        System.out.println("启动 MQ 消费者");
    }

    @Override
    public void stop() {
        System.out.println("优雅停止 MQ 消费者");
        running = false;
    }

    @Override
    public boolean isRunning() {
        return running;
    }

    @Override
    public int getPhase() {
        return 0;
    }
}
```

`phase` 越小，启动越早，停止越晚。比如：

- 数据源连接池：`phase = 0`，最早启动，最晚关闭；
- MQ 消费者：`phase = 100`，晚点启动，早点停止；
- Web 服务器：`phase = Integer.MAX_VALUE`，最后启动，最先停止。

关闭阶段顺序：

```text
ContextClosedEvent
  -> SmartLifecycle.stop
  -> @PreDestroy
  -> DisposableBean.destroy
  -> destroy-method
```

所以优雅停机可以这样写：

```java
@Component
public class GracefulShutdown {

    @PreDestroy
    public void shutdown() {
        System.out.println("等待任务完成，关闭线程池");
    }
}
```

注意：`ContextClosedEvent` 发布时，`SmartLifecycle.stop` 还没执行。  
所以资源停止优先用 `SmartLifecycle`，最后的清理用 `@PreDestroy` 或 `DisposableBean`。

---

## 一张地图：Spring 接入点全景

| 阶段 | 接入点 | 一句话 | TaskPilot 例子 |
|---|---|---|---|
| 配置环境 | `EnvironmentPostProcessor` | 改 Environment | 从配置中心拉租户配置 |
| 早期事件 | `ApplicationListener` + `addListeners` | 监听 Bean 出生前事件 | 监听 `ApplicationEnvironmentPreparedEvent` |
| 初始化上下文 | `ApplicationContextInitializer` | refresh 前改 context | 激活 tenant profile |
| 加图纸 | `BeanDefinitionRegistryPostProcessor` | 注册 BeanDefinition | 动态注册租户数据源 |
| 改图纸 | `BeanFactoryPostProcessor` | 修改 BeanDefinition | 把 Service 改成懒加载 |
| 改成品 | `BeanPostProcessor` | 修改 Bean 实例 | 给 `@Audit` 加代理 |
| 拿容器资源 | `Aware` | 注入容器资源 | `ApplicationContextAware`、`ApplicationEventPublisherAware` |
| 初始化 | `@PostConstruct` / `InitializingBean` / `init-method` | Bean 初始化 | 初始化连接池 |
| 全局校验 | `SmartInitializingSingleton` | 所有单例创建完 | 校验任务类型唯一 |
| 容器就绪 | `ContextRefreshedEvent` | 容器刷新完成 | 预热本地缓存 |
| 启动后一次 | `ApplicationRunner` / `CommandLineRunner` | Boot 启动后执行 | 初始化管理员账号 |
| 应用就绪 | `ApplicationReadyEvent` | 可以对外服务 | 注册到注册中心 |
| 事件监听 | `ApplicationListener` / `@EventListener` | 听广播 | 审计事件、租户事件 |
| 事件发布 | `ApplicationEventPublisher` | 发广播 | 发布 `TaskAuditEvent` |
| 启停组件 | `Lifecycle` / `SmartLifecycle` | 随容器启停 | MQ 消费者优雅启停 |
| 销毁 | `@PreDestroy` / `DisposableBean` / `destroy-method` | 释放资源 | 关闭线程池 |
| 条件装配 | `Condition` / `@Conditional` | 按条件注册 | 只有多租户模式才加载 |
| 配置导入 | `ImportSelector` / `ImportBeanDefinitionRegistrar` | `@EnableXxx` 底层 | 自定义 `@EnableTaskPilot` |

---

## 最后送你一个决策树

以后写代码前，先问自己：

```text
要改配置环境？          -> EnvironmentPostProcessor
要监听早期事件？         -> ApplicationListener + addListeners / spring.factories
要初始化 context？      -> ApplicationContextInitializer
要新增 BeanDefinition？ -> BeanDefinitionRegistryPostProcessor
要修改 BeanDefinition？ -> BeanFactoryPostProcessor
要修改 Bean 实例？      -> BeanPostProcessor
要拿容器资源？          -> Aware
要所有单例创建完做校验？ -> SmartInitializingSingleton
要监听事件？            -> ApplicationListener / @EventListener
要发事件？              -> ApplicationEventPublisher
随容器启停？            -> SmartLifecycle
启动后执行一次？         -> ApplicationRunner / CommandLineRunner
销毁清理？              -> @PreDestroy / DisposableBean
```

口诀：

> 环境上下文，定义实例化；  
> Aware 拿资源，前后处理器；  
> 单例完成后，刷新发事件；  
> 监听与发布，启停生命周期；  
> Boot 有 Runner，关闭做销毁。

现在再看到 `BeanPostProcessor`，你不会觉得它只是一个名词。  
你会知道：它站在“Bean 已经造出来，正在初始化”的那个路口。

现在再看到 `ApplicationListener`，你不会觉得它只是“监听器”。  
你会知道：它是容器一生广播系统里的收音机，而且——

> **它能不能收到，只看发布的那一刻 Bean 出生了没有。**

现在再看到 `ApplicationContextAware`，你不会觉得它只是“一个接口”。  
你会知道：它是 Bean 在生命周期中领取容器资源的补给站。

Spring 接入点不是一张名词表，而是一条时间线。  
记住容器走到哪一步，你就知道该用谁。  
记住事件在广播什么，你就知道该听什么。  
记住 Bean 在领取什么，你就知道该拿什么。