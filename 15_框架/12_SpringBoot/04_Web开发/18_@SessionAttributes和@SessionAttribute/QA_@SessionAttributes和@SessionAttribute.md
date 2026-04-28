# @SessionAttributes和@SessionAttribute - QA

## Q1：@SessionAttributes 和 HttpSession 的区别？

**A**：

| 对比 | @SessionAttributes | HttpSession |
|------|-------------------|-------------|
| 声明方式 | 注解 | 手动操作 |
| 类型安全 | ✅ 自动类型转换 | ❌ 需要手动转型 |
| 清除方式 | SessionStatus.complete() | session.invalidate() |
| 作用范围 | 仅 Controller | 全局 |

---

## Q2：如何清除 @SessionAttributes 存储的数据？

**A**：

```java
@Controller
@SessionAttributes({"user", "cart"})
public class UserController {
    
    @GetMapping("/logout")
    public String logout(SessionStatus status) {
        status.setComplete();  // 清除 @SessionAttributes 存储的数据
        return "redirect:/login";
    }
}
```

---

## Q3：@SessionAttribute 获取不到会怎样？

**A**：

| required | 结果 |
|----------|------|
| true（默认） | 400 Bad Request |
| false | null |

```java
@GetMapping("/profile")
public String profile(@SessionAttribute(required = false) User user) {
    if (user == null) {
        return "redirect:/login";
    }
    return "profile";
}
```

---

## Q4：Session 数据能跨 Controller 共享吗？

**A**：

```java
// 方式1：HttpSession（跨 Controller）
@GetMapping("/set")
public String setSession(HttpSession session) {
    session.setAttribute("global", "value");
    return "ok";
}

@GetMapping("/get")
public String getSession(HttpSession session) {
    String value = (String) session.getAttribute("global");
    return value;
}

// 方式2：@SessionAttributes（仅当前 Controller 类）
@Controller
@SessionAttributes({"data"})
public class ControllerA {}

@Controller
public class ControllerB {}  // 访问不到 ControllerA 的 @SessionAttributes 数据
```

---

## Q5：会话数据过多会影响性能吗？

**A**：是的。

**建议**：
- 只存必要数据
- 大对象使用懒加载
- 会话超时设置合理
- 敏感数据加密存储

```yaml
server:
  servlet:
    session:
      timeout: 30m    # 会话超时
      cookie:
        http-only: true
        secure: false  # 生产环境设为 true
```
