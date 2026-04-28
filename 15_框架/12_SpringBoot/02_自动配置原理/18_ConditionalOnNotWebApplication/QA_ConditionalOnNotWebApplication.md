# @ConditionalOnNotWebApplication 条件判断

## Q1：@ConditionalOnNotWebApplication 有什么用？

**A**：用于非 Web 应用的场景：

```java
// 只有非 Web 环境才加载
@ConditionalOnNotWebApplication
@Configuration
public class BatchAutoConfiguration {
    // 批处理任务专用配置
}
```

常见场景：
- Spring Boot CLI 工具
- 批处理任务（Spring Batch）
- 消息队列消费者
- 定时任务应用

---

## Q2：可以同时使用 @ConditionalOnWebApplication 和 @ConditionalOnNotWebApplication 吗？

**A**：**不可以**，逻辑互斥，同时使用永远不满足条件：

```java
// ❌ 错误：不可能同时满足
@ConditionalOnWebApplication
@ConditionalOnNotWebApplication
public class MyConfiguration { }

// ✅ 正确：分开两个配置类
@ConditionalOnWebApplication
public class WebConfig { }

@ConditionalOnNotWebApplication
public class NonWebConfig { }
```

---

## Q3：如何判断当前是 Web 还是非 Web 环境？

**A**：

```java
@Autowired
private ApplicationContext context;

@PostConstruct
public void check() {
    // 检查是否是 Web 环境
    ConfigurableWebApplicationContext wac =
        (context instanceof ConfigurableWebApplicationContext) ? (ConfigurableWebApplicationContext) context : null;

    if (wac != null) {
        System.out.println("Web 应用环境");
    } else {
        System.out.println("非 Web 应用环境");
    }
}
```

---

## Q4：非 Web 环境有哪些 Spring Boot 自动配置不加载？

**A**：

| 不加载 | 说明 |
|--------|------|
| `WebMvcAutoConfiguration` | Spring MVC 配置 |
| `WebFluxAutoConfiguration` | WebFlux 配置 |
| `DispatcherServletAutoConfiguration` | DispatcherServlet |
| `ServletWebServerFactoryAutoConfiguration` | 嵌入式容器 |
| `ErrorMvcAutoConfiguration` | 错误页面配置 |

