# 🗺️ Spring Boot 知识地图 v2.0

> 本地图涵盖 Spring Boot 框架核心知识点，按模块分类，共 **~263个**。
>
> v2.0 优化：整合碎片化模块，突出面试重点，结构更清晰。

---

## 一、Spring Boot 概述与启动

| # | 知识点 |
|---|--------|
| 1 | Spring Boot 核心理念（约定大于配置） |
| 2 | Spring Boot vs Spring MVC vs SSM |
| 3 | Spring Boot 核心优势（快速构建、自动配置、内嵌容器） |
| 4 | Spring Boot 主启动类规范 |
| 5 | @SpringBootApplication 组合注解（@Configuration + @EnableAutoConfiguration + @ComponentScan） |
| 6 | @EnableAutoConfiguration vs @SpringBootApplication 区别 |
| 7 | SpringApplication.run() 启动流程 |
| 8 | SpringApplication 静态方法（run vs create） |
| 9 | SpringApplicationBuilder 链式构建 |
| 10 | Spring Boot Banner 自定义 |
| 11 | Spring Boot 退出机制 |
| 12 | SpringApplication 构造器（primarySources / sources） |
| 13 | 懒加载初始化（spring.main.lazy-initialization） |
| 14 | 启动失败分析（FailureAnalyzers） |
| 15 | Spring Boot 可执行 JAR 原理 |

---

## 二、自动配置原理 ⭐核心

| # | 知识点 |
|---|--------|
| 1 | @EnableAutoConfiguration 启用自动配置 |
| 2 | AutoConfigurationImportSelector 源码流程 |
| 3 | AutoConfigurationImportFilter 过滤机制 |
| 4 | spring.factories 文件（JDK 6-2.6） |
| 5 | AutoConfiguration_imports 文件（JDK 2.7+） |
| 6 | Spring Boot 2.7 自动配置变化 |
| 7 | @AutoConfigureBefore / @AutoConfigureAfter 排序 |
| 8 | @AutoConfigureOrder 精确排序 |
| 9 | @AutoConfigurationPackage 自动配置包扫描 |
| 10 | 自动配置条件注解概述 |
| 11 | @ConditionalOnClass — 存在类时生效 |
| 12 | @ConditionalOnBean — 存在 Bean 时生效 |
| 13 | @ConditionalOnMissingClass — 不存在类时生效 |
| 14 | @ConditionalOnMissingBean — 不存在 Bean 时生效 |
| 15 | @ConditionalOnProperty — 配置属性匹配 |
| 16 | @ConditionalOnResource — 存在资源文件 |
| 17 | @ConditionalOnWebApplication — Web 应用类型 |
| 18 | @ConditionalOnNotWebApplication — 非 Web 应用 |
| 19 | @ConditionalOnExpression — SpEL 表达式 |
| 20 | @ConditionalOnJndi — JNDI 存在 |
| 21 | @ConditionalOnSingleCandidate — 单个候选 Bean |
| 22 | 自动配置报告详解（debug 模式查看） |
| 23 | ConditionEvaluationReport 使用 |
| 24 | 自动配置排除详解 |
| 25 | spring.autoconfigure.exclude 排除配置 |
| 26 | spring-boot-configuration-processor 元数据生成 |
| 27 | 自定义 Starter 自动配置开发 |
| 28 | Starter 依赖传递与冲突处理 |
| 29 | 条件注解总结对比 |
| 30 | 自动配置与手动配置优先级 |
| 31 | 自定义条件注解开发 |

---

## 三、配置管理

| # | 知识点 |
|---|--------|
| 1 | application.properties 格式 |
| 2 | application.yml / application.yaml 格式 |
| 3 | YAML 多文档配置（--- 分隔） |
| 4 | properties vs YAML 对比 |
| 5 | @Value 注解读取配置 |
| 6 | @Value ${} vs #{} 区别 |
| 7 | @ConfigurationProperties 配置绑定 |
| 8 | @EnableConfigurationProperties 启用配置绑定 |
| 9 | @ConfigurationProperties vs @Value 区别 |
| 10 | @NestedConfigurationProperty 嵌套配置属性 |
| 11 | @PropertySource 加载外部配置文件 |
| 12 | @PropertySources 多文件加载 |
| 13 | @ImportResource 导入 XML 配置 |
| 14 | Spring Boot 配置优先级（15级） |
| 15 | Spring Boot 外部化配置（Externalized Configuration） |
| 16 | 命令行参数配置（--xxx=yyy） |
| 17 | 环境变量自动映射（LOGGING_PATTERN_CONSOLE → logging.pattern.console） |
| 18 | RandomValuePropertySource 随机值 |
| 19 | 多环境配置（dev/test/prod） |
| 20 | @Profile 激活环境 |
| 21 | spring.profiles.active 激活 |
| 22 | application-{profile}.properties/yml |
| 23 | spring.config.activate.on-profile（2.4+） |
| 24 | spring.config.import 导入配置 |
| 25 | 配置占位符（${app.name:默认值}） |
| 26 | Relaxed 属性名绑定（驼峰/横线/下划线） |
| 27 | 配置元数据 spring-configuration-metadata.json |
| 28 | 配置加密（Jasypt） |
| 29 | Spring Boot 2.4 配置变更总结 |

