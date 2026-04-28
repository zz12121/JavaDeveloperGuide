# EnableAutoConfiguration 启用自动配置

## Q1：@EnableAutoConfiguration 的核心原理是什么？

**A**：核心在于 `@Import(AutoConfigurationImportSelector.class)`：

1. Spring Boot 启动时，`@Import` 注解将 `AutoConfigurationImportSelector` 导入到容器
2. 该类实现 `DeferredImportSelector` 接口，在所有 `@Configuration` 类处理完成后执行
3. `getAutoConfigurationEntry()` 方法从 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`（2.7+）或 `META-INF/spring.factories` 读取候选配置类列表
4. 返回的配置类会被当作 `@Configuration` 类处理，执行 `@Conditional*` 条件判断
5. 通过条件的配置类 → 注册为 Bean；未通过 → 静默跳过

---

## Q2：@EnableAutoConfiguration 和 @ComponentScan 会冲突吗？

**A**：不会冲突，各自职责不同：

| 注解 | 扫描范围 | 来源 |
|------|----------|------|
| `@ComponentScan` | 用户代码中的 `@Component` / `@Configuration` 等 | 应用本身的类 |
| `@EnableAutoConfiguration` | `spring-boot-autoconfigure` 中的自动配置类 | Spring Boot 提供 |

两者互补：用户代码负责业务组件，Spring Boot 自动配置负责基础设施配置。

---

## Q3：排除自动配置有哪几种方式？

**A**：四种方式：

```java
// 1. 主启动类 exclude 属性
@SpringBootApplication(exclude = { DataSourceAutoConfiguration.class })

// 2. 主启动类 excludeName 属性
@SpringBootApplication(excludeName = "org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration")

// 3. @EnableAutoConfiguration exclude 属性
@EnableAutoConfiguration(exclude = { WebMvcAutoConfiguration.class })

// 4. 配置文件（Spring Boot 2.x）
spring:
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

**注意**：Spring Boot 3.x 移除了 `exclude` 属性，只能用 `excludeName` 或配置文件。

---

## Q4：为什么排除的类名写错了不报错？

**A**：排除类名只是作为"黑名单"使用，Spring Boot 并不会验证这些类是否真实存在。如果类不存在，排除操作相当于"什么都没做"，不会抛异常。这可能导致配置意外失效，需要注意。

---

## Q5：@EnableAutoConfiguration 可以单独使用吗？

**A**：可以，但不推荐：

```java
@Configuration
@EnableAutoConfiguration
public class MyConfig {
    public static void main(String[] args) {
        SpringApplication.run(MyConfig.class, args);
    }
}
```

**问题**：
- 少了 `@ComponentScan`，不会扫描同包及子包下的组件
- 少了默认的包扫描，所有 `@Component` 都不会被发现
- 通常直接使用 `@SpringBootApplication` 更简洁

