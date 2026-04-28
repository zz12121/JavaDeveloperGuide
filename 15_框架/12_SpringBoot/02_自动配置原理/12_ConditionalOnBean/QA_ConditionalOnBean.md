# @ConditionalOnBean 条件判断

## Q1：@ConditionalOnBean 在什么时机检查 Bean？

**A**：`DeferredImportSelector` 机制保证了自动配置类在所有 `@Configuration` 类**之后**加载，但检查 `@Bean` 方法时 Bean 的注册顺序仍需注意：

```java
@AutoConfiguration
@ConditionalOnBean(DataSource.class)  // 此时 DataSource 可能尚未注册
public class MyAutoConfiguration {

    @Bean
    public MyService myService(DataSource dataSource) {  // 参数注入更安全
        return new MyService(dataSource);
    }
}
```

**最佳实践**：通过方法参数注入而非类级别条件判断。

---

## Q2：@ConditionalOnBean 和 @ConditionalOnMissingBean 有什么区别？

**A**：

| 注解 | 条件 | 典型场景 |
|------|------|----------|
| `@ConditionalOnBean(Xxx.class)` | 容器中有 Xxx 才加载 | 依赖其他 Bean 的功能 |
| `@ConditionalOnMissingBean(Xxx.class)` | 容器中没有 Xxx 才加载 | 提供默认实现，避免覆盖用户配置 |

```java
// 场景：只有存在 CacheManager 才注册缓存切面
@ConditionalOnBean(CacheManager.class)
public class CacheAspectAutoConfiguration { }

// 场景：只有没有自定义 RedisTemplate 才注册默认实现
@ConditionalOnMissingBean
public RedisTemplate redisTemplate() {
    return new RedisTemplate();
}
```

---

## Q3：如何检查 Bean 是否存在而不只是类型？

**A**：使用 `name` 属性指定 Bean 名称：

```java
@ConditionalOnBean(name = "primaryDataSource")
// 只有名为 primaryDataSource 的 Bean 存在时才加载
public class DataSourceMonitorAutoConfiguration { }
```

---

## Q4：@ConditionalOnBean 在 @Bean 方法上使用会有什么坑？

**A**：常见问题：

```java
@Configuration
public class MyConfig {

    @Bean
    @ConditionalOnBean(DataSource.class)  // ✅ 有效，DataSource 之前已注册
    public MyService myService(DataSource ds) {
        return new MyService(ds);
    }
}
```

但如果：

```java
@Configuration
public class MyConfig {

    @Bean
    public DataSource dataSource() { return new HikariDataSource(); }

    @Bean
    @ConditionalOnBean(DataSource.class)  // ⚠️ 可能失败，取决于 @ComponentScan 顺序
    public MyService myService() {
        return new MyService();
    }
}
```

**建议**：始终通过方法参数注入依赖，而不是在方法上使用 `@ConditionalOnBean`。

---

## Q5：search 属性怎么用？

**A**：`SearchStrategy` 控制搜索范围：

```java
public enum SearchStrategy {
    CURRENT,      // 只搜索当前容器
    ANCESTORS,    // 搜索当前容器 + 父容器
    ALL           // 搜索所有容器（默认）
}
```

```java
// 在 Spring Boot 应用中，通常只搜索当前容器
@ConditionalOnBean(value = DataSource.class, search = SearchStrategy.CURRENT)
public class MyAutoConfiguration { }
```

在 Spring MVC + Spring Boot 组合中，可能需要搜索父容器来找到根上下文的 Bean。