---

## 四、Web 开发

| # | 知识点 |
|---|--------|
| 1 | Spring MVC 自动配置原理 |
| 2 | WebMvcAutoConfiguration 生效条件 |
| 3 | WebMvcConfigurer 接口（2种实现方式） |
| 4 | addViewControllers 视图控制器 |
| 5 | addResourceHandlers 静态资源处理 |
| 6 | 静态资源目录优先级（/static /public /resources /META-INF/resources） |
| 7 | 欢迎页配置 |
| 8 | Favicon 图标配置 |
| 9 | configureContentNegotiation 内容协商 |
| 10 | addFormatters 格式化器 |
| 11 | configureMessageConverters 消息转换器 |
| 12 | HttpMessageConverter 原理 |
| 13 | Jackson 日期格式化（spring.jackson.date-format） |
| 14 | addInterceptors 拦截器注册 |
| 15 | 拦截器执行流程（preHandle → postHandle → afterCompletion） |
| 16 | Filter 过滤器（@WebFilter + @Order） |
| 17 | Filter 执行顺序（FilterRegistrationBean） |
| 18 | 拦截器 vs 过滤器 区别 |
| 19 | configureHandlerExceptionResolvers 异常处理 |
| 20 | @ControllerAdvice 全局异常处理 |
| 21 | @ExceptionHandler 局部异常 |
| 22 | BasicErrorController 默认错误处理 |
| 23 | 自定义 ErrorAttributes |
| 24 | 自定义 Error 页面（/error/404.html） |
| 25 | 文件上传配置（multipart） |
| 26 | 文件上传大小限制（spring.servlet.multipart.max-file-size） |
| 27 | CORS 跨域配置（WebMvcConfigurer） |
| 28 | @CrossOrigin 注解跨域 |
| 29 | 路径匹配策略 |
| 30 | PathPatternParser vs AntPathMatcher（2.6+ 变化） |
| 31 | @RequestMapping 请求映射 |
| 32 | RESTful 风格接口设计 |
| 33 | 请求参数绑定（@RequestParam / @PathVariable / @RequestBody） |
| 34 | 请求验证（@Valid / @Validated） |
| 35 | BindingResult 参数绑定错误处理 |
| 36 | WebFlux 自动配置（响应式 Spring Boot） |

---

## 五、数据访问

| # | 知识点 |
|---|--------|
| 1 | DataSource 自动配置原理 |
| 2 | HikariCP 连接池（默认） |
| 3 | Druid 连接池配置与监控 |
| 4 | 连接池对比（HikariCP vs Druid vs Tomcat） |
| 5 | spring.datasource 配置属性 |
| 6 | JdbcTemplate 自动配置 |
| 7 | JdbcTemplate 使用方法 |
| 8 | NamedParameterJdbcTemplate 命名参数 |
| 9 | RowMapper 结果映射 |
| 10 | TransactionManager 自动配置 |
| 11 | @EnableTransactionManagement |
| 12 | @Transactional 事务注解 |
| 13 | 事务传播行为（7种） |
| 14 | 事务隔离级别（5种） |
| 15 | 事务失效场景（同类调用、异常被 catch、rollbackFor） |
| 16 | spring-boot-starter-data-jpa 自动配置 |
| 17 | spring.jpa.hibernate.ddl-auto 选项 |
| 18 | spring.jpa.show-sql 日志输出 |
| 19 | JPA Entity 映射（@Table / @Column） |
| 20 | JPA 关联关系（@OneToMany / @ManyToOne） |
| 21 | MyBatis 自动配置（SqlSessionFactory） |
| 22 | @MapperScan 自动扫描 |
| 23 | Mapper XML 配置 |
| 24 | MyBatis TypeHandler 类型转换 |
| 25 | 多数据源配置（@Primary） |
| 26 | application.yml 多数据源 |
| 27 | Flyway 数据库迁移 |
| 28 | Liquibase 数据库迁移 |
| 29 | DataSourceInitializer 执行 SQL |
| 30 | spring-boot-starter-data-jdbc（轻量级） |

