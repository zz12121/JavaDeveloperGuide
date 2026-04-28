# @ConditionalOnWebApplication 条件判断

## Q1：如何区分 Servlet 和 Reactive Web 应用？

**A**：通过 `type` 属性：

```java
// Servlet Web 应用（Spring MVC + Tomcat）
@ConditionalOnWebApplication(type = Type.SERVLET)
public class WebMvcAutoConfiguration { }

// Reactive Web 应用（WebFlux + Netty）
@ConditionalOnWebApplication(type = Type.REACTIVE)
public class WebFluxAutoConfiguration { }

// 任何 Web 应用
@ConditionalOnWebApplication(type = Type.ANY)
public class CommonWebAutoConfiguration { }
```

---

## Q2：Spring Boot 如何判断是否为 Web 应用？

**A**：通过检查以下条件（满足任一即判定为 Web）：

| 条件 | Web 类型 |
|------|----------|
| `javax.servlet.http.HttpServletRequest` 存在 classpath | SERVLET |
| `org.springframework.web.context.support.StandardServletEnvironment` | SERVLET |
| `org.springframework.web.reactive.HandlerResult` | REACTIVE |
| `spring-webflux` 依赖存在 | REACTIVE |

---

## Q3：命令行应用会加载 Web 配置吗？

**A**：**不会**：

```bash
# 普通命令行工具（非 Web）
$ java -jar myapp.jar
# Spring Boot 检测到不是 Web 环境
# WebMvcAutoConfiguration 等配置不会加载

# Web 应用
$ java -jar mywebapp.jar
# 检测到 Web 环境
# WebMvcAutoConfiguration 等配置会加载
```

---

## Q4：如何让同一个配置在 Web 和非 Web 环境都生效？

**A**：不标注此注解即可，或明确使用 `Type.ANY`：

```java
// 不标注此条件 → Web 和非 Web 都生效
@Configuration
public class CommonAutoConfiguration { }

// 明确指定任意 Web 环境
@ConditionalOnWebApplication(type = Type.ANY)
@Configuration
public class WebCommonAutoConfiguration { }
```

---

## Q5：Type.REACTIVE 和 Type.SERVLET 互斥吗？

**A**：是的。一个应用要么是 Servlet 类型，要么是 Reactive 类型，不会同时满足：

```java
// WebMvcAutoConfiguration 只加载 Servlet 环境
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass(HttpServletRequest.class)
public class WebMvcAutoConfiguration { }

// WebFluxAutoConfiguration 只加载 Reactive 环境
@ConditionalOnWebApplication(type = Type.REACTIVE)
@ConditionalOnClass(HttpHandler.class)
public class WebFluxAutoConfiguration { }
```

