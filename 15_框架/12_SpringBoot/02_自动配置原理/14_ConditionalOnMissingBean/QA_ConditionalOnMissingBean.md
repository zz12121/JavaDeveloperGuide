# @ConditionalOnMissingBean 条件判断

## Q1：@ConditionalOnMissingBean 为什么是自动配置最重要的注解？

**A**：它是"用户优先"原则的核心保证：

```java
// Spring Boot 自动配置
@AutoConfiguration
public class RedisAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean(StringRedisTemplate.class)  // 关键
    public StringRedisTemplate stringRedisTemplate(...) {
        return new StringRedisTemplate(...);  // 提供默认实现
    }
}
```

```java
// 用户自定义配置
@Configuration
public class MyConfig {
    @Bean
    public StringRedisTemplate stringRedisTemplate(...) {
        // 自定义实现
        return new CustomStringRedisTemplate(...);
    }
}
```

结果：用户的 `MyConfig` 中的 `StringRedisTemplate` 先生效，自动配置提供的被跳过。

---

## Q2：@ConditionalOnMissingBean 在类上和方法上有什么区别？

**A**：

| 位置 | 效果 |
|------|------|
| 类上 | 整个配置类的所有 @Bean 都不注册 |
| 方法上 | 只有该 @Bean 方法不注册 |

```java
// ❌ 类级别：条件失败，整个类跳过
@ConditionalOnMissingBean(DataSource.class)
public class JdbcAutoConfiguration {
    @Bean public JdbcTemplate jdbcTemplate() { ... }  // 也不注册
}

// ✅ 方法级别：条件失败，只有这个方法不注册
public class JdbcAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean(JdbcTemplate.class)
    public JdbcTemplate jdbcTemplate() { ... }  // 只有这个不注册
}
```

---

## Q3：为什么有时候 @ConditionalOnMissingBean 不生效？

**A**：常见原因：

| 原因 | 说明 |
|------|------|
| 用户代码在自动配置**之前**执行 | `@ConditionalOnMissingBean` 检测时，用户的 Bean 还未注册 |
| @ComponentScan 顺序问题 | 先扫描到的包先处理 |
| 父子容器 | 子容器检测不到父容器的 Bean |

**最佳实践**：不要过度依赖条件注解，通过**参数注入**更可靠：

```java
@Bean
public MyService myService(@Autowired(required=false) MyDep dep) {
    // 无论 MyDep 是否存在都能正常工作
}
```

---

## Q4：ignored 属性有什么用？

**A**：

```java
// 即使存在 HikariDataSource，也视为"不存在"
@ConditionalOnMissingBean(
    value = DataSource.class,
    ignored = {HikariDataSource.class}
)
public class DruidAutoConfiguration {
    @Bean
    public DataSource dataSource() {
        return new DruidDataSource();
    }
}
```

**场景**：假设 HikariCP 是默认配置，但你希望用户在 pom 中引入 Druid 时自动切换。

---

## Q5：@ConditionalOnMissingBean 和 @Primary 一起用需要注意什么？

**A**：

```java
@Bean
@Primary
@ConditionalOnMissingBean(MyService.class)
public MyService myService() {
    return new DefaultMyService();
}
```

**问题**：如果用户配置了另一个 `MyService` 但没有标记 `@Primary`，而自动配置的也没有被排除，Spring 会报错"有多个相同类型的 Bean"。

**解决**：
1. 始终确保自动配置是兜底方案，配合 `@ConditionalOnMissingBean`
2. 用户配置应该更早加载或使用 `@Primary` 显式标记

