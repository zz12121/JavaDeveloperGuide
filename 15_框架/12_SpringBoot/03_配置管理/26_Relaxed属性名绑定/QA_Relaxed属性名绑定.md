# Relaxed 属性名绑定 QA

## Q1：为什么需要 Relaxed 绑定？

**A**：
- 环境变量不支持 `-` 和 `.`，Relaxed 绑定允许用下划线代替
- 提高配置灵活性，照顾不同命名习惯

---

## Q2：哪些格式可以混用？

**A**：常见格式：
- 驼峰：`contextPath`
- 横线：`context-path`
- 下划线：`context_path`
- 大写下划线：`CONTEXT_PATH`

---

## Q3：@ConfigurationProperties 推荐用哪种格式？

**A**：推荐用横线分隔：
```yaml
app:
  config-file: /path
```

因为这是 Spring Boot 官方推荐的格式，也是 IDE 提示的标准格式。

---

## Q4：Relaxed 绑定会影响 @Value 吗？

**A**：会。`@Value("${app.name}")` 和 `@Value("${app-name}")` 在某些情况下可以互换。

---

## Q5：为什么有时候提示不准确？

**A**：IDE 提示基于元数据文件（`spring-configuration-metadata.json`），可能有延迟或不完整。

---

## Q6：可以禁用 Relaxed 绑定吗？

**A**：Spring Boot 不提供官方方式禁用 Relaxed 绑定。如果需要严格绑定，可以自定义 `ConfigurationPropertySource`。
