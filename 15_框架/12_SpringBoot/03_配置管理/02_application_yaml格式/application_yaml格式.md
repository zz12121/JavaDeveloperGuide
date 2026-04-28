# application.yml / YAML 格式

## 先说结论

Spring Boot 推荐格式，层级清晰、支持复杂结构、中文无需转义，适合配置项较多或嵌套较深的场景。

## 深度解析

### 基础语法

```yaml
server:
  port: 8080
  servlet:
    context-path: /api
```

### 特点

| 特性 | 说明 |
|------|------|
| 缩进 | 用空格，禁止用 Tab（至少2个空格） |
| 大小写 | 敏感，`Port` 和 `port` 不同 |
| 层级表示 | 缩进即层级，无需符号 |
| 中文支持 | 直接写，无需转义 |
| 注释 | 用 `#` |

### 常见配置示例

```yaml
server:
  port: 8080
  servlet:
    context-path: /api

spring:
  datasource:
    url: jdbc:mysql://localhost:3306/test
    username: root
    password: 123456
  jpa:
    show-sql: true
    hibernate:
      ddl-auto: update

logging:
  level:
    root: INFO
    com.example: DEBUG
```

### 对象展开写法

```yaml
# 标准写法
server:
  port: 8080

# 展开写法（inline），等价
server: { port: 8080 }
```

### 多行字符串

```yaml
description: |
  第一行
  第二行
  第三行

# 保留末尾换行
description: >+
  第一行
  第二行
```

## 易错点/踩坑

- ❌ 使用 Tab 缩进 — YAML 严格禁止，必须用空格
- ❌ 缩进不一致 — 同级必须相同空格数
- ❌ 键名带空格 `port : 8080` — 冒号后要空格，键名不含空格
- ❌ 字符串不加引号 — 特殊字符（如 `:`, `#`）需要引号

## 关联知识点

- [[01_application_properties格式]] — properties 格式对比
- [[04_properties_vs_YAML对比]] — 选择建议
- [[03_YAML多文档配置]] — YAML 多文档语法
