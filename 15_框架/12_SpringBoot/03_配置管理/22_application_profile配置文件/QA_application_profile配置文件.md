# application-{profile} 配置文件 QA

## Q1：profile 文件和公共文件冲突时谁生效？

**A**：profile 文件优先，同名属性会覆盖公共配置。

加载顺序：
1. `application.yml`
2. `application-{profile}.yml`

---

## Q2：properties 和 YAML 可以混用吗？

**A**：可以，Spring Boot 会合并两者的配置。但不推荐，混用会增加维护难度。

---

## Q3：profile 文件中可以引用环境变量吗？

**A**：可以：

```yaml
spring:
  datasource:
    password: ${DB_PASSWORD}
```

---

## Q4：profile 文件可以有多个吗？

**A**：可以：

```yaml
spring:
  profiles:
    active: dev,common
```

会同时加载 `application-dev.yml` 和 `application-common.yml`。

---

## Q5：profile 文件支持 YAML 多文档吗？

**A**：支持：

```yaml
# application-dev.yml
server:
  port: 8080
---
spring:
  config:
    activate:
      on-profile: dev
server:
  port: 8081
```

---

## Q6：不同 profile 的配置文件放不同目录可以吗？

**A**：可以：

```
src/main/resources/
├── application.yml
└── config/
    ├── application-dev.yml
    └── application-prod.yml
```
