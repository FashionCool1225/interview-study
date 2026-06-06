# Spring / Spring Boot / Spring Cloud 面试知识卡片

> 适用对象：3年经验 Java 开发者 | 目标公司：中型互联网公司
> 格式：问答卡片，每个答案适合面试口头回答 2-3 分钟

---

## 一、Spring 核心

### 1.1 什么是 IOC 和 DI？它们的原理是什么？

**IOC（控制反转）** 是一种设计思想，核心是将对象的创建和管理权从代码中转移到 Spring 容器。传统开发中，对象自己 new 依赖对象；IOC 模式下，容器负责创建对象并注入依赖。

**DI（依赖注入）** 是 IOC 的具体实现方式，包括构造器注入、Setter 注入和字段注入（@Autowired）。

**原理简述：**
- 容器启动时，通过 `BeanDefinitionReader` 读取配置（注解/XML），将 Bean 的定义信息封装为 `BeanDefinition` 对象，注册到 `BeanDefinitionRegistry` 中
- `BeanFactory` 在需要时通过反射调用构造器或工厂方法创建 Bean 实例
- 创建完成后，容器自动将依赖对象注入到 Bean 中（通过字段赋值、Setter 方法或构造器参数）

**面试加分点：** IOC 的本质是"好莱坞原则"——Don't call me, I'll call you。对象不再主动查找依赖，而是被动接收容器注入的依赖。

---

### 1.2 Spring Bean 的生命周期是怎样的？

Bean 的完整生命周期可以分为以下阶段：

1. **实例化（Instantiation）**：通过反射调用构造器创建 Bean 的原始实例（此时属性都为默认值）
2. **属性注入（Populate Properties）**：DI 注入，填充 Bean 的属性值和依赖
3. **Aware 接口回调**：如果 Bean 实现了 `BeanNameAware`、`BeanFactoryAware`、`ApplicationContextAware` 等接口，Spring 会依次调用对应方法，传入容器引用
4. **BeanPostProcessor 前置处理**：调用所有 `BeanPostProcessor.postProcessBeforeInitialization()` 方法，可以对 Bean 做自定义修改（`@PostConstruct` 就是在这一步由 `CommonAnnotationBeanPostProcessor` 处理的）
5. **初始化（Initialization）**：调用 `InitializingBean.afterPropertiesSet()`，然后调用 `@Bean(initMethod=...)` 指定的方法
6. **BeanPostProcessor 后置处理**：调用 `BeanPostProcessor.postProcessAfterInitialization()`，AOP 代理就在这一步生成
7. **使用阶段**：Bean 就绪，可以被应用使用
8. **销毁（Destruction）**：容器关闭时，调用 `@PreDestroy` 标注的方法、`DisposableBean.destroy()` 以及 `@Bean(destroyMethod=...)` 指定的方法

**面试技巧：** 可以简化记忆为"实例化 -> 属性注入 -> Aware回调 -> 前置处理 -> 初始化 -> 后置处理 -> 使用 -> 销毁"。重点关注 BeanPostProcessor，因为 AOP 代理是在后置处理阶段创建的。

---

### 1.3 Spring 的三级缓存是如何解决循环依赖的？

**循环依赖**：A 依赖 B，B 又依赖 A。如果不用缓存，创建 A 时需要 B，创建 B 时又需要 A，会陷入死循环。

Spring 使用三级缓存解决单例 Bean 的 Setter/字段注入循环依赖：

| 缓存 | 名称 | 存储内容 |
|------|------|---------|
| 一级缓存 | `singletonObjects` | 完全初始化好的 Bean（成品） |
| 二级缓存 | `earlySingletonObjects` | 早期暴露的 Bean（已实例化，未填充属性） |
| 三级缓存 | `singletonFactories` | Bean 的 ObjectFactory（工厂对象，用于生成早期引用） |

**解决流程（A 依赖 B，B 依赖 A）：**

1. 创建 A：实例化 A -> 将 A 的 ObjectFactory 放入三级缓存 -> 开始填充属性，发现依赖 B
2. 创建 B：实例化 B -> 将 B 的 ObjectFactory 放入三级缓存 -> 开始填充属性，发现依赖 A
3. 获取 A 的早期引用：从三级缓存获取 A 的 ObjectFactory，调用 `getObject()` 生成 A 的早期引用（如果 A 需要被 AOP 代理，这一步会返回代理对象），放入二级缓存，删除三级缓存
4. B 拿到 A 的早期引用，完成属性填充，B 初始化完成，放入一级缓存
5. A 拿到完整的 B，完成属性填充，A 初始化完成，放入一级缓存

