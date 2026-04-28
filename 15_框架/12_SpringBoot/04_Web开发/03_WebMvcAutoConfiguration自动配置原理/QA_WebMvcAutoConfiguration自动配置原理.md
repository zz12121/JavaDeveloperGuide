# WebMvcAutoConfiguration自动配置原理 - QA

## Q1：自动配置和用户自定义配置哪个优先？

**A**：用户配置优先。

**原理**：
- 自动配置类使用 `@ConditionalOnMissingBean`
- 只有当用户**没有**自定义某个组件时，自动配置才会生效
- Spring 先加载自动配置，再加载用户配置，用户配置会覆盖自动配置

**示例**：
```java
// 用户自定义后，WebMvcAutoConfiguration 中的默认配置不生效
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {
    @Override
    public void addViewControllers(ViewControllerRegistry registry) {
        // 覆盖默认的视图控制器
    }
}
```

---

## Q2：@ConditionalOnWebApplication 是什么条件？

**A**：判断应用是否为 Web 应用。

**类型**：
- `Type.SERVLET`：基于 Servlet 的 Web 应用（SpringBoot 默认）
- `Type.REACTIVE`：响应式 Web 应用
- `Type.ANY`：任意 Web 类型

**源码示例**：
```java
@ConditionalOnWebApplication(type = Type.SERVLET)
public class WebMvcAutoConfiguration {}
```

---

## Q3：如何查看哪些自动配置生效了？

**A**：三种方式：

```bash
# 方式1：启动参数
java -jar app.jar --debug

# 方式2：配置文件
spring:
  debug: true
  autoconfigure:
    exclude:  # 排除的自动配置
      - org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration

# 方式3：代码中获取
@Autowired
private ApplicationContext context;

public void listBeans() {
    String[] beans = context.getBeanNamesForType(WebMvcConfigurer.class);
    Arrays.stream(beans).forEach(System.out::println);
}
```

---

## Q4：WebMvcAutoConfiguration 什么时候不会生效？

**A**：以下情况自动配置失效：

| 条件 | 说明 |
|------|------|
| 添加 `@EnableWebMvc` | 完全接管，禁用自动配置 |
| 存在 `WebMvcConfigurationSupport` Bean | 检测到已自定义，条件不满足 |
| 排除 `WebMvcAutoConfiguration` | `@SpringBootApplication(exclude = ...)` |
| `spring.main.web-application-type=none` | 设为非 Web 应用 |

---

## Q5：自动配置类的加载顺序是怎样的？

**A**：按以下顺序：

1. `WebMvcAutoConfiguration` 基础配置
2. `WebMvcAutoConfigurationAdapter` 适配配置
3. `EnableWebMvcConfiguration` 启用配置
4. 用户自定义 `WebMvcConfigurer`

**原则**：先加载基础，后加载用户配置，用户配置覆盖基础。
