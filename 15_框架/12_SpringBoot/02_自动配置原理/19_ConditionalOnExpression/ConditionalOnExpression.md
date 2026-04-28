# @ConditionalOnExpression 条件判断

## 先说结论

`@ConditionalOnExpression` 使用 SpEL（Spring Expression Language）表达式作为条件，是最灵活的条件注解，支持复杂的表达式逻辑，但过度使用会增加理解和维护成本。

## 深度解析

### 注解源码

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnExpressionCondition.class)
public @interface ConditionalOnExpression {
    String value();
}
```

### SpEL 表达式示例

```java
// 属性值为 true
@ConditionalOnExpression("${my.feature.enabled:true}")

// 属性值为 false
@ConditionalOnExpression("!${my.feature.enabled}")

// 多个属性组合
@ConditionalOnExpression(
    "${my.feature.enabled} and ${my.other.enabled}"
)

// 数值比较
@ConditionalOnExpression("${my.thread.pool-size:4} > 10")

// Bean 存在性
@ConditionalOnExpression("!${my.custom.skip:false}")
```

### 环境变量引用

```java
// 引用环境变量
@ConditionalOnExpression("${SERVER_PORT:8080} == 80")

// 引用系统属性
@ConditionalOnExpression("${user.dir} contains '/app'")
```

### 常见表达式模式

| 表达式 | 含义 |
|--------|------|
| `${prop:default}` | 带默认值的属性 |
| `${prop}` | 属性值 |
| `!${prop}` | 属性值取反 |
| `> 10`, `< 100` | 数值比较 |
| `contains 'xxx'` | 字符串包含 |
| `matches 'regex'` | 正则匹配 |

## 易错点/踩坑

- ❌ **表达式写错不会报错** — 只返回 false，不生效
- ❌ **过度使用** — 复杂的 SpEL 表达式难以维护，优先用其他条件注解
- ❌ **没有默认值** — 属性不存在会抛异常

## 关联知识点

- [[ConditionalOnProperty条件判断]] — 属性条件，更简洁
- [[自动配置条件注解概述]] — 条件注解体系