**为什么需要三级而不是两级？** 三级缓存的关键作用是延迟代理对象的创建。如果没有 AOP，二级缓存就够了。但有了 AOP，需要在三级缓存中存 ObjectFactory，在真正需要时才决定是否创建代理对象，避免过早创建代理导致的问题。

**注意：** 构造器注入的循环依赖无法解决（因为实例化阶段就需要依赖），可以用 `@Lazy` 延迟加载来规避。`prototype` 作用域的循环依赖也无法解决。

---

### 1.4 Spring AOP 的实现原理是什么？JDK 动态代理和 CGLIB 有什么区别？

**AOP 本质：** 在运行时动态生成代理对象，在不修改源代码的情况下，对方法进行增强（前置、后置、环绕、异常通知等）。

**JDK 动态代理：**
- 基于接口，要求被代理类必须实现至少一个接口
- 通过 `java.lang.reflect.Proxy` 类和 `InvocationHandler` 接口实现
- 运行时动态生成一个实现了目标接口的代理类，调用方法时通过 `InvocationHandler.invoke()` 转发
- 生成速度较快，但运行时反射调用性能略低

**CGLIB 动态代理：**
- 基于继承，通过 ASM 字节码框架在运行时生成目标类的子类
- 通过 `MethodInterceptor.intercept()` 拦截方法调用
- 不能代理 `final` 类和 `final` 方法（因为无法继承/覆写）
- 生成速度较慢（需要操作字节码），但运行时性能更好

**Spring 的选择策略：**
- 目标对象实现了接口 -> 默认使用 JDK 动态代理
- 目标对象没有实现接口 -> 使用 CGLIB
- 可以通过 `@EnableAspectJAutoProxy(proxyTargetClass=true)` 强制使用 CGLIB
- **Spring Boot 2.x 起默认使用 CGLIB 代理**（即使目标实现了接口）

**面试加分点：** Spring AOP 的增强逻辑是在 `BeanPostProcessor.postProcessAfterInitialization()` 阶段完成的，由 `AnnotationAwareAspectJAutoProxyCreator` 负责判断是否需要代理以及创建代理对象。

---

### 1.5 @Transactional 在哪些场景下会失效？

这是面试高频题，常见失效场景如下：

1. **方法不是 public 的**：Spring AOP 默认只代理 public 方法。`protected`、`private`、`default` 的方法上加 `@Transactional` 无效
2. **同一个类中方法内部调用**：A 方法（无事务）调用 B 方法（有事务），因为调用的是 `this.B()`，走的是目标对象本身而不是代理对象，事务不会生效。解决方法：注入自身、使用 `AopContext.currentProxy()` 获取代理对象、或将方法拆到不同类
3. **异常被 catch 吞掉了**：事务方法内部捕获了异常且没有重新抛出，Spring 感知不到异常，不会回滚
4. **异常类型不匹配**：`@Transactional` 默认只对 `RuntimeException` 和 `Error` 回滚，抛出 `checked exception`（如 `IOException`）不会回滚，需要显式指定 `rollbackFor = Exception.class`
5. **数据库引擎不支持事务**：如 MySQL 的 MyISAM 引擎，需要改为 InnoDB
6. **没有配置事务管理器**：Spring 容器中未注册 `PlatformTransactionManager`
7. **传播行为配置不当**：如使用了 `NOT_SUPPORTED`（非事务方式运行）
8. **多数据源时未指定事务管理器**：需要明确指定使用哪个事务管理器

**最佳实践：** 统一使用 `@Transactional(rollbackFor = Exception.class)`，确保所有异常都能回滚。

---

### 1.6 Spring 事务传播机制有哪些？

事务传播行为定义了当一个事务方法被另一个事务方法调用时，事务应该如何传播。共 7 种：

**常用（前3种）：**

