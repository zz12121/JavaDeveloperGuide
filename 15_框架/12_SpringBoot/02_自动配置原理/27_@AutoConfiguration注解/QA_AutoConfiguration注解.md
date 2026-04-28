# @AutoConfiguration 注解

## Q1：@AutoConfiguration 和 @Configuration 完全一样吗？

**A**：基本一样，但 `@AutoConfiguration` 增强了以下功能：

| 特性 | @Configuration | @AutoConfiguration |
|------|---------------|-------------------|
| 基础功能 | ✅ 完全相同 | ✅ 继承 @Configuration |
| @Indexed 支持 | ❌ 不自动支持 | ✅ 自动支持 |
| 语义区分 | ❌ 无 | ✅ 明确标识自动配置类 |
| IDE 友好 | 混在一起 | 便于区分业务配置和自动配置 |

**内部实现**：`@AutoConfiguration` 本身标注了 `@Configuration`，所以行为完全相同。

---

## Q2：proxyBeanMethods 是什么意思？

**A**：`proxyBeanMethods = true`（默认）保证 @Bean 方法调用走 Spring 代理：

```java
@AutoConfiguration(proxyBeanMethods = true)  // 默认
public class MyConfig {
    @Bean
    public A a() { return new A(); }

    @Bean
    public B b() {
        return new B(a());  // 走代理，返回同一个 A 实例
    }
}
```

```java
@AutoConfiguration(proxyBeanMethods = false)
public class MyConfig {
    @Bean
    public A a() { return new A(); }

    @Bean
    public B b() {
        return new B(a());  // 不走代理，创建新的 A 实例
    }
}
```

---

## Q3：应该用 @Configuration 还是 @AutoConfiguration？

**A**：按场景选择：

| 场景 | 推荐 |
|------|------|
| 用户自定义配置类 | `@Configuration` |
| Spring Boot Starter 中的配置 | `@AutoConfiguration` |
| 第三方库的自动配置 | `@AutoConfiguration` |

---

## Q4：@AutoConfiguration 可以放在用户代码上吗？

**A**：可以，但**不建议**。@AutoConfiguration 的语义是"自动配置类"，放在用户代码上会让其他开发者困惑：

```java
// ✅ 推荐：用户代码用 @Configuration
@Configuration
public class UserConfig { }

// ✅ 推荐：自动配置用 @AutoConfiguration
@AutoConfiguration
public class RedisAutoConfiguration { }
```

---

## Q5：enforceUniqueMethods 是什么？

**A**：保证同一个配置类中不会有重复的 @Bean 方法名：

```java
@AutoConfiguration(enforceUniqueMethods = true)  // 默认
public class MyConfig {
    @Bean
    public A a() { return new A(); }

    @Bean
    public A a() { return new A(); }  // ❌ 编译错误
}
```

