# spring.config.import 导入配置

## 先说结论

`spring.config.import` 是 Spring Boot 2.4 引入的配置导入机制，用于导入外部配置文件，比传统的 `@PropertySource` 更强大。

## 深度解析

### 基本语法

```yaml
# application.yml
spring:
  config:
    import: optional:file:./db-config.yml
```

### 导入位置

```yaml
spring:
  config:
    import:
      - optional:file:./db.yml
      - optional:classpath:extra.properties
      - optional:file:./redis.yml
```

### optional 前缀

```yaml
# 可选导入，文件不存在不报错
spring:
  config:
    import: optional:file:./optional.yml

# 必须导入，文件不存在报错
spring:
  config:
    import: file:./required.yml
```

### 支持的类型

| 类型 | 示例 |
|------|------|
| classpath | `classpath:config.yml` |
| file | `file:./config.yml` |
| jar | `jar:file:./lib/config.jar!/config.yml` |
| url | `https://example.com/config.yml` |

### 与 @PropertySource 对比

| 特性 | spring.config.import | @PropertySource |
|------|---------------------|-----------------|
| YAML 支持 | ✅ | ❌ |
| profile 条件 | ✅ | ❌ |
| 导入顺序控制 | ✅ | ❌ |
| 优先级管理 | ✅ | ❌ |

## 易错点/踩坑

- ❌ 文件路径错误 — 加 `optional:` 前缀避免报错
- ❌ 循环导入 — 配置 A 导入 B，B 导入 A
- ❌ 导入顺序导致覆盖问题 — 注意加载顺序

## 关联知识点

- [[11_@PropertySource加载外部配置]] — 传统加载方式
- [[14_SpringBoot配置优先级]] — 优先级规则
- [[29_SpringBoot2_4配置变更]] — 2.4 变化
