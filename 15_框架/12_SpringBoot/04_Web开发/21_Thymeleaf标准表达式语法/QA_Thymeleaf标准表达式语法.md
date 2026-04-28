# Thymeleaf标准表达式语法 - QA

## Q1：${} 和 *{} 有什么区别？

**A**：

| 对比 | ${} | *{} |
|------|-----|-----|
| 名称 | 变量表达式 | 选择表达式 |
| 作用对象 | 整个上下文 | 当前选中对象 |
| 常见用法 | 获取任意变量 | th:object 后获取属性 |

```html
<!-- ${}：直接获取变量 -->
<p th:text="${user.name}"></p>

<!-- *{}：先选中对象，再获取属性 -->
<div th:object="${user}">
    <p th:text="*{name}"></p>  <!-- 等价于 ${user.name} -->
</div>
```

---

## Q2：如何处理 null 值？

**A**：使用 Elvis 运算符或安全导航。

```html
<!-- Elvis 运算符 -->
<p th:text="${name ?: '默认值'}"></p>

<!-- 安全导航 -->
<p th:text="${user?.address?.city}"></p>
```

---

## Q3：URL 表达式如何传递中文参数？

**A**：自动 URL 编码。

```html
<!-- Thymeleaf 自动编码 -->
<a th:href="@{/search(keyword=${keyword})}">搜索</a>

<!-- 手动编码 -->
<a th:href="@{/search(keyword=${T(java.net.URLEncoder).encode(keyword, 'UTF-8')})}">
    搜索
</a>
```

---

## Q4：消息表达式支持多语言吗？

**A**：支持。

```properties
# messages.properties
user.title=User List

# messages_zh_CN.properties
user.title=用户列表
```

```html
<p th:text="#{user.title}"></p>
```

---

## Q5：如何在 JavaScript 中使用 Thymeleaf 表达式？

**A**：

```html
<script th:inline="javascript">
    var username = [[${user.name}]];  // 自动转 JS
    var users = [[${users}]];  // 自动转 JSON
</script>
```
