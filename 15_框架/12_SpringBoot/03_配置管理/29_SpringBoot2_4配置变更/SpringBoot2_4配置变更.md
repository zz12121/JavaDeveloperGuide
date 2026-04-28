# Spring Boot 2.4 配置变更总结

## 先说结论

Spring Boot 2.4 在配置加载机制上有重大变化，引入了 `spring.config.activate.on-profile` 和 `spring.config.import`，替代了部分旧用法。

## 深度解析

### 主要变化

| 变化 | 旧用法 | 新用法（2.4+） |
|------|--------|----------------|
| 条件 profile | 分离文件 | `spring.config.activate.on-profile` |
| 配置导入 | `@PropertySource` | `spring.config.import` |
| profile 包含 | `spring.profiles.include` | `spring.profiles.add` |
| bootstrap.yml | 支持 | **已移除** |

### 新增功能

#### 1. spring.config.activate.on-profile

```yaml
# application.yml
spring:
  config:
    activate:
      on-profile: dev
```

#### 2. spring.config.import

```yaml
spring:
  config:
    import: optional:file:./db.yml
```

#### 3. profile 多文档

```yaml
# 所有配置在一个文件中
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

### 移除功能

#### bootstrap.yml 移除

Spring Cloud 2.4+ 需要显式引入：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bootstrap</artifactId>
</dependency>
```

### 配置优先级变化

2.4+ 中，properties 和 YAML 优先级相同，按加载顺序覆盖。

## 易错点/踩坑

- ❌ 还在用 bootstrap.yml — 2.4+ 需要单独引入依赖
- ❌ profile 激活不生效 — 检查 on-profile 拼写
- ❌ 配置导入失败 — 加 `optional:` 前缀避免报错

## 关联知识点

- [[23_spring_config_activate_on_profile]] — 条件激活
- [[24_spring_config_import导入配置]] — 配置导入
- [[22_application_profile配置文件]] — profile 文件
