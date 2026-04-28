# @ConditionalOnProperty 条件判断

## 先说结论

`@ConditionalOnProperty` 通过检查 Spring Environment 中的配置属性值来决定是否加载自动配置，是实现"开关控制"的核心注解，支持灵活的功能启用/禁用机制。

## 深度解析

### 注解源码

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
@Documented
@Conditional(OnPropertyCondition.class)
public @interface ConditionalOnProperty {
    // 属性名
    String[] name() default {};

    // 期望的值
    String havingValue() default "";

    // 属性不存在时的行为
    boolean matchIfMissing() default false;

    // 是否反转结果
    boolean relaxed() default true;
}
```

### 配置示例

```java
// 只有 spring.cache.enabled=true 时才启用缓存
@AutoConfiguration
@ConditionalOnProperty(name = "spring.cache.enabled", havingValue = "true")
public class CacheAutoConfiguration { }

// 只有设置了 my.feature.enabled 才启用（不设置则不启用）
@ConditionalOnProperty(name = "my.feature.enabled", havingValue = "true")

// 即使属性不存在也启用（不推荐）
@ConditionalOnProperty(name = "my.feature.disabled", havingValue = "false", matchIfMissing = true)
```

### 属性名匹配规则

```java
// 完整属性名
@ConditionalOnProperty(name = "spring.cache.enabled")

// 松弛匹配（relaxed=true，默认）
@ConditionalOnProperty(name = "spring.cache.enabled")
// 等效于检查：
// - spring.cache.enabled
// - spring.cache.enable    (下划线/驼峰互换)
// - SPRING_CACHE_ENABLED   (大写)

@ConditionalOnProperty(name = "spring.cache.enabled", relaxed = false)
// 必须精确匹配
```

### 多个属性组合

```java
@ConditionalOnProperty(name = "spring.cache.type", havingValue = "redis")
@ConditionalOnProperty(name = "spring.redis.host", havingValue = "localhost")
public class RedisCacheAutoConfiguration { }
// 两个条件都满足才加载
```

## 易错点/踩坑

- ❌ **`havingValue` 不设置** — 默认空字符串，意味着只有属性值也是空字符串才匹配
- ❌ **`matchIfMissing=true` 过于宽松** — 容易导致意外启用功能
- ❌ **属性名写错** — 不会报错，只会条件不满足

## 关联知识点

- [[属性名Relaxed绑定]] — 属性名匹配规则
- [[application_properties]] — 配置文件格式
