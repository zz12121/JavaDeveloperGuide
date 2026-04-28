# @ConfigurationProperties vs @Value QA

## Q1：可以同时使用两者吗？

**A**：可以，但通常没必要。特殊场景除外：

```java
@ConfigurationProperties(prefix = "app")
public class AppProperties {
    @Value("${app.computed:#{2*3}}")
    private int computed;  // 配置属性 + SpEL
}
```

---

## Q2：性能上有差异吗？

**A**：几乎无差异。两者都在应用启动时绑定一次，都是单例。

---

## Q3：哪个更适合配置第三方库？

**A**：**@ConfigurationProperties**。Spring Boot 自动配置都用这种方式：

```java
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration { }
```

---

## Q4：可以用 @ConfigurationProperties 读取 List/Map 吗？

**A**：可以：

```java
@ConfigurationProperties(prefix = "app")
public class AppProperties {
    private List<String> servers;
    private Map<String, Integer> ports;
}
```

```yaml
app:
  servers:
    - server1
    - server2
  ports:
    http: 80
    https: 443
```

---

## Q5：@ConfigurationProperties 支持默认值吗？

**A**：支持，在属性定义处设置：

```java
public class AppProperties {
    private int timeout = 5000;  // 默认值
    private String name = "default";
}
```

---

## Q6：@Value 的 `:default` 语法能在 @ConfigurationProperties 用吗？

**A**：不能。@ConfigurationProperties 的默认值在 Java 代码中定义：

```java
@Value("${app.timeout:5000}")  // ✅ @Value 支持
```

```java
// @ConfigurationProperties
private int timeout = 5000;  // ✅ 定义默认值
```
