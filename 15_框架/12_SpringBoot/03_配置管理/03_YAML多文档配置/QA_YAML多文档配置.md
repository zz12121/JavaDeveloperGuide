# YAML 多文档配置 QA

## Q1：多文档和 profile 同时用，优先级是怎样的？

**A**：
1. 无 profile 的默认文档（最先加载）
2. profile 文档（按出现顺序，后面的覆盖前面的）
3. 命令行参数（最高优先级）

---

## Q2：多文档适合什么场景？

**A**：
- **默认配置 + 环境覆盖**：公共配置放前面，环境特定配置放后面
- **配置与元数据分离**：主配置 + 激活条件分离

```yaml
# 主配置
app:
  version: 1.0
---
# 开发环境
spring:
  config:
    activate:
      on-profile: dev
app:
  debug: true
```

---

## Q3：三个 `---` 就是三个文件吗？

**A**：不是。`---` 只是分隔符，表示单个 YAML 文件中的多个文档。Spring Boot 会按顺序解析并合并。

---

## Q4：多文档和 `spring.config.import` 有什么区别？

**A**：
- **多文档**：同一文件内的逻辑分区
- **spring.config.import**：导入其他独立文件

```yaml
# 多文档
server:
  port: 8080
---
# 导入外部文件
spring:
  config:
    import: optional:file:./db-config.yaml
```
