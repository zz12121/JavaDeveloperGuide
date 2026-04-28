# Spring Boot 核心优势

## 先说结论

Spring Boot 的三大核心优势：**快速构建**、**自动配置**、**内嵌容器**，让 Java 开发从"繁琐配置"升级为"开箱即用"。

## 深度解析

### 1. 快速构建

#### Create New Project

```
IntelliJ IDEA / Spring Initializr
    │
    ├── 选择依赖（Web/JPA/Redis/...）
    │
    └── 生成项目结构：
         src/
         ├── main/
         │   ├── java/
         │   │   └── com.example.demo/
         │   │       └── DemoApplication.java
         │   └── resources/
         │       └── application.properties
         └── pom.xml
```

#### 时间对比

| 任务 | SSM | Spring Boot |
|------|-----|-------------|
| 创建空项目 | 30 分钟 | 1 分钟 |
| 整合框架 | 2-3 天 | 1 小时 |
| 跑通 CRUD | 1 周 | 1 天 |
| 部署上线 | 2 天 | 30 分钟 |

### 2. 自动配置

#### 原理示意

```
┌─────────────────────────────────────────────────────────┐
│                   自动配置流程                            │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  启动类 @SpringBootApplication                          │
│       │                                                │
│       ├── @EnableAutoConfiguration                     │
│       │       │                                        │
│       │       └──→ 扫描 spring-boot-autoconfigure.jar  │
│       │                    │                           │
│       │                    ├──→ 读取 spring.factories  │
│       │                    │        或                   │
│       │                    │   AutoConfiguration.imports│
│       │                    │                           │
│       │                    └──→ @Conditional* 判断      │
│       │                             │                   │
│       │                             ↓                   │
│       │                        生效则注册 Bean           │
│       │                                                │
│       └── @ComponentScan 扫描自定义 Bean               │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

#### 自动配置的 Bean

| 类别 | 自动配置的 Bean |
|------|----------------|
| Web | DispatcherServlet、HandlerMapping、ViewResolver |
| JSON | JacksonHttpMessageConverters |
| 数据 | DataSource、HikariCP、JdbcTemplate |
| 事务 | DataSourceTransactionManager |
| 缓存 | CacheManager（根据类路径自动选择） |
| 安全 | SpringSecurityAutoConfiguration |

### 3. 内嵌容器

#### 支持的容器

```xml
<!-- 默认 Tomcat -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <!-- 包含 spring-boot-starter-tomcat -->
</dependency>

<!-- 切换 Jetty -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>

<!-- 切换 Undertow -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```

#### 容器对比

| 容器 | 吞吐量 | 内存 | 特点 |
|------|--------|------|------|
| Tomcat | 中 | 中 | 官方默认，生态完善 |
| Jetty | 高 | 低 | 轻量，适合嵌入式 |
| Undertow | 最高 | 最低 | 高性能，不支持 JSP |

### 4. 其他优势

#### starter 生态

```
spring-boot-starter-*          → 官方维护
spring-boot-starter-web        → Web 开发
spring-boot-starter-data-redis  → Redis 访问
spring-boot-starter-security   → 安全控制

*-spring-boot-starter           → 第三方维护
mybatis-spring-boot-starter     → MyBatis 集成
druid-spring-boot-starter      → Druid 连接池
```

#### 运维支持

```
Spring Boot Actuator
    ├── /actuator/health      → 健康检查
    ├── /actuator/metrics    → 性能指标
    ├── /actuator/env        → 环境变量
    └── /actuator/beans      → Bean 列表
```

## 易错点/踩坑

- ❌ 忽视自动配置报告，排查问题困难
- ❌ 不理解条件注解，导致配置不生效
- ❌ 随意切换容器，不做性能测试

## 代码示例

```java
// 10 分钟完成一个 Web 服务
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}

@RestController
class DemoController {
    @GetMapping("/hello")
    public String hello() {
        return "Hello, Spring Boot!";
    }
}
```

```yaml
# application.yml - 极简配置
server:
  port: 8080
spring:
  application:
    name: demo
```

## 关联知识点

- `@SpringBootApplication` 组合注解
- 自动配置原理
- 嵌入式容器配置
- Spring Boot Starter