---

## 六、缓存

| # | 知识点 |
|---|--------|
| 1 | @EnableCaching 开启缓存 |
| 2 | CacheAutoConfiguration 自动配置 |
| 3 | @Cacheable 缓存注解 |
| 4 | @CacheEvict 缓存清除 |
| 5 | @CachePut 更新缓存 |
| 6 | @Caching 组合缓存注解 |
| 7 | @CacheConfig 类级别配置 |
| 8 | @Cacheable 属性详解（unless / key / condition / sync） |
| 9 | KeyGenerator 自定义 key 生成策略 |
| 10 | CacheManager 缓存管理器 |
| 11 | ConcurrentMapCacheManager（内存缓存） |
| 12 | EhCacheCacheManager |
| 13 | RedisCacheManager |
| 14 | CaffeineCacheManager |
| 15 | 缓存序列化（JSON / JDK / Kryo） |
| 16 | 缓存过期时间配置（expireAfterWrite / expireAfterRead） |
| 17 | 缓存集群配置 |
| 18 | Spring Cache 注解原理（AOP 代理） |
| 19 | @Cacheable 失效场景（同类调用） |
| 20 | 缓存穿透（空值缓存 / BloomFilter） |
| 21 | 缓存雪崩（过期时间随机） |
| 22 | 缓存击穿（互斥锁 / 逻辑过期） |
| 23 | 分布式缓存 vs 本地缓存 |

---

## 七、异步与调度

| # | 知识点 |
|---|--------|
| 1 | @EnableAsync 启用异步 |
| 2 | @Async 异步方法注解 |
| 3 | AsyncConfigurer / AsyncConfigurerSupport 配置 |
| 4 | TaskExecutor 线程池配置 |
| 5 | @Async 线程池选择（SimpleAsyncTaskExecutor vs ThreadPoolTaskExecutor） |
| 6 | @Async 失效场景（同类调用 / 私有方法 / 未启用代理） |
| 7 | 异步方法返回值（void / Future / CompletableFuture） |
| 8 | AsyncUncaughtExceptionHandler 异常处理 |
| 9 | @EnableScheduling 启用调度 |
| 10 | @Scheduled 定时任务注解 |
| 11 | @Scheduled(cron = "...") Cron 表达式 |
| 12 | fixedDelay / fixedRate 区别 |
| 13 | initialDelay 初始延迟 |
| 14 | SchedulingConfigurer 编程式配置 |
| 15 | TaskScheduler 接口 |
| 16 | Trigger / CronTrigger 触发器 |
| 17 | Cron 表达式详解（7位 vs 6位） |
| 18 | 线程池隔离（不同业务用不同池） |
| 19 | 异步日志优化性能 |

---

## 八、运维与部署

| # | 知识点 |
|---|--------|
| 1 | Spring Boot 打包 Jar |
| 2 | 可执行 JAR 结构（BOOT-INF / META-INF） |
| 3 | spring-boot-maven-plugin 插件 |
| 4 | spring-boot-loader 加载器原理 |
| 5 | Spring Boot War 打包 |
| 6 | 部署到外部 Tomcat |
| 7 | 部署到 Jetty / Undertow |
| 8 | Spring Boot Docker 化 |
| 9 | Dockerfile 编写（多阶段构建） |
| 10 | layers 分层镜像（2.3+） |
| 11 | 镜像优化（减少体积 / 利用缓存） |
| 12 | bootstrap.yml 加载顺序（2.4 移除） |
| 13 | Spring Boot Admin 监控平台 |
| 14 | 优雅关闭（GracefulShutdown） |
| 15 | shutdown 配置（immediate / graceful） |
| 16 | K8s 健康检查探针（liveness / readiness） |
| 17 | 启动失败分析（FailureAnalyzer） |
| 18 | JNLP 部署 |
| 19 | WebSphere 部署 |

---

## 九、Spring Boot Actuator

