---
title: Nacos是怎么样和Jasypt结合的？
date: 2026-09-11 17:18:46
tags:
  - spring
  - nacos
  - jasypt
categories:
  - develop
---

# Nacos 和 Jasypt 怎么样结合的？

## 一、从一个尴尬的场景说起

假设你是一个微服务项目的开发，某天安全部门发来一份整改通知：

> 检查发现，Nacos 配置中心中 `spring.datasource.password` 为明文存储，存在敏感信息泄露风险，请于本周内完成整改。

你打开 Nacos 控制台一看：

```yaml
spring:
  datasource:
    url: jdbc:mysql://10.64.3.151:3306/cpc
    username: root
    password: MySecret@123   # ← 赤裸裸地躺在这里
```

改就改呗，把密码挪到环境变量里？但运维说不行，环境变量在 K8s 里也是明文；放到 K8s Secret？那还得改部署流程。折腾一圈，同事丢过来一个方案：**用 Jasypt 加密，Nacos 里只存密文。**

于是配置变成了这样：

```yaml
spring:
  datasource:
    password: ENC(Q9PNYb+QmZOd9cdlX8b3zXK7S3eGKsQlY5z2sCewoRo=)
```

应用启动后，程序拿到的是明文 `MySecret@123`，但 Nacos 里看到的只是一串乱码。完美。

这背后的功臣，就是 **Jasypt**。而它和 **Nacos** 的配合，并不只是"加密存储"这么简单——里面还藏着一个关于"加载顺序"的经典问题。这篇文章就从原理到实践，把整个链路讲清楚。

---

## 二、Nacos 配置管理是怎么实现的？

要理解 Jasypt 怎么和 Nacos 结合，得先知道 Nacos 的配置是怎么"进"到你的 Spring 应用里的。

### 1. 启动时的引导阶段

Spring Cloud 应用启动时，会先创建一个**引导上下文（Bootstrap Context）**。它的使命很单纯：**在应用主上下文创建之前，把远程配置中心里的配置拉下来**。

加载顺序大致是这样：

```
Spring Boot 启动
│
├─ 进入引导阶段，读取 bootstrap.yml
│   └─ 拿到 Nacos 服务器地址、namespace、username、password
│
├─ Nacos 客户端拿着这些信息去连接 Nacos 服务器
│   └─ 拉取对应的远程配置（Data ID = cpc-console-server-local.yaml）
│
├─ 把拉回来的配置塞进 Environment
│
└─ 创建主 ApplicationContext，开始正常的 Bean 装配
```

**关键点**：Nacos 的连接信息（`server-addr`、`password` 等）必须在**引导阶段就能拿到**，否则根本连不上服务器。这就是为什么它们通常写在 `bootstrap.yml` 里——这个文件的加载时机足够早。

### 2. 启动后的动态刷新

配置拉下来之后不是就完事了。Nacos 客户端会和服务端保持一个**长轮询（Long Polling）** 通道（新版使用Grpc长连接，对后续理解无影响）：

- 客户端不断问服务端："我关心的这些配置变了吗？"
- 没变就挂起等待；变了就立即把新配置拉回来
- 更新本地 `Environment` 和 `@RefreshScope` 的 Bean

这就是你在 Nacos 控制台改完配置、刷新一下页面就能看到效果的原因。

### 3. 小结

Nacos 配置管理的本质是：**引导阶段拉配置 + 运行时长轮询刷新**。它介入 Spring 的时机非常早，早到很多"常规"的初始化逻辑都还没跑。这为后面的问题埋下了伏笔。

---

## 三、Jasypt 是什么？

Jasypt 全称 **Java Simplified Encryption**，是一个 Java 加解密库。而在 Spring Boot 场景下，大家通常用的是它的封装：`jasypt-spring-boot-starter`。

它的用法极其简单——**你只管写密文，它负责解密**：

1. 用工具把明文加密，得到 `ENC(xxx)` 格式的字符串
2. 把密文写进配置文件
3. 启动时提供一个解密密钥（`jasypt.encryptor.password`）
4. 之后程序里任何 `@Value("${xxx}")` 或 `environment.getProperty("xxx")` 拿到的，都是解密后的明文

