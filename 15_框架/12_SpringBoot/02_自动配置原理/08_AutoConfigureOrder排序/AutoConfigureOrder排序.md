# @AutoConfigureOrder 排序

## 先说结论

`@AutoConfigureOrder` 允许显式指定自动配置类的加载顺序，数值越小越先加载。相比 `@AutoConfigureBefore/After` 的"相对顺序"，`@AutoConfigureOrder` 更适合批量调整优先级。

## 深度解析

### 注解源码

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.ANNOTATION_TYPE})
@Documented
@AutoConfigureOrder(Ordered.LOWEST_PRECEDENCE)
public @interface AutoConfigureOrder {
    int value() default Ordered.LOWEST_PRECEDENCE;  // 默认 Integer.MAX_VALUE
}
```

### 顺序值参考

| 常量 | 值 | 说明 |
|------|-----|------|
| `Ordered.HIGHEST_PRECEDENCE` | `Integer.MIN_VALUE` | 最高优先级 |
| `Ordered.HIGHEST_PRECEDENCE + n` | `Integer.MIN_VALUE + n` | 较高优先级 |
| `Ordered.LOWEST_PRECEDENCE` | `Integer.MAX_VALUE` | 最低优先级（默认） |
| `Ordered.LOWEST_PRECEDENCE - n` | `Integer.MAX_VALUE - n` | 较低优先级 |

### 使用示例

```java
// 最高优先级（最先加载）
@AutoConfiguration
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
public class FirstAutoConfiguration { }

// 较低优先级（最后加载，可覆盖其他配置）
@AutoConfiguration
@AutoConfigureOrder(Ordered.LOWEST_PRECEDENCE - 10)
public class OverrideAutoConfiguration { }
```

### 与 @AutoConfigureBefore/After 的优先级

- `@AutoConfigureBefore/After` 的默认顺序是 `LOWEST_PRECEDENCE - 1`
- 如果同时使用，**显式的 `@AutoConfigureOrder` 优先级更高**

```java
// 显式顺序优先于隐式顺序
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
@AutoConfigureAfter(SomeConfig.class)  // 被 @AutoConfigureOrder 覆盖
public class MyConfig { }
```

## 易错点/踩坑

- ❌ **数值写错导致意外顺序** — 如 `-1` 会被视为 `HIGHEST_PRECEDENCE - 1`，不是较低优先级
- ❌ **数值差异过大** — 建议使用 `Ordered` 常量而非魔法数字

## 关联知识点

- [[AutoConfigureBefore_After]] — 相对顺序控制
- [[自动配置条件注解概述]] — 注解体系