| # | 知识点 |
|---|--------|
| 1 | spring-boot-starter-actuator 引入 |
| 2 | /actuator 端点列表 |
| 3 | /actuator/health 健康检查 |
| 4 | /actuator/info 应用信息 |
| 5 | /actuator/beans Bean 列表 |
| 6 | /actuator/env 环境变量 |
| 7 | /actuator/configprops 配置属性 |
| 8 | /actuator/mappings 请求映射 |
| 9 | /actuator/heapdump 堆 Dump |
| 10 | /actuator/threaddump 线程 Dump |
| 11 | /actuator/metrics 指标 |
| 12 | /actuator/scheduledtasks 定时任务 |
| 13 | /actuator/caches 缓存信息 |
| 14 | /actuator/loggers 日志级别 |
| 15 | /actuator/conditions 自动配置报告 |
| 16 | 自定义端点开发 |
| 17 | @Endpoint 端点注解 |
| 18 | @ReadOperation / @WriteOperation / @DeleteOperation |
| 19 | @Selector 路径参数 |
| 20 | 端点暴露配置（management.endpoints.web.exposure.include） |
| 21 | 端点安全配置（Spring Security） |
| 22 | Prometheus 集成 |
| 23 | Micrometer 指标体系 |
| 24 | 自定义健康检查指标 |

---

## 十、Spring Boot 高级特性

| # | 知识点 |
|---|--------|
| 1 | SpringApplication 事件体系 |
| 2 | ApplicationStartingEvent |
| 3 | ApplicationEnvironmentPreparedEvent |
| 4 | ApplicationPreparedEvent |
| 5 | ApplicationReadyEvent |
| 6 | ApplicationFailedEvent |
| 7 | ApplicationRunner / CommandLineRunner |
| 8 | @RefreshScope 刷新作用域 |
| 9 | Spring Boot 初始化器 |
| 10 | ApplicationContextInitializer |
| 11 | EnvironmentPostProcessor |
| 12 | BeanFactoryPostProcessor vs @PostConstruct |
| 13 | @Conditional 扩展自定义条件 |
| 14 | Banner 多媒体支持（图片 / 动画） |
| 15 | SpringApplicationBuilder / FluentBuilder API |
| 16 | 启动链路源码分析 |
| 17 | AOT 编译支持 |
| 18 | GraalVM Native Image 构建 |
| 19 | native-image-agent 代理 |
| 20 | Spring Boot 2.7 变化总结 |
| 21 | Spring Boot 2.4 变化总结 |

---

## 十一、Spring Boot 3.x 新特性

| # | 知识点 |
|---|--------|
| 1 | Spring Boot 3.0 要求 JDK 17+ |
| 2 | Jakarta EE 9+（javax → jakarta） |
| 3 | GraalVM Native Image 官方支持 |
| 4 | observability 可观测性增强 |
| 5 | Micrometer 1.11+ 集成 |
| 6 | @AutoConfiguration 替代 @Configuration |
| 7 | @AutoConfiguration 替代自动配置注解 |
| 8 | configurationProperties 验证增强 |
| 9 | PatternDefaultsProperties |
| 10 | Proxy-JavaDiagnostics |
| 11 | AOT 预编译增强 |
| 12 | 更好的故障分析报告 |
| 13 | WebFlux 自动配置改进 |
| 14 | Virtual Threads（Project Loom）支持 |
| 15 | Jakarta Data |
| 16 | Spring Authorization Server 集成 |
| 17 | 依赖升级总结（Spring Framework 6 / Spring Security 6） |

---

## 🎯 复习建议

### 第一优先级（必考 🔴）

| 模块 | 知识点 |
|---|--------|
| 一、概述与启动 | 1-15 |
| 二、自动配置原理 | 1-30 ⭐⭐⭐ |
| 三、配置管理 | 1-29 |
| 四、Web开发 | 1-36 |

### 第二优先级（高频 🟡）

| 模块 | 知识点 |
|---|--------|
| 五、数据访问 | 1-30 |
| 六、缓存 | 1-23 |
| 七、异步与调度 | 1-19 |
| 九、Actuator | 1-24 |

### 第三优先级（加分项 🟢）

| 模块 | 知识点 |
|---|--------|
| 八、运维与部署 | 1-19 |
| 十、高级特性 | 1-21 |
| 十一、3.x新特性 | 1-17 |

---

## 📁 对应文件夹结构

```
12_SpringBoot/
├── 00_SpringBoot知识地图.md
├── 01_SpringBoot概述与启动/          (15)
├── 02_自动配置原理/                   (30) ⭐
├── 03_配置管理/                       (29)
├── 04_Web开发/                        (36)
├── 05_数据访问/                       (30)
├── 06_缓存/                           (23)
├── 07_异步与调度/                     (19)
├── 08_运维与部署/                     (19)
├── 09_SpringBootActuator/            (24)
├── 10_SpringBoot高级/                (21)
└── 11_SpringBoot3_x新特性/           (17)

合计：~263 个知识点
```

---

*最后更新：2026-04-28 | v2.0*
