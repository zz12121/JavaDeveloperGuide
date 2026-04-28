# @ConditionalOnWebApplication 条件判断

## 先说结论

`@ConditionalOnWebApplication` 用于判断当前应用是否是 Web 应用环境（Servlet 或 Reactive），只有 Web 应用才会加载 Web 相关的自动配置，非 Web 应用（如命令行工具、批处理任务）不会加载这些配置。

## 深度解析

### 注解源码

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnWebApplicationCondition.class)
public @interface ConditionalOnWebApplication {
    Type type() default Type.ANY;
}

public enum Type {
    ANY,        // 任何 Web 应用
    SERVLET,    // 仅 Servlet Web 应用
    REACTIVE    // 仅 Reactive Web 应用
}
```

### Web 应用类型

| 类型 | 说明 | 典型场景 |
|------|------|----------|
| `SERVLET` | 基于 Servlet API（Tomcat/Jetty/Undertow） | Spring MVC 应用 |
| `REACTIVE` | 基于 Reactive（Netty + WebFlux） | Spring WebFlux 应用 |
| `ANY` | 任何 Web 类型 | 不区分具体类型 |

### 代码示例

```java
// 只有 Web 应用才加载 WebMvcAutoConfiguration
@AutoConfiguration
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
@ConditionalOnClass(HttpServletRequest.class)
public class WebMvcAutoConfiguration { }

// 只有 Reactive 应用才加载 WebFluxAutoConfiguration
@AutoConfiguration
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.REACTIVE)
@ConditionalOnClass(HttpHandler.class)
public class WebFluxAutoConfiguration { }
```

### 判断依据

Spring Boot 通过以下方式判断 Web 环境：

| 检查项 | 说明 |
|--------|------|
| `HttpServletRequest` 存在 | Servlet 环境 |
| `HttpSession` 存在 | Servlet 环境 |
| `WebApplicationContext` | Servlet 环境 |
| `ReactiveWebApplicationContext` | Reactive 环境 |
| `spring-webflux` 依赖 | Reactive 环境 |

## 易错点/踩坑

- ❌ **`Type.ANY` 包含所有 Web 类型** — 如果只想针对 Servlet，需要明确指定 `Type.SERVLET`
- ❌ **纯 Spring Boot CLI 工具** — 不是 Web 环境，不会加载任何 Web 相关配置

## 关联知识点

- [[ConditionalOnNotWebApplication条件判断]] — 非 Web 环境判断
- [[自动配置条件注解概述]] — 条件注解体系