| 传播行为 | 说明 |
|---------|------|
| `REQUIRED`（默认） | 如果当前有事务，加入该事务；如果没有，创建一个新事务。绝大多数场景使用这个 |
| `REQUIRES_NEW` | 无论当前有没有事务，都创建一个新事务。当前事务会被挂起。典型场景：记录日志（不管主事务是否回滚，日志都要保存） |
| `NESTED` | 如果当前有事务，在嵌套事务（Savepoint）中执行；如果没有，等同于 REQUIRED。嵌套事务回滚不影响外层事务，但外层事务回滚会带动嵌套事务 |

**不常用（后4种）：**

| 传播行为 | 说明 |
|---------|------|
| `SUPPORTS` | 有事务就加入，没有就以非事务方式运行 |
| `NOT_SUPPORTED` | 无论有没有事务，都以非事务方式运行，会挂起当前事务 |
| `MANDATORY` | 必须在已有事务中运行，否则抛出异常 |
| `NEVER` | 以非事务方式运行，如果当前有事务则抛出异常 |

**面试常考场景：** `REQUIRED` vs `REQUIRES_NEW` vs `NESTED` 的区别。举例：外层方法事务中调用内层方法，内层抛异常：
- `REQUIRED`：内外共用一个事务，内层异常导致整个事务回滚
- `REQUIRES_NEW`：内外是两个独立事务，内层回滚不影响外层
- `NESTED`：内层在 Savepoint 上运行，内层回滚到 Savepoint，外层可以选择继续或也回滚

---

### 1.7 @Autowired 和 @Resource 有什么区别？

| 对比项 | @Autowired | @Resource |
|--------|-----------|-----------|
| 来源 | Spring 提供的注解 | JDK 提供的注解（JSR-250，`javax.annotation`） |
| 注入方式 | 默认按类型（byType）注入 | 默认按名称（byName）注入 |
| 配合使用 | 可配合 `@Qualifier("name")` 指定名称 | 可通过 `name` 属性直接指定 Bean 名称 |
| 可用位置 | 字段、构造器、Setter 方法 | 字段、Setter 方法（不支持构造器） |
| 找不到 Bean | 默认抛异常（可设 `required=false`） | 抛出 `NoSuchBeanDefinitionException` |

**@Autowired 按类型注入的流程：**
1. 先按类型查找 Bean
2. 如果找到多个同类型 Bean，按参数名/字段名匹配
3. 如果还匹配不上，使用 `@Qualifier` 指定的名称
4. 如果还是不行，抛出异常

**面试建议：** 推荐优先使用构造器注入 + `@Autowired`，因为可以保证 Bean 的不可变性和依赖的必要性。Spring 官方也推荐构造器注入。

---

## 二、Spring Boot

### 2.1 Spring Boot 的自动配置原理是什么？

Spring Boot 自动配置的核心是 **`@EnableAutoConfiguration`** 注解（包含在 `@SpringBootApplication` 中）。

**工作流程：**

1. `@EnableAutoConfiguration` 通过 `@Import(AutoConfigurationImportSelector.class)` 引入自动配置选择器
2. `AutoConfigurationImportSelector` 会扫描所有 jar 包下 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件（Spring Boot 2.7+），读取其中列出的自动配置类全限定名
3. 这些配置类并不是全部生效，每个配置类上都有 `@Conditional` 系列注解做条件判断（如 `@ConditionalOnClass`、`@ConditionalOnMissingBean`、`@ConditionalOnProperty` 等）
4. 只有满足条件的配置类才会生效，向容器中注册对应的 Bean

**举例：** 当 classpath 下存在 `DispatcherServlet.class` 时，`DispatcherServletAutoConfiguration` 就会生效，自动注册 `DispatcherServlet`。

**一句话总结：** 自动配置 = SPI 机制加载配置类 + `@Conditional` 条件过滤，实现"约定优于配置"。

---

### 2.2 Spring Boot Starter 的工作机制是什么？

**Starter 是什么：** 一个 Starter 就是一组预定义的依赖集合 + 自动配置类，用来简化某个技术的集成。比如 `spring-boot-starter-web` 包含了 Tomcat、Spring MVC、Jackson 等依赖和对应的自动配置。

**工作机制：**

1. 引入 Starter 的 Maven/Gradle 依赖后，相关的 jar 包都被引入 classpath
2. Spring Boot 启动时，`@EnableAutoConfiguration` 会扫描所有 jar 包中的自动配置文件
3. 每个 Starter 的 jar 包中 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件声明了该 Starter 提供的自动配置类
4. Spring Boot 根据条件注判断哪些配置类生效，自动注册对应的 Bean
5. 开发者可以通过 `application.yml` 中的属性覆盖默认配置（`@ConfigurationProperties` 绑定）

