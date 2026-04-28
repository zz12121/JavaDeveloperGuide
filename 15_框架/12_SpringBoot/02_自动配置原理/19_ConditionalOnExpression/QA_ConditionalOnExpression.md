# @ConditionalOnExpression 条件判断

## Q1：@ConditionalOnExpression 和 @ConditionalOnProperty 哪个更好？

**A**：优先使用 `@ConditionalOnProperty`：

| 对比 | @ConditionalOnProperty | @ConditionalOnExpression |
|------|----------------------|-------------------------|
| 可读性 | 好 | 差（SpEL 语法） |
| 功能 | 有限 | 强大（支持复杂表达式） |
| IDE 支持 | 属性名自动补全 | 无 |
| 性能 | 快 | 需解析表达式，稍慢 |

**何时用 @ConditionalOnExpression**：
- 需要多个属性组合判断
- 需要数值比较
- 需要正则匹配

---

## Q2：如何正确使用带默认值的表达式？

**A**：

```java
// ❌ 属性不存在会报错
@ConditionalOnExpression("${my.enabled}")

// ✅ 带默认值
@ConditionalOnExpression("${my.enabled:false}")

// ✅ 带默认值 + 布尔表达式
@ConditionalOnExpression("${my.enabled:true} and ${my.other:true}")

// ✅ 使用字符串属性
@ConditionalOnExpression("'${spring.profiles.active}' == 'prod'")
```

---

## Q3：可以引用 Bean 吗？

**A**：可以，但不推荐：

```java
// 引用 Bean 的属性
@ConditionalOnExpression("@myService.enabled")
public class MyAutoConfiguration { }

// 引用其他 Bean 是否存在
@ConditionalOnExpression("@myService != null")
public class DependentAutoConfiguration { }
```

**问题**：`@ConditionalOnBean` 更明确、更安全。

---

## Q4：如何调试表达式不生效？

**A**：

1. **简化表达式**：先用一个简单条件测试
2. **查看日志**：开启 debug 看条件判断过程
3. **单元测试**：单独测试 SpEL 表达式

```java
// 测试表达式
ExpressionParser parser = new SpelExpressionParser();
StandardEvaluationContext context = new StandardEvaluationContext();
context.setVariable("enabled", true);

Boolean result = parser.parseExpression("#enabled").getValue(context, Boolean.class);
System.out.println(result);  // true
```

