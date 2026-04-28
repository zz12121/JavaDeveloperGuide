# @ConditionalOnJndi

## 先说结论

- `@ConditionalOnJndi` 是条件注解之一，**只有当 JNDI 资源存在时才注册 Bean**
- 用于检测特定 JNDI 名称是否可用，常用于需要 JNDI 资源的自动配置

## 深度解析

### 核心概念

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Conditional(OnJndiCondition.class)
public @interface ConditionalOnJndi {
    // JNDI 资源名称
    String[] value() default {};
    
    // 资源类型
    Class<?>[] type() default {};
}
```

### 使用场景

| 场景 | 示例 |
|------|------|
| 检测 DataSource | `@ConditionalOnJndi("java:comp/env/jdbc/myDS")` |
| 检测 JMS | `@ConditionalOnJndi("java:comp/env/jms/myCF")` |
| 验证多个 | `@ConditionalOnJndi(value = {"java:comp/env/jdbc/ds", "java:comp/env/jms/cf"})` |

### 源码解析

```
OnJndiCondition
  ├── getConfiguration() → 获取 @ConditionalOnJndi 注解
  ├── getJndiNames() → 解析 JNDI 名称
  └── matches() → 执行 JNDI 查找
      └── InitialContext.lookup(name) → 尝试查找
```

## 易错点/踩坑

- ❌ **依赖 JNDI 容器环境**：在独立应用中 JNDI 通常不可用
- ❌ **忽略类型匹配**：即使 JNDI 存在，类型不匹配也会失败
- ❌ **JNDI 查找耗时**：每次条件判断都会执行 lookup，可能影响启动时间

## 代码示例

```java
@Configuration
@ConditionalOnJndi("java:comp/env/jdbc/myDS")
public class JndiDataSourceConfiguration {
    @Bean
    public DataSource jndiDataSource() {
        // 从 JNDI 获取 DataSource
        return (DataSource) new InitialContext().lookup("java:comp/env/jdbc/myDS");
    }
}
```

## 关联知识点

- `@ConditionalOnBean` — 存在 Bean 时生效
- `@ConditionalOnClass` — 存在类时生效
- `@ConditionalOnMissingBean` — 不存在 Bean 时生效
