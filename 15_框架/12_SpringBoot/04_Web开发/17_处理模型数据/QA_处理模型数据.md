# 处理模型数据 - QA

## Q1：Model、ModelMap、Map 哪个更好？

**A**：功能相同，按场景选择。

| 类型 | 推荐场景 |
|------|----------|
| Model | 推荐使用，接口更清晰 |
| ModelMap | 需要更多 Map 方法时 |
| Map | 简化场景，代码更简洁 |

```java
// 推荐：Model（最清晰）
@GetMapping("/list")
public String list(Model model) {
    model.addAttribute("users", userService.list());
    return "user/list";
}
```

---

## Q2：Model 数据能跨请求共享吗？

**A**：不能，Model 是请求作用域。

**共享方案**：

| 方案 | 作用域 | 适用场景 |
|------|--------|----------|
| Model | 请求 | 单次请求 |
| @SessionAttributes | 会话 | 同一用户多请求 |
| HttpSession | 会话 | 跨页面 |
| @ApplicationScope | 应用 | 全局共享 |

---

## Q3：ModelAndView 和 Model 的区别？

**A**：

| 对比 | Model | ModelAndView |
|------|-------|--------------|
| 视图设置 | 通过返回值 | 构造函数或 setter |
| 灵活性 | 较低 | 高 |
| 代码风格 | 函数式 | 对象式 |
| 使用场景 | 简单 | 复杂 |

```java
// Model：函数式
@GetMapping("/list")
public String list(Model model) {
    model.addAttribute("users", userService.list());
    return "user/list";
}

// ModelAndView：对象式
@GetMapping("/list")
public ModelAndView list() {
    ModelAndView mav = new ModelAndView("user/list");
    mav.addObject("users", userService.list());
    return mav;
}
```

---

## Q4：如何在视图中访问 Model 数据？

**A**：Thymeleaf 示例：

```html
<!-- 方式1：直接访问 -->
<p th:text="${user.name}">用户名称</p>

<!-- 方式2：安全访问 -->
<p th:text="${user?.name}">用户名称</p>

<!-- 方式3：遍历 -->
<tr th:each="user : ${users}">
    <td th:text="${user.name}">名称</td>
</tr>
```

---

## Q5：Model 数据何时被清除？

**A**：

| 情况 | 行为 |
|------|------|
| 请求结束 | 自动清除 |
| 重定向 | 清除（可使用 RedirectAttributes） |
| forward | 保留 |
| 异常 | 取决于异常处理器 |
