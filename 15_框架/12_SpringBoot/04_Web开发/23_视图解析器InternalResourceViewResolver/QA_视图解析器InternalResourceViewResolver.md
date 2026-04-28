# 视图解析器InternalResourceViewResolver - QA

## Q1：Thymeleaf 和 JSP 可以同时使用吗？

**A**：可以，但通常不推荐。

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {
    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        registry.jsp("/WEB-INF/views/", ".jsp");
        registry.enableContentNegotiation();  // 开启内容协商
    }
}
```

---

## Q2：多个 ViewResolver 的优先级？

**A**：Bean 名称决定优先级，Bean 名称为 `viewResolver` 的优先级最高。

| 顺序 | ViewResolver | Bean 名称 |
|------|-------------|-----------|
| 1 | BeanNameViewResolver | beanName |
| 2 | InternalResourceViewResolver | viewResolver |
| 3 | ... | ... |

---

## Q3：JSP 文件放在哪里？

**A**：

```yaml
spring:
  mvc:
    view:
      prefix: /WEB-INF/views/
```

**目录结构**：
```
src/main/
├── webapp/
│   └── WEB-INF/
│       └── views/
│           └── user/
│               └── list.jsp
```

**注意**：Spring Boot 建议使用 `src/main/resources/templates/` 放置 Thymeleaf。