**自定义 Starter 的步骤：**
1. 创建自动配置类，使用 `@Configuration` + `@Conditional` 注解
2. 创建 `AutoConfiguration.imports` 文件，写入自动配置类的全限定名
3. 打包发布为一个独立的 jar

---

### 2.3 Spring Boot 的启动流程是怎样的？

Spring Boot 启动的入口是 `SpringApplication.run()` 方法，核心流程如下：

1. **创建 SpringApplication 对象**：
   - 推断应用类型（Servlet 应用、响应式应用还是普通应用），根据 classpath 中的类判断
   - 加载所有 `ApplicationContextInitializer`（用于在容器刷新前做自定义初始化）
   - 加载所有 `ApplicationListener`（事件监听器）
   - 推断主类（包含 `main` 方法的类）

2. **执行 run() 方法**：
   - 创建并启动 `StopWatch` 计时器
   - 获取 `SpringApplicationRunListeners`（运行监听器，用于在启动各阶段发布事件）
   - 准备 `Environment` 环境：加载配置文件（application.yml 等）、解析命令行参数
   - 发布 `ApplicationEnvironmentPreparedEvent` 事件
   - **创建 ApplicationContext**：根据应用类型创建对应的容器（`AnnotationConfigServletWebServerApplicationContext` 等）
   - **准备上下文**：设置环境、执行 Initializer、加载主配置类
   - **刷新上下文（refresh）**：这是核心，调用 `AbstractApplicationContext.refresh()`，完成 Bean 的扫描、注册、实例化、依赖注入等（和传统 Spring 一致）
   - 启动内嵌 Web 服务器（Tomcat/Netty 等）
   - 发布 `ApplicationStartedEvent` 和 `ApplicationReadyEvent`
   - 执行 `ApplicationRunner` 和 `CommandLineRunner`

**面试加分点：** 整个启动过程大量使用了事件机制（Observer 模式），通过 `SpringApplicationRunListener` 在不同阶段发布事件，扩展性非常强。

---

### 2.4 Spring Boot 中有哪些条件注解（@Conditional 系列）？

条件注解用于控制配置类或 Bean 是否生效，是自动配置的核心机制：

| 注解 | 生效条件 |
|------|---------|
| `@ConditionalOnClass` | classpath 下存在指定的类 |
| `@ConditionalOnMissingClass` | classpath 下不存在指定的类 |
| `@ConditionalOnBean` | 容器中存在指定的 Bean |
| `@ConditionalOnMissingBean` | 容器中不存在指定的 Bean（常用于自动配置，提供默认实现） |
| `@ConditionalOnProperty` | 配置文件中指定的属性满足条件（`havingValue`、`matchIfMissing`） |
| `@ConditionalOnExpression` | SpEL 表达式为 true |
| `@ConditionalOnWebApplication` | 当前是 Web 应用 |
| `@ConditionalOnNotWebApplication` | 当前不是 Web 应用 |
| `@ConditionalOnJava` | JDK 版本满足条件 |

**底层原理：** 所有条件注解都通过 `@Conditional` 元注解标注一个 `Condition` 实现类。Spring 在解析配置类时，会调用 `Condition.matches()` 方法判断条件是否满足。不满足的配置类整个跳过，不会注册其中的任何 Bean。

**面试加分点：** `@ConditionalOnMissingBean` 是自动配置的关键——框架提供默认 Bean，但开发者只要自定义了同类型 Bean，自动配置的 Bean 就不会生效，实现了"约定优于配置"。

---

### 2.5 Spring Boot 配置文件的加载优先级是怎样的？

Spring Boot 支持多种配置来源，优先级从高到低如下（高优先级会覆盖低优先级）：

1. **命令行参数**（`--server.port=8081`）
2. **ServletConfig 初始化参数**
3. **ServletContext 初始化参数**
4. **JNDI 属性**
5. **Java 系统属性**（`System.getProperties()`，即 `-D` 参数）
6. **操作系统环境变量**
7. **jar 包外的 `application-{profile}.yml/properties`**（指定 profile 的配置文件）
8. **jar 包内的 `application-{profile}.yml/properties`**
9. **jar 包外的 `application.yml/properties`**
10. **jar 包内的 `application.yml/properties`**
11. **`@PropertySource` 注解引入的配置**
12. **`SpringApplication.setDefaultProperties` 设置的默认属性**

