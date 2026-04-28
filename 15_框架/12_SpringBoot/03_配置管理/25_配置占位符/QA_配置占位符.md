# 配置占位符 QA

## Q1：占位符和 SpEL 有什么区别？

**A**：

| 特性 | 占位符 `${}` | SpEL `#{}` |
|------|-------------|-----------|
| 功能 | 引用配置值 | 执行表达式 |
| 计算 | ❌ | ✅ |
| 默认值 | ✅ | ❌ |

---

## Q2：默认值可以是另一个占位符吗？

**A**：可以，嵌套使用：

```yaml
app:
  db: ${DB_HOST:${DEFAULT_HOST:localhost}}
```

---

## Q3：占位符解析失败会怎样？

**A**：启动报错：
```
Could not resolve placeholder 'xxx' in value "${xxx}"
```

可以加默认值避免：
```yaml
app:
  name: ${xxx:default}
```

---

## Q4：占位符可以跨文件引用吗？

**A**：可以，只要配置被加载就可以引用：

```yaml
# application.yml
app:
  name: myapp

# application-dev.yml
app:
  desc: ${app.name} is running  # 引用 application.yml
```

---

## Q5：占位符和 YAML 多文档配合使用？

**A**：可以：

```yaml
---
app:
  db: ${DB_URL:default}
---
spring:
  config:
    activate:
      on-profile: dev
app:
  db: jdbc:mysql://localhost:3306/dev
```