### 一个最小的例子

```yaml
# application.yml
spring:
  datasource:
    password: ENC(Q9PNYb+QmZOd9cdlX8b3zXK7S3eGKsQlY5z2sCewoRo=)

jasypt:
  encryptor:
    password: my-master-key   # 解密密钥，强烈建议用环境变量传入
```

```java
@RestController
public class DemoController {
    @Value("${spring.datasource.password}")
    private String password;  // 拿到的是明文 MySecret@123
}
```

### 它是怎么"透明解密"的？

Jasypt 的核心手段不是"修改配置文件"，而是**包装 PropertySource**。

Spring 的 `Environment` 里维护着一堆 `PropertySource`（比如 `application.yml` 对应一个、`bootstrap.yml` 对应一个、系统环境变量对应一个……）。每个 `getProperty()` 调用最终都会落到某个 `PropertySource` 上。

Jasypt 通过一个 `BeanFactoryPostProcessor`（简称 BFPP），把这些 `PropertySource` 全部包装成 `EncryptablePropertySourceWrapper`。之后任何 `getProperty()` 调用都会经过这层包装：

```
调用 getProperty("spring.datasource.password")
  → 拿到原始值 "ENC(Q9PNYb+...)"
  → 检测到 ENC() 前缀
  → 调用 StringEncryptor.decrypt() 解密
  → 返回明文 "MySecret@123"
```

**这就是它"透明"的原因**——不管是 `@Value` 注入、`Environment.getProperty()`，还是 Nacos 客户端内部读配置，只要走了 `Environment` 这条路，就都会被自动解密。

---

## 四、两者结合：加解密与存储分发各司其职

把 Nacos 和 Jasypt 放在一起，分工非常清晰：

| 角色 | 职责 |
|------|------|
| **Jasypt** | 负责**加解密**：在应用侧把 `ENC()` 还原成明文 |
| **Nacos** | 负责**存储和分发**：保存密文，通过长轮询推送给客户端 |

结合方式有两种场景：

### 场景一：加密 Nacos 里存放的业务配置

这是最常见的用法。数据库密码、Redis 连接串、第三方 API Key 等敏感配置，全部加密后存进 Nacos：

```yaml
# Nacos 上的 cpc-console-server-local.yaml
spring:
  datasource:
    url: jdbc:mysql://10.64.3.151:3306/cpc
    password: ENC(Q9PNYb+QmZOd9cdlX8b3zXK7S3eGKsQlY5z2sCewoRo=)
  redis:
    password: ENC(abcdefg1234567...)
```

应用启动后，Jasypt 自动解密，业务代码无感知。**这一步没有任何时序问题**——因为 Nacos 只是把密文当普通字符串传过来，解密是应用侧的事。

### 场景二：加密 Nacos 自身的连接密码

这就麻烦了。你的 `bootstrap.yml` 里写着：

```yaml
spring:
  cloud:
    nacos:
      config:
        password: ENC(Q9PNYb+QmZOd9cdlX8b3zXK7S3eGKsQlY5z2sCewoRo=)
```

问题来了：

- **Nacos 客户端要在引导阶段连服务器**，此时需要密码的明文
- **Jasypt 的解密器要在主上下文初始化时才生效**，此时还没轮到它

如果 Nacos 拿到的是字面量 `ENC(...)`，它会拿着这串"乱码"去连服务器，结果自然是认证失败。

**这就是"先有鸡还是先有蛋"的经典问题**：要解密需要 Jasypt 跑起来，要 Jasypt 跑起来需要主上下文初始化，要主上下文初始化需要先从 Nacos 拉到配置，要连 Nacos 需要先解密……

看似死循环。但实际情况是，**在某些版本组合下，它是能跑通的**。为什么？

---

## 五、Jasypt 是怎么"抢跑"的？

答案藏在 `jasypt-spring-boot-starter` 的一个隐藏入口里，以及 Spring Cloud 的启动机制里。我们一点点把链路接起来。

### 1. Spring Boot 启动时，先扫 spring.factories

`SpringApplication` 在**构造阶段**（还没执行 `run()` 之前），就会通过 `SpringFactoriesLoader` 扫描 classpath 下**所有 jar 包**的 `META-INF/spring.factories` 文件，找出所有注册了 `ApplicationListener` 接口的实现类，并实例化它们。

