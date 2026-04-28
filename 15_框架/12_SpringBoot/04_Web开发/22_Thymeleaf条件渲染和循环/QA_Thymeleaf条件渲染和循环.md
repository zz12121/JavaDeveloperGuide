# Thymeleaf条件渲染和循环 - QA

## Q1：th:if 和 th:unless 哪个更好？

**A**：根据语义选择。

```html
<!-- 语义清晰时使用 -->
<p th:if="${user.active}">已激活</p>
<p th:unless="${user.active}">未激活</p>

<!-- 复杂条件用 th:if 即可 -->
<p th:if="${user.active && user.role != 'guest'}">VIP 用户</p>
```

---

## Q2：循环中的状态变量有哪些？

**A**：

| 变量 | 说明 |
|------|------|
| index | 当前索引（从0） |
| count | 当前计数（从1） |
| size | 集合大小 |
| even | 是否偶数行 |
| odd | 是否奇数行 |
| first | 是否第一行 |
| last | 是否最后行 |

---

## Q3：如何实现表格斑马纹？

**A**：

```html
<tr th:each="user, stat : ${users}"
    th:class="${stat.odd} ? 'bg-gray' : ''">
    <td th:text="${user.name}"></td>
</tr>
```

---

## Q4：如何处理空集合？

**A**：

```html
<!-- th:if 判断 -->
<div th:if="${not #lists.isEmpty(users)}">
    <div th:each="user : ${users}" th:text="${user.name}"></div>
</div>
<div th:if="${#lists.isEmpty(users)}">
    <p>暂无数据</p>
</div>

<!-- th:remove 实现 -->
<tr th:each="user : ${users}" th:remove="tag">
    <td th:text="${user.name}"></td>
</tr>
```

---

## Q5：可以嵌套循环吗？

**A**：可以。

```html
<div th:each="category : ${categories}">
    <h3 th:text="${category.name}"></h3>
    <ul>
        <li th:each="item : ${category.items}" th:text="${item.name}"></li>
    </ul>
</div>
```
