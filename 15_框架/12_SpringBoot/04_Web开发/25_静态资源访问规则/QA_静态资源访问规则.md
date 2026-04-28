# 静态资源访问规则 - QA

## Q1：静态资源和 Controller 路径冲突怎么办？

**A**：静态资源优先。

**规则**：
- 静态资源存在 → 返回静态文件
- 静态资源不存在 → 交给 Controller 处理

---

## Q2：可以自定义静态资源路径吗？

**A**：可以。

```yaml
spring:
  web:
    resources:
      static-locations:
        - classpath:/static/
        - classpath:/custom-static/
```

---

## Q3：静态资源放在 webapp 目录可以吗？

**A**：可以，但不推荐。

```yaml
spring:
  mvc:
    static-path-pattern: /static/**
    web-resources:
      static-locations: file:/webapp/
```

---

## Q4：如何处理浏览器缓存？

**A**：

```yaml
spring:
  web:
    resources:
      cache:
        period: 3600  # 缓存时间（秒）
```

**或使用版本化资源**：
```html
<script src="/js/app.js?v=1.0.0"></script>
```