**简化记忆：** 命令行 > 系统属性 > 环境变量 > 外部配置 > 内部配置 > 默认配置。外部配置优先于内部配置，profile 配置优先于默认配置。

**实际开发中的用法：** 开发环境用 `application-dev.yml`，生产环境用 `application-prod.yml`，通过 `spring.profiles.active=prod` 切换。也可以用 Nacos 等配置中心统一管理。

---

### 2.6 Spring Boot 常用注解有哪些？

**核心注解：**

| 注解 | 作用 |
|------|------|
| `@SpringBootApplication` | 组合注解，包含 `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan` |
| `@Configuration` | 标记当前类为配置类，等同于 XML 中的 `<beans>` |
| `@Bean` | 在配置类中声明一个 Bean，方法返回值作为 Bean 注册到容器 |
| `@Component` / `@Service` / `@Repository` / `@Controller` | 组件扫描注解，标记类由 Spring 管理 |
| `@Autowired` | 自动注入依赖 |
| `@Value` | 注入配置属性值或 SpEL 表达式 |
| `@ConfigurationProperties` | 将配置文件中的属性批量绑定到 Bean 的属性上（类型安全） |

**Web 相关：**

| 注解 | 作用 |
|------|------|
| `@RestController` | `@Controller` + `@ResponseBody`，返回 JSON |
| `@RequestMapping` / `@GetMapping` / `@PostMapping` | 映射 HTTP 请求 |
| `@PathVariable` | 获取路径参数（`/users/{id}`） |
| `@RequestParam` | 获取查询参数（`?name=xxx`） |
| `@RequestBody` | 将请求体 JSON 反序列化为 Java 对象 |
| `@ExceptionHandler` | 处理控制器中抛出的异常 |
| `@ControllerAdvice` / `@RestControllerAdvice` | 全局异常处理 |

**其他常用：**

| 注解 | 作用 |
|------|------|
| `@Async` | 标记方法异步执行（需 `@EnableAsync`） |
| `@Scheduled` | 定时任务（需 `@EnableScheduling`） |
| `@Transactional` | 声明式事务 |
| `@Slf4j`（Lombok） | 自动注入日志对象 |

---

## 三、Spring Cloud

### 3.1 微服务架构的核心组件有哪些？

一个完整的微服务架构通常包含以下核心组件：

| 组件 | 作用 | 常用技术 |
|------|------|---------|
| **服务注册与发现** | 服务实例自动注册和发现，实现服务间寻址 | Nacos、Eureka、Consul |
| **负载均衡** | 在多个服务实例间分配请求 | Spring Cloud LoadBalancer（取代 Ribbon） |
| **服务调用** | 声明式的 HTTP 客户端，简化服务间调用 | OpenFeign |
| **API 网关** | 统一入口，负责路由、鉴权、限流、日志 | Spring Cloud Gateway |
| **熔断降级** | 防止雪崩效应，服务不可用时快速失败 | Sentinel、Hystrix（已停更） |
| **配置中心** | 集中管理配置，支持动态刷新 | Nacos Config、Apollo、Spring Cloud Config |
| **分布式链路追踪** | 追踪跨服务调用链路，定位性能瓶颈 | Sleuth + Zipkin、SkyWalking |
| **消息驱动** | 异步通信，解耦服务 | Spring Cloud Stream（对接 RabbitMQ/Kafka） |
| **分布式事务** | 保证跨服务数据一致性 | Seata |

**面试技巧：** 可以画一张架构图，从客户端 -> 网关 -> 微服务集群（注册中心 + 配置中心）-> 数据库/缓存/消息队列，串联所有组件。

---

### 3.2 Nacos/Eureka 服务注册与发现的原理是什么？

**Eureka 原理（AP 模型，已停更）：**
- 架构：Eureka Server 集群（Peer-to-Peer，互相同步注册信息）
- 服务注册：服务启动时向 Eureka Server 发送注册请求，注册信息包括 IP、端口、健康检查 URL 等
- 服务发现：客户端定时（默认 30s）从 Server 拉取注册表并本地缓存，调用时从本地缓存中获取地址
- 心跳续约：客户端每 30s 发送心跳，Server 90s 未收到心跳则剔除
- 自我保护机制：如果 15 分钟内超过 85% 的节点心跳不正常，Eureka 进入保护模式，不剔除任何服务（防止网络分区时误删健康服务）

