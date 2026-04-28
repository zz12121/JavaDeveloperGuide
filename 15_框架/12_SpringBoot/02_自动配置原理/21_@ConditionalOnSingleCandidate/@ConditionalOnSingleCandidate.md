# @ConditionalOnSingleCandidate

## 先说结论

- `@ConditionalOnSingleCandidate` 检查 IOC 容器中**是否存在指定类型且只有一个候选 Bean**
- 比 `@ConditionalOnBean` 更严格：不仅要求存在，还要求**唯一**

## 深度解析

### 核心概念

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Conditional(OnSingleCandidateCondition.class)
public @interface ConditionalOnSingleCandidate {
    
    // 要检测的 Bean 类型
    Class<?> value() default Object.class;
    
    // Bean 名称（可选）
    String name() default "";
}
```

### 条件判断逻辑

```
OnSingleCandidateCondition
  ├── 查找指定类型的所有 Bean
  ├── 检查 Bean 数量 == 1
  └── 检查该 Bean 是否被@Primary标记或唯一
```

### 与其他条件注解对比

| 注解 | 条件 | 场景 |
|------|------|------|
| `@ConditionalOnBean` | 存在 ≥1 个 | 存在即可 |
| `@ConditionalOnSingleCandidate` | 存在 = 1 个 | 需要唯一 |
| `@ConditionalOnMissingBean` | 不存在 | 需要自定义 |

## 易错点/踩坑

- ❌ **误用在有多个候选的场景**：如果配置了多个 DataSource，会跳过
- ❌ **忽略泛型信息**：`List<Object>` 和 `List<String>` 是不同类型
- ❌ **与 @Primary 混淆**：单候选不一定有 @Primary

## 代码示例

```java
@Configuration
@ConditionalOnSingleCandidate(DataSource.class)
public class SingleDataSourceConfiguration {
    
    @Bean
    @Primary
    public DataSource unifiedDataSource(DataSource dataSource) {
        // 只有当 DataSource 唯一时才会执行
        return new WrappedDataSource(dataSource);
    }
}
```

## 关联知识点

- `@ConditionalOnBean` — 存在 Bean 时生效
- `@ConditionalOnMissingBean` — 不存在 Bean 时生效
- `@Primary` — 主 Bean 标记