```java
// SpringApplication 构造函数中
setListeners((Collection) 
    getSpringFactoriesInstances(ApplicationListener.class));
this.bootstrapContext = new DefaultBootstrapContext();
```

只要你的 classpath 上有 `spring-cloud-context`（引入 Nacos 配置中心时会间接带入），它的 `spring.factories` 里就注册了：

```properties
# spring-cloud-context/META-INF/spring.factories
org.springframework.context.ApplicationListener=\
org.springframework.cloud.bootstrap.BootstrapApplicationListener,\
...
```

所以 **`BootstrapApplicationListener` 在 `SpringApplication` 构造完成的那一刻，就已经躺在 `listeners` 集合里了**。

### 2. 两个监听器的 Order 值决定了谁先跑

`ApplicationEnvironmentPreparedEvent` 发布后，所有监听该事件的监听器会按 `Order` 值排序执行，值越小越优先：

| 监听器 | Order 值 | 执行顺序 |
|---|---|---|
| `BootstrapApplicationListener` | `Ordered.HIGHEST_PRECEDENCE + 5` | **先执行** |
| `ConfigFileApplicationListener` | `Ordered.HIGHEST_PRECEDENCE + 10` | **后执行** |

这个顺序是**有意为之**的。`BootstrapApplicationListener` 必须抢在 `ConfigFileApplicationListener` 之前，先把 Bootstrap 上下文建起来，去加载 `bootstrap.yml` 并连接 Nacos。如果让 `ConfigFileApplicationListener` 先跑，本地的 `application.yml` 会先被加载，配置的优先级覆盖关系就乱了。

### 3. BootstrapApplicationListener 做了什么

`BootstrapApplicationListener` 收到事件后，会**创建一个全新的 `SpringApplication`（我们称之为 Bootstrap 应用），并调用它的 `run()` 方法**，从而创建一个独立的 **Bootstrap ApplicationContext**：

```java
// BootstrapApplicationListener 核心逻辑
private ConfigurableApplicationContext bootstrapServiceContext(...) {
    SpringApplicationBuilder builder = new SpringApplicationBuilder()
        .profiles(environment.getActiveProfiles())
        .environment(bootstrapEnvironment)
        .web(WebApplicationType.NONE)
        .sources(BootstrapImportSelectorConfiguration.class);
    return builder.run();
}
```

### 4. 嵌套的启动流程——关键的一环

这里是最容易绕晕的地方。`BootstrapApplicationListener` 不是"执行完就结束了"，它启动的 Bootstrap 应用**会重新走一遍完整的 `SpringApplication.run()` 流程**，包括：

- 再次发布 `ApplicationEnvironmentPreparedEvent` 事件
- 再次触发所有监听器（`BootstrapApplicationListener` 会检测到已在 Bootstrap 过程中，直接跳过，防止无限递归）
- 再次调用 `ConfigFileApplicationListener`，从而加载 `bootstrap.yml`，并执行所有 `EnvironmentPostProcessor`

```
主应用 SpringApplication.run()
│
├─ prepareEnvironment()
│   └─ 发布 ApplicationEnvironmentPreparedEvent
│       │
│       ├─ BootstrapApplicationListener 先执行
│       │   └─ 创建 Bootstrap 应用：new SpringApplication(...).run()
│       │       │
│       │       ├─ prepareEnvironment()
│       │       │   └─ 再次发布 ApplicationEnvironmentPreparedEvent
│       │       │       ├─ BootstrapApplicationListener 检测到递归 → 跳过 ✅
│       │       │       └─ ConfigFileApplicationListener 执行
│       │       │           ├─ 加载 bootstrap.yml
│       │       │           └─ 执行所有 EnvironmentPostProcessor
│       │       │
│       │       ├─ refresh()
│       │       │   └─ Jasypt 的 BFPP 执行
│       │       │       └─ 包装 Bootstrap Environment 的 PropertySource ✅
│       │       │
│       │       └─ Bootstrap 应用启动完成
│       │
│       └─ ConfigFileApplicationListener 再执行（主应用侧）
│
├─ prepareContext()
│   └─ PropertySourceBootstrapConfiguration.initialize()
│       └─ NacosPropertySourceLocator.locate()
│           └─ environment.getProperty("spring.cloud.nacos.config.password")
│               └─ 走的是被 Jasypt 包装过的 PropertySource
│                   └─ 识别 ENC() → 解密 → 返回明文 ✅
│
└─ 主应用 refresh()
```

