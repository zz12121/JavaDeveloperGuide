# spring.config.activate.on-profile（2.4+）

## 先说结论

这是 Spring Boot 2.4 引入的新语法，在 YAML 多文档中使用 `spring.config.activate.on-profile` 来条件激活配置，替代传统的 profile 文件命名方式。

## 深度解析

### 基本语法

```yaml
# application.yml
server:
  port: 8080
---
spring:
  config:
    activate:
      on-profile: dev
server:
  port: 8081
---
spring:
  config:
    activate:
      on-profile: prod
server:
  port: 80
```

### 激活条件

```yaml
# 仅激活 dev
on-profile: dev

# 激活多个
on-profile: [dev, test]

# 排除某个
on-profile: "!prod"

# 表达式
on-profile: "dev & !integration"
```

### 与传统方式对比

| 方式 | 文件 |
|------|------|
| 传统 | `application-dev.yml` |
| 2.4+ | `application.yml` + `on-profile: dev` |

### 完整示例

```yaml
# application.yml
spring:
  application:
    name: myapp
  datasource:
    url: jdbc:mysql://localhost:3306/default
---
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:mysql://localhost:3306/dev
---
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: jdbc:mysql://prod-server:3306/prod
```

## 易错点/踩坑

- ❌ 与传统 profile 文件混用 — 可能导致配置重复或覆盖混乱
- ❌ on-profile 拼写错误 — 不生效但不报错
- ❌ 忘记激活条件 — 需要 `spring.profiles.active` 或 `spring.config.activate.on-profile` 触发

## 关联知识点

- [[03_YAML多文档配置]] — YAML 多文档基础
- [[21_spring_profiles_active激活]] — profile 激活
- [[29_SpringBoot2_4配置变更]] — 2.4 变化总结
