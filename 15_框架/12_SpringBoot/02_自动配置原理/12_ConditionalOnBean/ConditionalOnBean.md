# @ConditionalOnBean 条件判断

## 先说结论

`@ConditionalOnBean` 用于判断 Spring 容器中是否已存在指定的 Bean，是自动配置中处理 Bean 依赖关系的核心手段。常与 `@ConditionalOnMissingBean` 配合，实现"用户优先"的配置策略。

## 深度解析

### 注解源码

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnBeanCondition.class)
public @interface ConditionalOnBean {
    Class<?>[] value() default {};
    Class<?>[] type() default {};
    String[] name() default {};

    // 搜索策略
    SearchStrategy search() default SearchStrategy.ALL;
}
```

### 搜索策略

| 策略 | 说明 |
|------|------|
| `CURRENT` | 只在当前配置类所在容器中搜索 |
| `ANCESTORS` | 向上搜索父容器 |
| `ALL` | 搜索所有容器（默认） |

### 核心用法

```java
@AutoConfiguration
// 只有容器中有 RedisConnectionFactory 才加载
@ConditionalOnBean(RedisConnectionFactory.class)
public class RedisAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public RedisTemplate redisTemplate(RedisConnectionFactory factory) {
        return new RedisTemplate(factory);
    }
}
```

### 典型场景

| 场景 | 条件注解 |
|------|----------|
| 只有配置了数据源才配置 JdbcTemplate | `@ConditionalOnBean(DataSource.class)` |
| 只有没有自定义 MyService 才提供默认实现 | `@ConditionalOnMissingBean(MyService.class)` |
| 只有启用了缓存才配置 CacheManager | `@ConditionalOnBean(CacheOperationSource.class)` |

### 与 @ConditionalOnClass 的区别

| 注解 | 检查时机 | 用途 |
|------|----------|------|
| `@ConditionalOnClass` | 加载配置类之前（classpath 检查） | 判断依赖 jar 是否存在 |
| `@ConditionalOnBean` | 注册 Bean 时（容器状态检查） | 判断其他 Bean 是否已注册 |

## 易错点/踩坑

- ❌ **在 @AutoConfiguration 类上使用 @ConditionalOnBean** — 自动配置类本身加载时，Bean 可能尚未注册完成
- ❌ **搜索不到父容器的 Bean** — 需设置 `search = SearchStrategy.ANCESTORS`
- ❌ **与 @AutoConfigureBefore/After 混用** — 条件注解比顺序更可靠

## 关联知识点

- [[ConditionalOnMissingBean条件判断]] — 相反条件
- [[ConditionalOnClass条件判断]] — classpath 检查
- [[自动配置条件注解概述]] — 条件注解体系