### 5. Jasypt 的 BootstrapConfiguration 在 Bootstrap 应用中被加载

Bootstrap 应用启动时，会去读取 `spring.factories` 中 `org.springframework.cloud.bootstrap.BootstrapConfiguration` 这个 key 下配置的类。

而 `jasypt-spring-boot-starter` 的 `spring.factories` 里正好注册了：

```properties
# jasypt-spring-boot-starter/META-INF/spring.factories
org.springframework.cloud.bootstrap.BootstrapConfiguration=\
com.ulisesbocchio.jasyptspringbootstarter.JasyptSpringCloudBootstrapConfiguration
```

这个类的源码很简洁：

```java
@Configuration
@ConditionalOnProperty(name = "jasypt.encryptor.bootstrap", 
                       havingValue = "true", matchIfMissing = true)
@Import(EnableEncryptablePropertiesConfiguration.class)
public class JasyptSpringCloudBootstrapConfiguration {
}
```

注意 `matchIfMissing = true`——**不写这个配置也默认生效**。

它 `@Import` 的配置类会向 Bootstrap 上下文注册一个 `static` 的 BFPP，在 Bootstrap 应用 `refresh()` 时执行，把 Bootstrap Environment 的所有 `PropertySource` 包装成解密壳。

**至此，Bootstrap Environment 具备了自动解密能力。** 之后 Nacos 读密码时，读到的是明文。

---

## 六、另一种注入 jasypt 密码的方式：自定义 EnvironmentPostProcessor

上面讲的是把 `jasypt.encryptor.password` 写在 `bootstrap.yml` 里的方案。但实际生产中，**解密密钥本身**也不该明文写在配置文件里——否则加密就失去意义了。

于是就有人想：能不能通过环境变量传入，在代码里动态注入？

### 一个真实的实现

```java
@Slf4j
public class DefensePostProcessor implements EnvironmentPostProcessor {

    private static final AtomicInteger COUNT = new AtomicInteger();
    private static final String DEFENSE_NAME = "defenseProperties";

    private static final Map<String, Object> JASYPT_MAP = new HashMap<String, Object>() {{
        put("jasypt.encryptor.ivGeneratorClassname", "org.jasypt.iv.NoIvGenerator");
        put("jasypt.encryptor.algorithm", "PBEWithMD5AndDES");
    }};

    private static final String JASYPT_PASSWORD_ENV = "YNLZX";
    private static final String JASYPT_PASSWORD_KEY = "jasypt.encryptor.password";

    @Override
    public void postProcessEnvironment(ConfigurableEnvironment environment, 
                                       SpringApplication application) {
        String jasyptPassword = (String) environment.getSystemEnvironment()
                                                    .get(JASYPT_PASSWORD_ENV);
        AtomicReference<String> jasyptPasswordRef = new AtomicReference<>();
        if (!StringUtils.isBlank(jasyptPassword)) {
            jasyptPasswordRef.set(new String(Base64.decodeBase64(jasyptPassword)));
        }
        if (!StringUtils.isBlank(jasyptPasswordRef.get())) {
            environment.getPropertySources().addFirst(
                new MapPropertySource(
                    String.format("%s-%d", DEFENSE_NAME, COUNT.incrementAndGet()),
                    new HashMap<String, Object>(JASYPT_MAP) {{
                        put(JASYPT_PASSWORD_KEY, jasyptPasswordRef.get());
                    }}
                )
            );
        }
    }
}
```

注册方式：

```properties
# META-INF/spring.factories
org.springframework.boot.env.EnvironmentPostProcessor=\
com.yourpackage.DefensePostProcessor
```

### 为什么这个方案能成功？

看到这里，你可能会有一个疑问：

