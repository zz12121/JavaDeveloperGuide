# @Profile 激活环境 QA

## Q1：@Profile 和 spring.profiles.active 有什么关系？

**A**：`spring.profiles.active` 是激活方式，`@Profile` 是使用方式：
- `spring.profiles.active=dev` 激活 dev 环境
- `@Profile("dev")` 的 Bean 在 dev 环境才生效

---

## Q2：@Profile 可以标注在接口上吗？

**A**：可以，用于接口/抽象类的不同实现：

```java
@Profile("dev")
public interface DataSource { }

public class DevDataSource implements DataSource { }

public class ProdDataSource implements DataSource { }
```

---

## Q3：默认 profile 是什么？

**A**：默认 profile 是 `default`。可以通过以下方式设置：

```yaml
spring:
  profiles:
    active: dev
```

---

## Q4：@Profile("!prod") 是什么意思？

**A**：表示"非 prod"。即除 prod 以外的所有环境都生效。

---

## Q5：可以同时激活多个 profile 吗？

**A**：可以：

```bash
--spring.profiles.active=dev,common
```

---

## Q6：profile 切换时 Bean 会重新创建吗？

**A**：会的。Spring Boot 重启应用，新的 profile 生效，创建新的 Bean。
