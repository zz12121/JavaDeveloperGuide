# SpringBoot自动配置WebMvc - QA

## Q1：SpringBoot 如何做到零配置开发 Web 应用？

**A**：
- SpringBoot 通过 `WebMvcAutoConfiguration` 自动配置了 Web 开发所需的核心组件
- 只要引入 `spring-boot-starter-web`，自动配置生效
- 用户只需写业务代码，无需手动配置 DispatcherServlet、视图解析器等

**要点**：
- 核心：`WebMvcAutoConfiguration` 类
- 触发：`@EnableAutoConfiguration` 扫描 `spring-boot-autoconfigure`
- 生效条件：`DispatcherServlet` 不存在、容器中有 `WebMvcConfigurer` 等

---

## Q2：什么时候会禁用自动配置？

**A**：添加 `@EnableWebMvc` 注解时会禁用。

**原因**：`@EnableWebMvc` 导入了 `DelegatingWebMvcConfiguration`，该类继承 `WebMvcConfigurationSupport`，当检测到 `WebMvcConfigurationSupport` 已存在时，`WebMvcAutoConfiguration` 的 `@ConditionalOnMissingBean` 条件不满足，自动配置失效。

**场景**：需要完全自定义 WebMvc 配置时使用。

---

## Q3：自定义 WebMvcConfigurer 会禁用自动配置吗？

**A**：不会。

**说明**：
- `WebMvcConfigurer` 是配置接口，实现它只是添加自定义配置
- 自动配置 + 用户配置 = 合并后的最终配置
- **只有** `@EnableWebMvc` 才会禁用自动配置

---

## Q4：静态资源无法访问怎么办？

**A**：常见原因及解决方案：

| 原因 | 解决方案 |
|------|----------|
| 添加了 `@EnableWebMvc` | 移除，或手动配置静态资源映射 |
| 覆盖了 `addResourceHandlers` | 检查实现逻辑 |
| 文件放错位置 | 放在 `/static/`、`/public/`、`/resources/` 下 |

---

## Q5：如何排除特定的自动配置？

**A**：

```java
@SpringBootApplication(exclude = {WebMvcAutoConfiguration.class})
// 或
@SpringBootApplication(excludeName = "org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration")
```

**注意**：排除后 WebMvc 功能完全由你接管，慎用。