> `ConfigFileApplicationListener`（Order = HIGHEST + 10）比 `BootstrapApplicationListener`（Order = HIGHEST + 5）后执行，所以 `DefensePostProcessor` 是在 `BootstrapApplicationListener` 之后才跑的。那 Jasypt 在 Bootstrap 上下文里执行 BFPP 时，`jasypt.encryptor.password` 还没被加进去，怎么解密？

这个问题问得非常好，答案是**前面讲的"嵌套启动流程"**：

1. **`DefensePostProcessor` 不是在主应用的启动流程里执行的，而是在 Bootstrap 应用的启动流程里执行的**。Bootstrap 应用是一个完整的 `SpringApplication`，它同样会走 `prepareEnvironment` → 发布事件 → `ConfigFileApplicationListener` 响应 → 执行所有 `EnvironmentPostProcessor` 这条链路。

2. **顺序完全正确**：在 Bootstrap 应用内部，
    - 先执行 `DefensePostProcessor`，从环境变量 `YNLZX` 读到 Base64 编码的密钥，解码后注入 `jasypt.encryptor.password`
    - 然后 Bootstrap 应用 `refresh()`，Jasypt 的 BFPP 执行，包装 PropertySource
    - 之后 Nacos 读 `bootstrap.yml` 里的 `ENC(...)` 密码时，解密条件已全部具备

3. **`BootstrapApplicationListener` 防止了递归**：它在 Bootstrap 应用的启动过程中再次收到 `ApplicationEnvironmentPreparedEvent` 时，会检测到当前已处于 Bootstrap 过程中，直接跳过，不会无限创建新的 Bootstrap 应用。

### 两种方案的对比

| 方案 | `jasypt.encryptor.password` 的来源 | 生效时机 | 密钥是否落地 |
|---|---|---|---|
| 写在 `bootstrap.yml` 里 | 配置文件直接读取 | Bootstrap 应用加载 `bootstrap.yml` 时 | ❌ 密钥明文写在配置里 |
| 自定义 `EnvironmentPostProcessor` | 从环境变量读取，Base64 解码后动态注入 | Bootstrap 应用的 `EnvironmentPostProcessor` 阶段 | ✅ 密钥只在运行时存在 |

两种方式**最终效果一样**——都是让 `jasypt.encryptor.password` 在 Bootstrap 应用 `refresh()` 之前就存在于 Environment 中，从而让 Jasypt 的 BFPP 能正常工作。但后者的安全性明显更高：**密钥不进代码库、不进配置文件，只通过环境变量在运行时注入。**

> ⚠️ **注意**：`DefensePostProcessor` 必须通过 `META-INF/spring.factories` 注册，否则不会被 `ConfigFileApplicationListener` 加载执行。

---

## 七、这个机制不是永久的——版本陷阱

上面这套机制能成立，有一个前提：**你的项目还在使用 Spring Cloud 的 bootstrap 机制**。

### 旧版本（Spring Cloud Alibaba ≤ 2023.x）

- `bootstrap.yml` 默认启用
- `spring.factories` 里的 `BootstrapConfiguration` 会被加载
- Jasypt 能在引导上下文抢跑
- **Nacos 连接配置写在 `bootstrap.yml`**

### 新版本（Spring Cloud Alibaba 2025.x+）

- 官方**废弃了 bootstrap 支持**，强制走 Spring Boot 2.4+ 的 `ConfigData` 机制
- `spring.factories` 里的 `BootstrapConfiguration` 入口不再被触发
- Jasypt 的"抢跑"失效
- Nacos 拿到的永远是字面量 `ENC(...)`，连接直接 403
- **Nacos 连接配置迁移到 `application.yml`，用 `spring.config.import` 导入**

```yaml
# 新版本写法
spring:
  application:
    name: cpc-console-server
  config:
    import:
      - "nacos:cpc-console-server.yaml?group=DEFAULT_GROUP&refreshEnabled=true"
  cloud:
    nacos:
      server-addr: ${NACOS_URL:10.64.3.151:30848}
      username: ${NACOS_USERNAME:nacos}
      password: ${NACOS_PASSWORD}   # ← 这里不能再写 ENC() 了
```

### 那新版本里 Nacos 密码怎么加密？

