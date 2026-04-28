# Thymeleaf模板引擎

## 先说结论

Thymeleaf 是 Spring Boot 官方推荐的模板引擎，支持自然模板，HTML 可直接在浏览器打开预览。**现代 Java Web 开发的首选模板引擎**。

## 深度解析

### 引入依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

### 默认配置

```yaml
spring:
  thymeleaf:
    prefix: classpath:/templates/     # 模板目录
    suffix: .html                     # 模板后缀
    mode: HTML                        # 模板模式
    encoding: UTF-8                   # 编码
    cache: true                       # 是否缓存（开发时 false）
```

### 目录结构

```
src/main/resources/
├── templates/              # Thymeleaf 模板目录
│   ├── user/
│   │   ├── list.html
│   │   └── detail.html
│   └── layout/
│       └── base.html
└── static/                 # 静态资源目录
    ├── css/
    ├── js/
    └── images/
```

### 基本使用

```java
@Controller
@RequestMapping("/user")
public class UserController {
    
    @GetMapping("/list")
    public String list(Model model) {
        model.addAttribute("users", userService.list());
        return "user/list";  // → templates/user/list.html
    }
}
```

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>用户列表</title>
</head>
<body>
    <table>
        <tr th:each="user : ${users}">
            <td th:text="${user.id}">1</td>
            <td th:text="${user.name}">张三</td>
        </tr>
    </table>
</body>
</html>
```

### 核心特性

| 特性 | 说明 |
|------|------|
| 自然模板 | HTML 可直接在浏览器打开，显示静态内容 |
| 表达式 | `th:text`、`th:each`、`th:if` 等 |
| 布局 | 模板继承、片段引用 |
| 国际化 | 支持多语言 |

## 易错点/踩坑

- ❌ 忘记添加 `xmlns:th` 命名空间 → Thymeleaf 语法不生效
- ❌ 生产环境开启缓存 `cache: false` → 性能下降
- ❌ 模板路径写错 → Template not found 错误

## 代码示例

### 模板片段引用

```html
<!-- layout/footer.html -->
<footer th:fragment="footer">
    <p>&copy; 2024 Company</p>
</footer>

<!-- 使用片段 -->
<div th:replace="layout/footer :: footer"></div>
```

### 模板继承

```html
<!-- layout/base.html -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head th:fragment="head">
    <meta charset="UTF-8">
    <title th:replace="~{::title}">默认标题</title>
</head>
<body>
    <div th:replace="~{::header}"></div>
    <main th:fragment="content">
        <!-- 子模板内容 -->
    </main>
</body>
</html>

<!-- user/list.html -->
<!DOCTYPE html>
<html th:replace="~{layout/base :: layout(~{::title}, ~{::#main})}">
<head>
    <title>用户列表</title>
</head>
<body>
    <main id="main" th:fragment="content">
        <h1>用户列表</h1>
    </main>
</body>
</html>
```

## 关联知识点

- `21_Thymeleaf标准表达式语法`：Thymeleaf 表达式详解
- `22_Thymeleaf条件渲染和循环`：条件与循环语法
