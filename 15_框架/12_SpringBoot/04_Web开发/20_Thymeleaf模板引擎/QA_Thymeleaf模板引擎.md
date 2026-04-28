# Thymeleaf模板引擎 - QA

## Q1：Thymeleaf 和 JSP 的区别？

**A**：

| 对比 | Thymeleaf | JSP |
|------|-----------|-----|
| 模板格式 | HTML5 | JSP 语法 |
| 浏览器预览 | ✅ 自然模板，可直接打开 | ❌ 无法直接运行 |
| 学习曲线 | 低 | 中 |
| 社区生态 | 活跃 | 成熟但老旧 |
| Spring Boot 官方支持 | ✅ 推荐 | ⚠️ 需要配置 |
| 性能 | 较好 | 更好 |

**推荐**：新项目使用 Thymeleaf，老项目迁移可选 JSP。

---

## Q2：Thymeleaf 表达式有哪些？

**A**：

| 表达式 | 说明 | 示例 |
|--------|------|------|
| `${...}` | 变量表达式 | `${user.name}` |
| `*{...}` | 选择表达式 | `*{name}` |
| `@{...}` | 链接表达式 | `@{/user/list}` |
| `#{...}` | 消息表达式 | `#{home.title}` |
| `~{...}` | 片段表达式 | `~{fragments :: footer}` |

---

## Q3：如何禁用 Thymeleaf 缓存？

**A**：

```yaml
# 开发环境
spring:
  thymeleaf:
    cache: false

# 或
spring:
  thymeleaf:
    prefix: file:src/main/resources/templates/
    cache: false
```

---

## Q4：Thymeleaf 如何处理 XSS？

**A**：默认自动转义 HTML 特殊字符。

```html
<!-- 默认转义 -->
<p th:text="${user.name}">  <!-- <script> 自动转义为 &lt;script&gt; -->

<!-- 不转义（谨慎使用） -->
<p th:utext="${user.name}">
```

---

## Q5：Thymeleaf 支持国际化吗？

**A**：支持。

```properties
# messages.properties
user.list.title=用户列表
user.list.empty=暂无数据

# messages_zh_CN.properties
user.list.title=用户列表
user.list.empty=暂无数据
```

```html
<p th:text="#{user.list.title}">用户列表</p>
```