既然 Jasypt 的解密赶不上了，最务实的方案是：**用环境变量或 JVM 参数直接传明文**。

```yaml
spring:
  cloud:
    nacos:
      config:
        password: ${NACOS_PASSWORD}
```

启动时：

```bash
java -jar app.jar -DNACOS_PASSWORD=MySecret@123
# 或在 K8s 里通过 Secret 挂载环境变量
```

**Nacos 连接密码用环境变量，Nacos 内部的业务配置继续用 Jasypt 加密**——这是新版下最稳妥的组合。

---

## 八、一个容易踩的坑：`${VAR:ENC(...)}` 的假象

最后说一个实战中容易迷惑人的写法：

```yaml
password: ${NACOS_PASSWORD:ENC(Q9PNYb+QmZOd9cdlX8b3zXK7S3eGKsQlY5z2sCewoRo=)}
```

有人可能会发现：**明明没配环境变量，怎么也能连上？**

要小心，这里有两种可能：

**可能一：环境变量确实存在。** `${VAR:默认值}` 的语义是——优先读环境变量 `NACOS_PASSWORD`，只有它不存在时才用冒号后面的默认值。如果你的部署环境里已经注入了这个变量（K8s Secret、启动脚本、CI/CD 变量等），Nacos 拿到的就是明文，`ENC()` 那串密文根本没被用到。

**可能二：环境变量不存在，但老版本 + Jasypt 抢跑。** 在旧版本下，Jasypt 会把 `ENC(...)` 解密，返回明文。此时即使环境变量没配，也能连上。

**如何辨别？** 去掉环境变量、同时升级到新版试试——如果连接立刻失败，说明之前一直靠的是"环境变量覆盖"这个障眼法，`ENC()` 只是个摆设。

这个写法本身没有错，但它会**掩盖问题**。如果你以为"ENC() 生效了"，实际靠的却是环境变量，一旦环境变量被移除或版本升级，故障会突然爆发。

---

## 九、总结

把整篇文章的核心串起来：

1. **Nacos 配置管理**：引导阶段拉配置，运行时长轮询刷新。
2. **Jasypt**：通过包装 `PropertySource` 实现透明解密，任何走 `Environment.getProperty()` 的读取都会被自动解密。
3. **两者结合**：Nacos 存密文、Jasypt 在应用侧解密，业务代码无感知。
4. **Nacos 密码加密的难题**：Jasypt 通常在主上下文才生效，赶不上 Nacos 的连接时机。
5. **旧版本的解法一**：Jasypt 通过 `spring.factories` 注册 `BootstrapConfiguration`，在引导上下文里也跑一次 BFPP，从而"抢跑"成功。
6. **旧版本的解法二**：自定义 `EnvironmentPostProcessor`，从环境变量动态注入 `jasypt.encryptor.password`，密钥不落地，安全性更高。它能生效的关键在于 **Bootstrap 应用会走一遍完整的启动流程，`EnvironmentPostProcessor` 在这个嵌套流程里被执行，早于 Jasypt 的 BFPP**。
7. **监听器顺序**：`BootstrapApplicationListener`（Order = HIGHEST + 5）先于 `ConfigFileApplicationListener`（Order = HIGHEST + 10）执行，这是保证 Bootstrap 上下文先建立的关键。
8. **新版本的现实**：bootstrap 机制被废弃，抢跑失效，Nacos 连接密码请改用环境变量或 JVM 参数传入。
9. **一个隐蔽的坑**：`${VAR:ENC(...)}` 可能让你误以为加密生效了，实际靠的是环境变量覆盖。

**一句话记住它**：

> Nacos 负责"送"，Jasypt 负责"解"。旧版本里 Jasypt 通过 `BootstrapApplicationListener` 触发的**嵌套启动流程**提前蹲守在引导上下文里，等着 Nacos 来读；密钥要么写在 `bootstrap.yml`，要么通过自定义 `EnvironmentPostProcessor` 从环境变量动态注入。新版本里蹲守点没了，Nacos 的密码就老老实实用环境变量吧。

希望这篇能帮你把这条链路彻底理清。如果项目还在旧版本、跑得好好的，那就先别急着升级——搞清楚自己依赖的是哪套机制，比盲目升级重要得多。