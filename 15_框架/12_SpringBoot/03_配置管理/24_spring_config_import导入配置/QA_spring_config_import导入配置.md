# spring.config.import 导入配置 QA

## Q1：spring.config.import 和 @PropertySource 哪个更好？

**A**：**spring.config.import** 更推荐：
- 支持 YAML
- 支持 profile 条件
- 配置更集中

---

## Q2：导入的文件会覆盖主配置吗？

**A**：是的。`spring.config.import` 导入的文件优先级高于主配置文件。

---

## Q3：可以导入远程配置文件吗？

**A**：可以：

```yaml
spring:
  config:
    import: https://example.com/config.yml
```

---

## Q4：导入失败会影响启动吗？

**A**：
- 不加 `optional:` — 会报错启动失败
- 加 `optional:` — 文件不存在不报错

---

## Q5：可以导入 properties 文件吗？

**A**：可以：

```yaml
spring:
  config:
    import: optional:file:./db.properties
```

---

## Q6：导入多个文件有顺序吗？

**A**：有。按声明顺序加载，后面的覆盖前面的同名属性。
