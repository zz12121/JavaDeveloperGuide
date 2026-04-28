# Spring Boot 核心理念

## 先说结论

**约定大于配置**（Convention Over Configuration）是 Spring Boot 的核心理念，通过提供合理的默认值和自动配置，减少开发者的配置负担，实现"零配置"或"少配置"开发。

## 深度解析

### 什么是约定大于配置

约定大于配置是一种软件设计范式，框架为常见场景提供默认配置，开发者只需在偏离默认行为时才进行配置。

### Spring Boot 的体现

| 方面 | 默认行为 | 可配置项 |
|------|----------|----------|
| 项目结构 | src/main/java、src/main/resources | 可自定义 |
| 打包方式 | Jar（可执行） | 可改为 War |
| 配置文件 | application.properties/yml | 可指定路径 |
| 端口 | 8080 | server.port |
| 静态资源 | /static、/public 等 | 可自定义目录 |
| 日志 | SLF4J + Logback | 可自定义配置 |

### 传统配置 vs Spring Boot

```
传统 SSM 配置（XML）：
┌─────────────────────────────────────────┐
│ web.xml                                 │
│   ├── ContextLoaderListener             │
│   ├── DispatcherServlet                │
│   └── CharacterEncodingFilter           │
├─────────────────────────────────────────┤
│ applicationContext.xml                  │
│   ├── DataSource                        │
│   ├── SqlSessionFactory                 │
│   ├── TransactionManager                │
│   └── MapperScanner                     │
├─────────────────────────────────────────┤
│ springmvc.xml                           │
│   ├── ViewResolver                      │
│   ├── HandlerMapping                    │
│   └── Interceptors                      │
└─────────────────────────────────────────┘

Spring Boot 配置（注解 + 少量配置）：
┌─────────────────────────────────────────┐
│ application.yml（可选）                  │
│   └── server.port: 8080                │
├─────────────────────────────────────────┤
│ @SpringBootApplication                  │
│   └── 一切自动配置完成                   │
└─────────────────────────────────────────┘
```

### 自动配置原理

Spring Boot 的自动配置通过 `@EnableAutoConfiguration` 触发：

1. `spring-boot-autoconfigure` 包中包含大量 `xxxAutoConfiguration` 类
2. 通过 `@Conditional*` 条件注解判断是否生效
3. 满足条件时自动注册相关 Bean

## 易错点/踩坑

- ❌ 以为 Spring Boot 真的"零配置"，忽视自定义配置需求
- ❌ 忽略自动配置报告，导致排查问题困难
- ❌ 默认配置不满足需求时，不知道如何覆盖

## 代码示例

```java
// Spring Boot 启动类 - 极简配置
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

```properties
# application.properties - 可选配置
server.port=8080
spring.application.name=demo
```

## 图解/流程

```
┌─────────────────────────────────────────────────────────────┐
│                     约定大于配置                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   常见场景 ──→ Spring Boot 提供默认值                        │
│       │                                                    │
│       ├── 端口 → 8080                                      │
│       ├── 上下文路径 → /                                   │
│       ├── 日志 → SLF4J + Logback                           │
│       ├── JSON → Jackson                                  │
│       └── 数据库 → HikariCP + JPA                          │
│                                                             │
│       ↓ 偏离默认时                                           │
│                                                             │
│   自定义配置 ──→ 只需配置不同的部分                           │
│       │                                                    │
│       └── server.port=9090                                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 关联知识点

- `@SpringBootApplication` 组合注解
- 自动配置原理
- `spring.factories` / `AutoConfiguration.imports`
