# @ConditionalOnMissingClass 条件判断

## 先说结论

`@ConditionalOnMissingClass` 是 `@ConditionalOnClass` 的反向条件，当 classpath 中**不存在**指定类时配置才生效。常用于提供默认实现，避免与用户的手动配置冲突。

## 深度解析

### 注解源码

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnClassCondition.class)  // 与 @ConditionalOnClass 共用同一个 Condition
public @interface ConditionalOnMissingClass {
    String[] value() default {};
}
```

### 典型使用场景

| 场景 | 用法 |
|------|------|
| 只有没有 MyService 才提供默认实现 | `@ConditionalOnMissingClass("com.example.MyServiceImpl")` |
| 只有 classpath 没有 JdbcTemplate 才提供替代 | `@ConditionalOnMissingClass("org.springframework.jdbc.core.JdbcTemplate")` |

### 与 @ConditionalOnMissingBean 对比

| 注解 | 检查内容 | 检查时机 |
|------|----------|----------|
| `@ConditionalOnMissingClass` | classpath 中是否有类 | 配置类加载前 |
| `@ConditionalOnMissingBean` | 容器中是否有 Bean | Bean 注册时 |

### 实际代码示例

```java
// 只有 classpath 没有 FastJson 时，才使用 Jackson
@AutoConfiguration
@ConditionalOnClass(ObjectMapper.class)
@ConditionalOnMissingClass("com.alibaba.fastjson.JSON")
public class JacksonAutoConfiguration {

    @Bean
    @Primary
    @ConditionalOnMissingBean
    public ObjectMapper objectMapper() {
        return new Jackson2ObjectMapperBuilder().createXmlMapper(false).build();
    }
}
```

## 易错点/踩坑

- ❌ **只能使用 String 类名** — `@ConditionalOnMissingClass` 没有 `Class<?>` 类型的属性
- ❌ **只检查类存在性，不检查重复** — 同一个 jar 多次引入不会影响判断

## 关联知识点

- [[ConditionalOnClass条件判断]] — 相反条件
- [[ConditionalOnMissingBean条件判断]] — Bean 级别相反条件
