# Thymeleaf条件渲染和循环

## 先说结论

Thymeleaf 提供 `th:if`、`th:unless`、`th:switch`、`th:each` 等指令处理条件逻辑和循环渲染。

## 深度解析

### 条件指令

```html
<!-- th:if：条件为 true 时显示 -->
<p th:if="${user.active}">用户已激活</p>

<!-- th:unless：条件为 false 时显示（等价于 not th:if） -->
<p th:unless="${user.active}">用户未激活</p>
```

### th:each 循环

```html
<!-- 基本遍历 -->
<tr th:each="user : ${users}">
    <td th:text="${user.name}">姓名</td>
</tr>

<!-- 状态变量 -->
<tr th:each="user, stat : ${users}">
    <td th:text="${stat.index}">索引（从0开始）</td>
    <td th:text="${stat.count}">计数（从1开始）</td>
    <td th:text="${stat.size}">总数</td>
    <td th:text="${stat.even}">是否偶数</td>
    <td th:text="${stat.odd}">是否奇数</td>
    <td th:text="${stat.first}">是否第一</td>
    <td th:text="${stat.last}">是否最后</td>
</tr>
```

### th:switch / th:case

```html
<div th:switch="${user.role}">
    <p th:case="'admin'">管理员</p>
    <p th:case="'vip'">VIP 用户</p>
    <p th:case="*">普通用户</p>
</div>
```

## 代码示例

### 表格斑马纹

```html
<table>
    <tr th:each="user, stat : ${users}"
        th:class="${stat.odd} ? 'odd' : 'even'">
        <td th:text="${user.name}">姓名</td>
    </tr>
</table>
```

### 空数据处理

```html
<!-- 空集合 -->
<div th:if="${users.empty}">
    <p>暂无数据</p>
</div>

<!-- 或使用 #lists.isEmpty() -->
<div th:if="${#lists.isEmpty(users)}">
    <p>暂无数据</p>
</div>
```

## 关联知识点

- `21_Thymeleaf标准表达式语法`：表达式基础
- `20_Thymeleaf模板引擎`：模板引擎配置