**Nacos 原理（支持 AP 和 CP 模型）：**
- 架构：Nacos Server 集群（Raft 协议保证一致性）
- 服务注册：服务启动时通过 HTTP/gRPC 向 Nacos Server 注册实例信息
- 服务发现：支持两种模式
  - **临时实例（默认，AP 模式）**：客户端定时发送心跳，Nacos 定时剔除无心跳实例。客户端本地缓存注册表，通过 UDP 接收变更通知
  - **永久实例（CP 模式）**：不依赖心跳，实例状态由服务端健康检查决定
- 优势：Nacos 集成了配置中心功能，支持 AP/CP 切换，社区活跃，是目前国内主流选择

**核心区别：** Eureka 只有 AP 模型且已停更；Nacos 同时支持 AP 和 CP，功能更全面（注册中心 + 配置中心二合一）。

---

### 3.3 Feign 的底层原理是什么？超时如何处理？

**Feign 原理：**
Feign 是一个声明式的 HTTP 客户端，通过接口 + 注解的方式定义 HTTP 请求，底层通过动态代理实现。

**工作流程：**
1. 使用 `@EnableFeignClients` 开启 Feign 支持
2. Spring 扫描 `@FeignClient` 标注的接口，为每个接口创建 JDK 动态代理对象
3. 调用接口方法时，`InvocationHandler` 拦截调用，根据方法上的注解（`@GetMapping`、`@PostMapping`、`@PathVariable` 等）构建 HTTP 请求的 URL、Header、Body
4. 通过集成的负载均衡器（Spring Cloud LoadBalancer）从注册中心获取服务实例地址
5. 底层默认使用 JDK `HttpURLConnection` 发送请求（可以替换为 Apache HttpClient 或 OkHttp 以获得连接池支持）
6. 响应结果通过 `Decoder` 反序列化为 Java 对象

**超时处理：**

```yaml
# 全局超时配置
feign:
  client:
    config:
      default:
        connectTimeout: 5000   # 建立连接超时（ms）
        readTimeout: 10000     # 读取响应超时（ms）
      # 针对特定服务的超时配置
      order-service:
        connectTimeout: 3000
        readTimeout: 5000
```

**最佳实践：**
- 替换底层 HTTP 客户端为 OkHttp 或 HttpClient，启用连接池提升性能
- 配合 Sentinel/Hystrix 做超时降级，Feign 的 fallback 或 fallbackFactory 实现降级逻辑
- 设置合理的超时时间，避免因下游服务慢导致线程池耗尽

---

### 3.4 Spring Cloud Gateway 的作用和过滤器链是什么？

**Gateway 的核心作用：**
- **统一入口**：所有外部请求通过网关进入，屏蔽内部服务细节
- **路由转发**：根据请求特征（路径、Header、参数等）将请求路由到不同的微服务
- **负载均衡**：集成 Spring Cloud LoadBalancer 实现客户端负载均衡
- **鉴权认证**：统一校验 Token，拦截非法请求
- **限流熔断**：集成 Sentinel/Redis 限流，保护后端服务
- **跨域处理**：统一配置 CORS
- **日志监控**：记录请求日志，集成链路追踪

**核心概念：**
- **Route（路由）**：由 ID、目标 URI、断言（Predicate）和过滤器（Filter）组成
- **Predicate（断言/谓词）**：匹配条件，决定请求是否走某条路由（如路径匹配、Header 匹配、时间匹配等）
- **Filter（过滤器）**：对请求/响应进行处理（如添加 Header、限流、重写路径等）

**过滤器链执行流程：**

```
请求 -> GatewayFilter（Pre 阶段）-> 路由到目标服务 -> GatewayFilter（Post 阶段）-> 响应
```

过滤器分为两类：
- **GatewayFilter**：作用于单个路由
- **GlobalFilter**：作用于所有路由（常用于全局鉴权、日志记录）

执行顺序由 `getOrder()` 方法决定，值越小优先级越高。Pre 阶段按 order 从小到大执行，Post 阶段按 order 从大到小执行。

