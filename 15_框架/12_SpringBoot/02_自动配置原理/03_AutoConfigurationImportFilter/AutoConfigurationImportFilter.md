# AutoConfigurationImportFilter

## 先说结论

`AutoConfigurationImportFilter` 是自动配置的第一道过滤器，在条件注解判断之前快速排除那些"基本条件不满足"的自动配置类（比如 classpath 中缺少关键依赖），减少后续条件判断的开销。

## 深度解析

### 过滤器接口

```java
@FunctionalInterface
public interface AutoConfigurationImportFilter {

    // 返回 boolean[]，true 表示该配置类通过过滤
    boolean[] match(String autoConfigurationClass,
                    AutoConfigurationMetadata autoConfigurationMetadata);
}
```

### Spring Boot 内置过滤器

Spring Boot 内置三个过滤器（按顺序执行）：

| 过滤器 | 作用 |
|--------|------|
| `OnClassCondition` | 检查 `@ConditionalOnClass` 的类是否存在于 classpath |
| `OnBeanCondition` | 检查 `@ConditionalOnBean` / `@ConditionalOnMissingBean` |
| `OnWebApplicationCondition` | 检查 `@ConditionalOnWebApplication` |

### match 方法执行流程

```java
// AutoConfigurationImportSelector 内部调用
List<AutoConfigurationImportFilter> filters = getConfigurationClassFilter();
// filters = [OnClassCondition, OnBeanCondition, OnWebApplicationCondition]

boolean[] matchResult = new boolean[configurations.size()];
for (AutoConfigurationImportFilter filter : filters) {
    for (int i = 0; i < configurations.size(); i++) {
        // 逐个过滤器判断
        // 如果之前已经被过滤为 false，后续不再重复判断
    }
}
```

### 与条件注解的关系

```
AutoConfigurationImportFilter（第一道过滤）
    │
    ├── OnClassCondition  → 快速排除 classpath 不满足的配置
    ├── OnBeanCondition  → 快速排除 Bean 依赖不满足的配置
    └── OnWebApplicationCondition → 快速排除 Web 环境不满足的配置
    │
    ▼
@Conditional* 条件注解（精细判断）
    │
    └── 逐个 @ConditionalOnXxx 验证
```

**为什么需要两层过滤？**

- `AutoConfigurationImportFilter`：在配置类加载 **之前** 执行，快速排除，节省 BeanDefinition 解析开销
- `@Conditional*`：在配置类加载 **过程中** 执行，作为注解存在，更灵活

## 易错点/踩坑

- ❌ **自定义过滤器不能替代条件注解** — 过滤器只做快速预判，详细条件逻辑仍在条件注解中
- ❌ **过滤器的 match 结果会被缓存** — 同一个配置类的过滤结果在整个启动过程中不会重新计算

## 关联知识点

- [[AutoConfigurationImportSelector]] — 过滤器的调用方
- [[ConditionalOnClass条件判断]] — OnClassCondition 检查的条件
