# 自动配置排除详解

## Q1：exclude 和 excludeName 有什么区别？

**A**：

| 属性 | 参数类型 | 优点 |
|------|----------|------|
| `exclude` | `Class<?>[]` | IDE 自动补全、重构支持 |
| `excludeName` | `String[]` | 不需要 import，避免编译依赖 |

```java
// exclude：需要 import
@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)

// excludeName：使用全限定类名
@SpringBootApplication(excludeName = "org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration")
```

---

## Q2：Spring Boot 3.0 如何排除自动配置？

**A**：Spring Boot 3.0 移除了 `exclude` 属性：

```java
// ❌ 不支持
@SpringBootApplication(exclude = { DataSourceAutoConfiguration.class })

// ✅ 使用 excludeName
@SpringBootApplication(excludeName = {
    "org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration"
})

// ✅ 配置文件方式
// application.yml
spring:
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

---

## Q3：排除后还想使用该功能怎么办？

**A**：排除只是跳过自动配置，手动配置仍然可以使用：

```java
@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
public class MyApplication {

    @Bean
    public DataSource dataSource() {
        // 手动配置数据源
        return new DruidDataSource();
    }
}
```

---

## Q4：如何验证排除是否生效？

**A**：开启 debug 模式查看报告：

```properties
debug: true
```

启动日志中应该看不到被排除的配置在 `positiveMatches` 中，或者查看 `negativeMatches` 中的原因。

---

## Q5：可以排除所有自动配置吗？

**A**：可以，但不推荐：

```java
@SpringBootApplication(excludeName = {
    "org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration",
    "org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration",
    // ... 需要列出所有
})
public class MyApplication { }

// ✅ 更简单的方式：禁用自动配置
@EnableAutoConfiguration(exclude = {
    DataSourceAutoConfiguration.class,
    HibernateJpaAutoConfiguration.class
})
```

