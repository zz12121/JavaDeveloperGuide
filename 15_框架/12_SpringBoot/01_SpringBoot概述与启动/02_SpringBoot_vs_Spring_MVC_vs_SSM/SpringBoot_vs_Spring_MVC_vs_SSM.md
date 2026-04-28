# Spring Boot vs Spring MVC vs SSM

## 先说结论

- **SSM**：Spring + Spring MVC + MyBatis，需要大量 XML 配置
- **Spring MVC**：Spring 的 Web 框架，处理 HTTP 请求/响应
- **Spring Boot**：基于 Spring MVC，简化配置，内嵌容器，快速开发

## 深度解析

### 技术栈对比

```
┌─────────────────────────────────────────────────────────────┐
│                        整体架构对比                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  SSM（传统）                Spring MVC             Spring Boot│
│  ┌─────────────┐          ┌─────────────┐       ┌──────────┐│
│  │   Spring    │          │   Spring    │       │  Spring  ││
│  │  (IoC/AOP)  │          │  (IoC/AOP)  │       │(IoC/AOP) ││
│  └──────┬──────┘          └──────┬──────┘       └────┬─────┘│
│         │                        │                   │     │
│  ┌──────┴──────┐          ┌──────┴──────┐           │     │
│  │ Spring MVC  │          │ Spring MVC  │←──────────┘     │
│  │  (XML配置)   │          │ (注解配置)   │                 │
│  └──────┬──────┘          └──────┬──────┘           │     │
│         │                        │                   │     │
│  ┌──────┴──────┐                 │           ┌──────┴─────┐ │
│  │   MyBatis   │                 │           │Auto Config │ │
│  │ (XML Mapper)│                 │           └──────┬─────┘ │
│  └─────────────┘                 │                  │       │
│                                  │           ┌──────┴─────┐  │
│                                  │           │ 内嵌容器   │  │
│                                  │           │ Tomcat/Jetty│ │
│                                  │           └─────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 详细对比表

| 维度 | SSM | Spring MVC | Spring Boot |
|------|-----|------------|-------------|
| **配置方式** | XML | 注解为主 | 注解 + 自动配置 |
| **依赖管理** | 手动指定版本 | 手动指定版本 | starter + parent |
| **容器** | 外部 Tomcat | 外部 Tomcat | 内嵌 Tomcat（默认） |
| **打包** | War | War | Jar（默认）/ War |
| **启动方式** | deploy 到容器 | deploy 到容器 | java -jar 运行 |
| **配置文件** | 多个 XML | application.properties | application.yml |
| **开发效率** | 慢 | 中 | 快 |
| **学习曲线** | 陡峭 | 较陡 | 平缓 |

### Spring MVC 与 Spring Boot 的关系

```
Spring MVC ⊂ Spring ⊂ Spring Boot

Spring Boot = Spring MVC + 自动配置 + 内嵌容器 + starter 依赖
```

**Spring Boot 对 Spring MVC 的增强**：
1. 自动配置 `WebMvcAutoConfiguration`
2. 自动配置静态资源处理
3. 自动配置视图解析器
4. 自动配置消息转换器
5. 提供 `WebMvcConfigurer` 扩展点

## 易错点/踩坑

- ❌ 混淆 Spring MVC 和 Spring Boot 的关系
- ❌ 认为 Spring Boot 取代了 Spring MVC（实际是增强）
- ❌ 忽视 Spring Boot 的自动配置可能覆盖自定义配置

## 代码示例

### SSM 配置（XML）

```xml
<!-- web.xml -->
<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:applicationContext.xml</param-value>
</context-param>
<servlet>
    <servlet-name>dispatcher</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:springmvc.xml</param-value>
    </init-param>
</servlet>
```

### Spring Boot 配置（极简）

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

```java
@RestController
@RequestMapping("/user")
public class UserController {
    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);
    }
}
```

## 关联知识点

- `@SpringBootApplication` 组合注解
- Spring Boot Starter
- 自动配置原理
- WebMvcAutoConfiguration
