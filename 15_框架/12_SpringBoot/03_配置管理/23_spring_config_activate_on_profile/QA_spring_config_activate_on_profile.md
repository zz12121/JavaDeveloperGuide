# spring.config.activate.on-profile QA

## Q1：spring.config.activate.on-profile 和传统 profile 哪个先加载？

**A**：传统 profile 文件（`application-{profile}.yml`）先加载，然后是 `on-profile` 条件激活的文档。

---

## Q2：on-profile 可以用表达式吗？

**A**：可以：

```yaml
on-profile: "dev & !integration"  # dev 且非 integration
on-profile: "[dev, test]"        # dev 或 test
on-profile: "!prod"              # 非 prod
```

---

## Q3：哪些版本支持这个语法？

**A**：Spring Boot **2.4.0+** 支持。

---

## Q4：可以同时使用传统 profile 和 on-profile 吗？

**A**：可以，但不推荐：

```yaml
# application-dev.yml（传统方式）
server:
  port: 8081
---
# 同文件中的 on-profile
spring:
  config:
    activate:
      on-profile: dev2
server:
  port: 8082
```

---

## Q5：on-profile 激活失败会怎样？

**A**：该文档不会被加载，不报错。

---

## Q6：为什么推荐用 on-profile 而不是传统 profile 文件？

**A**：
- 相关配置集中管理
- 减少文件数量
- 更清晰的依赖关系
