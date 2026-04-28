# 自定义条件注解开发

## Q1：什么情况下需要自定义条件注解？

**A**：当内置的 `@ConditionalOn*` 无法满足需求时：

| 场景 | 内置注解 | 自定义需求 |
|------|----------|------------|
| 特定的配置值组合 | `@ConditionalOnProperty` | 检查多个属性 |
| 特定环境检测 | `@ConditionalOnWebApplication` | 检测特定运行环境 |
| Bean 数量判断 | `@ConditionalOnBean` | 检查 Bean 个数 |
| 类加载器判断 | 无直接支持 | 自定义类加载器检查 |

---

## Q2：@ConditionalOnExpression 和自定义条件注解如何选择？

**A**：

| 场景 | 推荐 |
|------|------|
| 单一属性检查 | `@ConditionalOnProperty` |
| 多属性组合 | `@ConditionalOnProperty` 组合 |
| 复杂 SpEL 表达式 | `@ConditionalOnExpression` |
| 非属性类逻辑 | 自定义条件注解 |

```java
// 复杂条件：用 @ConditionalOnExpression
@ConditionalOnExpression(
    "${my.config.type} == 'advanced' and ${my.config.level} > 5"
)

// 同样的条件用自定义注解
@ConditionalOnMyConfig(type = "advanced", level = 5)
```

自定义条件注解更易读、可维护。

---

## Q3：自定义条件注解可以带属性吗？

**A**：可以：

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Conditional(OnMyCustomCondition.class)
public @interface ConditionalOnMyCustomCondition {
    String name();
    int minValue() default 0;
}

public class OnMyCustomCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        Map<String, Object> attrs = metadata
            .getAnnotationAttributes(ConditionalOnMyCustomCondition.class.getName());

        String name = (String) attrs.get("name");
        int minValue = (Integer) attrs.get("minValue");

        String configValue = context.getEnvironment().getProperty(name);
        return configValue != null && configValue.length() > minValue;
    }
}
```

---

## Q4：自定义条件注解在哪个阶段执行？

**A**：与 `@ConditionalOn*` 相同，在配置类处理阶段执行：

```
BeanDefinitionRegistryPostProcessor
    │
    ▼
ConfigurationClassPostProcessor
    │
    ▼
遍历所有 @Configuration 类
    │
    ▼
检查 @Conditional* 注解
    │
    ├── 内置条件 → OnXxxCondition.matches()
    └── 自定义条件 → OnMyCondition.matches()
    │
    ▼
通过 → 注册 BeanDefinition
```

---

## Q5：自定义条件注解有什么性能考虑？

**A**：条件检查在启动时执行，应注意：

| 问题 | 影响 | 建议 |
|------|------|------|
| 反射调用 | 开销小 | 可忽略 |
| 类加载 | 较大 | 避免不必要的 Class.forName |
| 文件读取 | 中等 | 缓存结果 |
| 网络请求 | 大 | **禁止**，启动时不应有网络请求 |

