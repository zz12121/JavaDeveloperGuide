# @ConditionalOnNotWebApplication 条件判断

## 先说结论

`@ConditionalOnNotWebApplication` 与 `@ConditionalOnWebApplication` 相反，只有当应用**不是** Web 环境时才加载配置，常用于提供非 Web 应用的默认配置。

## 深度解析

### 与 @ConditionalOnWebApplication 对比

| 注解 | 条件 |
|------|------|
| `@ConditionalOnWebApplication` | 是 Web 应用才加载 |
| `@ConditionalOnNotWebApplication` | **不是** Web 应用才加载 |

### 使用场景

```java
// 只有非 Web 环境（如批处理、命令行工具）才加载
@AutoConfiguration
@ConditionalOnNotWebApplication
public class BatchAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public BatchJobExecutor batchJobExecutor() {
        return new SimpleBatchJobExecutor();
    }
}
```

### 典型配置

```java
// 非 Web 环境下的数据源（不使用连接池）
@AutoConfiguration
@ConditionalOnNotWebApplication
@ConditionalOnClass(DataSource.class)
public class NonWebDataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public DataSource dataSource() {
        return new SingleConnectionDataSource();
    }
}
```

## 关联知识点

- [[ConditionalOnWebApplication条件判断]] — 相反条件
- [[自动配置条件注解概述]] — 条件注解体系
