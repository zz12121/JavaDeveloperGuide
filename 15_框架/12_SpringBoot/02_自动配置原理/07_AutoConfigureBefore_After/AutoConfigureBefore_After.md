# @AutoConfigureBefore / @AutoConfigureAfter

## 先说结论

`@AutoConfigureBefore` 和 `@AutoConfigureAfter` 用于控制自动配置类之间的加载顺序，确保依赖关系正确的配置优先或滞后加载。只对自动配置类有效，对普通 `@Configuration` 类无效。

## 深度解析

### 注解源码

```java
@AutoConfigureAfter(DataSourceAutoConfiguration.class)
@AutoConfigureBefore(WebMvcAutoConfiguration.class)
public @interface AutoConfigureOrder {
    int value() default Ordered.LOWEST_PRECEDENCE;
}
```

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.ANNOTATION_TYPE})
@Documented
@AutoConfigureOrder(Ordered.LOWEST_PRECEDENCE - 1)
public @interface AutoConfigureBefore {
    Class<?>[] value() default {};
}
```

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.ANNOTATION_TYPE})
@Documented
@AutoConfigureOrder(Ordered.LOWEST_PRECEDENCE - 1)
public @interface AutoConfigureAfter {
    Class<?>[] value() default {};
}
```

### 执行顺序规则

```
@AutoConfigureBefore(A.class)  →  本配置在 A 之前加载
@AutoConfigureAfter(A.class)   →  本配置在 A 之后加载

默认顺序：LOWEST_PRECEDENCE = Integer.MAX_VALUE
```

### 常见使用场景

| 场景 | 用法 | 示例 |
|------|------|------|
| 覆盖默认配置 | 在之后加载 | `@AutoConfigureAfter(WebMvcAutoConfiguration.class)` |
| 提供前置依赖 | 在之前加载 | `@AutoConfigureBefore(DataSourceAutoConfiguration.class)` |
| 多个配置顺序 | 组合使用 | `@AutoConfigureAfter({A.class, B.class})` |

### 实际案例

```java
// MyBatis 自动配置必须在 DataSource 之后
@AutoConfigureAfter(DataSourceAutoConfiguration.class)
public class MybatisAutoConfiguration {
    // 此时 DataSource Bean 已经存在
}
```

```java
// 自定义 WebMvc 配置覆盖默认
@AutoConfigureAfter(WebMvcAutoConfiguration.class)
@Configuration
public class MyWebMvcAutoConfiguration {
    // 此时 WebMvcAutoConfiguration 的 Bean 已注册
}
```

## 易错点/踩坑

- ❌ **对普通 @Configuration 类无效** — 这些注解只影响自动配置类
- ❌ **循环依赖** — A 在 B 之前，B 在 A 之前会导致启动失败
- ❌ **过度依赖顺序** — 应该用 `@ConditionalOnBean` 更优雅地处理依赖

## 关联知识点

- [[AutoConfigureOrder排序]] — 显式指定顺序值
- [[自动配置条件注解概述]] — 条件注解体系
- [[@ConditionalOnBean条件判断]] — 通过条件处理依赖关系
