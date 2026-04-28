# @ConditionalOnMissingBean 条件判断

## 先说结论

`@ConditionalOnMissingBean` 是 Spring Boot 自动配置中最重要的注解之一，用于判断容器中**不存在**指定 Bean 时才注册自动配置类提供的 Bean，实现"用户优先"原则——如果用户已经手动配置了某个 Bean，自动配置就不会覆盖它。

## 深度解析

### 注解源码

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnBeanCondition.class)
public @interface ConditionalOnMissingBean {
    Class<?>[] value() default {};
    Class<?>[] type() default {};
    String[] name() default {};

    // 忽略已排除的 Bean
    boolean ignored() default false;

    // 搜索策略
    SearchStrategy search() default SearchStrategy.ALL;
}
```

### 核心使用模式

```java
@AutoConfiguration
public class RedisAutoConfiguration {

    // 只有容器中没有 StringRedisTemplate 时才注册
    @Bean
    @Primary
    @ConditionalOnMissingBean(StringRedisTemplate.class)
    public StringRedisTemplate stringRedisTemplate(
            RedisConnectionFactory connectionFactory) {
        return new StringRedisTemplate(connectionFactory);
    }
}
```

### 执行时序

```
自动配置加载阶段（DeferredImportSelector 后置处理）
    │
    ▼
遍历自动配置类的 @Bean 方法
    │
    ▼
遇到 @ConditionalOnMissingBean
    │
    ▼
检查容器中是否已存在该 Bean
    │
    ├── 存在 → 跳过（不注册）
    └── 不存在 → 注册该 Bean
```

### ignored 属性

```java
// 忽略某些已标记为"可忽略"的 Bean
@ConditionalOnMissingBean(value = DataSource.class, ignored = {HikariDataSource.class})
// 即使 HikariDataSource 存在，也视为"不存在"
```

## 易错点/踩坑

- ❌ **类级别条件判断可能失效** — `@ConditionalOnMissingBean` 在类级别时，Bean 注册顺序可能导致误判
- ❌ **方法参数注入优于条件判断** — 更好的设计是通过参数注入而非条件判断
- ❌ **父子容器的 Bean** — 父容器中的 Bean 也会被检测到

## 关联知识点

- [[ConditionalOnBean条件判断]] — 相反条件
- [[自动配置报告详解]] — 查看 Bean 加载情况