**面试加分点：** Gateway 基于 WebFlux（Netty + Reactor），是异步非阻塞的，性能优于 Zuul 1.x 的同步阻塞模型。

---

### 3.5 Hystrix 和 Sentinel 的原理和区别是什么？

**熔断器原理（通用）：**
熔断器有三种状态，类似电路中的保险丝：
- **关闭（Closed）**：正常状态，请求正常通过。同时统计失败率
- **打开（Open）**：失败率超过阈值，熔断器打开，所有请求直接走 fallback（快速失败），不再调用下游服务
- **半开（Half-Open）**：经过一段休眠时间后，放行少量探测请求。如果成功则关闭熔断器恢复正常；如果失败则继续保持打开状态

**Hystrix vs Sentinel 对比：**

| 对比项 | Hystrix | Sentinel |
|--------|---------|----------|
| 隔离策略 | 线程池隔离（默认）/ 信号量隔离 | 信号量隔离（默认）/ 线程池隔离 |
| 熔断策略 | 基于异常比率 | 基于异常比率、异常数、慢调用比例 |
| 限流 | 简单限流（基于线程池大小） | 多种限流策略（QPS、线程数、调用关系、热点参数） |
| 实时指标 | 滑动窗口（RxJava） | 滑动窗口 + LeapArray |
| 动态规则 | 支持多种数据源 | 支持多种数据源（Nacos、Apollo、Zookeeper 等） |
| 控制台 | 简单监控 | 功能丰富的控制台（实时监控、规则管理、集群流控） |
| 维护状态 | **已停止维护**（2020年进入维护模式） | **活跃维护**（阿里巴巴开源） |

**Sentinel 的限流算法：**
- 固定窗口计数器
- 滑动时间窗口（LeapArray 实现）
- 令牌桶算法
- 漏桶算法

**面试建议：** 现在新项目推荐使用 Sentinel，功能更强大，社区活跃。如果老项目用 Hystrix，迁移到 Sentinel 的成本也不高。

---

### 3.6 分布式配置中心的作用是什么？

**为什么需要分布式配置中心：**
- 微服务数量多，每个服务都有配置文件，分散管理维护成本高
- 不同环境（dev/test/prod）的配置不同，需要统一管理
- 业务配置需要动态修改并实时生效（如开关、限流阈值），不能每次都重启服务
- 配置需要版本管理、权限控制、灰度发布

**主流配置中心对比：**

| 对比项 | Nacos Config | Apollo | Spring Cloud Config |
|--------|-------------|--------|-------------------|
| 配置推送 | 长轮询 | 长轮询 | 需要配合 Bus（MQ）推送 |
| 实时生效 | 支持 | 支持 | 需要手动刷新或配合 Bus |
| 版本管理 | 支持 | 支持 | 依赖 Git |
| 灰度发布 | 支持 | 支持 | 不支持 |
| 高可用 | 集群 | 多机房部署 | 依赖 Git + MQ |
| 控制台 | 有 | 功能丰富 | 无 |

**Spring Boot 中使用 Nacos Config：**
1. 引入 `spring-cloud-starter-alibaba-nacos-config` 依赖
2. 在 `bootstrap.yml` 中配置 Nacos 地址和 DataId
3. 使用 `@RefreshScope` 注解标注需要动态刷新配置的 Bean
4. Nacos 控制台修改配置后，通过长轮询机制通知客户端，配置自动刷新

---

### 3.7 分布式链路追踪（Sleuth/Zipkin）的原理是什么？

**解决的问题：** 在微服务架构中，一个请求可能经过多个服务，出问题时很难定位是哪个环节出了问题。链路追踪用于可视化请求在各个服务间的调用过程。

**核心概念（遵循 Google Dapper 论文）：**
- **TraceId**：一次完整请求链路的唯一标识，贯穿整个调用链
- **SpanId**：一次服务调用的唯一标识，每个 Span 记录调用的开始时间、结束时间、状态等信息
- **ParentSpanId**：父 Span 的 ID，用于构建调用树

**Sleuth + Zipkin 的工作流程：**

1. **Sleuth（数据采集）**：作为客户端 SDK 集成到每个微服务中
   - 自动拦截请求，生成/传递 TraceId 和 SpanId
   - 将追踪信息放入请求头（`X-B3-TraceId`、`X-B3-SpanId` 等）传递给下游服务
   - 将 Span 数据异步上报给 Zipkin

