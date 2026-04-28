# @ConditionalOnClass 条件判断

## 先说结论

`@ConditionalOnClass` 用于判断 classpath 中是否存在指定的类，是 Spring Boot 自动配置中最常用的条件注解。当依赖的 jar 包存在时，自动配置才生效，是"按需加载"机制的基础。

## 深度解析

### 注解源码

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnClassCondition.class)
public @interface ConditionalOnClass {
    // 要检查的类
    Class<?>[] value() default {};

    // 要检查的类名（使用字符串，避免 classpath 不存在时编译失败）
    String[] name() default {};
}
```

### 两种写法对比

```java
// 方式一：Class 引用（常用）
@ConditionalOnClass(DataSource.class)
public class DataSourceAutoConfiguration { }

// 方式二：String 类名（安全写法，避免编译错误）
@ConditionalOnClass(name = "com.zaxxer.hikari.HikariDataSource")
public class HikariAutoConfiguration { }
```

### 为什么需要两种写法？

```java
// 如果 HikariDataSource 不在 classpath 中：
// ❌ 编译错误：Class 引用方式无法编译
@ConditionalOnClass(HikariDataSource.class)  // 编译失败，类不存在

// ✅ 字符串方式：编译通过，运行时判断
@ConditionalOnClass(name = "com.zaxxer.hikari.HikariDataSource")
```

### 实际应用场景

```java
// Redis 自动配置：只有引入 spring-boot-starter-data-redis 才生效
@AutoConfiguration
@ConditionalOnClass(RedisOperations.class)
@EnableConfigurationProperties(RedisProperties.class)
public class RedisAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean(StringRedisTemplate.class)
    public StringRedisTemplate stringRedisTemplate(
            RedisConnectionFactory factory) {
        return new StringRedisTemplate(factory);
    }
}
```

### 与 @ConditionalOnMissingClass 对比

| 注解 | 条件 |
|------|------|
| `@ConditionalOnClass(Xxx.class)` | classpath 中**有** Xxx 才加载 |
| `@ConditionalOnMissingClass("com.xxx.Xxx")` | classpath 中**没有** Xxx 才加载 |

## 易错点/踩坑

- ❌ **Class 引用方式** — 如果类不在 classpath 会编译失败
- ❌ **只检查类是否存在，不检查实例化** — 即使类无法实例化也会通过
- ❌ **多个类用 AND 关系** — 所有类都存在才通过

## 关联知识点

- [[ConditionalOnMissingClass条件判断]] — 相反条件
- [[自动配置条件注解概述]] — 条件注解体系
- [[spring.factories文件]] — 配置文件位置
