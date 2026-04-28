# @EnableConfigurationProperties QA

## Q1：@EnableConfigurationProperties 和 @Component 哪个先用？

**A**：执行顺序：
1. `@EnableConfigurationProperties` 注册配置属性类为 Bean
2. 如果配置属性类同时有 `@Component`，效果相同

建议：用 `@EnableConfigurationProperties` 管理第三方配置，用 `@Component` 管理自定义配置。

---

## Q2：可以同时用两者吗？

**A**：不建议，会重复注册 Bean，可能导致问题。

---

## Q3：@EnableConfigurationProperties 可以启用多个类吗？

**A**：可以：

```java
@EnableConfigurationProperties({
    AppProperties.class,
    ServerProperties.class,
    DataSourceProperties.class
})
public class Application { }
```

---

## Q4：自动配置场景下一定要用 @EnableConfigurationProperties 吗？

**A**：不一定。Spring Boot 2.2+ 支持在配置属性类上直接加 `@ConfigurationProperties`，配合 `@Component` 即可：

```java
@Component
@ConfigurationProperties(prefix = "app")
public class AppProperties { }
```

`@EnableConfigurationProperties` 主要用于：
- 自动配置类中启用第三方配置属性
- 不希望配置属性类被 @Component 扫描到的场景

---

## Q5：@EnableConfigurationProperties 支持泛型吗？

**A**：支持：

```java
@EnableConfigurationProperties(AppProperties.class)
@EnableConfigurationProperties(ServerProperties.class)
public class Application { }
```

---

## Q6：配置属性类的 @Profile 注解会影响 @EnableConfigurationProperties 吗？

**A**：会。如果配置属性类标记了 `@Profile("dev")`，只有激活 dev profile 时才会注册该 Bean。
