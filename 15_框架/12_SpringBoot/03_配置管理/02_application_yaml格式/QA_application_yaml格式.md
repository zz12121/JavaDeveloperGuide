# application.yml / YAML 格式 QA

## Q1：YAML 缩进到底用几个空格？

**A**：至少2个空格，同一层级必须一致。推荐用 2 空格或 4 空格，保持团队统一即可。

---

## Q2：字符串要不要加引号？

**A**：
- 普通字符串：不需要
- 特殊字符：需要（`:`, `#`, `&`, `*` 等）
- 布尔值：`true/false` 或 `yes/no`

```yaml
# 需要引号
message: "Hello: World"
path: 'C:\Users\test'

# 不需要
name: 张三
```

---

## Q3：YAML 中 `true` 和 `True` 有区别吗？

**A**：YAML 区分大小写。`true` 是布尔值，`True` 在某些解析器中也会识别为 `true`，但建议统一用小写 `true/false`。

---

## Q4：如何验证 YAML 语法是否正确？

**A**：
```bash
# 在线验证：https://www.yamllint.com/
# 命令行
yamllint config.yaml

# Spring Boot 启动时会自动检查
```

---

## Q5：application.yml 和 application.yaml 有区别吗？

**A**：完全等价，Spring Boot 都支持。通常用 `.yml` 后缀。
