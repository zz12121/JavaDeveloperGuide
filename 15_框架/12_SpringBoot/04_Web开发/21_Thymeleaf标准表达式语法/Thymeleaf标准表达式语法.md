# Thymeleaf标准表达式语法

## 先说结论

Thymeleaf 提供四种标准表达式语法：变量表达式 `${}`、选择表达式 `*{}`、链接表达式 `@{}`、消息表达式 `#{}`，是模板渲染的核心。

## 深度解析

### 四种表达式

| 表达式 | 语法 | 用途 |
|--------|------|------|
| 变量表达式 | `${...}` | 获取变量值 |
| 选择表达式 | `*{...}` | 在上下文对象上执行 |
| 链接表达式 | `@{...}` | 处理 URL |
| 消息表达式 | `#{...}` | 国际化消息 |

### 变量表达式 `${}`

```html
<!-- 获取对象属性 -->
<p th:text="${user.name}">用户名</p>

<!-- 调用方法 -->
<p th:text="${user.getName().toUpperCase()}">用户名</p>

<!-- 三元运算符 -->
<p th:text="${user.status == 1 ? '启用' : '禁用'}">状态</p>

<!-- Elvis 运算符 -->
<p th:text="${user.name ?: '匿名用户'}">用户名</p>

<!-- 安全导航运算符 -->
<p th:text="${user?.address?.city}">城市</p>
```

### 选择表达式 `*{}`

```html
<!-- 先用 th:object 选择对象，再用 * 获取属性 -->
<div th:object="${user}">
    <p th:text="*{name}">姓名</p>
    <p th:text="*{age}">年龄</p>
    <!-- 等价于 ${user.name} -->
</div>
```

### 链接表达式 `@{}`

```html
<!-- 绝对路径 -->
<a th:href="@{https://example.com}">官网</a>

<!-- 相对路径（相对于上下文） -->
<a th:href="@{/user/list}">用户列表</a>

<!-- 带参数 -->
<a th:href="@{/user/{id}(id=${user.id})}">详情</a>
<!-- 结果: /user/123 -->

<!-- 多个参数 -->
<a th:href="@{/user/{id}(id=${user.id},type=1)}">详情</a>

<!-- 带查询参数 -->
<a th:href="@{/user/list(page=${page},size=10)}">列表</a>
```

### 消息表达式 `#{}`

```html
<!-- 简单消息 -->
<p th:text="#{user.welcome}">欢迎</p>

<!-- 带参数 -->
<p th:text="#{user.greeting(${user.name})}">你好</p>

<!-- 多语言切换 -->
<p th:text="#{|user.role.${user.role}|}">角色</p>
```

## 易错点/踩坑

- ❌ 表达式语法写错 → Thymeleaf 解析错误
- ❌ 使用 JavaScript 中的 `${}` → 与 Thymeleaf 冲突，需转义
- ❌ URL 中使用中文参数 → 需 URL 编码

## 代码示例

### 完整示例

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title th:text="#{user.list.title}">用户列表</title>
</head>
<body>
    <h1 th:text="${pageTitle}">页面标题</h1>
    
    <a th:href="@{/user/add}">新增用户</a>
    
    <table>
        <tr th:each="user : ${users}">
            <td th:text="*{user.id}">ID</td>
            <td th:text="${user.name}">姓名</td>
            <td th:text="#{|user.status.${user.status}|}">状态</td>
            <td>
                <a th:href="@{/user/{id}(id=${user.id})}">编辑</a>
            </td>
        </tr>
    </table>
</body>
</html>
```

## 关联知识点

- `20_Thymeleaf模板引擎`：模板引擎基础
- `22_Thymeleaf条件渲染和循环`：条件与循环语法
