# Favicon图标配置 - QA

## Q1：为什么浏览器不显示 Favicon？

**A**：

| 原因 | 解决方案 |
|------|----------|
| 缓存 | 清除浏览器缓存，Ctrl+F5 强制刷新 |
| 文件名错误 | 确保是 `favicon.ico` |
| 路径错误 | 放在 `static/` 目录 |
| 被禁用 | 检查 `spring.mvc.favicon.enabled` |

---

## Q2：支持 PNG 格式的 Favicon 吗？

**A**：支持。

```html
<link rel="icon" href="/images/favicon.png" type="image/png">
```
