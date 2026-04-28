# @ConditionalOnProperty 条件判断

## Q1：havingValue 和 matchIfMissing 有什么区别？

**A**：

| 属性 | 作用 | 默认值 |
|------|------|--------|
| `havingValue` | 属性必须匹配的值 | "" (空字符串) |
| `matchIfMissing` | 属性不存在时的行为 | `false` |

```java
// 示例 1：必须有 spring.my.feature=true 才启用
@ConditionalOnProperty(name = "spring.my.feature", havingValue = "true")

// 示例 2：属性不存在时默认启用
@ConditionalOnProperty(name = "spring.my.feature", havingValue = "true", matchIfMissing = true)

// 示例 3：属性为 false 时启用（反向逻辑）
@ConditionalOnProperty(name = "spring.my.feature", havingValue = "false")
```

---

## Q2：matchIfMissing = true 有什么风险？

**A**：

```java
// ⚠️ 不推荐：即使配置文件中没有 spring.cache.enabled，也会启用
@ConditionalOnProperty(
    name = "spring.cache.enabled",
    havingValue = "true",
    matchIfMissing = true  // 属性不存在时视为 true
)
public class CacheAutoConfiguration { }
```

**风险**：
- 用户不知道这个功能被默认启用了
- 可能加载不必要的组件，影响启动速度
- 不符合"最小惊讶原则"

**建议**：除非是明确的默认值，否则不要使用 `matchIfMissing = true`。

---

## Q3：name 属性支持多个属性名吗？

**A**：支持数组：

```java
// 满足任一属性即可
@ConditionalOnProperty(
    name = {"spring.redis.host", "spring.data.redis.host"},
    havingValue = "true",
    matchIfMissing = true
)
```

但通常更推荐单一属性：

```java
// 推荐：单一明确的属性
@ConditionalOnProperty(name = "spring.cache.type", havingValue = "redis")
```

---

## Q4：relaxed 属性是什么意思？

**A**：`relaxed = true`（默认）启用宽松匹配：

```java
@ConditionalOnProperty(name = "spring.mybatis.configuration", havingValue = "true")
// 支持匹配：
// - spring.mybatis.configuration
// - spring.mybatis.config  (下划线转驼峰)
// - SPRING_MYBATIS_CONFIGURATION  (全大写下划线)

// relaxed = false 则必须精确匹配
@ConditionalOnProperty(name = "spring.mybatis.configuration", havingValue = "true", relaxed = false)
```

---

## Q5：如何在配置文件中禁用某个自动配置？

**A**：三种方式：

```properties
# 方式一：直接禁用
spring.autoconfigure.exclude=\
  org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration

# 方式二：配置开关 + 条件注解配合
spring.my.feature.enabled=false
```

```java
// 自动配置中的条件
@ConditionalOnProperty(name = "spring.my.feature.enabled", havingValue = "true", matchIfMissing = false)
public class MyFeatureAutoConfiguration { }
```

