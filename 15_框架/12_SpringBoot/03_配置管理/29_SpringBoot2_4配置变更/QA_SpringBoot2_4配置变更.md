# Spring Boot 2.4 配置变更 QA

## Q1：2.4 之前创建的配置文件还能用吗？

**A**：可以。2.4 完全向后兼容，旧配置方式仍然支持。

---

## Q2：bootstrap.yml 为什么被移除？

**A**：Spring Boot 本身不需要 bootstrap。bootstrap 是 Spring Cloud 的功能，之前随 Spring Boot 一起加载。2.4 把它分离出来，需要显式引入。

---

## Q3：spring.profiles.include 和 spring.profiles.add 有什么区别？

**A**：功能相同，`add` 是 2.4+ 的推荐写法：

```yaml
# 旧写法
spring:
  profiles:
    include: common

# 新写法
spring:
  profiles:
    add: common
```

---

## Q4：spring.config.import 支持什么格式？

**A**：支持：
- `classpath:config.yml`
- `file:./config.yml`
- `optional:file:./config.yml`（文件不存在不报错）
- `https://example.com/config.yml`

---

## Q5：2.4 的配置优先级有什么变化？

**A**：properties 和 YAML 同级加载，谁后加载谁覆盖。

---

## Q6：升级到 2.4 需要改配置吗？

**A**：大多数情况下不需要。2.4 完全向后兼容。
