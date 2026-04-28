# @PropertySources 多文件加载 QA

## Q1：@PropertySources 和多个 @PropertySource 哪个更好？

**A**：Java 8+ 支持可重复注解，两种写法等价：

```java
// @PropertySources（显式容器）
@PropertySources({
    @PropertySource("classpath:db.properties"),
    @PropertySource("classpath:app.properties")
})

// 可重复注解（更简洁）
@PropertySource("classpath:db.properties")
@PropertySource("classpath:app.properties")
```

---

## Q2：Spring Boot 2.4 之后有什么变化？

**A**：推荐用 `spring.config.import` 替代：

```yaml
spring:
  config:
    import: 
      - optional:file:./db.properties
      - optional:file:./redis.properties
```

这是 2.4+ 的标准方式。

---

## Q3：加载失败会影响启动吗？

**A**：
- 默认会报错启动失败
- 加 `ignoreResourceNotFound = true` 可忽略不存在的文件

---

## Q4：可以加载不同位置的配置文件吗？

**A**：可以：

```java
@PropertySources({
    @PropertySource("classpath:db.properties"),
    @PropertySource("file:./config/local.properties"),
    @PropertySource("file:/etc/myapp/config.properties")
})
```

---

## Q5：@PropertySource 加载的配置文件能用 @ConfigurationProperties 绑定吗？

**A**：可以：

```java
@PropertySource("classpath:custom.properties")

@ConfigurationProperties(prefix = "custom")
public class CustomProperties { }
```

---

## Q6：多文件加载顺序是怎样的？

**A**：按注解顺序依次加载，后面的覆盖前面的同名属性。
