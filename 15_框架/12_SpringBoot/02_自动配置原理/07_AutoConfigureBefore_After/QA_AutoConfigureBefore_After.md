# @AutoConfigureBefore / @AutoConfigureAfter

## Q1：@AutoConfigureBefore 和 @AutoConfigureAfter 区别是什么？

**A**：

| 注解 | 作用 | 场景 |
|------|------|------|
| `@AutoConfigureBefore(X.class)` | 本配置在 X **之前**加载 | 需要先准备好某些基础设施 |
| `@AutoConfigureAfter(X.class)` | 本配置在 X **之后**加载 | 需要使用 X 已配置的 Bean |

```java
// 示例：MyAutoConfiguration 需要 DataSource 已配置好
@AutoConfigureAfter(DataSourceAutoConfiguration.class)
public class MyAutoConfiguration {
    @Autowired
    private DataSource dataSource;  // 此时一定不为空
}
```

---

## Q2：@AutoConfigureBefore/After 和 @ConditionalOnBean 如何选择？

**A**：推荐使用 `@ConditionalOnBean`，更优雅：

| 方案 | 优点 | 缺点 |
|------|------|------|
| `@AutoConfigureAfter` | 显式声明顺序 | 脆弱，依赖实现细节 |
| `@ConditionalOnBean` | 自动判断，灵活 | 仅针对 Bean 存在性 |

```java
// 推荐：条件判断（更健壮）
@AutoConfiguration
@ConditionalOnBean(DataSource.class)   // DataSource 存在才加载
public class MyAutoConfiguration { }

// 不推荐：强制顺序依赖
@AutoConfigureAfter(DataSourceAutoConfiguration.class)
public class MyAutoConfiguration { }
```

---

## Q3：可以同时使用 @AutoConfigureBefore 和 @AutoConfigureAfter 吗？

**A**：可以，但通常没必要：

```java
// 同时使用示例
@AutoConfigureBefore(A.class)    // 在 A 之前
@AutoConfigureAfter(B.class)     // 在 B 之后
public class MyAutoConfiguration { }
// 结果：在 B 之后、A 之前
```

**注意**：避免循环依赖。

---

## Q4：@AutoConfigureBefore/After 对用户自定义 @Configuration 生效吗？

**A**：**不生效**。这些注解只对自动配置类有效：

```java
// MyConfig 是用户自定义的普通配置类
@Configuration
@AutoConfigureAfter(WebMvcAutoConfiguration.class)  // ❌ 无效
public class MyConfig { }
```

用户自定义配置类的加载顺序由 `@ComponentScan` 顺序决定，与 `@AutoConfigure*` 无关。

---

## Q5：多个配置相互依赖怎么处理？

**A**：分层处理：

```java
// 第一层：基础设施
public class AAutoConfiguration { }        // 无依赖，最先加载

// 第二层：使用 A
@AutoConfigureAfter(AAutoConfiguration.class)
public class BAutoConfiguration { }        // 在 A 之后

// 第三层：使用 A 和 B
@AutoConfigureAfter({AAutoConfiguration.class, BAutoConfiguration.class})
public class CAutoConfiguration { }       // 在 A、B 之后
```

如果出现循环依赖（A 在 B 之前，B 在 A 之前），Spring Boot 会启动失败并报错。