2. **Zipkin（数据展示）**：独立部署的服务端
   - 接收各服务上报的 Span 数据
   - 存储到数据库（默认内存，生产可用 Elasticsearch）
   - 提供 Web UI，可视化展示调用链路、耗时分析、依赖拓扑

**面试加分点：** Spring Cloud Sleuth 已被 **Micrometer Tracing** 取代（Spring Boot 3.x / Spring Cloud 2022.x 起）。新项目推荐使用 Micrometer Tracing + Zipkin/Jaeger/SkyWalking。SkyWalking 是国产开源项目，功能更全面，支持多种探针（Java/Go/前端），在国内使用越来越广泛。

---

### 3.8 微服务之间的通信方式有哪些？同步和异步有什么区别？

**同步通信：**

| 方式 | 特点 |
|------|------|
| **HTTP REST**（RestTemplate / WebClient） | 最基础的方式，直接调用 HTTP 接口，简单但代码耦合度高 |
| **Feign / OpenFeign** | 声明式 HTTP 客户端，通过接口 + 注解定义调用，集成负载均衡和熔断 |
| **gRPC** | 基于 HTTP/2 + Protocol Buffers，高性能，强类型，适合内部服务通信 |
| **Dubbo** | 阿里开源的 RPC 框架，性能好，生态丰富 |

**异步通信：**

| 方式 | 特点 |
|------|------|
| **消息队列**（RabbitMQ / Kafka / RocketMQ） | 生产者发送消息到 MQ，消费者异步消费。服务间完全解耦 |
| **Spring Cloud Stream** | Spring 对消息队列的抽象层，统一 API，切换 MQ 实现不需要改代码 |
| **事件驱动**（Event-Driven） | 通过发布/订阅事件实现服务间通信 |

**同步 vs 异步对比：**

| 对比项 | 同步（HTTP/RPC） | 异步（MQ） |
|--------|-----------------|-----------|
| 实时性 | 实时获取结果 | 不保证实时，最终一致 |
| 耦合度 | 调用方需要知道被调用方地址 | 完全解耦，通过 MQ 中转 |
| 容错性 | 被调用方不可用时调用失败 | 消息持久化，消费者恢复后可重试 |
| 性能 | 受限于最慢的服务 | 削峰填谷，提升整体吞吐 |
| 适用场景 | 需要立即返回结果的场景（查询） | 不需要立即返回、可延迟处理的场景（通知、日志、数据统计） |

**实际项目中的选择：**
- 查询类接口（如获取商品信息）：使用 Feign 同步调用
- 写操作（如下单后发短信、加积分）：使用 MQ 异步处理
- 高并发场景（如秒杀）：MQ 削峰 + 异步处理
- 对性能要求极高的内部调用：考虑 gRPC

---

## 附：面试高频追问汇总

### Q：Spring 的 Bean 是线程安全的吗？

Spring 默认 Bean 是单例的（Singleton），如果 Bean 中有可变的成员变量，则不是线程安全的。解决方案：
1. 将 Bean 设为 `prototype`（每次获取创建新实例）
2. 使用 `ThreadLocal` 存储线程局部变量
3. 使用同步机制（`synchronized`、`ReentrantLock`）
4. 将 Bean 设计为无状态的（推荐）

### Q：Spring Boot 如何实现热部署？

使用 `spring-boot-devtools`：当类文件变更时自动重启应用（利用双类加载器机制，重启速度很快）。也可以使用 JRebel 实现真正的热替换（无需重启）。

### Q：什么是 Spring Cloud 中的服务雪崩？如何解决？

**服务雪崩**：A 调用 B，B 调用 C，C 响应变慢导致 B 的线程池耗尽，进而导致 B 不可用，最终导致 A 也不可用，故障级联扩散。

**解决方案：**
1. **超时控制**：为每个远程调用设置合理的超时时间
2. **熔断降级**：使用 Sentinel/Hystrix，失败率过高时快速失败，返回降级数据
3. **限流**：限制单位时间内的请求量，保护服务不被打垮
4. **线程池隔离**：每个服务调用使用独立的线程池，避免一个服务慢导致所有线程被占用
5. **异步解耦**：非核心链路使用 MQ 异步处理

---

> 最后更新：2026年6月
