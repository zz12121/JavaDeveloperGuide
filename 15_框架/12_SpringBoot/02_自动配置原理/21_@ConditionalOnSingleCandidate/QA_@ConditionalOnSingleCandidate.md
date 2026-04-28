# @ConditionalOnSingleCandidate

## Q1：@ConditionalOnSingleCandidate 和 @ConditionalOnBean 有什么区别？

**A**：关键区别在于**数量要求**：

```java
// @ConditionalOnBean：只要存在至少1个即可
@ConditionalOnBean(DataSource.class)

// @ConditionalOnSingleCandidate：必须恰好1个
@ConditionalOnSingleCandidate(DataSource.class)
```

**使用场景**：
- `@ConditionalOnBean`：只需要某种能力存在
- `@ConditionalOnSingleCandidate`：需要唯一处理（如统一包装、代理）

---

## Q2：什么情况下算"单候选"？

**A**：需要同时满足：
1. 存在该类型的 Bean
2. 只有 **1 个**该类型的 Bean
3. 该 Bean 是唯一候选（@Primary 或类型唯一）

```java
// 示例：单候选的情况 ✅
@Bean
public DataSource ds1() { ... }  // 唯一 DataSource

// 示例：多候选的情况 ❌
@Bean
public DataSource ds1() { ... }
@Bean
public DataSource ds2() { ... }  // 2个 DataSource，不满足
```

---

## Q3：实际应用场景？

**A**：常见于需要统一处理的场景：

```java
// Spring Session 自动配置
@ConditionalOnSingleCandidate(UnderlyingSessionRepository.class)
public class SessionRepositoryWrapper<S extends Session> {
    // 只有唯一候选时才包装
}
```

---

## Q4：如果 Bean 有 @Primary 标记呢？

**A**：@Primary 只是解决优先级，不改变数量：

```java
@Bean @Primary
public DataSource ds1() { ... }

@Bean
public DataSource ds2() { ... }  // 2个，仍然不是单候选
```

**单候选 ≠ 主 Bean**，而是**唯一 Bean**。
